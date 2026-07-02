# EAR System — Greenfield Development Plan (v1)
### Master plan: ground-up build, fresh repo per component, Opus-agent build sessions
**Supersedes:** `archive/PROJECT-PLAN-v1.md` and the roadmap half of `archive/EARSYSTEM-PRODUCTION-PLAN-v3.md`.
**Companions:** `DECISION-LOG-v1.md` (every decision + rationale, including ones made unilaterally during this planning effort — flagged `[CLAUDE-DECIDED]` for post-review) · `OPEN-QUESTIONS-v1.md` (clarifications wanted from Guy).

---

## 1. What is being built (one paragraph)

A multi-facility RF tracker-detection platform. Python agents on warehouse PCs drive HackRF radios sweeping LTE uplink bands, prepare the feed at the edge (≈90% of raw traffic is noise), and upload signal + context streams to a central Cloudflare R2 bucket under facility-scoped identities. A containerized Django/Pegasus server ingests per-agent data through an onboarding-gated, two-table pipeline into per-agent DuckDB files, constructs device **events** (burst groups) via two-pass power binning and pattern-aware noise filtering, and exports an immutable Parquet **event dataset** — the handoff to a future, separately-chartered Level 2 analysis effort that produces forensic reports. Everything is governed by written contracts with an executable golden fixture as the enforcement gate.

## 2. Component & repository topology (fresh repos — see D-030…D-036)

| Repo | Component | Contents | Visibility |
|---|---|---|---|
| **`ear-docs`** | Knowledge base (THIS repo, renamed) | Contracts, component specs, plans, decision log, questions. **Source of truth** — any contract change lands here in the same PR-cycle as the code change. | private |
| **`ear-agent`** | Capture/upload agent v5 (Python) | Scanner + uploader packages, edge filter + feed split, L4.5 overrides, SAF/DLQ, heartbeat; its own systemd templates + install script; agent-level tests consuming `ear-fixture`. Spec: `components/EAR-AGENT-SPEC-v1.md`. | private |
| **`ear-server`** | Level 1 server (fresh SaaS Pegasus generation) | Django host + `rfanalysis` app: Sensor registry/gate, sync, landing→enrich, ephemeral pipeline job (D-044), per-agent DuckDB, event construction, Parquet handoff export; compose + GHCR. Spec: `components/RFANALYSIS-SCOPE-v3.md`. | private |
| **`ear-fixture`** | Golden fixture & validation data | Deterministic generator (**19 personas**), committed expected outputs, whole-run assertions, **legacy-corpus adapter** (old-bucket data → v5 shape, quarantined from production code). Versioned releases consumed by agent + server CI. Spec: `components/GOLDEN-FIXTURE-SPEC-v1.md`. | private |
| **`ear-fleet`** | Operational state (NO code) | Per-facility/per-host git-tracked `.env` configs, sensor inventory seed, L4.5 override rule files, provisioning checklists, labeled ground-truth corpus log. | private |
| **`ear-analysis`** | Level 2 (best-effort v1, D-055) | Django app installed into the ear-server deployment (D-042). L2 DuckDB reads the Parquet handoff only; feature extraction, rule-based G1–G6 classification, four-factor scoring, **provisional** correlation, results to Postgres; dashboard pages. Spec: `level2/DEVICE-BEHAVIOR-TAXONOMY-v1.md`. | private |

**Dependency direction:** `ear-docs` ← referenced by all. `ear-fixture` ← test-dep of agent & server. `ear-fleet` ← consumed at provision/ops time. Agent and server never depend on each other — only on the R2 contract.

## 3. Cross-cutting engineering conventions (D-037…D-041)

- **Versioning:** semver + git tags per repo; `schema_version=v5` on wire records; `dataset_version` on the handoff; docs keep version-numbered filenames.
- **CI gate = the fixture.** Every repo's CI runs its slice of the golden fixture; a red persona blocks merge. New rule ⇒ new persona in the same change.
- **Machine-readable contracts:** `ear-docs/contracts/schemas/` will hold JSON Schemas for the observation record (signal + context feeds), heartbeat, and the handoff Parquet column spec — generated during WP-D1, kept beside the prose contracts. Prose explains; schema enforces.
- **Config:** pydantic-settings (agent, server) reading env + TOML; no hardcoded tunables; every tunable listed in `operations/PARAMETER-TUNING-v1.md`.
- **Branching:** trunk-based, short-lived feature branches, tags for releases. Conventional commits.

## 4. Work breakdown — packages sized for Opus build sessions

