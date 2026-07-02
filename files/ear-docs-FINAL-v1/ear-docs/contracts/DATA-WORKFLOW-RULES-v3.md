# EAR System — Data Workflow Rules (v3 — greenfield consolidated edition)

> **SCOPE (this effort):** Covers **Level 1 — dataset preparation**: the local-host capture/upload chain and the server-side steps that build the **event dataset** (power-bin grouping, boundary reconciliation, event construction, noise filtering) up to the analysis handoff. **Level 2 — analysis** is a **separate component** (`ear-analysis`) consuming only the event dataset via the handoff contract; an initial best-effort version (feature extraction, G1–G6 classification, scoring, provisional correlation, dashboard) is IN scope of this build (D-055). The data boundary is unchanged.

### Contracts the data must satisfy at every hop, from antenna to report

**Purpose.** Every major breakage in this system's history was a *silent contract violation*, not a code bug: `time_bucket` silently changed meaning and broke `hours_present`; the edge filter changed what a "row" means and the server never learned; storage paths and bucket prefixes drifted between builds. This document defines the data chain as a sequence of **hops**, and at each hop fixes a **contract** — rules a design must meet. A change that violates a rule is rejected by design, not discovered in production.

**How to read this.** The data makes nine hops from antenna to report. Each hop has one rule-set (R1–R9), preceded by the meta-rule (R0) that governs how all contracts may change. Every rule-set uses the same five headings:

- **MOVES** — what data crosses this hop, in what shape.
- **PRODUCER GUARANTEES** — what the upstream side promises.
- **CONSUMER MAY ASSUME** — what the downstream side is entitled to rely on (and nothing more).
- **NEVER** — invariants that may never change without a version bump.
- **ENFORCED BY** — the automated check that makes the rule real. *A rule with no check is not a rule.*

**The chain at a glance:**

```
 [antenna] ──R1──► JSONL row ──R2──► edge-filtered row ──R3──► rotated .gz file
     ──R4──► transport (R2 object store) ──R5──► ingest (landing → observations)
     ──R6──► analytical columns ──R7──► pipeline stages ──R8──► results (Postgres)
     ──R9──► forensic report
```

---

## R0 — Change discipline (meta-rule, governs R1–R9)

- **R0.1 — Meaning is immutable.** The meaning of an existing field, column, or value never changes. To evolve, *add* a new field/column and deprecate the old one; never repurpose. (This one rule prevents the `time_bucket` → `hours_present` class of bug.)
- **R0.2 — Everything is versioned.** Every record and every file carries a `schema_version`. Consumers reject versions they do not recognize rather than guessing.
- **R0.3 — Every rule has a check.** Each rule below names an automated enforcement: a shared schema, a contract test, a runtime assertion, or a CI gate. Unenforced rules are documentation, and documentation is what failed before.
- **R0.4 — Paired change.** A producer and its consumer are never changed in the same commit unless the contract test between them changes in that commit too.
- **R0.5 — Rules travel with the code.** This document lives in both the agent repo and the server repo, and is referenced from each `CLAUDE.md`, so automated (Claude Code) and human edits are bound by it.

---

## R1 — Observation record  ·  *antenna → JSONL row*

**MOVES.** One RF observation, serialized as a single JSONL line, written by the scanner.

