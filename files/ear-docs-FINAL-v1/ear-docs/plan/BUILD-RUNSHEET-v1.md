# EAR System — Build Runsheet (v1)
### Session-by-session execution order for Claude CLI (Opus/Sonnet agents). Start state: Ubuntu LTS box + historic HackRF log folders + this repo. Every session starts by pasting `operations/BUILD-SESSION-BOOTSTRAP-v2.md` with the brackets filled from this sheet.

**Model assignment rule:** **Opus** for contract-critical logic (edge filter, ingest, event construction, fixture expected-outputs) and anything where a subtle semantic error corrupts data silently. **Sonnet** for skeletons, boilerplate, deploy plumbing, CRUD dashboards. When in doubt, Opus.

**Question-gate rule:** each session lists `ASK` (must have Guy's answer) or `DEFAULT` (proceed on the logged provisional, flag in the session summary). Unanswered `ASK` = do a different session.

| # | Session (WP) | Repo | Model | Gate to pass | Questions |
|---|---|---|---|---|---|
| S00 | Provision box (DEV-ENVIRONMENT-v2 step 0): layout, bare repos, mirrors, Docker | — (Guy + Sonnet assist) | Sonnet | `git clone` from local origin works; compose hello-world | ASK Q-01/Q-02 (defaults exist: `ear-` names, private) |
| S01 | **Historic-log intake**: inventory Guy's raw folders → classify (raw sweep stdout? legacy JSONL? fake-gzip?) → curate into `ear-fixture/replay/` with a manifest (source host, date range, format, size) | ear-fixture | Sonnet | Manifest committed; samples validate against nothing yet (raw intake only) | DEFAULT Q-20 (classify what exists) |
| S02 | WP-D1: repo skeletons + CI stubs (pytest + fixture hook), pyproject, lint | all | Sonnet | CI runs empty-green in each repo | — |
| S03 | WP-F1: fixture generator + ingest personas P9–P13 + expected outputs | ear-fixture | **Opus** | Generator deterministic (two runs byte-identical); expected outputs committed | — |
| S04 | WP-F2: event personas P1–P8 + **P15–P19** + whole-run assertions (P5 `xfail`) | ear-fixture | **Opus** | Expected outputs double-derived (`expected/derivations/`); **Guy spot-reviews P1/P2/P15 derivations** | DEFAULT Q-05/Q-06 |
| S05 | WP-F3: legacy-corpus adapter (historic JSONL/fake-gzip → v5 shape) using S01 manifest | ear-fixture | **Opus** | Sample of real historic files converts + validates against `contracts/schemas/` | DEFAULT Q-12 (uses local folders) |
| S06 | WP-A1: agent skeleton, config surface, identity module, test harness | ear-agent | Sonnet | Identity tests green (regex, env-pinning, facility) | — |
| S07 | WP-A2: scanner — CaptureSource seam, hackrf driver, parser, enricher, writer, rotation, `--record` | ear-agent | **Opus** | Replay of S01 raw captures → valid v5 rows (schemas); rotation correct; **record→replay ROUND-TRIP: replaying a recording reproduces the original run's rows byte-identically** | DEFAULT Q-20 |
| S08 | WP-A3: queue lifecycle + SAF (pending-as-retry, poison DLQ, disk ceiling) | ear-agent | **Opus** | Crash-requeue + ceiling tests green | DEFAULT Q-09 remainder |
| S09 | WP-A4: uploader oneshot — edge filter + feed split, L4.5 engine, gzip, keys, heartbeat | ear-agent | **Opus** | Personas P4/P7/P12/P14 green against MinIO; obs/ctx/health keys correct | DEFAULT Q-08 (carry v4.3 thresholds, re-derive in soak) |
| S10 | WP-A6: dev harness wiring — MinIO profile, replay end-to-end, timers on the box | ear-agent(+server compose stub) | Sonnet | **Replay agents → MinIO loop runs on the box** | — |
| S11 | WP-S1: fresh Pegasus gen, rfanalysis app, models+admin, compose per parity table, GHCR | ear-server | Sonnet | Stack up; Sensor admin usable; images build | ASK Q-11 only for PROD deploy — dev proceeds |
| S12 | WP-S2: sync + onboarding gate + pending-surfacing + qa isolation | ear-server | **Opus** | Persona P11 green (pending never ingested) against MinIO | — |
| S13 | WP-S3: two-table ingest into per-agent L1 files (landing, enrich, quarantine, ledger, both feeds) | ear-server | **Opus** | P9/P10/P12/P13 green; quarantine visible in admin | — |
| S14 | WP-S4: ephemeral pipeline job + **health sweep (D-059)**: run_level1, pool, lock, timers, notify(), check_health + HealthSweep + banner | ear-server | Sonnet | Job runs end-to-end; non-zero-exit alert fires; **staged stale-sensor/failed-run → correct sweep verdicts + one state-change notify** | DEFAULT Q-19; ASK ntfy topic (or default) |
| S15 | WP-S5+S6: Pass-2 budget benchmark; power clustering + Pass-2 seam | ear-server | **Opus** | Benchmark report committed; P5 still `xfail`, seam swappable | **ASK Q-03** (hunt result / candidate choice) |
| S16 | WP-S7+S8: noise filter (exception+safeguard) + events (collapse→burst) | ear-server | **Opus** | P1–P3, P8 green; P2 protection asserted | DEFAULT Q-06 |
| S17 | WP-S9: Parquet handoff export + L2 DuckDB smoke + conformance | ear-server | **Opus** | Whole-run assertions green (determinism, conservation, conformance) | — |
| S18 | WP-S10: **legacy-corpus end-to-end** (S05 adapter output through the full pipeline); human review vs known truth (20595) | ear-server | **Opus** | Event dataset sane vs Guy's knowledge — **Guy reviews** | ASK Q-13 (band plan confirm) |
| S19 | WP-O1: ops dashboard (sensor board w/ heartbeat staleness, pending review, runs, quarantine) | ear-server | Sonnet | Usable against dev data | — |
| S19a | WP-L1: L2 feature extraction over handoff Parquet | ear-analysis | **Opus** | Features match hand-computed values for personas P1–P3 | — |
| S19b | WP-L2: classifier v1 (G1–G6) + scoring; Postgres models | ear-analysis | **Opus** | P1→G1, P2→G1, P3→G4 assertions green; unclassified queue works | DEFAULT Q-T1..T4 (rules provisional until field answers) |
| S19c | WP-L3: provisional correlation (seam, marked) | ear-analysis | **Opus** | Same-device-two-sensors fixture case correlates; seam swappable | DEFAULT (real method open) |
| S19d | WP-L4: analysis dashboard + run_level2 job/timer | ear-analysis | Sonnet | **Legacy corpus classifies sensibly — Guy reviews (20595→G1)** | ASK Q-16 (summary page shape) |
| S19e | **UAT gate (D-056):** Guy runs journeys J1–J7 against dev data (fixture + legacy corpus) per DASHBOARD-DESIGN-v1 | ear-server/ear-analysis | Guy + Sonnet (fixes) | All journeys ✅ or filed fixes; **M5 blocked on open red journey** | ASK Q-16 informs J6 |
| S20 | HackRF live step: plug into box, `--source hackrf --record`, real capture joins replay sets | ear-agent (Guy + Opus) | **Opus** | Live RF → valid v5 rows → pipeline events; **round-trip on live recording** (replay == live output) | ASK Q-07 (floor for the room) |
| S21+ | WP-A5 soak on real test host → WP-O2 first facility (checklists, ear-fleet, drills) → WP-O3 remainder | mixed | mixed | M5 exits per plan | ASK Q-04/Q-14/Q-15/Q-21 before facility |

**Parallelism:** after S05, the A-track (S06–S10) and S-track (S11–S14) can interleave or run in parallel CLI sessions. S15–S18 need both tracks' outputs.

## Verification protocol (D-058 — how "functioning as spec'd" is confirmed)
1. **One gate command per session.** Every exit gate reduces to a single reproducible command (`make gate` in each repo → runs the session's persona slice + schema validation + unit tests). If the gate can't be run as one command, the session isn't gate-able and isn't done.
2. **Evidence committed.** The gate command writes a **gate report** (`gate-reports/S{nn}-{date}.txt`: what ran, pass/fail, fixture release version, xfail list) committed with the session's final commit. Agent claims are not evidence; reports are.
3. **Guy re-runs independently.** A session is marked complete only after Guy runs the same gate command in a fresh shell and sees green. Thirty seconds of trust-but-verify per session.
4. **Human-oracle points** (machine can't confirm — Guy's field truth is the test): S04 derivation spot-review · **S18 corpus events vs known truth (20595)** · S19d classification sanity · S19e UAT journeys · S20 live RF. These five cannot be delegated.
5. **Regression floor:** the full persona suite (not just the session's slice) runs in CI on every commit — a session can't pass its own gate while breaking a previous one.

**Standing session rules:** bootstrap pasted first · fixture green (or xfail-only) to close · push local origin **and** GitHub mirror at close (D-048) · new decisions → DECISION-LOG, new questions → OPEN-QUESTIONS, both in the same commit.