Each work package (WP) is written to hand directly to a build agent: scope, contract references, definition of done. **Execution order + Opus/Sonnet assignment per session: `plan/BUILD-RUNSHEET-v1.md`.** **DoD for every WP = code + fixture personas green + docs updated + committed.** Session bootstrap: `operations/BUILD-SESSION-BOOTSTRAP-v2.md` (parameterized per WP).

### M0 — Foundation (Guy + 1 session)
- **WP-D0 (Guy):** Provision the Strix box per `operations/DEV-ENVIRONMENT-v2.md` step 0 (local bare repos = origin, D-048); create the **6** GitHub mirrors (incl. ear-analysis, D-055); seed `ear-docs` from the zip and push both remotes. Phase-0 items: hunt for Pass-2 calc code; decide server topology; decide false-split/merge cost; confirm sweep-collapse window. Answer `OPEN-QUESTIONS-v1`.
- **WP-D1:** Repo skeletons + CI stubs. ~~JSON Schemas~~ **DONE** — authored in `contracts/schemas/` (signal/context v5, heartbeat v1, handoff columns v1; D-052 pins canonical field names).

### M1 — Fixture first (1–2 sessions) — `ear-fixture`
- **WP-F1:** Deterministic generator + personas P9–P13 (ingest-class) + committed expected outputs.
- **WP-F2:** Event-class personas P1–P8 **and P15–P19 (D-057)** + whole-run assertions (determinism, conservation, handoff conformance). P5 ships `xfail`.
- **WP-F3:** Legacy-corpus adapter (old bucket → v5 files) for M4 validation. Kept out of any production path.

### M2 — Agent (`ear-agent`, 4–5 sessions, ~2–2.5 wk elapsed part-time)
- **WP-A1:** Package skeleton, pydantic-settings config surface, identity module (facility+host from env, regex-validated), structlog, pytest harness wired to `ear-fixture`.
- **WP-A2:** Scanner: `hackrf_sweep` subprocess driver, line parser, enrichment, buffered JSONL writes, rotation (5 min/50 MB), per-device folders, serial-pinned binding + startup verification, staggered start.
- **WP-A3:** Queue lifecycle + store-and-forward (D-045): pending→processing→completed, crash re-queue; `pending/` as retry queue (no circuit breaker, no watcher); poison-file DLQ + replay **to R2 only**; **disk-space ceiling** checked per run with eviction+alert.
- **WP-A4:** Uploader as **ephemeral timer oneshot** (D-045, ~5-min cadence): edge filter with **feed split** (`obs_`/`ctx_`), L4.5 override engine (rules re-read each run from `ear-fleet`), real gzip, config/schema stamping, identity-derived R2 keys, **heartbeat written per run** (incl. queue depth, disk %).
- **WP-A6 (dev harness, D-046/D-047):** dev environment per `operations/DEV-ENVIRONMENT-v2.md` (Strix Halo, container parity, MinIO profile, bring-up + live-capture HackRF step); capture-source seam (`hackrf`/`replay` behind one interface), replay driver (timestamp shift, speed factor, raw-stdout + legacy-JSONL inputs), `--record` mode, dev compose (MinIO + optional server stack) for the full local loop. Replay datasets committed to `ear-fixture`.
- **WP-A5:** systemd templates + installer; **`ear-agent check` self-check command (D-059)** wired into the provisioning checklist; 48 h test-host soak; kill-WAN drill. Optional Node reference diff.

### M3 — Server foundation (`ear-server`, 3–4 sessions)
- **WP-S1:** Fresh Pegasus generation; `rfanalysis` app; Postgres models (Sensor/AnalysisRun/IngestLedger/InfrastructureSignature) + admin; **compose per D-047 parity table** (proxy/web/pipeline/postgres + minio dev profile) + GHCR images.
- **WP-S2:** Sync + onboarding gate (active-only iteration; pending-prefix surfacing; qa isolation path).
- **WP-S3:** Two-table ingest into per-agent DuckDB files: landing (as-is, compression-detected) → enrichment (casts, folder-authoritative identity, quarantine-with-reason, ledger-in-transaction). Signal and context feeds land separately.
- **WP-S4:** **Ephemeral pipeline job (D-044) + health sweep (D-059):** `check_health` command + timer + `HealthSweep` model + dashboard banner + state-change notify (spec: `operations/HEALTH-CHECKS-v1.md`); and: `run_level1` management command — per-agent stages as sequential code, agents parallelized via process pool (open→work→close, `memory_limit`), halt-on-failure = non-zero exit + alert, run lock against overlap, systemd timer/cron schedule, `docker compose run --rm pipeline`. No Celery/Redis in Level 1. + `check_config`.

