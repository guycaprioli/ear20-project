# EAR System — Independent Steelman Audit (Fable-on-Cowork, sub-agent edition)

Pre-code expert audit of the `ear-docs-FINAL-v1` design set. Seven independent reviewer sub-agents (isolated), adversarial cross-examination, and lead whole-system passes. **Start with [`synthesis/REPORT.md`](synthesis/REPORT.md).**

## Read order
1. **[`synthesis/REPORT.md`](synthesis/REPORT.md)** — the orchestrator synthesis (A: top risks · B: change-before-repo-creation · C: over-confidence audit · D: full findings table · E: what to protect · F: dissents · G: coverage matrix). **This is the deliverable.**
2. **[`synthesis/lead-consistency.md`](synthesis/lead-consistency.md)** — Pass 0: 11 grep-cited cross-document contradictions (the lead's whole-system review).
3. **[`synthesis/lead-seams.md`](synthesis/lead-seams.md)** — Phase 1.5: cross-domain seams, emergent/systemic risks (incl. the Primary-Target Gauntlet), and the panel meta-review.
4. **[`synthesis/load-bearing-decisions.md`](synthesis/load-bearing-decisions.md)** — per-reviewer decision-ID / contract map.

## Structure
```
audit/
  synthesis/       REPORT.md + the three lead passes (above)
  reviewers/       the seven Phase-1 independent reviews (full findings, schema-per-finding):
                   rf-sigint · data-eng · reliability · security · test-qa · sre-ops · arch-pm
  rebuttals/       the eight Phase-2 adversarial cross-examinations (verdicts: killed/softened/
                   survives/hardened/rationale_addressed/author-dissent)
```

## Numbers
- **89** reviewer findings + **11** lead-consistency + **7** lead-seam findings.
- Phase-2 cross-exam of the 30 most contested → **37 verdicts: 0 killed, 10 survives, 14 softened, 13 hardened.**
- Every finding: severity · steelman · failure_scenario · proposed_alternative · downstream_impact · confidence (+ verdict where cross-examined).

## Method (per REVIEW-PROMPT-COWORK-FABLE)
Pass 0 lead consistency → Phase 1 seven isolated deep reviewers (each: enumerate surface → ≥5 passes → steelman-then-attack → findings → what-to-protect → coverage) → Phase 1.5 lead seam review + panel meta-review → Phase 2 adversarial cross-examination → Phase 3 synthesis. Steelman-first throughout; DECISION-LOG treated as authoritative for *why*; nothing built, so every finding costs one edit.

## Next step (D-060)
Feed `REPORT.md` §A–G into the review-disposition process: triage each finding ACCEPT / ACCEPT-DEFER / REJECT / NEEDS-INFO, adjudicate the four §F dissents, and apply the §B change-before-repo-creation items before WP-D0 creates repos.
