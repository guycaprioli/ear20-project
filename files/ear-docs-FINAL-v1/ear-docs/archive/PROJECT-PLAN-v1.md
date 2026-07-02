# EAR System — Project Plan (v1)
### The working plan: from documented design to Level 1 in production

**Context: GREENFIELD REDEPLOY.** This is a ground-up rebuild using the lessons of the prior 6 months — the contract documents ARE those lessons distilled. Nothing is currently deployed: no migration, no cutover, no legacy compatibility. The old repos are reference/donors; the old R2 data (bare `rf1a/` prefixes, incl. the known 20595 target) is retained as a **historical test corpus** for validating event construction against reality — production starts in a **fresh bucket/prefix root** with facility-scoped ids and `schema_version=v5` from day one (enrichment rejects anything else; no COALESCE drift absorbed).

**Ground rules.** Scope = **Level 1** (dataset prep through the event handoff). Level 2 is chartered separately (Phase L2 below is a placeholder, not scheduled). Each phase has an exit criterion — don't start the next until it's met. Source docs: `RFANALYSIS-SCOPE-v2` (build spec), `DATA-WORKFLOW-RULES-v2`, `LOCAL-HOST-DATA-FLOW-v1`, `EVENT-CONSTRUCTION-METHODOLOGY-v1`, `EVENT-DATASET-HANDOFF-v1`.

---

## Phase 0 — Decisions & recovery hunt (Guy, ~2–4 hrs, BEFORE any build)

The cheap items that unblock or improve everything downstream.

| # | Task | Output |
|---|---|---|
| 0.1 | **Hunt for the Pass-2 boundary-reconciliation code** in repos/branches/scratch not among the 8 reviewed. Fingerprint: a 2nd step selecting observations by distance-to-bucket-boundary, then a heavier calc (centroid/valley/tolerance) on that subset. | Found → becomes the spec. Not found → pick a candidate method (boundary-zone / centroid-nearest / valley-test) in Phase 3. |
| 0.2 | **Decide server topology**: central multi-facility server vs per-facility appliance. | One line in the production plan. Shapes deployment, not the app (app runs either way). |
| 0.3 | **Decide false-split vs false-merge cost** (assumed: false-split/miss is worse). | Sets Pass-2 conservatism default. |
| 0.4 | **Confirm sweep-collapse window** (2s?) — ideally from sweep revisit time. | Config default. |
| 0.5 | **Save all 11 docs to local machine + push `production-plan` repo to GitHub.** | Docs survive. (Do this first.) |

**Exit:** topology + cost decided; Pass-2 either recovered or a method chosen; docs safe.

## Phase 1 — Agent re-platform to Python (uplink-agent v5, ~2–2.5 wk part-time)

**DECISION (revised after architecture review):** re-platform the agent from Node to **Python**, aligning with the Python backend — single-maintainer stack unification. De-risked by (a) the L-chain contracts specifying exact behavior, (b) the golden fixture providing pass/fail validation, (c) Phase 1 already gutting the uploader regardless. **Conditions: port, don't redesign** — implement the L-chain contracts exactly; all hardening items land as part of the port because they ARE the contracts.