### M4 — Event construction (`ear-server`, 3–4 sessions)
- **WP-S5:** Pass-2 compute-budget benchmark (structure is LOCKED; benchmark sizes the method).
- **WP-S6:** Power clustering: Pass-1 bucketing + Pass-2 seam (recovered calc or chosen candidate; stub loudly marked; P5 `xfail` until real).
- **WP-S7:** Noise filtering: presence×activity + low-duty-cycle exception + signature filter + safeguard.
- **WP-S8:** Events: sweep-collapse → burst grouping (600 s) = event.
- **WP-S9:** Handoff export: atomic partitioned Parquet + dedicated L2 DuckDB read-smoke + conformance test.
- **WP-S10:** **Corpus validation:** run the legacy corpus (incl. 20595) end-to-end; human review of the event dataset against known truth.

### M4.5 — Level 2 best-effort (`ear-analysis`, 2–3 sessions, D-055)
- **WP-L1:** L2 DuckDB feature extraction over handoff partitions (gap stats/CV, duty, presence, power variance, first/last-seen, bursts); read-only, glob by facility.
- **WP-L2:** Rule-based classifier v1 (taxonomy decision order G4→G5/G6→G3→G1/G2→unclassified queue) + four-factor candidate scoring; persist `DeviceClassification`/`TrackerCandidate`.
- **WP-L3:** PROVISIONAL correlation (time-overlap within `(facility, band_group)`) behind a seam — marked provisional like Pass-2; persist `CorrelatedDetection`.
- **WP-L4:** Analysis dashboard per `components/DASHBOARD-DESIGN-v1.md` (Facility summary, Classifications, Candidates, Unclassified queue, Device detail w/ event timeline); `run_level2` job + timer. UAT journeys J3/J6 are the acceptance scripts.
- **Exit:** legacy corpus (S18 output) classifies sensibly — 20595 lands in G1, known infra in G4, phones in G3 — reviewed by Guy; unclassified queue populated, not empty-by-force.

### M5 — Operations & first facility (2–3 sessions + Guy on-site)
- **WP-O1:** Ops dashboard per `components/DASHBOARD-DESIGN-v1.md` (Sensors board, Onboarding queue, Runs, Quarantine, Suppression). UAT journeys J1/J2/J4/J5/J7 are the acceptance scripts.
- **WP-O2:** `ear-fleet` provisioning: per-host checklist, config templates, first-facility rollout, corpus logging live from day one.
- **WP-O3:** ~~Recovery + retention~~ **PRE-WRITTEN** (`operations/RECOVERY-RETENTION-v1.md`). Remaining: **time-sync** tolerance + drift detection, **security/access** basics. Verify/refine during M5: `RUNBOOK-v1`, `BACKUP-DR-v1` (both drills), `HOST-PROVISIONING-CHECKLIST-v1` (template → ear-fleet copies).

**Milestone exits:** M1 = fixture releases consumable; M2 = fresh bucket receiving valid v5 feeds from soaked test host; M3 = gated ingest of real data, quarantine visible; M4 = corpus produces a sane, deterministic, conformant event dataset; M5 = first facility live + corpus collection running. **Level 1 done = M5.**

## 5. Dependency graph & rough calendar

```
M0 ─► M1 ─► M2 (agent)  ─┐
        └─► M3 (server) ─┴─► M4 ─► M5        M2 ∥ M3 possible (different repos/sessions)
```
Part-time, single operator orchestrating Opus sessions: **M0–M4 ≈ 5–7 weeks; M5 +1–2 weeks** for the first facility. Additional facilities repeat WP-O2 only.

## 6. Risk register (top items)

| Risk | Mitigation |
|---|---|
| Pass-2 calc never found & wrong candidate chosen | Seam + `xfail` P5 keeps it swappable; corpus validation (WP-S10) is the acceptance check; split/merge-cost decision sets conservatism |
| Opus sessions drift from contracts | Bootstrap v2 per WP; fixture as merge gate; port-don't-redesign rule; decision log forbids silent reopening |
| Fleet config drift (60 hosts) | `ear-fleet` git-tracked configs; config_version stamped in batches; server flags drift |
| WAN outages at facilities | SAF is load-bearing (tested by kill-WAN drill); disk ceiling bounds the blast radius |
| Single maintainer bus-factor | Everything in `ear-docs`; decision log; fixture encodes intent executably |
