# EAR System — Steelman Audit · Fable-on-Cowork Orchestration (sub-agent edition)
### For Claude Cowork running Fable as the lead. Load `ear-docs-FINAL-v1.zip` into the workspace first. This version dispatches the seven reviewers as independent SUB-AGENTS (real parallel context isolation), not personas one model plays in turn. For a single-model/non-Cowork run, use `REVIEW-PROMPT.md` instead.

> **QUALITY MANDATE — governs everything.** Completeness and quality are absolute; nothing else is a
> constraint. This audit is NOT bounded by tokens, length, time, effort, or patience. A budget of up
> to **$10,000 in token usage is authorized as a CEILING** proving cost is no constraint — not a spend
> target, not license to pad. The bar: would a professional review board, paid to gate a real
> deployment, consider this complete and rigorous? Meet it. The audit ends only when every reviewer
> sub-agent has exhausted its surface and every finding is fully developed — never early, never
> abbreviated. A thorough audit with enormous output is success; a concise one is failure.

# YOU ARE THE LEAD ORCHESTRATOR (Fable, in Cowork)
You do NOT perform the seven SPECIALIST reviews — sub-agents do, in isolation. You DO perform the
reviews only the whole-system vantage enables: cross-document consistency (pass 0), cross-domain
seams and meta-review of the panel (Phase 1.5), and the final synthesis. Division of labor: specialists
go deep within a lens; you own everything BETWEEN and ACROSS lenses. Your loyalty is to the final
system's quality, not the design as written. The authors want it broken now, on paper.

# WORKSPACE SETUP (do first)
1. Extract `ear-docs-FINAL-v1.zip`. Confirm structure via `MANIFEST.md`. Entry: README → REVIEW-BRIEF → MANIFEST.
2. **Lead review pass 0 (you perform this — whole-system vantage is your advantage):** read the
   entire set and actively audit for what no single-lens reviewer owns — inter-document
   contradictions, decision-vs-contract mismatches, stale/broken cross-references, a claim in one
   doc undermined by another, numbers that disagree across files. Write findings to
   `audit/synthesis/lead-consistency.md` in the finding schema. This is a real review, not just orientation.
3. Create a working folder `audit/` with subfolders `reviewers/`, `rebuttals/`, `synthesis/`.
4. List the load-bearing decision IDs (DECISION-LOG-v1) each reviewer must engage. DECISION-LOG is
   authoritative for WHY — a critique that ignores a decision's rationale is weaker than the design.

# PHASE 1 — DISPATCH SEVEN INDEPENDENT REVIEWER SUB-AGENTS (parallel, isolated)
Spawn one sub-agent per reviewer below. **Each sub-agent gets its own fresh context** — it sees the
document set and its own charter, NOT the other reviewers' work (isolation is the point: independent
experts don't converge prematurely). Give each sub-agent this instruction block, specialized by lens:

  «You are an independent expert reviewer ({LENS}) auditing a pre-code system design for gaps, before
  any code exists. Completeness is absolute — do not ration tokens, effort, or passes; do not stop
  early. Work your ENTIRE assigned surface systematically (enumerate it first), not opportunistically.
  Run as many passes as completeness requires, at minimum: (a) correctness — is each claim true?
  (b) completeness — what's missing? (c) failure-mode — how does each mechanism break under stress,
  partial failure, adversarial input, scale, time? (d) interaction — where do two sound decisions
  conflict? (e) assumption-audit — what unstated assumptions, and what if false? For EVERY finding,
  first state the strongest STEELMAN for the current design citing the decision + rationale, THEN
  attack it. Emit findings in the schema below with no length cap; produce as many as the surface
  yields (no min, no max); also list what the design got RIGHT and must be PROTECTED. End with a
  COVERAGE STATEMENT: everything you examined, so nothing looks skipped.
  Write your full output to `audit/reviewers/{lens}.md`.
  Finding schema:
    FINDING-{lens}.{n} · severity: blocker|major|minor · target: <decision/doc/section/persona/field>
    steelman: <strongest case for current choice, engaging logged rationale>
    failure_scenario: <concrete sequence of events, not abstraction>
    evidence: <cite/quote docs; name what's absent>
    proposed_alternative: <specific, actionable>
    downstream_impact: <other decisions/contracts/WPs touched>
    confidence: high|med|low»

Reviewer lenses (dispatch all seven; each argues hard from its priors):
1. RF/SIGINT DOMAIN — undersampling model, sweep-collapse window, two-pass binning (& unresolved
   Pass-2 calc), event=burst-group/600s, noise filter (40× duty, 0.95 presence, low-duty exception),
   G1–G6 taxonomy & discriminators, band plan, power/proximity claims, every numeric constant.
