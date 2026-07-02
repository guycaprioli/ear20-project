# Adversarial cross-examination — Cluster: conservation / ingest-accounting (SEAM-1)

Examiner stance: try to REFUTE each finding. A finding survives only if it withstands a genuine kill attempt.
Findings under exam: **DE.1** (R5.1 two-term post-load assertion), **RF.5** (four stage orders), **QA.7** (P17 vs whole-run conservation, L4.5×FILTER double-count).

Primary evidence read: DATA-WORKFLOW-RULES-v3 R5.1 (l.169), R7 (l.196); EVENT-CONSTRUCTION-METHODOLOGY §19 (l.19–24) vs §100; EVENT-DATASET-HANDOFF §8 (l.8); RFANALYSIS-SCOPE §54 (l.54); GOLDEN-FIXTURE P9 (l.24), P17 (l.32), whole-run Conservation (l.39); observation-context.v5.schema.json (l.19, 25); LOCAL-HOST-DATA-FLOW L4 (l.150–176), L4.5 (l.185–216); DECISION-LOG D-012, D-013, D-021, D-026, D-028, D-044, D-049; lead-seams §SEAM-1.

---

## TARGET: FINDING-DE.1 — R5.1 post-load assertion `observations delta == Σ ledger row_counts` is two-term, breaks on any quarantine

### Rebuttal (strongest kill attempt)

Three lines of attack were tried:

**(a) "The assertion is aspirational prose, not a halt."** Refuted by the documents themselves. R5.1 lists the assertion under **ENFORCED BY** (DATA-WORKFLOW l.169), the same clause family as the ledger uniqueness *constraint* and the transactional insert — i.e. it is stated as a runtime enforcement mechanism, not narrative. D-044 is explicit that the pipeline is `docker compose run --rm … run_level1` where "**Non-zero exit = alert; next scheduled run retries**" and "halt-on-failure maps to exit codes." A post-load *assertion* that evaluates false is, by construction, a halt-on-failure. There is no text anywhere carving quarantine out of the halt path. So the "aspirational, non-halting" reading has no textual support and the design's own execution model (D-044) turns a false assertion into a non-zero exit → `notify()` (D-049 emits on "pipeline non-zero exit"). The halt reproduces.

**(b) "Quarantine is rare / edge-case, so severity is inflated."** Refuted by the design's own stated priors. R5.1's rationale for the two-table split is that "raw data is dirty in ways transform-on-read can't handle safely" and the entire reason `raw_landing` is persistent is that dirty rows are *expected and routine*. P9 is a first-class golden-fixture persona precisely because malformed rows are normal operating condition, not exception. A fleet of ~60 sensors emitting field-captured RF data will produce at least one castable-timestamp/missing-power/agent-mismatch row in essentially every window. The failure is not a corner — it is the modal run.

**(c) "The arithmetic is actually fine because ledger row_count is recorded post-enrichment, so it already excludes quarantined rows."** Refuted by the ledger definition. R5.1 (l.157): the ledger records `row_count` at **landing** ("each file lands exactly once, tracked in an `ingest_ledger` (filename, size/etag, row_count, ingested_at)") — landing is defined (l.158) as "as-is import, minimal typing … **no filtering, no casting**." Quarantine happens later, at *enrichment* (l.159). So ledger `row_count` = landed (pre-quarantine) count, and `observations` delta = landed − quarantined. The two terms are provably unequal by exactly `quarantined_count`. The arithmetic gap is real and structural, not a misreading.

**(d) DECISION-LOG rescue?** D-021 mandates "quarantine-with-reason" and bans `ignore_errors` — it *requires* the quarantine path to exist and be counted, but it says nothing to reconcile the R5.1 two-term assertion with that path. The DECISION-LOG does not address the arithmetic; if anything D-021 *sharpens* the contradiction (it mandates the very quarantine flow that breaks the two-term equation). Not `rationale_addressed`.

**(e) Does the fixture line 39 already supersede R5.1, making this a doc-clarity nit?** No — and this cuts the OTHER way (see cross-exam below). The fixture whole-run Conservation (l.39) is a *different scope* (server whole-run, three-term: `input = observations + quarantined + suppressed-summarized`) than R5.1's ENFORCED-BY (per-batch post-load, two-term). They do not supersede one another; they coexist in two different documents and the R5.1 one is still the enforcement the ingest stage will actually implement. A build engineer coding the R5.1 stage assertion writes the two-term form as literally specified and ships a pipeline that halts on every dirty run. The fixture line 39 being *correct* is what proves R5.1 line 169 is *wrong* — it does not repair it.

