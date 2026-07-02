# EAR System — Local-Host Data Flow (v2 — greenfield consolidated edition)

> **IMPLEMENTED BY:** `ear-agent` v5 (Python, spec `components/EAR-AGENT-SPEC-v1`). The Node `uplink-monitor` v4.3 is a **reference/behavior donor only** — 'current code' notes below describe the donor, not the build target. Greenfield: nothing is ported by default; the contracts here ARE the build spec.

> **SCOPE (this effort):** Covers **Level 1 — dataset preparation**: the local-host capture/upload chain and the server-side steps that build the **event dataset** (power-bin grouping, boundary reconciliation, event construction, noise filtering) up to the analysis handoff. **Level 2 — analysis** is a **separate component** (`ear-analysis`) consuming only the event dataset via the handoff contract; an initial best-effort version (feature extraction, G1–G6 classification, scoring, provisional correlation, dashboard) is IN scope of this build (D-055). The data boundary is unchanged.

### From antenna to object store: capture, enrich, write, buffer, upload

**Scope.** This document captures the **local-host half** of the data chain — everything that happens on a warehouse monitoring PC, from HackRF capture to the batch landing in object storage. It stops at the upload boundary; server-side sync/ingest/pipeline (R5–R9) is covered in `DATA-WORKFLOW-RULES-v1.md`. Read alongside that document — the rule numbers align (L1–L6 here feed R5 there).

**Deployment shape (established, corrected against code v4.3.0).** Multi-facility by design: many hosts across sites (`ral`, `cht`, `dal`, …) push to one central Cloudflare R2 bucket; the server is often remote and pulls by facility prefix. Each monitoring PC runs **four independent scanner processes** (systemd template `uplink-scanner@{a..d}`, per-device env, staggered start) writing to per-device folders, **plus ONE shared EPHEMERAL uploader per host (D-045)** — a systemd **timer oneshot** (`uplink-upload.timer`, ~5-min cadence matching rotation) that scans all four devices' folders, filters/splits/uploads, writes the heartbeat, and exits — plus one shared DLQ (poison files only). *(Per-device isolation is at the folder/data level. A failed upload run simply exits; the next timer tick retries — no resident uploader to crash or restart.)* Scanners are otherwise fully independent — one per HackRF (a/b/c/d). They share only the physical machine and its network link. There is **no shared queue and no fan-in**: each device has its own scanner process, its own local folders, its own uploader, and its own dead-letter queue. Staggered startup (0/3/6/9s) is a **HackRF driver/USB initialization requirement**, not queue coordination.

```
ONE WAREHOUSE PC (×7–15 hosts)
┌───────────────────────────────────────────────────────────────────────┐
│  Pipeline A (HackRF a, low band)     Pipeline B (HackRF b, high band)    │
│  ┌──────────────────────────┐        ┌──────────────────────────┐       │
│  │ scanner  → pending/       │        │ scanner  → pending/       │       │
│  │ uploader → edge filter    │        │ uploader → edge filter    │  …C,D │
│  │   → processing/→completed/│        │   → processing/→completed/│       │
│  │   → DLQ/ (on failure)     │        │   → DLQ/ (on failure)     │       │
│  │   → PUT object store      │        │   → PUT object store      │       │
│  └──────────────────────────┘        └──────────────────────────┘       │
│  (a,c = low band 662–850)             (b,d = high band 1709–1911)         │
└───────────────────────────────────────────────────────────────────────┘
                                    │  (4 independent uploaders per host)
                                    ▼
                       central Cloudflare R2 bucket (R2-PRIMARY — all facilities)
                       keys: {agent_id}/obs_{agent_id}_{date}_{seq}.jsonl.gz
```

---

## L0 — Device identity  ·  *which radio is which agent*  (MULTI-FACILITY)

**MOVES.** No data — this is the identity binding every downstream record depends on. **Designed for multiple facilities from the start:** the server is often remote and many facilities push to one shared object store, so identity must be globally unique across all sites.

