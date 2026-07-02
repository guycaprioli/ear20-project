# EAR System — Session Handoff (Level 1 documentation effort)
### Read this first. Then read the eight documents it points to.

**What happened this session.** Reviewed 8 experimental repos, converged on a canonical architecture, and produced a complete **Level 1 (dataset-preparation)** specification for the EAR System — from RF capture through to the analysis handoff. The chain that "kept breaking silently" is now pinned as explicit contracts. This session was documentation + design, not implementation.

**Deployment context: GREENFIELD.** Guy is redeploying from scratch — a ground-up rebuild applying six months of lessons (which these documents distill). No running fleet, no migration/cutover; fresh production bucket with facility ids + `schema_version=v5` from day one; old R2 data retained as a historical test corpus (validates Phase 3 against reality, incl. the 20595 target).

**What EAR is.** A multi-facility RF tracker-detection system. HackRF SDR sensors sweep LTE **uplink** channels in warehouses, upload observations to Cloudflare **R2** (S3-compatible), a **Django + DuckDB** server (hosted in the `bss_earv1` Pegasus project) builds device **events** and filters noise, and produces **periodic forensic reports** (not real-time alerting). Goal of Level 1: turn raw sweeps into a clean, trustworthy **event dataset**; the analysis of that dataset is Level 2 (separate effort).

---

## The documents (source of truth — all in this folder)

Read in this order:
1. **`EARSYSTEM-PRODUCTION-PLAN-v3.md`** — repo consolidation, architecture, phased rollout, parts-donor map.
2. **`DATA-WORKFLOW-RULES-v2.md`** — the full R0–R9 contract chain (change-discipline through reports), enforcement table, R5.0 onboarding gate.
3. **`LOCAL-HOST-DATA-FLOW-v1.md`** — L0–L6 local host chain (identity, capture config, capture, write, queue, edge filter, summarization override, store-and-forward, upload).
4. **`EVENT-CONSTRUCTION-METHODOLOGY-v1.md`** — the heart: event = burst group; uplink undersampling physics; two-pass power binning; noise filtering; the open Pass-2 calc.
5. **`EVENT-DATASET-HANDOFF-v1.md`** — the Level 1 → Level 2 interface (`events` + `event_gaps`). The contract that keeps the two efforts separate.
6. **`PARAMETER-TUNING-v1.md`** — forward-looking A/B/validation plan; the labeled-corpus requirement.
7. **`RFANALYSIS-SCOPE-v2.md`** — the real buildable app spec against all the above; supersedes the v1 scaffold. **This is the build entry point.**
7a. **`PROJECT-PLAN-v1.md`** — sequenced phases 0–6 with exit criteria.
7b. **`GOLDEN-FIXTURE-SPEC-v1.md`** — the executable form of every contract (13 personas + whole-run assertions); phase gates.
7c. **`BUILD-SESSION-BOOTSTRAP-v1.md`** — paste at the start of every Claude Code build session.
8. Superseded, kept for history: `EARSYSTEM-PRODUCTION-PLAN-v1/v2.md`, `DATA-WORKFLOW-RULES-v1.md` (v2/v3 reflect the R2-primary reversal, facility identity, two-table ingest, Level-1 R7 rescope).

The working git repo (`/home/claude/production-plan/`, 14 commits) does **not** persist. `/mnt/user-data/outputs/` does **not** persist between sessions. **Guy's local machine is the only source of truth** — these files must be saved there.

---

## Key decisions locked this session (do not silently reopen)

- **Multi-facility identity (Option A):** `agent_id = {facility}-{host}{device}`, lowercase, regex `^[a-z]{3}-rf\d+[a-d]$` (e.g. `ral-rf2a`). Facility = 3-letter code (ral, cht, dal…) from `FACILITY` env, immutable. Globally unique by construction.
- **Transport = R2-primary** (central cloud hub — correct for many-facilities/remote-server, not a compromise). Per-agent folders in the bucket.
- **Correlation scope = `(facility, band_group)`** only. Bands: a/c = low (662–850 MHz), b/d = high (1709–1911 MHz).
- **Binning + event construction is SERVER-owned.** The agent's `power_bin` is advisory only.
- **An event IS a burst group** (10-min / 600s gap, validated, tunable). Sweep-collapse (seconds) ≠ burst-grouping (minutes). Analysis operates only on events.
- **Two-pass power binning:** fast fixed-bucket Pass 1 over everything + expensive boundary reconciliation Pass 2 over edge cases only. The split is load-bearing for processing time.
- **Noise filter = hourly presence × activity** with the **low-duty-cycle exception** (present-but-sparse = power-saving tracker = KEEP, never filter as infrastructure). This protects the primary target.
- **Prep/analysis boundary is hard.** Level 1 stops at the event dataset. CV/periodicity/scoring/classification/reports/correlation-method = Level 2, separate effort.
- **Ingest = two-table** (persistent raw landing → batched enrich) + **onboarding gate** (Sensor registry allowlist: pending→qa→active→disabled) + **ledger** (idempotency). Per-agent for bin/consolidation; load/enrich may batch.
- **Greenfield design review (decided):** edge filter KEPT — noise ≈90% of raw traffic, feed prep is load-bearing (ship-raw proposal rejected on this number; rationale now a NEVER in R2/L4). **Feed split added**: signal feed `obs_…` (FORWARD only, sole input to event construction) vs context feed `ctx_…` (summaries/heartbeats/config) — false-CRITICAL bug made structurally unrepresentable. Plus: agent health heartbeat to `_health/`; containerized server (compose+GHCR) with bare-metal agent; two-pass binning LOCKED (Guy); Phase 3.0 benchmark sizes the Pass-2 compute budget only. Heartbeat + containerized server confirmed.
- **Scale structure: per-agent Level 1 DuckDB files** (`l1/{facility}/{agent_id}.duckdb`, disposable, parallel chains) → **partitioned Parquet handoff** (`handoff/{facility}/{agent_id}/{date}/`) → **dedicated Level 2 DuckDB** reading the Parquet. Physically enforces the handoff boundary; throttle = worker concurrency × memory_limit.
- **rfanalysis v1 scaffold = superseded-but-referenced.** Build fresh in `bss_earv1`; carry forward only known-good fragments (hours_present fix, low-duty-cycle exception, DuckDB bindings, sqlsafe).