### Verdict: **hardened**

The finding survives every kill attempt, and the cross-exam surfaces an aggravator the reviewer under-stated: the halt is not merely a false page — under D-044 "next scheduled run retries," a **deterministic** dirty row (the P9 agent-id-mismatch row, or any un-castable value that recurs every window) makes the assertion fail identically on every retry. The pipeline never produces a handoff until a human intervenes. That is the same deterministic-poison infinite-fail mode DE.6 identifies, reached here through the accounting assertion rather than a worker crash. So DE.1 is not just "operator paged every run"; it is "fleet produces no handoff at all while any deterministic dirty row exists" — a data-availability outage, worse than the filed "noisy page."

### Residual claim (survives at blocker)

R5.1 ENFORCED-BY must be restated as a three-term identity: `observations_delta + quarantined_delta == Σ ledger.row_count` for the batch, with `quarantined_count` recorded per-run so the equality is checkable; and quarantine must be explicitly declared a **non-halting** outcome (exit 0, surface count in run report, alert only on a quarantine-ratio threshold — mirroring the D-028 over-filter safeguard). Absent both edits, the first dirty production run halts.

### Downstream
R5 ENFORCED-BY summary row; D-044 needs an explicit "quarantine is not a failure exit" carve-out; D-049 needs a quarantine-ratio emitter; P9 expected-output wording must state the three-term reconciliation; RUNBOOK quarantine-review. Composes with QA.7 and RF.5 into the single SEAM-1 identity (below).

---

## TARGET: FINDING-RF.5 — four docs specify four different stage orders; noise-filter activity count is order-dependent

### Rebuttal (strongest kill attempt)

**(a) "The ordering differences are presentational, not results-affecting."** This is the reviewer's own steelman and it is the strongest kill line, so it deserves the hardest push. The claim would be: all four docs agree on the *set* {power-bin, sweep-collapse, burst-group, noise-filter} and on the two operative invariants (filtered devices don't flow forward via INNER JOIN filter='pass'; events built only from FORWARD passing signal), so the diagrams are just informal.

This kill **fails on the numbers.** The noise filter's "total activity" is defined (methodology §100, l.102) as `SUM(observation_count)` over the **hourly rollup** (`hourly_stats`) per `(agent_id, earfcn, power_bin)`. Whether that sum is taken over **raw enriched observations** or over **sweep-collapsed detections** changes its value — sweep-collapse (methodology §19 step 2) explicitly "merges the undersampled sweep-hits of a single transmission into one detection," so a device with 12 raw sweep-hits becomes 4 detections. `SUM(observation_count)` differs by the collapse factor. The filter compares that sum against `timeframe × ACTIVITY_PER_HOUR` and against `MIN_OBSERVATIONS`. A P15 deep-sleep tracker (4 short bursts across 10 days) sits *near the min-obs floor by design* — that is the entire point of P15 (GOLDEN-FIXTURE l.: "Survives noise filter (NOT dropped by min-obs)"). If the build engineer computes activity over collapsed detections rather than raw observations, the P15 survival result can flip. So the order is **results-affecting on the single most important persona in the system** (the primary-target hardest case). Not presentational.

**(b) "But R7 is the authoritative pipeline-stage contract and it is unambiguous — so there is no real conflict."** This is the correct partial defense and it *narrows* the finding. R7 PRODUCER GUARANTEES (l.196) states a **"Fixed order: power clustering → noise filtering → sweep-collapse → burst grouping"** and R7 NEVER says "Stage order is never rearranged silently … a missing predecessor is a hard error." HANDOFF §8 (l.8) matches R7. RFANALYSIS-SCOPE §54 matches R7. And — decisively — the methodology's **own prose** (§100, l.100) matches R7: "After power-bin grouping, **before** event construction." So **four of five statements agree** (R7, HANDOFF, RFANALYSIS-SCOPE, methodology-prose): noise-filter is second, over raw observations, before collapse. The lone outlier is the methodology **numbered list** (§19 step 4), which puts noise-filter *last*, after burst-grouping — and it is contradicted by prose 80 lines below it in the same file.

