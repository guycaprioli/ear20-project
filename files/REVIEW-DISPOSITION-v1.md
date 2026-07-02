# EAR System — Review Disposition Process (v1)
### What happens when the expert audit returns. The review is not done until its findings are triaged, adjudicated, and folded back into the design. (D-060)

## Inputs
The panel's `REPORT.md` (sections A–G) + reviewer files + rebuttals. Findings carry severity, steelman, failure_scenario, proposed_alternative, downstream_impact.

## Step 1 — Triage (Guy + one Claude session)
Sort every surviving finding into exactly one bucket:
- **ACCEPT** — valid; will change the design. → Step 2.
- **ACCEPT-DEFER** — valid but not now (post-first-facility, or Level 2 era). → OPEN-QUESTIONS with a trigger condition.
- **REJECT** — steelman holds / rationale already answered / reviewer erred. → record WHY in the disposition log (rejecting a finding is itself a logged decision).
- **NEEDS-INFO** — can't decide without field truth or a spike. → becomes a question or a small investigation task.

## Step 2 — Adjudicate dissents (Section F)
For each unresolved reviewer disagreement, Guy makes the call; the reasoning is logged as a new decision (D-nnn). Dissents are never left open into build.

## Step 3 — Fold accepted findings into the design
Each ACCEPT becomes doc edits in the SAME discipline as the rest:
- contract/decision change → the contract doc + a new/amended decision (D-nnn) in the SAME commit;
- new failure mode → a new fixture persona (R0.3: every rule has a check);
- new open item → OPEN-QUESTIONS;
- structural (pre-repo) change → applied BEFORE WP-D0 creates repos.
Version-bump any materially changed doc; note "revised per review finding-{id}" in the commit.

## Step 4 — Disposition log
`plan/REVIEW-DISPOSITION-LOG.md`: one row per finding — id, bucket, decision/persona/question it produced, one-line rationale. This is the audit trail proving every finding was handled, not dropped.

## Step 5 — Re-review trigger
If Step 3 produces **structural changes** (repo topology, a contract's core semantics, the L1/L2 boundary, identity), re-run the affected reviewer lens on the changed sections only — not the whole panel. Minor/localized changes need no re-review; the fixture is the regression guard.

## Exit (planning phase truly complete when)
Every finding is in a bucket with a disposition-log row; all dissents adjudicated; all ACCEPTs folded; all pre-repo structural changes applied. THEN WP-D0 (repo creation) proceeds. The review is a gate on build, not a parallel nicety.
