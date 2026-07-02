# EAR System — Knowledge Base (`ear-docs`)
### Multi-facility RF tracker detection · **FINAL v1 (pre-code, for expert review)** · greenfield build ready
> **Reviewers:** read `REVIEW-BRIEF.md` then `MANIFEST.md`. 62 decisions (D-001…D-062), 19 personas, runsheet S00–S21 — nothing built yet; every gap you find costs one edit, not a rewrite.
**Entry point.** Supersedes `archive/SESSION-HANDOFF-v1.md`. This repo is the **source of truth**: contracts change here in the same cycle as code (D-033).

## What this is
Six months of experimental lessons distilled into contracts, a plan, and an executable fixture spec for a ground-up rebuild: Python agents (HackRF capture, edge feed-prep, R2 upload) → containerized Django/DuckDB Level 1 server (gated ingest → event construction → Parquet handoff (ephemeral job model, D-044)) → future Level 2 analysis (separate effort). Noise ≈90% of raw traffic; the system's job is preparing and trusting the other 10%.

## Reading order
| # | Doc | What it is |
|---|---|---|
| 1 | `plan/GREENFIELD-DEVELOPMENT-PLAN-v1.md` | **The master plan**: 6-repo topology (incl. `ear-analysis`, D-055), work packages, milestones M0–M5, risks |
| 1a | `plan/BUILD-RUNSHEET-v1.md` | **Execute this**: sessions S00–S21, Opus/Sonnet per session, question gates |
| 1b | `plan/REVIEW-DISPOSITION-v1.md` | **After the audit**: how findings are triaged/folded before build proceeds (D-060) |
| 2 | `DECISION-LOG-v1.md` | Every decision + rationale; **all `[CLAUDE-DECIDED]` items D-030…D-062** await Guy's post-effort review (esp. D-042/D-050/D-052/D-061) |
| 3 | `OPEN-QUESTIONS-v1.md` | Q-01…Q-18 for Guy, ordered by what they block |
| 4 | `components/` | Build specs: `EAR-AGENT-SPEC-v1` · `RFANALYSIS-SCOPE-v3` (server) · `GOLDEN-FIXTURE-SPEC-v1` · `DASHBOARD-DESIGN-v1` (screens + UAT journeys J1–J7) |
| 5 | `contracts/` | The law: `DATA-WORKFLOW-RULES-v3` (R-chain) · `LOCAL-HOST-DATA-FLOW-v2` (L-chain) · `EVENT-CONSTRUCTION-METHODOLOGY-v1` · `EVENT-DATASET-HANDOFF-v1` (+ **`schemas/`** — machine-readable, D-052) |
| 6 | `operations/` | `BUILD-SESSION-BOOTSTRAP-v2` (paste into every Opus session) · `DEV-ENVIRONMENT-v2` · `RUNBOOK-v1` · `HEALTH-CHECKS-v1` · `BACKUP-DR-v1` · `RECOVERY-RETENTION-v1` · `HOST-PROVISIONING-CHECKLIST-v1` · `PARAMETER-TUNING-v1` |
| 7 | `level2/` | `DEVICE-BEHAVIOR-TAXONOMY-v1` — **the v1 classification spec** (G1–G6; Level 2 un-deferred to best-effort build, D-055) |
| 8 | `archive/` | Superseded docs, kept for history |

## Repo map (see D-030)
`ear-docs` (this) · `ear-agent` (Python agent v5) · `ear-server` (fresh Pegasus + rfanalysis) · `ear-fixture` (golden fixture + legacy-corpus adapter) · `ear-fleet` (private per-host configs, override rules, corpus log) · `ear-analysis` (Level 2 best-effort v1, D-055).

## Where things stand
- **Done:** full Level 1 architecture; contracts L0–L6 / R0–R9; event methodology (event = burst group, 600 s); handoff contract (atomic Parquet); fixture spec (19 personas); greenfield review (edge filter kept on the 90% number, feed split, heartbeat, containers); this plan.
- **Next:** **M0/WP-D0 (Guy):** apply the audit Phase-A pre-repo fixes (`audit/BUILD-READINESS-PLAN.md`); create repos + push; hunt Pass-2 code; answer Q-01…Q-05 (+ `MIN_OBSERVATIONS`); review D-030…D-062. Then WP-D1 → M1 fixture-first.
- **Open algorithms:** Pass-2 boundary calc (seam + `xfail` until resolved) · Level 2 correlation method (deferred with Level 2).
- **From day one of real data:** log every physically-confirmed device into the labeled corpus (`ear-fleet`) — the future tuning ground truth that can't be reconstructed later.

> **Persistence warning:** Claude session outputs do NOT persist. This repo on Guy's machine + GitHub is the only source of truth.