---

## OUTSTANDING ITEMS (all tracked; none lost)

### A. Missing-code algorithms (may exist in code Guy hasn't found — HUNT FIRST)
- **Pass-2 boundary-reconciliation calc.** The math that reassigns power-boundary-straddling observations. Not in any of the 8 repos. Was designed/explored (performance-motivated). Fingerprint: a 2nd step that cheaply selects observations by distance-to-bucket-boundary, then applies a heavier calc (centroid/valley/tolerance) to only that subset. Build the app with this as a clearly-marked STUB behind a seam.
- **Cross-sensor correlation method** (Level 2). What makes two per-sensor events "the same device" across same-band antennas. Also unspecified; same class.

### B. Judgment calls (only Guy can decide)
- **False-split vs false-merge cost** — sets Pass-2 conservatism. Higher stakes than usual because data is already undersampled (a wrong split can shatter a sparse event). Assumed lean: false-split (miss) worse than false-merge.
- **Sweep-collapse window value** — earnest-v3 used 2s; confirm target (may tie to sweep revisit time).
- **Server topology** — central multi-facility vs per-facility appliance. Decide before Phase 1. App is built to run either way.

### C0. Agent tooling review → DECISION: RE-PLATFORM TO PYTHON (uplink-agent v5)
Node agent reviewed end-to-end. Initial verdict was keep-Node (workload fit, resilience stack, solid systemd model). **Revised on Guy's stack-alignment call**: single Python-centric maintainer → one stack; rewrite de-risked because the L-chain contracts now specify exact behavior and the golden fixture provides pass/fail validation, and Phase 1 was gutting the uploader anyway (legacy Postgres path through 6 modules, fake-gzip, identity-regex landmine `[a-z0-9]+` lacking `-`). **Conditions locked:** port-don't-redesign (implement L-chain exactly); keep the systemd model verbatim (templated scanners + 1 shared uploader/DLQ per host — process model corrected in docs); harvest gittest2 (resilience/boto3/structlog/pydantic-settings); side-by-side Node-vs-Python soak on the test host before cutover. Legacy Postgres path is not ported (excision by omission). See PROJECT-PLAN Phase 1.

### C. Local-side CODE BUGS (agent code, bite in deployment — fix before fleet rollout)
- **`dlqReplay` imports legacy `../database` (Postgres), not the object store** — recovered-after-outage data may replay to a destination not in the current architecture. HIGH.
- **No disk-space ceiling on `pending/` + DLQ overflow** — multi-day WAN outage grows local storage unbounded until disk fills. HIGH. Needs cap + eviction/alert policy.

### D. Per-host config verification (before each host goes live)
- `HACKRF_SERIAL_A..D` explicitly populated (not first-found fallback); `FACILITY` set; `R2_PREFIX`/agent_id correct (with facility); live noise-floor value confirmed (constants say −50, docs say −60 — pick and record); crash-recovery of `processing/` re-queues (not orphans).

### E. Undocumented operational areas (identified in gap audit, NOT yet written)
- **Server pipeline recovery semantics** — what happens to a half-built batch if a mid-chain task crashes (partial DuckDB state? resume vs restart?).
- **Data retention / lifecycle** — DuckDB growth, raw-landing retention, Postgres results, report retention over months × 60 sensors.
- **Time-sync tolerance/detection** — NTP assumed; no contract for detecting drift or acceptable tolerance (drift corrupts gap-based events).
- **Security/access** — R2 credentials, report access, sensitivity of surveillance data.

### F. Level 2 (deferred by design)
Reporting format (harvest eartest `ForensicReportGenerator`), correlation, CV/periodicity/scoring/classification, A/B framework. Starts from the handoff contract.

### G. Start collecting NOW (costs nothing, impossible to reconstruct later)
**Labeled ground-truth corpus** — every physically-confirmed device (tracker or benign) logged against its event signature. The basis for all future objective parameter tuning.

---

## Recommended next moves
**See `PROJECT-PLAN-v1.md` — the sequenced working plan (Phases 0–6 + L2).** Summary of the original options:
1. **Hunt for the Pass-2 calc code** (item A) — could close the biggest correctness gap with real logic instead of a stub.
2. **Fix the two local-side code bugs** (item C) — they bite in the deployment done first.
3. **Document the operational gaps** (item E) — recovery/retention/time-sync/security — to complete the picture before running unattended.
4. **Begin the build** against `RFANALYSIS-SCOPE-v2.md` inside `bss_earv1`.

## Working style (for the next agent)
Guy: terse, direct, sequential questions building to decisions. Wants **anti-sycophantic pushback** — challenge premises, don't fold under pressure, no flattery. Values correctness/traceability over speed: file:line refs, versioned docs (version numbers in filenames), git discipline, present tradeoffs for his decision, document findings. Fix bugs immediately, prefer the correct approach over the easy one. Exclude guidance docs from RF/device analysis reports. Always present files for download and remind him outputs don't persist — his local machine is the source of truth.