**CONTRACT.**
- `agent_id = {facility}-{host}{device}`, all lowercase — e.g. `ral-rf2a`, `cht-rf1b`, `dal-rf4d`.
- **Format regex:** `^[a-z]{3}-rf\d+[a-d]$`.
- **`facility`** = designated three-letter site code (`ral`, `cht`, `dal`, …), sourced from `FACILITY` in each host's `.env`. **Immutable once assigned** — renaming a facility orphans all its historical data under the old prefix.
- **Host segment (`rf{N}`) is unique within a facility**; the facility prefix makes the full id globally unique (two facilities may both have an `rf2` — `ral-rf2a` and `cht-rf2a` are distinct sensors and must never merge).
- **Device-letter → HackRF binds by hardware address / serial** (`HACKRF_SERIAL_A..D`), not USB enumeration order — survives re-plugging/reordering.
- **Casing is lowercase at every boundary** (agent write, R2 key, ingest) — object-store keys and SQL joins are case-sensitive, so `RAL` vs `ral` would silently split a facility's data. Normalize to lowercase; reject mixed case.
- Band group fixed by letter: **a, c = low band (662–850 MHz); b, d = high band (1709–1911 MHz)**. A given EARFCN appears on at most the two same-band devices of a host.

**NEVER.**
- A `{facility}-{host}` combination is never reused across two physical PCs.
- A facility code is never renamed or reused (immutable; would orphan or collide historical data).
- Device-letter → radio is never bound by "first device found" in production (fallback exists in code; must not be relied on).
- Correlation never crosses facilities or band groups — it scopes to `(facility, band_group)`.

**ENFORCED BY.** `FACILITY` + config-pinned serials per host `.env`; agent_id validated against the regex at startup **and** at ingest (a host missing its facility prefix fails loudly rather than uploading under a colliding bare id). Scanner **verifies its expected HackRF is present at startup and fails if not**. Casing normalized/asserted lowercase at write and ingest.

> **BUILD NOTE (ear-agent v5):** identity module composes `{facility}-{host}{device}` from env (`FACILITY` + host prefix — never hostname-derived, D-003). Server `Sensor`/`AnalysisRun` carry a queryable `facility` column (split on first `-`).
> **OPEN:** confirm every production host has `HACKRF_SERIAL_A..D` explicitly populated (not relying on first-found fallback), and `FACILITY` set.

---

## L0.5 — Capture configuration  ·  *per-agent tuning, and it must travel with the data*

**MOVES.** No observation data — this is the per-agent tuning that determines *what gets captured and what leaves the host*, plus the requirement that it accompany the data downstream.

**Why it's a contract, not a detail.** Tuning is **per-agent**: a sensor near an HVAC unit or a metal rack has a genuinely different noise environment than one in open space, and antenna/gain differ per mount. A sensor at noise floor −55 produces a **fundamentally different dataset** than one at −70 — the floor below which no data exists is per-sensor. Therefore a candidate's meaning depends on the capture config that produced it, and the server cannot honestly interpret or correlate data without knowing those conditions.

**Config fields (per agent).** `noise_floor_dbm`, HackRF gains (`lna_gain`, `vga_gain`, amp/antenna power), `bin_width`, edge-filter thresholds (`filter`/`aggregate` minCount/cv/dutyCycle).

**CURRENT STATE (honest).** Hand-edited in each host's `.env`, per host. It shapes capture but **does not travel with the data** — the server has no record of the conditions each sensor was operating under. This is a known gap.