**PRODUCER GUARANTEES (scanner).**
- Emits these core fields: `observed_at`, `agent_id`, `band`, `earfcn`, `frequency_mhz`, `power_dbm`, `config_version`, `schema_version`. *(The canonical field set is `contracts/schemas/observation-signal.v5.schema.json` — authoritative over this prose per D-052; this list is descriptive.)* `observation_count`, `duty_cycle`, `filter_action`, and the advisory `power_bin` are **not** emitted by the scanner — they are added downstream at the edge-filter hop (R2/L4). There is **no** `direction` field (capture is uplink-only; the v5 signal schema forbids extra properties via `additionalProperties:false`). *(Reconciled per audit finding LEAD.2/DE.11/QA.8 — prose previously said `time`/`direction`, which the schema rejects.)*
- `observed_at` is ISO-8601 UTC, taken from the host's NTP-synced clock, and represents the **observation** moment.
- `agent_id` matches `^[a-z]{3}-rf\d+[a-d]$`, lowercase — facility-scoped (`ral-rf2a`); facility immutable (see L0).
- Every row carries `schema_version="v5"` (greenfield). Enrichment REJECTS any other version to quarantine — no COALESCE absorption of legacy shapes (D-021). Legacy-corpus data enters only via the `ear-fixture` adapter, never production ingest (D-035).
- On the signal feed (post-edge-filter, `obs_` files) `observation_count = 1` and `filter_action = 'FORWARD'`; `duty_cycle ∈ [0,1]` when present. These are edge-filter-added (R2/L4), not scanner-emitted (finding LEAD.3).

**CONSUMER MAY ASSUME.** The fields above exist and are typed as stated. Nothing about calibration of `power_dbm`. Nothing about ordering within a file.

**NEVER.**
- `power_dbm` is **uncalibrated relative** dB; it is never re-interpreted as absolute/calibrated power. Calibration, if ever added, is a *new* field.
- `observed_at` is never the write time or upload time. The legacy name `time` is not valid v5 (D-052); the fixture/adapter maps it.
- A field's meaning never changes (R0.1); unknown new fields are ignored by consumers, missing required fields fail loudly.

**ENFORCED BY.** The shared JSON-Schemas `contracts/schemas/observation-signal.v5.schema.json` (signal feed) and `observation-context.v5.schema.json` (context feed) — committed to both repos and **authoritative over this prose** (D-052). A CI check diffs this prose field list against the schemas so they cannot drift again (finding LEAD.2/DE.11/QA.8). Agent validates at write time (unit test). Loader validates at ingest: violations are counted, logged, and quarantined — **never silently dropped** (replaces the current `ignore_errors=true`).

> **CHANGES from current behavior (choose consciously):** (1) add `schema_version` to the row — small agent change; (2) NTP-synced clocks become a *provisioning requirement*, because CV analysis of inter-event gaps across 60 sensors is meaningless if clocks drift; (3) loader stops silently discarding malformed rows.

---

## R2 — Edge-filter semantics  ·  *JSONL row → edge-filtered row*

**MOVES.** The uploader's EdgeFilter groups each 5-minute file's rows into `(agent_id, earfcn, power_bin)` clusters and emits one of three record shapes per cluster — **split across two physical feeds**: FORWARD rows in the **signal feed** (`obs_…`), AGGREGATE/FILTER summaries (+ L4.5 heartbeats, config stamps) in the **context feed** (`ctx_…`). *Rationale (NEVER remove this stage): noise can be ~90% of raw traffic; edge filtering is load-bearing feed preparation targeting the signal records worth shipping.* The server's event path ingests only the signal feed — summaries are structurally excluded from event construction. *(The agent's `power_bin` here is a **transient/advisory** grouping key for this decision only — authoritative binning and event construction are SERVER-owned; see the event-construction methodology.)*

**PRODUCER GUARANTEES (uploader).**
- `filter_action = 'FORWARD'` → one output row **per raw observation**, `observation_count = 1`. Full fidelity.
- `filter_action = 'AGGREGATE'` → **one** row per cluster per file; `observation_count` = true count in the cluster; `observed_at` = representative (median) observation time.
- `filter_action = 'FILTER'` → **one** summary row per cluster; `observation_count` = true count; power = cluster average.
- The thresholds that produced the classification are recorded.

**CONSUMER MAY ASSUME.** `filter_action` states whether a row is one event (`FORWARD`) or a summary of many (`AGGREGATE`/`FILTER`). `observation_count` is authoritative for volume regardless of shape.

**NEVER.**
- An `AGGREGATE`/`FILTER` row is **never** treated as a single detection event for periodicity/CV analysis — doing so detects the 5-minute file cadence, not the signal (the false-CRITICAL bug).
- The three action values never change meaning; new behavior = new action value.

