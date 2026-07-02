# EAR System — rfanalysis App Scope (v3 — ear-server greenfield edition)
### The real build spec, against the finalized contracts. Supersedes the v1 scaffold.

> **SCOPE:** Level 1 — dataset preparation. Builds the event dataset up to the handoff contract; Level 2 (CV/periodicity/scoring/classification/reports) is a separate effort. See `EVENT-DATASET-HANDOFF-v1.md`.

**What this is.** The v1 scaffold (`rfanalysis-scaffold-v1.zip`) was built early this session, *before* ~a dozen design decisions that changed its foundations. It succeeded as a spike — it found the `hours_present` bug, validated DuckDB Python bindings, and drove the whole methodology reconstruction — but it now **predates the design**. This document defines the app to build inside **`ear-server`** — a **fresh SaaS Pegasus generation** (D-032; `bss_earv1` and the v1 scaffold are reference donors only) — against the finalized contract documents.

**Repo:** `ear-server` (fresh SaaS Pegasus generation per D-032; `bss_earv1` is a reference donor only). **Contracts (in `ear-docs`):** `contracts/DATA-WORKFLOW-RULES-v3` · `contracts/LOCAL-HOST-DATA-FLOW-v2` · `contracts/EVENT-CONSTRUCTION-METHODOLOGY-v1` · `contracts/EVENT-DATASET-HANDOFF-v1` · `operations/PARAMETER-TUNING-v1`. Plan: `plan/GREENFIELD-DEVELOPMENT-PLAN-v1` (WP-S1…S10). Two landing paths: signal `obs_` + context `ctx_` feeds (D-012).

---

## Disposition of the v1 scaffold

| v1 piece | Disposition |
|---|---|
| `hours_present` bugfix (`COUNT(DISTINCT date_trunc('hour', …))`) | **CARRY FORWARD** — validated, now a contract (R6). |
| Low-duty-cycle noise-filter exception + over-filter safeguard | **CARRY FORWARD** — the power-saving-tracker protection; documented as NEVER-remove. |
| DuckDB via official Python bindings | **CARRY FORWARD** — architecture confirmed. |
| `sqlsafe` identifier + numeric validation | **CARRY FORWARD** — good, extend for facility regex. |
| Django + DuckDB structure, StageResult/PipelineConfig dataclasses | **CARRY FORWARD as reference** — reshape into Pegasus app. |
| Bare `agent_id` (`^rf\d+[a-d]$`), no facility | **REBUILD** — facility-scoped identity. |
| Single-pass `read_json → INSERT`, `ignore_errors=true`, delete-and-reload | **REBUILD** — two-table landing→enrich, ledger, quarantine. |
| Single-pass `bin_mappings` fold | **REBUILD** — two-pass boundary reconciliation (Pass-2 stubbed). |
| Time-clustering / CV / scoring bundled in pipeline | **REMOVE from this app** — that's Level 2. This app stops at the handoff. |
| No Sensor registry, no onboarding gate, no correlation | **BUILD NEW** — per finalized design. |

**Bottom line:** carry forward the known-good *logic fragments*; rebuild the *structure* fresh inside `ear-server`. The identity change alone touches every table and query, so patching v1 would be more churn than a clean build.

---

## Target app: `ear-server/<pegasus_project>/apps/rfanalysis/` (fresh Pegasus generation, D-032)

### Postgres models (Django)
- **`Sensor`** (registry / allowlist): `agent_id` (unique, `^[a-z]{3}-rf\d+[a-d]$`), `facility`, `host`, `device_letter`, `band_group` (low/high), `status` (`pending|qa|active|disabled`), `hackrf_serial`, `bearing_deg`, `offset_feet`, `capture_config_version`, `notes`. Admin-editable. **This is the onboarding gate and the identity source of truth.**
- **`AnalysisRun`**: run_id, facility, agent_id, window, `config_snapshot`, status, stage results, timing. (carry from v1, add facility.)
- **`IngestLedger`**: filename, agent_id, size/etag, row_count, ingested_at. Idempotency (R5).
- **`InfrastructureSignature`**: curated known-infra (earfcn, power_bin, facility?) for signature filtering. (carry from v1.)
- **Event/handoff tables** live in DuckDB (analytical), not Postgres — see handoff contract. Results/candidate summaries that the dashboard reads may be mirrored to Postgres per production plan.

### DuckDB analytical layer — PER-AGENT FILES (scale structure, DECIDED)
**One DuckDB file per agent:** `l1/{facility}/{agent_id}.duckdb` — that agent's landing, observations, clustering, filter_decisions, events. Rationale: Level 1 has **zero cross-agent operations**, so per-agent files convert the single-writer bottleneck into **parallel per-agent pipeline chains (process pool, D-044)**; a wedged/corrupt file affects one agent and is **disposable** (rebuild from R2 + ledger — recovery = delete and re-run). Level 2 gets its **own dedicated DuckDB** reading the Parquet handoff — it physically cannot reach back into L1 tables (enforces the handoff NEVER structurally).