**TARGET CONTRACT.**
- **PRODUCER GUARANTEES.** Each uploaded batch is **self-describing**: it carries the agent's capture config (the fields above) plus a `config_version` (monotonic version or hash of the config). Batch-stamped, so late/out-of-order files (the store-and-forward reality, L5) still carry their own truth without a separate registration channel.
- **CONSUMER MAY ASSUME.** For any observation/batch, the server can determine the capture conditions and config version that produced it.
- **NEVER.**
  - Two sensors are correlated as directly comparable without accounting for capture-config differences (a −55-floor and a −70-floor sensor are not the same instrument).
  - A candidate is reported without its capture context (floor/gain it was detected under).
  - `config_version` semantics change (R0.1); a config change always bumps the version.
- **ENFORCED BY.** `config_version` + capture fields stamped in each batch (header record or sidecar); recorded on the server `Sensor` row / run provenance; correlation flags incomparable capture settings; report formatter emits the floor/gain a detection was made under.

> **CHANGES from current code:** (1) agent stamps capture config + `config_version` into each batch — small uploader addition; (2) server records and versions it.
> **RECOMMENDED EVOLUTION (not Phase-1 blocking):** keep `.env` as source of truth for now, but **git-track per-host `.env`** (config repo, or a central registry that renders them). At 60+ agents across facilities, hand-edited-and-untracked tuning is exactly the drift that "kept breaking the chain" — versioning gives an audit trail, fleet-wide drift diffing, and the ability to correlate a threshold change with a shift in results.
> **NOTE:** this elevates the earlier single "noise floor −50 vs −60" open item — with per-agent tuning there is no single floor; it is per-sensor, which is itself the strongest argument for capture config traveling with the data sooner rather than later.

---

## L1 — Capture + enrich  ·  *HackRF → enriched observation (in memory)*

**MOVES.** One RF reading per frequency bin per sweep, enriched into an observation object.

**PRODUCER GUARANTEES (scanner process, one per device).**
- Spawns `hackrf_sweep` directly for its device's band range (one-shot mode).
- Parses sweep output (`date, time, hz_low, hz_high, bin_width, num_samples, dB…`), maps each bin frequency → EARFCN.
- Applies the noise-floor keep-decision at parse time (readings below floor are **not** emitted). Default floor is device config (`NOISE_FLOOR_DBM`); see open item on the −50 vs −60 discrepancy.
- Enriches via `fromReading(reading, agentId)` → stamps: `time` (ISO-8601 UTC, from host clock), `agent_id`, `band`, `earfcn`, `frequency_mhz` (2-dp), `power_dbm` (1-dp), `direction: 'ul'`.