**ENFORCED BY.** Pipeline contract test: consolidation and time-clustering stages must restrict to `filter_action = 'FORWARD' OR filter_action IS NULL`; presence/infrastructure classification weights by `observation_count`. A regression fixture (AGGREGATE-heavy file) asserts zero false CRITICALs.

---

## R3 — File format & rotation  ·  *edge-filtered rows → rotated `.jsonl.gz` file*

**MOVES.** Buffered rows are written to a local JSONL file, rotated, and gzipped for upload.

> **GREENFIELD NOTE:** `ear-agent` v5 ships **real gzip** by contract (verified in WP-A5 soak; fixture P12). Server ingest still **content-detects** compression rather than trusting extensions — the historical fake-gzip corpus (donor-era files named `.gz` but uncompressed) is handled by the `ear-fixture` legacy adapter. File names: signal `obs_{agent_id}_…`, context `ctx_{agent_id}_…` (D-012).

**PRODUCER GUARANTEES (scanner/uploader).**
- Filename: `obs_{agent_id}_{YYYY-MM-DD}_{NNNNNN}.jsonl` → `.gz` on upload. Date is UTC.
- Rotation on 5 minutes **or** 50 MB, whichever first.
- Files are **write-once**: once rotated and moved past `pending/`, content never changes.
- Local lifecycle: `pending/ → processing/ → completed/`; `completed/` retained 24 h.

**CONSUMER MAY ASSUME.** A file that exists in transport is complete and immutable. The filename encodes agent and calendar date, but **date-in-name is a labeling convenience, not an ingest-window authority** (a backlog file can arrive late).