| # | Task | Contract |
|---|---|---|
| 1.1 | **Skeleton**: `uplink-agent` Python package, two entrypoints (`scanner`, `uploader`) mirroring the systemd units; pydantic-settings config; structlog; pytest. **Harvest gittest2**: `resilience.py` (circuit breaker/retry), boto3 patterns, test conventions. | — |
| 1.2 | **Scanner port**: subprocess `hackrf_sweep`, line-parse, enrich, buffered JSONL writes, 5-min/50-MB rotation, per-device folders, staggered start, serial-pinned device binding, startup HackRF verification. Identity from env (`FACILITY` + host prefix — **never hostname-derived**), `^[a-z]{3}-rf\d+[a-d]$` validated at startup. | L0/L1/L2 |
| 1.3 | **Queue + store-and-forward port**: pending→processing→completed lifecycle, crash-recovery re-queues `processing/`, DLQ + replay **to the object store only** (no Postgres path exists to excise — it simply isn't ported), circuit breaker, **disk-space ceiling with eviction/alert** on pending+DLQ. | L3/L5 |
| 1.4 | **Uploader port**: edge filter (FORWARD/AGGREGATE/FILTER — KEPT: noise ≈90% of traffic, feed prep is load-bearing) with the **FEED SPLIT**: signal feed `obs_…` (FORWARD only) + context feed `ctx_…` (summaries, L4.5 heartbeats, config stamp). L4.5 override (versioned rules), **real gzip**, `config_version` + `schema_version`, R2 keys derived from identity config — not filename regex. Shared uploader + DLQ per host. | L4/L4.5/L0.5/L6 |
| 1.4b | **Agent heartbeat**: small JSON to `_health/{agent_id}.json` each interval (status, disk %, queue depth, DLQ count, config_version) — distinguishes "sensor dead" from "RF quiet"; feeds the Sensor dashboard. | new |
| 1.5 | **Deployment**: keep the systemd model verbatim (templated `uplink-scanner@{a..d}` + `uplink-uploader`, resource caps, hardening, plugdev) — swap `ExecStart` to python. Per-host config template (`FACILITY`, host prefix, `HACKRF_SERIAL_A..D`, noise floor); git-track per-host `.env`. | L0/L0.5 |
| 1.6 | **Validation (greenfield)**: agent-level fixture personas (P4 override, P7 edge semantics, P12 gzip, rotation/idempotency) green + 48 h soak on the test host. *Optional:* run the old Node agent briefly as a **reference generator** and diff semantics — an aid, not a gate (nothing to cut over from). | fixture |

**Exit:** Python agent runs 48 h on the test host; fresh production bucket shows facility-prefixed, gzipped, config-stamped `schema_version=v5` batches; kill-WAN test shows bounded disk + successful replay to R2.

## Phase 2 — Server foundation in bss_earv1 (~1 wk)

Per `RFANALYSIS-SCOPE-v2`. No detection logic yet — plumbing with contracts.

- 2.0 **Containerized server from day one**: compose + GHCR images (dev/prod parity per Guy's deployment direction). Agent stays bare systemd+venv (USB passthrough not worth it).
- 2.1 Create `pegasus/apps/rfanalysis/`; Postgres models: **`Sensor`** (registry/allowlist, facility, band_group, status pending|qa|active|disabled), `AnalysisRun`, `IngestLedger`, `InfrastructureSignature`. Admin for Sensor.
- 2.2 **Sync service** (harvest gittest2 `sync.py`): iterates `Sensor.status='active'` only; discover-pending surfaces new prefixes, never ingests. (R5.0)
- 2.3 **Two-table ingest into PER-AGENT DuckDB files** (`l1/{facility}/{agent_id}.duckdb`): persistent `raw_landing` (as-is, compression-detected) → batched **enrichment** to `observations` (casts, folder-authoritative identity, quarantine table with reasons, ledger in same transaction). Stage base class enforces open→work→close + `memory_limit`. (R5.1)
- 2.4 **Celery per-agent chain** skeleton: sync → land → enrich, completion-gated, halt-on-failure. Chains run **in parallel across agents** (per-agent files remove the single-writer lock); throttle = worker concurrency × per-connection memory_limit. `rfanalysis.toml` + `check_config`.
- 2.5 Contract tests via **golden fixture** (`GOLDEN-FIXTURE-SPEC-v1`): build the generator + ingest personas P9–P13 (quarantine, idempotency, gate, fake-gzip, config drift).

**Exit:** test host's real R2 data lands → enriches on a schedule; a fake new prefix shows as `pending` and is never ingested; quarantine counts visible.

## Phase 3 — Event construction + noise filtering (~1–1.5 wk)

The methodology, per `EVENT-CONSTRUCTION-METHODOLOGY-v1`.

- 3.0 **Benchmark to size the Pass-2 budget** (two-pass itself is LOCKED — see methodology NEVER): measure candidate edge-reconciliation calcs on DuckDB per-agent files at realistic volume to establish how expensive Pass 2 can afford to be per agent. Informs the method choice, never the structure.
- 3.1 Power clustering: Pass-1 fast bucketing + **Pass-2 seam** (recovered calc or chosen candidate per 0.1/3.0; placeholder never silently canonical).
- 3.2 **Noise filtering**: hourly presence×activity, **low-duty-cycle exception**, signature filter, over-filter safeguard. (carry rfanalysis-v1 logic + hours_present fix)
- 3.3 **Events**: sweep-collapse (window from 0.4) → **burst grouping 600s = the event**.
- 3.4 **Handoff EXPORT**: write `events` + `event_gaps` as partitioned Parquet `handoff/{facility}/{agent_id}/{date}/` with `run_id`/`config_version`/`dataset_version`; stand up the **dedicated Level 2 DuckDB** reading the partitions (read-only smoke query); handoff contract test runs against the Parquet.
- 3.5 Edge-semantics enforcement: FORWARD-only into events; AGGREGATE/FILTER weight presence via `observation_count` (false-CRITICAL regression fixture).
- 3.6 Extend golden fixture with event personas P1–P8 + whole-run assertions (determinism, conservation, handoff contract, safeguard). P5 (boundary straddler) is `xfail` until Pass-2 lands.

**Exit:** the **historical test corpus** (old-bucket real data, incl. the 20595 target — ingested via a one-off legacy-schema adapter kept out of production paths) produces a sane, deterministic event dataset passing the handoff contract test; power-saving-pattern fixture survives the infrastructure filter.

## Phase 4 — Onboarding & QA workflow (~0.5 wk)

- 4.1 `qa` path: isolated per-agent run (staging context), excluded from anything production.
- 4.2 Dashboard minimal: Sensor status board (harvest earnest-v3 `rf-device-status` for health/staleness), pending-prefix review, run history, quarantine/suppression counts.
- 4.3 Promotion flow pending→qa→active documented as an ops checklist.

**Exit:** bring a second host online end-to-end *through the gate*: appears pending → QA'd in isolation → promoted → in production runs.

## Phase 5 — Fleet rollout (per facility, ~1 wk for first facility)

- 5.1 Template install (Phase-1 agent release) across the first facility's PCs; per-host config checklist (1.8) each.
- 5.2 Seed `Sensor` registry; onboard each host via the Phase-4 gate.
- 5.3 Run 1 week: volume/timing/disk validation; **start the labeled ground-truth corpus** — log every physically-confirmed device from day one.
- 5.4 Ops hardening: backups (DuckDB file, Postgres, R2 lifecycle), restore drill, health alerts.

**Exit:** first facility fully onboarded; a week of clean events; corpus collection running; restore drill passed.

## Phase 6 — Operational documentation (parallel with 4–5, ~2–3 days)

Close the audit's item-E gaps as short contract docs: pipeline **recovery semantics** (largely solved by per-agent disposable files: delete file → re-run chain from R2+ledger; document the manifest + partial-export cleanup), **retention/lifecycle** (landing, DuckDB, Postgres, reports), **time-sync** tolerance + drift detection, **security/access** basics.

**Exit:** four short docs committed; recovery semantics implemented in the Celery chain.

## Phase L2 — Analysis (SEPARATE EFFORT — chartered later, not scheduled here)

Consumes the handoff contract. Backlog seeded: correlation method (open algorithm), CV/periodicity port, scoring/classification, `ForensicReportGenerator` harvest, per-facility reports, A/B framework (needs the corpus from 5.3).

---

## Sequence & dependencies

```
0 (decisions/hunt) ──► 1 (agent) ──► 2 (server foundation) ──► 3 (events) ──► 4 (onboarding) ──► 5 (fleet, per facility)
                                                                     6 (ops docs) runs parallel with 4–5
0.1/0.3/0.4 feed 3.1/3.3.   0.2 feeds 5 (deployment shape).   L2 starts any time after 3 (handoff exists).
```

**Rough calendar (part-time, single operator + Claude Code):** Phases 0–3 ≈ 4–5.5 weeks to a correct end-to-end Level 1 on real data (Python agent re-platform adds ~1–1.5 wk over hardening-in-place); +2 weeks through first-facility rollout. Additional facilities repeat Phase 5.

## Standing rules while executing
- Every build session starts by pasting `BUILD-SESSION-BOOTSTRAP-v1.md`.
- Fixture is the gate: a phase isn't done until its personas are green (`GOLDEN-FIXTURE-SPEC-v1`).
- Contract change ⇒ doc change in the same commit (R0). Version numbers in filenames.
- Placeholders (Pass-2) stay loudly marked; never silently canonical.
- Every session ends: outputs presented + saved locally (nothing persists).
- Log every physically-confirmed device into the corpus from the first day of real data.