So the finding as filed ("four docs specify four *different* orders") is **over-counted**: it is really one canonical order (attested 4×) plus one self-contradicting document (the methodology, list-vs-prose). That reduces the *doc-drift* surface but does **not** kill the operative risk, because R7's authority is not self-executing: a build engineer implementing the algorithm reads the methodology (the document literally titled EVENT-CONSTRUCTION-METHODOLOGY, the natural home for "how to build events"), hits the numbered recipe first, and codes noise-filter last — over collapsed detections. The very document that is *supposed* to be authoritative for the algorithm carries the wrong order in its most prescriptive form (a numbered list).

**(c) DECISION-LOG rescue?** D-026 records "sweep-collapse ≠ burst-grouping (two scales)" — it distinguishes two of the four stages but says **nothing** about where noise-filter sits relative to them, and nothing about what the activity count is computed over. D-028 defines the noise filter's *metric* (presence × activity, low-duty exception) but not its *position*. So the DECISION-LOG does not adjudicate the order. Not `rationale_addressed`.

**(d) "The failure is speculative — no engineer would collapse before filtering when the INNER JOIN pattern makes intent obvious."** Weak: the numbered list *is* the collapse-first instruction, stated imperatively, in the authoritative algorithm doc. Betting the P15 result on an engineer noticing that the prose 80 lines down overrides the numbered recipe is exactly the "silent-contract-via-a-number" failure mode R0 exists to prevent.

### Verdict: **softened**

The finding survives as a real, results-affecting defect, but its scope narrows: it is **not** "four contradictory orders" — it is **one canonical order (R7/HANDOFF/RFANALYSIS-SCOPE/methodology-prose, 4× attestation) versus one self-contradicting document (the methodology numbered list §19 vs its own prose §100)**, plus the still-missing explicit statement of *what the noise filter's activity count is computed over*. Severity stays **major** because the ambiguity lands on the P15 primary-target survival guarantee, but the fix is smaller than filed: correct the one outlier list and pin the activity-count basis — not "reconcile four documents."

### Residual claim (survives, narrowed)

(1) Correct EVENT-CONSTRUCTION-METHODOLOGY §19 numbered list to match its own §100 prose and R7: `power-bin → noise-filter → sweep-collapse → burst-group`. (2) State explicitly, in one canonical place (R7), that the noise filter's `total activity = SUM(observation_count)` is computed over **raw enriched observations (pre-collapse), weighted per R2** — and bind that basis into P15's derivation so the survival guarantee is order-independent. Doc edit now; a pipeline rewrite + every-fixture re-derivation post-build.

### Downstream
Methodology §19; R7 canonical order; P15 derivation; D-028 (add activity-count basis); the noise-filter's `filter_decisions` materialization. Composes into SEAM-1: "total activity" must have one meaning for the conservation identity's `suppressed_summarized` term to be stable.

---

## TARGET: FINDING-QA.7 — P17 (agent) vs whole-run (server) conservation never shown to compose; L4.5-forced EARFCN may double-count or gap

### Rebuttal (strongest kill attempt)

**(a) "The L4.5 double-count cannot happen: L4.5 REPLACES the natural FILTER classification, it does not ADD to it — so a forced EARFCN's volume appears exactly once."** This is the strongest kill and it is *substantially* supported by the text. LOCAL-HOST-DATA-FLOW L4.5 (l.185): the rule set "**forces** known noisy infrastructure signals to summary (FILTER) treatment … **regardless of how the edge filter would naturally classify them**." And (l.189-ish): "A matched signal **is collapsed to the same one-record-per-window summary** that an auto-classified FILTER cluster produces." The forcing is a classification *override*, not an additional emission — the natural classification is *superseded*, not duplicated. Moreover the context schema `record_type` enum is **disjoint** — `["summary","l45_heartbeat","config_stamp"]` — a given window-cluster carries exactly one record_type. So a forced EARFCN emits an `l45_heartbeat` and does **not** also emit a `summary` for the same window. Under this reading the double-count QA.7 fears does **not** reproduce: volume is counted once.