**Operating rules:**
- Each task: **open its file → work → close** (no long-lived cached connections across tasks) — enforce in the stage base class.
- `SET memory_limit` per connection; **process-pool size is the throttle** (D-044). Peak footprint = pool size × memory_limit (the only two scaling tunables; file count is irrelevant at rest).
- No query ever opens two L1 files together; cross-agent reads go through the Parquet handoff, never L1 files.
- L1 files are cache, not backup surface (backup = Postgres + handoff Parquet).

Per-agent file contents:
- **Landing table** (`raw_landing`) — persistent, as-is import, mostly VARCHAR, no casting. (R5.1)
- **Enrichment step** → `observations` — date/numeric casting, `power_bin` (advisory), agent identity **folder-authoritative**, malformed rows → quarantine table with reason (NOT `ignore_errors`). (R5.2)
- **power_clustering** — Pass-1 fast bucketing + **Pass-2 boundary reconciliation (STUB — see open items)**.
- **filter_decisions** — hourly presence × activity, low-duty-cycle exception, safeguard, signature filter. (carry logic from v1)
- **events** — sweep-collapse then **burst grouping (600s gap) = the event**. (build per methodology)
- **handoff EXPORT** — each completed chain exports `events` + `event_gaps` as **partitioned Parquet**: `handoff/{facility}/{agent_id}/{date}/…` (immutable, columnar, no writer contention when chains finish concurrently). The **dedicated Level 2 DuckDB** reads this via glob/ATTACH — works for either server topology (per-facility L2 reads its partition; central reads all).

### Ingest + pipeline orchestration (EPHEMERAL JOB, D-044 — no Celery/Redis in Level 1)
**Execution model:** resident Postgres + Django web; the pipeline is a scheduled ephemeral container (`docker compose run --rm pipeline manage.py run_level1`) that processes everything new and exits — non-zero exit alerts, next run retries, run lock prevents overlap, systemd timer schedules. Per-agent stages are sequential code; agents parallelized with a process pool.
- **Per-agent stage sequence** (function-order gating, halt-on-failure via exit code):
  `sync(active agent) → land → enrich → bin+reconcile → filter → events → export handoff Parquet`. Agents run **in parallel** via process pool (per-agent DuckDB files); concurrency bounded by pool size × memory_limit.
- Load/enrich **may** batch across agents; **bin/consolidation/events MUST be per-agent** (correctness boundary).
- Correlation + reporting are **separate, later** (Level 2 / distinct effort), not in this job.
- Onboarding: sync iterates `Sensor.status='active'`; new R2 prefixes surface as `pending`; `qa` agents run in isolation, excluded from production correlation/reports.

### Config
- `rfanalysis.toml` (git-tracked, mounted) — prep parameters: bin size, sweep-collapse window, **burst gap 600s**, presence 0.95, duty-cycle multiplier 40, min-obs, safeguard ratio, Pass-2 threshold (when built). Analysis params (CV etc.) belong to Level 2, not here. `check_config` command. Precedence: defaults → TOML → per-Sensor overrides (noise floor) → CLI.

### Harvest from other repos (per production plan)
- `gittest2`: `sync.py` (boto3 R2, typed, retry), `resilience.py`, `metrics.py`, `health.py`, structlog, test patterns.
- `earnest-v3`: `rf-device-status` (agent health/staleness observability) → feeds Sensor status dashboard.
- `eartest`: `ForensicReportGenerator` shape → Level 2 reporting effort (not this app).

---

## Explicitly OUT of scope (Level 2 / separate efforts)
Time-clustering, CV/periodicity, tracker scoring/classification, threat reports, cross-sensor correlation method, A/B framework. This app **stops at the handoff contract.**

## Open items this app must accommodate (build interface, defer internal)
- **Pass-2 boundary reconciliation calc** — build the two-pass *structure* with Pass-2 as a stubbed/simple step (e.g. current single-pass fold as placeholder) behind a clear seam; swap in the real calc when found/decided. Do **not** let the placeholder become canonical (mark it clearly).
- **Sweep-collapse window** — parameterize (default 2s pending confirmation).
- **False-split vs false-merge cost** — sets Pass-2 conservatism; parameter, default pending decision.
- **Server topology** (central vs per-facility) — app must run either way; keep facility filtering explicit so a per-facility deployment is just a config scope.

## Definition of done (this app)
Given `active` agents' data in R2, the app: gates by onboarding status, syncs per-agent idempotently, lands→enriches with quarantine, bins with the two-pass seam, filters infra/noise preserving power-saving trackers, builds burst-group events, and produces the handoff dataset (`events`+`event_gaps`) with full provenance (`run_id`, `config_version`) — all per-agent, completion-gated, inside the ephemeral pipeline job (D-044) in `ear-server`. Nothing beyond the handoff.