**CONSUMER MAY ASSUME (this host's file writer).** Each object has the seven core fields, typed. Nothing about calibration of `power_dbm`. No `power_bin`, `observation_count`, `duty_cycle`, or `filter_action` yet — those are added later at the edge-filter step (L4/R2).

**NEVER.**
- `power_dbm` is uncalibrated relative dB; never treated as absolute (any calibration = new field).
- `time` is the observation moment, never the write/upload moment.
- Sub-noise-floor readings are never written (they don't exist on disk).

**ENFORCED BY.** `observationSchema.validate()` (existing) checks required fields, ISO time, band/earfcn/frequency/power types, direction ∈ {ul,dl}. Startup device-presence check (L0).

> **OPEN:** live noise floor — constants say `NOISE_FLOOR_DBM: -50`, docs/eartest say −60. Confirm the deployed value and record it as the contract. (Matters: it's the floor below which *no data exists*, so the server's −110 references are dead.)
> **NOTE:** schema has optional `carrier, cell_id, pci, latitude, longitude, metadata` fields — not populated at capture in this build (no GPS tagging observed). Listed for completeness.

---

## L2 — Local write + rotation  ·  *observation → JSONL file in this device's `pending/`*

**MOVES.** Enriched observations are buffered and written as JSONL to **this device's own** `pending/` folder.

**PRODUCER GUARANTEES (file writer).**
- Buffers in memory (`BUFFER_SIZE`), flushes every `FLUSH_INTERVAL_MS` (5 s) with a throttle delay.
- Filename: `obs_{agent_id}_{YYYY-MM-DD}_{NNNNNN}.jsonl` (UTC date, zero-padded sequence).
- Rotates on **5 minutes or 50 MB**, whichever first.
- **Each device writes to its own folder tree** — not shared with the other three devices on the host.

**CONSUMER MAY ASSUME (this device's uploader).** A rotated file is complete and immutable. Filename encodes agent + calendar date, but date-in-name is a label, not an authority on freshness (backlog can arrive late — L5).

**NEVER.**
- A filename is never reused with different content (write-once).
- The four devices' streams never interleave into one file (per-device isolation).

**ENFORCED BY.** Filename regex (existing). `MIN_FILE_AGE_MS` (2 s) settle guard before a file is considered complete. Rotation on size/age (existing `fileWriter`).

---

## L3 — Local queue lifecycle  ·  *`pending/` → `processing/` → `completed/` (or `failed/`)*

**MOVES.** Whole files transition through queue states as the uploader drains them. This is the first-layer local buffer.

**PRODUCER GUARANTEES (uploader, one per device).**
- Moves a file `pending/ → processing/` **before reading it** (write-complete barrier; a file being processed is out of the producer's way).
- On successful upload → `completed/` (retained 24 h, then cleaned).
- On failure → `failed/` and/or DLQ (L5). `failed/` retained 7 days.
- If the uploader is slower than production or paused, files accumulate in `pending/` — this **is** the file-level overflow buffer.

**CONSUMER MAY ASSUME.** A file in `completed/` was successfully uploaded. A file in `processing/` is mid-flight (crash-recovery should re-examine these on restart).

**NEVER.**
- A file is never deleted from `pending/`/`processing/` until its upload is confirmed or it is safely in DLQ.
- Queue folders are never shared between the four devices (each pipeline independent).

**ENFORCED BY.** `fileQueue` state-dir moves (existing). Retention cleanup (`cleanupCompleted` 24 h, `cleanupFailed` 7 d). 

> **OPEN:** crash-recovery of `processing/` on uploader restart — confirm files stuck in `processing/` after a crash are re-queued, not orphaned.

---

## L4 — Edge filter  ·  *raw rows → FORWARD / AGGREGATE / FILTER rows*  (= R2)

**MOVES.** During upload prep, the uploader classifies each 5-min file's clusters and rewrites rows. **This is where `observation_count`, `duty_cycle`, and `filter_action` are added.**

**WHY THIS STAGE EXISTS (NEVER remove it):** noise/infrastructure can be **~90% of raw traffic**. Edge filtering is load-bearing **feed preparation** — it targets the signal records worth shipping and collapses constant chatter to summaries before WAN/storage/ingest pay for it. A greenfield review considered dropping it (ship-raw); rejected on this number.

**FEED SPLIT (v5, structural fix for the false-CRITICAL bug class):** the uploader emits **two physically distinct streams per device**:
- **Signal feed** — `obs_{agent_id}_…` files: FORWARD rows only, full fidelity. The server's event-construction path ingests **only** this feed.
- **Context feed** — `ctx_{agent_id}_…` files: AGGREGATE/FILTER summary rows, L4.5 heartbeats, capture-config stamp. Lands in its own table; drives presence weighting (`observation_count`), suppression reporting, volume honesty.
It is thereby **structurally impossible** to construct an event from a summary row — the historic false-CRITICAL bug becomes unrepresentable rather than merely forbidden.

**BINNING OWNERSHIP (critical correction).** The edge filter computes a coarse `power_bin = floor(power_dbm/2)*2` **only as a transient grouping key for its own summarization decision** — it decides whether a cluster is FORWARD/AGGREGATE/FILTER within one 5-min file. This bin is **advisory, not authoritative**, and is NOT the classification the detection methodology uses. The **authoritative power binning and the merging of neighboring observations into events happen on the SERVER** (see event-construction methodology, server half). The agent forwards raw `power_dbm` + `earfcn` + `frequency_mhz`; it never decides what an "event" is. *(Why: RF power readings jitter — the same emitter reads −14.2/−14.8/−13.9 across sweeps — so grouping energy into a device/event is a tolerance-based judgment, not a fixed-range lookup. That judgment is deliberately the server's job.)*

**PRODUCER GUARANTEES (uploader edge filter).**
- Transient grouping key `power_bin = floor(power_dbm/2)*2` used internally to decide the action (not authoritative).
- `FORWARD` → one row per raw observation, `observation_count=1` (full fidelity, tracker candidates).
- `AGGREGATE` → one median-representative row per cluster, `observation_count`=true count.
- `FILTER` → one summary row per cluster, `observation_count`=true count, avg power.
- Classification thresholds (count/CV/duty-cycle) recorded; configurable via `thresholds.json` / env.

**CONSUMER MAY ASSUME (server).** `filter_action` tells whether a row is one event-candidate or a summary of many; `observation_count` is authoritative for volume. The server does its own authoritative binning/event-construction from these rows — it does not treat the agent's transient `power_bin` as final.

**NEVER.**
- `AGGREGATE`/`FILTER` rows are never treated as single detection events downstream — **enforced structurally by the feed split** (summaries travel in `ctx_` files the event path never reads), plus server-side R2/R7 checks.
- The three action values never change meaning.
- The agent's transient `power_bin` is never treated as the authoritative bin (server owns binning).

**ENFORCED BY.** (Server-side, per R2.) Locally: edge-filter unit test on cluster classification + reduction stats logging (existing `edgeFilter` stats).

> **NOTE:** this is the L-chain's copy of R2; the enforcement that prevents the false-tracker bug lives on the server (it's the consumer that must honor `filter_action`).

---

## L4.5 — Curated summarization override  ·  *tune out known noisy infrastructure before forwarding*

**MOVES.** No new data shape — a curated rule set that **forces known noisy infrastructure signals to summary (FILTER) treatment** so they don't flood upload volume, regardless of how the edge filter would naturally classify them.

**The problem it solves.** Some known facility-infrastructure emitters produce constant high-volume traffic. Even when the edge filter would eventually summarize them, operators need to *explicitly* tune specific known signals down so they aren't forwarded at full volume. Runs on the **upload side** (the confirmed pain point is forwarding/WAN volume, not local disk).

**Not a hard drop — a forced summary.** A matched signal is collapsed to the same one-record-per-window summary that an auto-classified FILTER cluster produces. It is **never suppressed to nothing.** You keep presence, volume (`observation_count`), and average power forever; you drop only the redundant per-sample detail. If a "known constant" emitter later changes behavior (e.g. goes intermittent — possibly meaning it isn't what you labeled), the per-window summary count shifts and reveals it; a hard drop would have hidden it.

**Matching key — MUST be identity-based, in the agent's native units.**
- Rules match on **EARFCN (frequency segment)**, optionally scoped by band — **per-EARFCN**, forcing all of that frequency's observations to summary.
- Rules do **NOT** match on `power_bin` (transient/advisory at the agent — L4) and do **NOT** match on a narrow power window. RF power smears across levels (radio jitter), so a tight power bracket would gut a signal's center and leave anomalous-looking tails. Per-EARFCN avoids the smear-fragmentation problem and keeps the agent rule dumb — all nuanced power-grouping stays server-side where the methodology lives.
- Rules never match on behavioral/temporal pattern (a tracker mimicking that pattern could hide); identity only.

**PRODUCER GUARANTEES.**
- Override rules are an explicit, per-agent, **versioned** list (EARFCN, optional band), stamped with `config_version` and traveling with the batch (same discipline as capture config, L0.5).
- A matched signal always emits its periodic summary heartbeat (rule matched, EARFCN, count, avg power, window) — **never silence.**

**CONSUMER MAY ASSUME (server).** It can see, per agent, exactly which EARFCNs were force-summarized and how much volume each represented — so a report states "ral-rf2a: EARFCN 20420 force-summarized (known facility infra), 47k obs/window," not a silent omission.

**NEVER.**
- A signal is never suppressed to zero — always reduced to the summary heartbeat.
- Override rules never match on power_bin or behavioral pattern — EARFCN/band identity only.
- A rule is never applied without its `config_version` traveling to the server.

**ENFORCED BY.** Versioned rule file (git-tracked, per L0.5 recommendation); summary heartbeat records stamped in the batch; server-side check that flags when a correlated same-band sensor pair has *different* override rules (one summarizing an EARFCN its partner forwards would skew correlation).

> **NEW capability (not in current build):** curated per-EARFCN summarization override on the upload side. Small uploader addition riding on the existing FILTER machinery.
> **RELATIONSHIP to server-side `InfrastructureSignature`:** distinct. L4.5 controls *upload volume* (force-summarize before forwarding). The server's `InfrastructureSignature` controls *analysis* (exclude known infra from candidate lists but retain full record). Volume control is local and irreversible-ish (raw detail not forwarded); analysis exclusion is server-side and fully reversible. Use L4.5 only for genuine volume problems; prefer server-side exclusion for "keep but don't flag."

---

## L5 — Local durability & store-and-forward  ·  *surviving connectivity outages*

> **v2 REFRAME (D-045):** with the ephemeral-uploader model, **`pending/` IS the store-and-forward buffer and retry queue by construction**. A run that cannot reach R2 exits leaving files in place; the timer cadence is the backoff. The circuit breaker is retired; the DLQ handles only **poison files** (malformed / repeatedly rejected by R2), never mere connectivity failure. The disk-space ceiling (with eviction policy + alert) is checked at the start of every upload run. The donor's watcher/breaker mechanics below are retained as reference only.

**MOVES.** On upload failure, batches persist locally and are replayed when connectivity returns — "catch up on the side."

**PRODUCER GUARANTEES (circuit breaker + DLQ, per device).**
- **Circuit breaker** wraps uploads: CLOSED (normal) → OPEN after N consecutive failures (rejects immediately for 30 s, stops hammering a dead link) → HALF_OPEN (tests recovery) → CLOSED.
- **Dead-letter queue**: failed batches persisted to a local DLQ folder (JSONL batch files + metadata: error, `retryCount`).
- **Replay** (`dlqReplay`): on recovery, drains DLQ back out; supports date-range filter, batch size, dry-run.
- A batch is never lost to an outage — it stays on local disk (pending/ or DLQ) until confirmed uploaded.

**CONSUMER MAY ASSUME (server).** The object store **eventually** receives every batch — possibly **out of order** and possibly **delayed by hours or days** after an outage. This is precisely why server ingest must key on file identity, not arrival time or filename date (R5 ledger).

**NEVER.**
- The circuit breaker never discards a failed batch — it defers to DLQ.
- A batch is never deleted from DLQ until replay confirms upload.

**ENFORCED BY.** Circuit-breaker state-transition tests; DLQ persistence; replay idempotency. Recommended integration test: *kill connectivity mid-batch, restore, assert every batch arrives exactly once.*

> **OPEN (priority):** **no known disk-space ceiling** on `pending/` + DLQ. A multi-day WAN outage could grow local storage unbounded until the disk fills — a real warehouse failure mode. Decision needed: cap size + oldest-drop policy, or alert-and-hold. **Recommend resolving before fleet rollout.**
> **RESOLVED (greenfield, D-017):** the donor's `dlqReplay` targeted a legacy Postgres path. In `ear-agent` v5 the Postgres path does not exist — replay targets **R2 only**, by construction. The **disk-space ceiling** on pending+DLQ is a v5 build requirement (WP-A3), not an open bug.

---

## L6 — Upload to object store  ·  *confirmed batch → S3-compatible object*  (= R4)

**MOVES.** A confirmed-complete, gzipped batch is PUT to the object store.

**PRODUCER GUARANTEES (uploader).**
- Object keys, all identity-derived (never filename-regex): **signal feed** `{agent_id}/obs_{agent_id}_….jsonl.gz` · **context feed** `{agent_id}/ctx_{agent_id}_….jsonl.gz` (D-012) · **heartbeat** `_health/{agent_id}.json` (D-015). The bucket self-organizes by facility+sensor; a server filters a facility with a prefix match (`ral-*`).
- Every batch row carries `schema_version=v5` (greenfield; server enrichment rejects anything else, D-021).
- gzip compression before PUT.
- Atomic PUT — object is fully present or absent, never partial.
- Retried (backoff) until confirmed; unconfirmed → DLQ (L5).

**CONSUMER MAY ASSUME (server sync).** Listing a prefix yields only complete, immutable objects with stable keys/sizes. Facility is derivable from the key prefix.

**NEVER.**
- Prefix scheme never drifts from `agent_id` (historic `earcore/` vs bare-`rf1a/` confusion banned; must include facility).
- An object is never mutated in place.

**ENFORCED BY.** Size-verified download + `verify-sync` audit (server side). Startup validation of endpoint/bucket/prefix (pydantic-settings on server; `.env` on agent).

> **TRANSPORT = R2-PRIMARY (design of record).** Multiple facilities with an often-remote server make a **central cloud object store the correct hub** — all facilities push to one Cloudflare R2 bucket; server(s) pull by facility prefix, location-independent. This is no longer a pending decision. MinIO/LAN-primary is dropped (it only made sense for a single on-site facility); a per-facility local cache could return later *only* as an optimization if a specific site's WAN is chronically bad. Store-and-forward (L5) is therefore **load-bearing infrastructure**, not a safety net — inter-facility WAN is far less reliable than a LAN.
> **OPEN:** confirm `R2_PREFIX`/agent_id (with facility) is correctly set on every host (misconfig = silent mislabeling).
> **OPEN (server topology, decide before Phase 1):** one **central server** pulling all facilities (cross-facility visibility, single deployment, commingled data, single point of failure) vs a **replicable per-facility server** pulling only its own prefix (isolation, data residency, N deployments, no built-in cross-facility view). Determines whether the server is one multi-tenant system or a per-facility appliance.

---

## Consolidated open questions (to verify before fleet rollout)

| # | Item | Where | Priority |
|---|---|---|---|
| 1 | Overflow pool disk-space ceiling / eviction policy | L5 | **High** — unbounded growth on long WAN outage |
| 2 | ~~dlqReplay target~~ RESOLVED greenfield (D-017): R2-only replay in v5 | L5 | closed |
| 3 | All hosts have explicit `HACKRF_SERIAL_A..D` (no first-found fallback) | L0 | Medium |
| 4 | Live noise-floor value (−50 vs −60) | L1 | Medium — defines the "no data exists below" line |
| 4b | Batch-stamped capture config + `config_version` (not yet sent) | L0.5 | **High** — server can't interpret/correlate without it |
| 4c | Curated per-EARFCN summarization override (upload side, not yet built) | L4.5 | Medium — volume control for known infra |
| 4d | Binning/event-construction is server-owned; agent bin is advisory | L4 | **High** — ownership seam, must not drift |
| 5 | `processing/` crash-recovery on uploader restart | L3 | Medium |
| 6 | Server topology: central multi-facility vs per-facility appliance | L6 | **High** — architectural, decide before Phase 1 |
| 7 | `R2_PREFIX = {agent_id}` verified per host | L6 | Low (but silent if wrong) |

## What this document deliberately does NOT cover
Server-side sync, ingest ledger, DuckDB pipeline, correlation, results persistence, and reporting are R5–R9 in `DATA-WORKFLOW-RULES-v1.md`. The handoff point is L6/R4: once an object is in the store, the server owns it.