So on the mechanism, the *most alarming* version of QA.7 (P17's sum literally exceeds raw on a correct implementation) is **defeated** — the design does specify replacement, if you read L4 + L4.5 + the schema enum together.

**(b) BUT — the finding is not only "double-count happens." Its filed claim is "field ownership is not pinned precisely enough to close, and neither doc PROVES the two forms compose."** That survives (a) intact, because:

- P17 (GOLDEN-FIXTURE l.32) writes the agent identity as `raw == FORWARD + Σsummary_obs_count + ΣL4.5_heartbeat_count` — listing **summary** and **L4.5 heartbeat** as *separate additive terms*. Nothing in P17 or its "known raw input" derivation states the precedence rule that an L4.5-forced EARFCN's volume belongs to the heartbeat term and is *excluded* from the summary term. The correct behavior (replacement) is inferable only by cross-reading three other documents (L4, L4.5, the schema enum). The *persona itself*, which is the committed-truth artifact the code must conform to, does not encode the disjointness. Under the double-derivation rule (D-057), whoever hand-derives P17's expected sum must pick a convention; if they pick "forced EARFCN counts in both terms" (a defensible misreading of the two-term-listing), the committed expected output is wrong and the conforming code is wrong forever (this is exactly QA.14's "wrong-and-invisible accounting identity" risk). The ambiguity is real even though the *intended* answer is knowable.

- The **compose** claim is untouched by (a). P17 is agent-side `raw == FORWARD + Σsummary + Σheartbeat`. Whole-run (l.39) is server-side `input = observations + quarantined + suppressed-summarized`. These are genuinely different equations over different term-sets, and **no document proves they reconcile.** The specific unproven bridge QA.7 names is sharp and correct: server `observations` counts FORWARD rows *post-ingest*; a FORWARD row that **quarantines** at enrichment (bad timestamp on a FORWARD row — entirely possible; FORWARD rows are full-fidelity raw and can be malformed) moves from the `observations` term into the `quarantined` term. The two equations compose only if `quarantined ∩ summarized = ∅` — i.e. only if no summarized/heartbeat volume can also be quarantined and no FORWARD-that-agent-counted can vanish server-side without a matching term. **That disjointness is asserted nowhere.** This is precisely the DE.1 quarantine-term gap re-surfacing at the compose seam: the server-side identity needs the quarantine term (DE.1) *and* needs it proven disjoint from summarization (QA.7).

**(c) DECISION-LOG rescue?** D-013 (L4.5 override: "per-EARFCN, identity-based, forced-summary, never silent drop, heartbeat always") establishes the *never-silence* and *forced-summary* intent but does **not** state the FILTER-vs-L4.5 precedence in accounting terms, nor prove agent/server composition. D-012 (feed split) guarantees summaries and heartbeats travel in `ctx_` and never enter the event path — orthogonal to *volume accounting*. So the DECISION-LOG supplies the intent that kills the double-count (supports rebuttal (a)) but does **not** address the composition/field-ownership gap. Partial, not full `rationale_addressed`.

**(d) The observation_count=0 heartbeat sub-point.** The schema allows `l45_heartbeat.observation_count minimum: 0` (l.25) vs `summary minimum: 1` (l.19). QA.7's point (4) is that the "never silence even at zero" invariant (a rule matched with zero obs in the window) is schema-*allowed* but not *tested* by any persona. This is valid and small — it survives as a coverage gap, not a contradiction.

### Verdict: **softened**

The most severe sub-claim (an L4.5-forced EARFCN double-counts on a *correct* implementation, so conservation fails on correct code) is **killed** by the replacement semantics (L4.5 forces/overrides, schema record_type is disjoint, DECISION-LOG D-013 intent). But the finding's core — **the two conservation equations are never proven to compose, and P17 does not itself pin the field-ownership/precedence needed to make its sum unambiguous** — **survives**, now sharpened to two concrete unproven bridges: (i) P17 must state the L4.5-forced-EARFCN counts once (in the heartbeat term, not also the summary term); (ii) the agent↔server composition requires `quarantined ∩ summarized = ∅` and the presence of the quarantine term, which is the DE.1 gap. Severity drops from "conservation fails on correct code" to **major: conservation identity is under-specified and un-composed; a wrong committed derivation (D-057) or a future L4.5-precedence change breaks it invisibly.**

### Residual claim (survives, narrowed)

State ONE end-to-end conservation identity and prove the agent and server forms compose: (1) pin that an L4.5-forced EARFCN's volume is counted exactly once (l45_heartbeat term), excluded from the FILTER-summary term, and encode that precedence in P17's derivation + a new persona P17b (L4.5 × natural-FILTER overlap → volume appears once); (2) make the whole-run assertion `server.observations + server.quarantined + server.suppressed_summarized == agent.raw` with an explicit note `quarantined ∩ summarized = ∅`; (3) add the DE.1 quarantine term so the server side balances; (4) pin the obs_count=0 heartbeat case in a persona.

### Downstream
R1 volume-honesty ENFORCED-BY; L4.5 vs edge-FILTER precedence; observation-context obs_count semantics; P4/P17/new-P17b; R5.1 row-delta (DE.1); Suppression dashboard J4; D-057 double-derivation scope (QA.14). Composes into SEAM-1.

---

## COMPOSITION: do DE.1 + RF.5 + QA.7 collapse into ONE conservation identity fix?

**Yes.** The three findings are not independent — they are three different terms/inputs of a single end-to-end conservation identity that no current document writes correctly, and none of the three lenses could specify the whole identity alone. Adjudication:

- **DE.1 owns the QUARANTINE term.** The identity's server side is unbalanced without it; the two-term R5.1 form is provably wrong and halting.
- **RF.5 owns the meaning of the SUPPRESSED-SUMMARIZED input.** The `suppressed_summarized` / activity volume is only stable if "total activity" (`SUM(observation_count)`) has one definition — which requires a pinned stage order (noise-filter over raw observations, pre-collapse). If order is ambiguous, the volume that feeds the identity is ambiguous.
- **QA.7 owns COMPOSITION + the L4.5/FILTER precedence.** It binds the agent-side (P17) and server-side (l.39) forms together and requires (a) volume counted once across L4.5-vs-FILTER, and (b) `quarantined ∩ summarized = ∅` — which structurally *depends on DE.1's quarantine term existing*.

**The single identity that satisfies all three at once:**

```
agent.raw  ==  server.observations
             + server.quarantined            ← DE.1 (missing term; must be non-halting)
             + server.suppressed_summarized   ← RF.5 (volume well-defined only if stage
                                                 order pins "total activity" = SUM(obs_count)
                                                 over RAW observations, pre-collapse)

with, for composition (QA.7):
  server.observations        == agent.FORWARD  minus any FORWARD row that quarantined
  server.suppressed_summarized == Σ agent.summary_obs_count + Σ agent.l45_heartbeat_count
  an L4.5-forced EARFCN is counted ONCE (heartbeat term), never also as a FILTER summary
  quarantined ∩ summarized == ∅   (a summarized/heartbeat row is never also quarantined)
```

This is **one contract edit that must satisfy three lenses simultaneously**, and it is the correct scope: the per-batch R5.1 ENFORCED-BY assertion (DE.1) becomes a *projection* of this whole-run identity onto one batch (with the quarantine term), the noise-filter stage-order/activity-basis (RF.5) fixes what `suppressed_summarized` counts, and the P17 + whole-run reconciliation (QA.7) is the proof that the agent and server projections of this one identity agree. Fix any two and the third still breaks it. The lead-seams synthesis (§SEAM-1, l.45–49) independently reached the same conclusion; this cross-exam confirms it and adds the precise term-ownership map above.

**Author action:** write the single identity once (natural home: extend the GOLDEN-FIXTURE whole-run Conservation at l.39, since it already has the correct three-term skeleton), then make R5.1 ENFORCED-BY, the methodology noise-filter step, and P17 all *reference* it rather than restate partial forms. Do not fix DE.1, RF.5, QA.7 as three separate doc patches — the whole point of SEAM-1 is that partial fixes leave an un-composed identity.

---

## Conflicts for the author (cross-domain)

None of the three findings conflicts with another *within* this cluster — they compose. Two adjacent conflicts touch this cluster and are flagged for the author (not adjudicated here, belong to other clusters):

1. **DE.1's "quarantine must be non-halting (exit 0)" vs D-044 fail-loud / SRE reliance on non-zero-exit paging (DE.6/REL.6).** DE.1's fix requires quarantine to NOT trip the halt-on-failure exit. This is the same tension the lead flagged (lead-seams §Cross-lens 1): resolvable as "quarantine exits 0 but surfaces count + alerts on ratio threshold," which preserves fail-loud for genuine failures while stopping quarantine from halting. Real design decision for the author; not a defect in DE.1.

2. **QA.7's L4.5-counted-once precedence vs any future change to L4.5-vs-FILTER precedence.** Not a present conflict; a durability note — once the identity pins "counted once," the L4.5 precedence rule becomes load-bearing for conservation and must be paired-change protected (R0.4).