**NEVER.**
- A filename is never reused with different content.
- The date in the filename is never used as the sole "what's new" mechanism (that is R5's ledger job).

**ENFORCED BY.** Filename regex validated on write and on ingest. Uploader moves file to `processing/` before reading (write-complete barrier). `MIN_FILE_AGE_MS` guard already enforces settle time.

---

## R4 — Transport  ·  *file → central R2 object store (R2-PRIMARY, design of record)*

**Key namespace (complete):** `{agent_id}/obs_…` (signal) · `{agent_id}/ctx_…` (context) · `_health/{agent_id}.json` (heartbeat, D-015). All identity-derived; never filename-regex.

**MOVES.** The gzipped file becomes an object under a per-sensor prefix in the object store.

**PRODUCER GUARANTEES (uploader).**
- Object key: `{agent_id}/{filename}.gz` (per-sensor prefix).
- PUT is atomic — an object is either fully present or absent; no partial objects are ever visible.
- Upload is retried (backoff, circuit breaker, dead-letter queue); a file not confirmed uploaded stays queued.

**CONSUMER MAY ASSUME (sync service).** Listing a prefix yields only complete objects. Object size is stable once listed. Keys are unique and stable.

**NEVER.**
- The prefix scheme never drifts (historic `earcore/` vs `rf1a/` confusion is banned — prefix = `agent_id`, full stop).
- An object is never mutated in place; re-upload of changed content uses a new key or is treated as an error (R5 keys on filename+size/etag).

**ENFORCED BY.** Sync service integrity check (size match on download; the `verify-sync` audit compares store vs local). Endpoint/prefix are config, validated at startup (pydantic-settings). Atomicity is a property of S3/R2 PUT — relied upon, documented.

---

## R5 — Ingest  ·  *object store → DuckDB `observations` table*

### R5.0 — Agent authorization gate (onboarding) — runs BEFORE any fetch/ingest

**MOVES.** No data — the gate that decides *which* agents' folders the server is allowed to fetch and ingest at all. R2 is segmented into per-agent folders (`{agent_id}/…`), so selection is by explicit prefix; this gate governs which prefixes are authorized.

**PRODUCER GUARANTEES (server).**
- Fetch/ingest iterates an **explicit allowlist** — the `Sensor` registry filtered to authorized status — never raw R2 discovery.
- Agent lifecycle (status on the `Sensor` row): **`pending → qa → active → disabled`**.
  - `pending`: folder seen in R2 but not authorized; **not fetched, not ingested.**
  - `qa`: synced and analyzed **in isolation** (staging context) for onboarding checks — confirm facility, serial→device mapping, band group, capture config, noise-floor sanity, event sanity. **Not in production reports or correlation.**
  - `active`: in production ingest, analysis, correlation, reports.
  - `disabled`: retired/suspended; not fetched.
- Discovery of a *new* R2 prefix produces a **"pending onboarding" notice**, surfacing the agent for review — it never enqueues it for production.

**CONSUMER MAY ASSUME.** Any data in production analysis came from an `active` agent that passed onboarding/QA. A new or misconfigured agent cannot silently enter production.

**NEVER.**
- Auto-discovery never equals auto-production. An unregistered/`pending` folder is never ingested into production.
- A `qa` agent's data never enters production reports or cross-sensor correlation until promoted to `active`.
- Agent status semantics never change under the same names (R0.1).

**ENFORCED BY.** `Sensor` registry as the allowlist; sync iterates `Sensor.objects.filter(status='active')` (plus a separate `qa` path to an isolated staging area); a "discover pending" function *surfaces* new prefixes but cannot enqueue them. Per-agent R2 folders make isolation structural (a `pending` agent's files never intermingle with production).

> **INTENT vs ACTUAL (gap to build, not port):** this gate **did not exist as code** in any reviewed repo. earnest-v3's `refreshAgentList` (`SELECT DISTINCT agent_id FROM ear_observations WHERE time > NOW()-24h`) and gittest2's `_discover_agents` (any `rf*` folder) both **auto-discover with no gate** — a new folder would auto-enter production on the next run. The ingredients existed (earnest-v3 `v3_staging` schema; `rf-device-status` health monitoring module) but were never crystallized into an agent allowlist. Onboarding was enforced *manually* (operator configured what ran). For multi-facility scale this must become a coded gate. Harvest `rf-device-status` (agent health/staleness observability) and the staging-schema concept (isolated QA analysis) into the implementation.

### R5.1 — Fetch + ingest (for authorized agents) — TWO-TABLE: landing → enrich

**MOVES.** New objects are synced to the local raw-data volume, bulk-loaded **as-is** into a persistent `raw_landing` table, then a **separate batched enrichment step** promotes them into `observations` (dates, numerics, identity). Two distinct, completion-gated tasks — never one collapsed hop. *(Why: raw data is dirty/inconsistent in ways transform-on-read can't handle safely — the original design landed raw first, then processed in SQL where failures can be inspected and counted. The eartest/gittest2 rewrites collapsed this into a temp table; the persistent landing table is a feature, not overhead: safety net, quarantine point, reprocess-without-refetch, audit of what files actually contained.)*

**PRODUCER GUARANTEES (sync + landing + enrichment).**
- Sync downloads only missing/size-mismatched objects (skip-if-exists).
- Each file lands **exactly once**, tracked in an `ingest_ledger` (filename, size/etag, row_count, ingested_at); list → anti-join ledger → land new only → record in the **same transaction**.
- **Landing:** as-is import, minimal typing (VARCHAR-heavy), no filtering, no casting; verify actual compression (files have been observed as **fake-gzip** — `.gz` name, uncompressed content — detect, don't assume from extension).
- **Enrichment (batched, may span agents):** date/timestamp normalization, numeric casting, advisory `power_bin`; **agent identity is FOLDER-authoritative** — the R2 prefix wins; a row whose internal agent_id disagrees is flagged (never silently kept, never `'unknown'`); malformed rows → **quarantine table with reason + count** (R1) — `ignore_errors=true` is banned.
- Legacy multi-schema drift (time/observed_at, power/power_dbm, band/bandwidth) is resolved explicitly at enrichment with `schema_version`, not hopeful COALESCE.

**CONSUMER MAY ASSUME.** `observations` contains each uploaded row exactly once. Late-arriving backlog files are ingested whenever they appear (ledger, not filename date, decides). No row silently vanished.

**NEVER.**
- A file is never ingested twice (would inflate every downstream count).
- Malformed rows are never silently dropped — counted, logged, quarantined (R1 enforcement).
- The date-in-filename is never the newness test.

**ENFORCED BY.** Ledger uniqueness constraint on filename(+size/etag). Transactional insert+ledger write (crash cannot record an unfinished load). Post-load **conservation identity** (finding DE.1/QA.7, SEAM-1): `observations_delta + quarantined_delta == Σ ledger.row_count` for the batch — quarantined rows are **counted, not silently subtracted** from the identity (the two-term form `observations_delta == Σ row_count` was arithmetically false the instant any row quarantined — a designed path — and halted every dirty-row run). Quarantine is a **non-halting** outcome: a run with quarantined rows exits 0, records `quarantined_count` (with reason breakdown) on the run, and alerts only when the quarantine **ratio** exceeds a configured threshold (mirrors the over-filter safeguard). This is the same identity the fixture whole-run "Conservation" assertion states end-to-end (`agent.raw == observations + quarantined + suppressed_summarized`, with an L4.5-forced EARFCN counted once and `quarantined ∩ summarized = ∅`).

---

## R6 — Analytical column semantics  ·  *`observations` → derived working tables*

**MOVES.** The pipeline builds `power_clustering`, `hourly_stats`, `filter_decisions`, `core_activity_stats`, consolidation and time-clustering tables from `observations`.

**PRODUCER GUARANTEES (stages).** Each column's meaning is documented once and frozen. Specifically the two that bit us:
- `time_bucket` in `power_clustering`/`core_activity_stats` holds a **raw observation timestamp** (the historic convention), NOT an hour bucket.
- `hours_present` is derived as `COUNT(DISTINCT date_trunc('hour', time_bucket))` — hours, not observation count. (This is the bugfix; it is now a *contract*, not an implementation detail.)

**CONSUMER MAY ASSUME.** A documented column means exactly what the contract says, forever. Anything needing "hours" derives them explicitly with `date_trunc('hour', …)`; nothing assumes a column is pre-bucketed by reading its name.

**NEVER.**
- A column's semantic never changes under the same name (R0.1). If `time_bucket` ever needs to mean an hour bucket, that is a **new column** (`hour_bucket`), not a redefinition.
- `hours_present` is never computed as `COUNT(DISTINCT time_bucket)` again.

**ENFORCED BY.** A `SCHEMA-SEMANTICS.md` column dictionary + a unit test that feeds a known high-frequency signal and asserts `hours_present ≤ timeframe_hours` (catches any regression to observation-counting). Column comments in `schema.sql`.

---

## R7 — Pipeline stages  ·  *working tables → EVENT DATASET (Level 1 stops here)*

**MOVES.** Ordered, completion-gated bulk stages transform working tables into the **event dataset** defined by `EVENT-DATASET-HANDOFF-v1`. **This chain ends at the handoff.** Time-clustering / CV / scoring / candidate selection are **Level 2** (separate effort) and are NOT stages of this pipeline.

**PRODUCER GUARANTEES (pipeline).**
- Fixed order: power clustering (Pass 1 + Pass-2 boundary reconciliation) → noise filtering (presence×activity, low-duty-cycle exception) → sweep-collapse → burst grouping (= events) → persist handoff (`events` + `event_gaps`).
- **Per-agent** for binning/consolidation/events (correctness boundary); load/enrich may batch across agents.
- Each run is a full rebuild over the configured window (deterministic given data + config).
- Only `filter_action='FORWARD'`/NULL rows enter event/CV analysis (R2); `AGGREGATE`/`FILTER` inform presence, weighted by `observation_count`.
- The resolved config for the run is captured.

**CONSUMER MAY ASSUME.** The event dataset reflects the whole configured window, produced by that exact config. Two runs on identical data+config produce identical events (determinism).

**NEVER.**
- Stage order is never rearranged silently (each stage declares its input tables; a missing predecessor is a hard error, not empty output).
- A stage never partially writes on error (transaction per stage); no silent no-ops (the historic chunked-merge no-op is banned).

**ENFORCED BY.** Stage base class asserts required input tables exist/are non-empty before running. Per-stage transaction. Determinism test: same fixture twice → identical event set. Config snapshot attached to the run (R8). Handoff contract test (see `EVENT-DATASET-HANDOFF-v1`).

---

## R8 — Results persistence  ·  *candidates → Postgres (`AnalysisRun`, `TrackerCandidate`, `CorrelatedDetection`)*

**MOVES.** Candidates and run metadata are written to Postgres; cross-sensor correlation is computed.

**PRODUCER GUARANTEES (runner).**
- Every candidate is linked to its `AnalysisRun`; every run stores `config_snapshot`, window, workflow, agent, timing, stage results.
- Correlation joins candidates within the **same band group** (a/c low, b/d high) and overlapping time windows — never across band groups (physically impossible to share an EARFCN).
- Postgres holds results durably; DuckDB remains the queryable raw/analytical store.

**CONSUMER MAY ASSUME (dashboard/report).** Every result traces to the run and config that produced it. The web tier reads only Postgres — no DuckDB dependency. Correlation reflects real same-band sensor agreement.

**NEVER.**
- A candidate is never stored without its run + config linkage (no orphan results, no "which settings produced this?" mystery).
- Correlation never pairs low-band with high-band sensors.

**ENFORCED BY.** FK constraints (candidate → run). `config_snapshot` non-null. Correlation query joins on `Sensor.band_group`; a test asserts no cross-band pairs. Read-path test: report generation touches only Postgres.

---

## R9 — Forensic report  ·  *results → report artifact*

**MOVES.** A run (or comparison) becomes a structured, methodology-traceable report written to the `/reports` volume and surfaced in the dashboard.

**PRODUCER GUARANTEES (report generator).**
- Every claim traces to a candidate/run/config (threat priority, classification, behavior, recommendation — the proven `FINAL_THREAT_ASSESSMENT` shape).
- Power-based proximity/bearing statements are labeled **coarse and uncalibrated** (R1 forbids treating dBm as absolute).
- Report records which sensors/agents and which time window it covers.

**CONSUMER MAY ASSUME (analyst).** Any figure can be traced back to source rows. Proximity language is qualitative, not metric.

**NEVER.**
- A report never asserts a calibrated distance/power figure from uncalibrated dBm.
- A report never presents a candidate without its methodology and confidence.

**ENFORCED BY.** Report template pulls only from persisted run data (R8), never recomputes ad hoc. A generation test asserts every reported candidate ID exists in the run. Proximity fields render through a formatter that always emits the "uncalibrated/coarse" qualifier.

---

## Enforcement summary (the checks that make these rules real)

| Rule | Check | Where it runs |
|---|---|---|
| R0.2 versioning | `schema_version` present + known | agent write test, loader ingest |
| R1 record shape | shared `observation-signal.v5.schema.json` + `observation-context.v5.schema.json` (D-052) | agent unit test + loader |
| R2 edge semantics | FORWARD-only CV; AGGREGATE fixture → 0 false CRITICAL | pipeline contract test |
| R3 file rules | filename regex, write-once, settle barrier | agent test |
| R4 transport | prefix=agent_id, size-verify, atomic PUT | sync integrity check |
| R5.0 onboarding gate | sync iterates active allowlist; pending folders surfaced not ingested | Sensor registry + sync test |
| R5 idempotency | ledger uniqueness + transactional load + row-delta assert | loader test + runtime |
| R6 column meaning | `hours_present ≤ timeframe` test, column comments | pipeline unit test |
| R7 stages | input-table assertions, per-stage tx, determinism test | pipeline test |
| R8 persistence | FK + non-null config_snapshot + no cross-band correlation | DB constraints + test |
| R9 reports | trace-to-source test, uncalibrated formatter | report test |

**Rule of thumb for any future change:** find the hop it touches, read that hop's NEVER list, and if the change would violate it, the change is wrong — evolve by adding, per R0.1. Producer and consumer never move without the contract test moving too (R0.4).