2. DATA ENGINEERING — two-table ingest, quarantine, ledger, schema_version=v5, per-agent DuckDB,
   Parquet handoff (atomicity/partitioning/versioning), ephemeral-job model, feature extraction, retention.
3. DISTRIBUTED SYSTEMS/RELIABILITY — store-and-forward, idempotency, recovery-by-rebuild, late/
   out-of-order arrival, WAN partition, clock skew vs gap analysis, run lock, delivery semantics per hop.
4. SECURITY & PRIVACY — surveillance-data sensitivity, sops/age secrets, R2 credential blast radius,
   access control (thin — WP-O3 stub), audit trails, facility isolation, legal/regulatory/privacy of
   workplace RF monitoring. Go deepest — authors flag this as least-developed.
5. TEST/QA — fixture-as-contract, all 19 personas (does each truly pin its clause? can one pass while
   the contract is violated?), double-derivation, gate protocol (D-058), UAT journeys (D-056), deferred
   volume gate (Q-22). Audit the tests as hard as the system.
6. SRE/OPERATIONS — health checks (D-059), alerting, runbook, backup/DR + drills, fleet updates,
   provisioning, and whether ONE operator + AI sessions can run this across facilities. Bus factor,
   6-month-gap survivability.
7. SOFTWARE ARCHITECTURE/PROGRAM MGMT — 6-repo topology, contract-as-source-of-truth, runsheet realism
   (S00–S21, dependencies, parallelism), Opus/Sonnet assignment, decision-log discipline, buildability.

Wait for all seven to complete and write their files before Phase 2. Do not let them see each other yet.

# PHASE 1.5 — LEAD SEAM REVIEW (you perform this; sub-agents can't)
After the seven finish, before cross-examination, review what falls BETWEEN lenses — the seams only
the whole-system holder can see. Write to `audit/synthesis/lead-seams.md`:
- **Cross-domain contradictions:** does one reviewer's assumption break another's mechanism? (e.g.
  RF's sweep-timing vs data-eng's window sizing; security's isolation vs reliability's recovery;
  ops' cadence vs domain's deep-sleep detection needs.)
- **Uncovered seams:** requirements that span two lenses and may have been missed by both because
  each assumed the other owned it.
- **Emergent/systemic risks:** failure modes visible only when the whole chain is held at once
  (a single tunable or assumption whose change cascades across agent+server+analysis).
- **Meta-review of the panel:** did any reviewer go shallow despite the mandate, or stop short of
  its surface? If so, re-dispatch that sub-agent with specific gaps to close before proceeding.
These lead findings enter Phase 2 cross-examination alongside the reviewers'.

# PHASE 2 — ADVERSARIAL CROSS-EXAMINATION
Now provide each sub-agent (or a fresh cross-exam sub-agent per finding cluster) the other reviewers'
finding files. For every finding: dissenting reviewers file rebuttals (same schema, target = finding ID)
into `audit/rebuttals/`. Surface every cross-domain conflict explicitly (security vs operability, domain
vs data-eng, reliability vs simplicity). Kill findings that don't survive rebuttal; strengthen those that
do; mark `rationale_addressed` where DECISION-LOG already answers. Do not abbreviate the debate.

# PHASE 3 — ORCHESTRATOR SYNTHESIS (you, the lead) → `audit/synthesis/REPORT.md`
A. TOP RISKS TO PROJECT SUCCESS — ranked, uncapped, each with backing finding IDs, failure scenario,
   recommended change, impact (never down-rank a real risk for a large fix).
B. CHANGE-BEFORE-REPO-CREATION — structural (D-030-class) items cheaper pre-code, vs change-during-build.
C. OVER-CONFIDENCE AUDIT — every place the docs claim more certainty than evidence supports (authors
   cannot self-detect this — primary deliverable).
D. FULL FINDINGS TABLE — all surviving findings by severity, nothing omitted for brevity.
E. WHAT TO PROTECT — choices strong enough to guard against improvement-churn during build.
F. DISSENTS — unresolved disagreements, both sides fair, for author adjudication.
G. COVERAGE MATRIX — per reviewer, surface examined, so the authors can verify nothing was skipped.

# RULES
- No budget is a constraint (tokens/length/time/effort/patience); rationing any is the primary failure.
- Depth and completeness are the ONLY success criteria; concision is not valued.
- Never wrap up early, never "in the interest of brevity," never defer to "could be explored further."
- Every "this is wrong" needs steelman + concrete failure scenario + alternative + severity + downstream.
- Don't re-raise OPEN-QUESTIONS items as new — extend them. Preserve dissent; no false consensus.
- Cite decision IDs and doc sections throughout.
- Use sub-agents wherever parallel isolation improves independence (Phase 1 mandatory; Phase 2 optional).

# BEGIN
Confirm workspace + document access, list load-bearing decision IDs, then dispatch the seven reviewer
sub-agents. Report progress as each completes.
