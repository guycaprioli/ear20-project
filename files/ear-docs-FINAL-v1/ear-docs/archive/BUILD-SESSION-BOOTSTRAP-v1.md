# EAR System — Build Session Bootstrap (v1)
### Paste this at the start of every Claude Code / build session. It is the session's operating contract.

---

You are working on the **EAR System** — a multi-facility RF tracker-detection platform (HackRF sensors → Cloudflare R2 → Django+DuckDB server → event dataset → forensic analysis). The design phase is COMPLETE and documented. Your job is to **build against the documents, not redesign**.

## Read in this order (docs/ folder)
1. `SESSION-HANDOFF-v1.md` — state, locked decisions, outstanding items
2. `PROJECT-PLAN-v1.md` — find the current phase; work ONLY that phase
3. `RFANALYSIS-SCOPE-v2.md` — the app build spec (carry-forward vs rebuild table)
4. The contract for whatever you touch: `DATA-WORKFLOW-RULES-v2` (R-chain) · `LOCAL-HOST-DATA-FLOW-v1` (L-chain, agent) · `EVENT-CONSTRUCTION-METHODOLOGY-v1` (events) · `EVENT-DATASET-HANDOFF-v1` (L1/L2 interface) · `GOLDEN-FIXTURE-SPEC-v1` (the tests you must keep green)

## Locked decisions — do NOT reopen (challenge Guy only with new evidence, never silently)
- `agent_id = {facility}-{host}{device}`, lowercase, `^[a-z]{3}-rf\d+[a-d]$`; facility immutable
- R2-primary transport (central hub); per-agent bucket folders
- Per-agent Level 1 DuckDB files → partitioned Parquet handoff → dedicated Level 2 DuckDB
- Event = burst group (600 s gap, tunable config); binning/events are SERVER-owned (agent bin advisory)
- Two-pass power binning (Pass 2 currently a marked stub — never let the stub become canonical)
- Noise filter keeps the low-duty-cycle exception (removing it = filtering out the primary target)
- Onboarding gate: Sensor allowlist pending→qa→active→disabled; discovery never auto-ingests
- Two-table ingest (persistent landing → batched enrich, quarantine-not-drop, folder-authoritative identity)
- Level 1 stops at the handoff. CV/scoring/classification/correlation/reports = Level 2, out of scope

## Standing rules (Guy's, non-negotiable)
- Correct over easy; fix bugs immediately; verify claims with file:line refs
- Contract change ⇒ doc change + fixture persona change in the SAME commit (R0)
- git discipline; version numbers in filenames; tradeoffs presented for Guy's decision, not decided silently
- Anti-sycophancy: challenge reasoning when warranted; don't fold under pushback
- No hardcoded tunables (burst gap, thresholds) — config only, versioned
- Stage tasks: open DuckDB file → work → close; `memory_limit` set; never open two L1 files in one query
- End of session: `present_files` on deliverables + remind Guy that `/mnt/user-data/outputs/` does NOT persist — his local machine is the source of truth

## Definition of done (any task)
Code + tests (fixture personas green, new rule ⇒ new persona) + doc updated + committed + presented. A task without its test and doc update is not done.

## Session close checklist
1. Fixture suite green (or xfail-only for the Pass-2 stub)
2. Docs updated for anything that changed; versions bumped if material
3. Commits pushed / files presented for download
4. `SESSION-HANDOFF` updated if state or decisions moved
5. One-paragraph summary: what moved, what's next, what's blocked on Guy
