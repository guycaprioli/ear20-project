# Lead Seam Review — Phase 1.5 (whole-system vantage; the reviews only the panel-holder can do)

**Reviewer:** Lead Orchestrator (Fable). **Written after** all seven reviewer sub-agents completed (89 findings: 16 blocker, 40 major, 33 minor across the returned index; the reviewer files themselves tally 13 blocker / 33 major / 27 minor as written — the difference is a few reviewers labelling a compound finding's sub-parts). **Purpose:** the things no single lens can see — cross-domain contradictions (one reviewer's assumption breaks another's mechanism), uncovered seams (a requirement spanning two lenses that both assumed the other owned), emergent/systemic risks (visible only holding the whole chain), and a meta-review of the panel's depth. These lead findings enter Phase 2 alongside the reviewers'.

---

## A. Meta-review of the panel (did anyone go shallow? re-dispatch?)

**Verdict: no re-dispatch.** Every reviewer exhausted its surface. Word counts 7.5k–10.7k each (60k total), 12–16 findings each, every finding steelman-first with a concrete failure scenario and file:line citations, each with an explicit WHAT-TO-PROTECT and a COVERAGE STATEMENT enumerating areas found sound and *why*. Each of the REVIEW-BRIEF's eight pre-flagged weak spots was not merely re-raised but *extended*:

| Brief's suspected weak spot | Extended by |
|---|---|
| Pass-2 fallback honesty | RF.7 (stub can false-*merge* — worse error), QA.3 (whole pipeline runs on stub-binned "authoritative" power_bin) |
| Correlation provisional-marking | REL.4 (correlation ships now and is clock-drift-sensitive before the time-sync contract exists) |
| Cold-start G1-vs-G5 over-claim | RF.6 (3 of 4 G5 discriminators unavailable day one; the four-factor score fires high on benign telemetry, poisoning the seed corpus) |
| Security least-developed | SEC.1–11 (Q-23 legal gate, key blast-radius, credential scope, dashboard authZ, at-rest, audit, physical compromise) |
| Single-operator bus factor | OPS.1/4/7/15 (unstaffed on-call, invisible total-box-death, no cold-return runbook, build bus-factor = 1) |
| Volume deferred (Q-22) | DE.4 (memory_limit ≠ RSS), REL.6 (stale-lock murders a long run), QA.13 (concurrency-correctness ≠ volume) |
| Fixture-defines-truth | QA.1–16 (P7 proves an absence; conservation doesn't close; determinism unachievable; origin bug R6 has no persona) |
| Time-sync under-specified | REL.4 (drift corrupts 600 s burst grouping, not just cross-sensor CV; no numeric tolerance, no hard gate) |

**Isolation integrity.** The security reviewer self-reported seeing two incidental grep-preview lines from `audit/synthesis/load-bearing-decisions.md` and explicitly did not open it or use it. Immaterial — its 11 findings are independently derived and grep-cited. All other reviewers confirmed `audit/` unread. Independence held.

**Convergence map (the strongest validity signal — isolated reviewers hitting the same seam).** Where ≥2 lenses independently found the same defect, confidence is very high and these anchor the synthesis:

| Defect | Independent finders | Note |
|---|---|---|
| **Uploader cardinality** (1 shared vs 4 vs "its own") | LEAD.1, REL.1 | REL.1 tied it to the *frozen* heartbeat schema — the shared uploader cannot author 4 device-lettered `_health/` files with per-device queue depth |
| **Record field drift** (`observed_at`/`time`, `direction`, `duty_cycle`) | LEAD.2, DE.11, QA.8 | triple convergence; D-052 says schema wins, so the prose contract R1 is unenforced-as-written |
| **presence_ratio 0.95 vs 0.8** | LEAD.4, RF.4, QA.2 | 0.8 is earnest-v3's *rejected* value; gates whether the primary target enters the exception path |
| **Undefined rebuild window × late arrival × 90-day prune** | REL.2, DE.5 | the 13-month R2 retention is defeated by a 90-day landing prune + skip-if-exists |
| **Disk-ceiling eviction = silent data loss** | REL.7, OPS.1, SEC.5 | reliability, ops, AND security each reached "don't silently oldest-drop" independently |
| **Halt-on-failure undoes per-agent isolation** | DE.6, REL.6 | one corrupt disposable file → whole-fleet run failure; can loop forever |
| **Determinism byte-identity unachievable** | QA.4, DE (note) | contradicts the per-row `run_id` requirement; undermines Guy's gate re-run |
| **Deep-sleep primary-target starvation** | RF.3/RF.4, QA.5/QA.16, DE.12 | see §D.1 — the systemic risk |
| **qa→active promotion folds onboarding data into production** | REL.8, DE.10, QA.11 | window-vs-status rebuild; skip-if-exists |
| **Server-topology decision (D-004) is under-weighed** | SEC.8, OPS.8, REL.12, DE.3 | four lenses each add a decisive criterion the docs don't weigh — see §B.5 |

Convergence also means Phase 2 should *stress the convergent findings hardest* (if they survive multi-lens agreement they are near-certain) and reserve most rebuttal effort for the genuine cross-lens *conflicts* (§E).

---

## B. Cross-domain contradictions (one reviewer's assumption breaks another's mechanism)

**SEAM-1 · "Conservation" is claimed as a whole-run invariant but three lenses each found a different reason it cannot close — and the three interlock.**
- RF.5: the noise filter's "total activity" count differs depending on whether it runs over *raw observations* or *sweep-collapsed detections* — and four docs specify four different stage orders.
- DE.1: the R5.1 post-load assertion (`observations delta == Σ ledger row_counts`) is arithmetically wrong the moment any row is quarantined (the quarantine term is missing) — so *every* run with one dirty row halts.
- QA.7: the agent-side (P17) and server-side (whole-run) conservation equations are different and never shown to compose, and an L4.5-forced EARFCN that would *also* be naturally FILTER-classified may be double-counted or gapped.
- **The seam:** no single reviewer saw that "conservation" is defeated *three times over* — stage-order-dependence (RF), the quarantine term (DE), and L4.5/FILTER precedence (QA). The conservation identity cannot be written until all three are fixed *together*: it must be `agent.raw == server.observations + server.quarantined + server.suppressed_summarized`, with (a) a pinned stage order so "total activity" has one meaning, (b) the quarantine term present, (c) L4.5 vs FILTER precedence decided so volume is counted once. This is one contract edit that must satisfy three lenses at once, and none of them could specify it alone.

**SEAM-2 · Oldest-drop eviction destroys the exact signal the detection score most depends on — an RF consequence of a reliability/ops policy that neither RF nor the policy-owners connected.**
- REL.7/OPS.1/SEC.5 converged on "don't silently oldest-drop `pending/`" for reliability (blast radius), ops (operator absence), and security (evidence destruction) reasons.
- But RF.6 and the taxonomy establish that **`first-seen` recency is a load-bearing factor in the four-factor score (NEW + stationary + low-duty + periodic)** and the G1 signature ("sudden first-seen date — it arrived").
- **The seam none of them closed:** oldest-drop eviction during a WAN outage *specifically deletes a device's earliest observations* — i.e. it destroys the *first-seen* evidence that the arrival happened, which is precisely the highest-value signal for catching a newly-planted tracker. So a WAN outage during a tracker's arrival week both loses the data (reliability) *and* silently corrupts the one scoring factor (RF) that would have flagged it. The eviction policy is therefore not just a data-retention choice; it is a *detection-quality* choice. This strengthens the tiered-eviction recommendation (drop `ctx_` first, `obs_` last, DLQ never) with an RF rationale the reliability reviewer couldn't supply and elevates "resolve the eviction policy" from ops-hygiene to detection-integrity.

**SEAM-3 · Correlation ships in this build and is time-sync-sensitive, but the time-sync *contract* was deferred to WP-O3 when correlation was still "future Level 2."**
- REL.4: clock drift corrupts 600 s burst grouping (drift, not offset) and cross-sensor correlation; `ntp_offset_ms` has no numeric tolerance and no hard gate; WP-O3 time-sync is "a stub not a contract."
- D-055 un-defers Level 2 (including provisional correlation) *into this build*; D-029 previously placed correlation in a separate future effort.
- **The seam:** the "time-sync can wait for WP-O3 / M5" assumption was safe *when correlation was a future effort*. D-055 silently invalidated it — correlation now ships at M4.5 (S19c), but its time-sync precondition is still scheduled for after it. RF owns 600 s semantics, reliability owns drift, ops/security own NTP provisioning — and the un-deferral that broke the ordering (D-055, an Arch/PM decision) is owned by yet another lens. Only the whole-system view sees that **a decision in one lens (un-defer L2) moved a dependency across another lens's deferral boundary (time-sync in WP-O3)**. Fix: promote the numeric NTP tolerance + hard degrade-gate ahead of S19c, not to WP-O3.

**SEAM-4 · Disposability is simultaneously the best operational decision and the enabler of an infinite-loop failure — the same mechanism, two lenses, opposite valence.**
- OPS protects per-agent disposability (D-023) as "the single best operational decision" (recovery = delete + rebuild, never a 3am surgical repair).
- REL.6 shows the same disposability enables a *thundering-herd self-stealing-lock loop*: a long run outlives the (wall-clock) stale-lock threshold, the next tick steals the lock, two processes open the same DuckDB file, one corrupts, the "cheap" delete-and-rebuild kicks in, the rebuild is *also* long, and it loops forever — precisely *because* corrupt files are cheerfully rebuilt.
- DE.6: one agent's failure halts all 60 under `compose run --rm` halt-on-failure, contradicting the isolation D-023 sells.
- **The seam:** a strength (disposability) and two failure modes (infinite rebuild loop, whole-fleet halt) are *the same property* seen from ops vs reliability vs data-eng. The synthesis must hold both: protect disposability (ops is right) *and* fix the lock semantics (liveness not wall-clock, REL.6) *and* make the pool per-agent fault-isolating (DE.6) — otherwise "protect disposability" and "fix the loop" read as contradictory recommendations when they are complementary.

**SEAM-5 · The server-topology decision (D-004/Q-04) is framed as reliability/visibility, but FOUR lenses each add a decisive, unweighed criterion.**
- SEC.8: jurisdiction/isolation — if facilities span jurisdictions with different Q-23 legal answers, central commingling is a legal liability; per-facility appliance isolates by construction.
- OPS.8: SPOF — if prod shares the dev box (Q-11 open, D-047 parity), one Strix Halo failure takes dev *and* prod dark; prefer a separate prod host.
- REL.12: partial-fetch — a central server fetching all facilities hits R2 egress/rate limits on a post-outage backlog and produces a *partial-but-green* dataset.
- DE.3: filesystem — the atomic-rename handoff assumes temp and final on one filesystem; a central deploy with mounted volumes can hit `EXDEV` (non-atomic).
- **The seam:** the docs weigh D-004 as "central = cross-facility visibility + SPOF vs per-facility = isolation + N deployments." Four lenses each supply a criterion that *changes the answer* and none of which is in the docs' framing. This makes D-004 a far more loaded pre-repo structural call than it appears — it should be decided with the legal map (Q-23), the SPOF posture (Q-11), the egress budget (Q-14/Q-22), and the filesystem-atomicity constraint all in hand.

**SEAM-6 · The strongest structural decision has the weakest guardian test.**
- RF, DE, QA, REL, OPS all independently list the feed split (D-012) in WHAT-TO-PROTECT — it converts the false-CRITICAL bug class into an *unrepresentable* state. It is the most-praised decision in the corpus.
- QA.1: but its guardian persona P7 asserts an *absence* ("zero events"), which is unfalsifiable as a mechanism check — zero-because-the-split-works is indistinguishable from zero-because-the-signal-feed-was-empty.
- **The seam:** the decision everyone protects is guarded by the test QA proved is hollow. Not a contradiction — the design is good and the test is weak — but the synthesis must flag that *the most load-bearing structural invariant has no real regression guard*, so a future refactor could silently break the thing all seven reviewers told the authors to protect. QA.1's two-armed differential P7 (negative control: same rows in the signal feed → assert an event *would* form) is the fix.

---

## C. Uncovered seams (spanning two lenses; each may have assumed the other owned it)

**SEAM-7 · L2 result-persistence recovery — DE and OPS each saw one half.**
- DE.7: the DuckDB→Postgres *write* is an unguarded dual-write (compute 5,000 candidates, write 3,000, crash → dashboard renders a partial analysis as complete; no run-supersession semantics).
- OPS.9: L2 has *no recovery runbook* and no stated retention for its result tables.
- Neither owns the whole: the fix is a transactional per-run replace with a `results_complete` flag *and* a RUNBOOK clause stating L2 recovery = delete-current-run + re-run (symmetric with L1 disposability). DE specifies the transaction; OPS specifies the runbook; the seam is that "L2 persistence integrity" needs both and was assigned to neither.

**SEAM-8 · The heartbeat is over-loaded across three lenses and no one holds its full contract.**
- REL.1/LEAD.1: uploader cardinality makes the device-lettered heartbeat un-authorable by a shared uploader.
- OPS.10: the "capture advancing" health check keys on `obs_` arrival, which is *correctly near-zero* for the quiet primary target (D-028/P15) — so it false-reds healthy quiet sensors.
- OPS.2: a facility-WAN outage turns 48 heartbeats stale at once; the alert cardinality (1 facility-level push vs 48) is undefined.
- **The seam:** the heartbeat is simultaneously an identity artifact (REL.1), a liveness-vs-signal discriminator (OPS.10), and a WAN-rollup input (OPS.2). Its schema is frozen (D-052) but its *semantics* are contested from three directions. The fix must be holistic: decide cardinality (host vs device scope), base liveness on `ctx_`/heartbeat freshness not `obs_` volume, and compute a facility-level roll-up before per-sensor alerts — one coherent heartbeat-semantics contract, not three patches.

**SEAM-9 · "Reproducibility" is the claimed foundation of three things and is weaker than claimed for two different reasons.**
- QA.4: byte-identical determinism is unachievable (DuckDB ordering, Parquet metadata/compression, per-row `run_id`) — so the *determinism test* is mis-specified.
- QA.12: Guy's independent gate re-run (D-058) presumes determinism — so the *verification protocol* inherits the flaw.
- DE.9: the A/B framework's "re-process the same landing data without re-fetching" holds only 90 days (landing prune) — so *A/B reproducibility* is time-bounded.
- **The seam:** determinism/reproducibility underpins the determinism assertion, the gate protocol, *and* the A/B framework — three pillars — and QA (test lens) + DE (data lens) each found it weaker than claimed from a different angle. The fix (logical determinism: canonical total sort, exclude `run_id`, pinned codec+version) must be applied consistently to all three, or "reproducible" keeps meaning three different things.

---

## D. Emergent / systemic risks (visible only holding the whole chain at once)

**D.1 · THE PRIMARY-TARGET GAUNTLET — the deep-sleep 20595-class tracker, the entire reason the system exists, can be lost at *seven* independent points, each owned by a different lens, and no single reviewer sees the whole gauntlet.** This is the headline emergent finding of the audit.

A power-saving deep-sleep tracker (G1, multi-hour-to-multi-day beacon, "tolerates very few events," the 20595 class the system was built to catch) must survive this entire chain to be detected — and each link was flagged by a different lens as broken or unpinned:

1. **Capture/eviction (REL.7 + SEAM-2):** if a WAN outage triggers oldest-drop eviction during its arrival week, its first-seen data — the highest-value scoring signal — is silently destroyed.
2. **Sweep-collapse window (RF.2):** a wrong/global 2 s window mis-reconstructs its transmissions (over- or under-collapsing), corrupting `detection_count`.
3. **Clock drift (REL.4):** its multi-day-spaced beacons are exactly the regime where slow clock drift accumulates and lands them on the wrong side of the 600 s burst boundary — splitting or merging its events — before any drift gate exists.
4. **Burst gap 600 s (RF.3):** one global gap across a class spanning 60 s–multi-day cadences can shatter (motion-activated) or over-merge its bursts.
5. **MIN_OBSERVATIONS floor — Level 1 (QA.16):** a deep-sleeper's tiny total (≈12–16 obs) sits near the noise floor; the floor's value is *unstated*, and QA.16 shows P15's own claimed protection mechanism (the low-duty *presence* exception) **does not even apply** to a multi-day-silent device (it isn't omnipresent) — it survives only via `total ≥ MIN_OBSERVATIONS` + ELSE, so an unstated floor set slightly high drops it as noise.
6. **Rebuild window (REL.2):** if its sparse/late data falls outside an undefined rebuild window, it lands but is never grouped into events.
7. **min-events floor — Level 2 (QA.5):** a 4-event stream has 3 gaps, right at the `min-events=3` floor; the classifier may refuse to compute CV and starve it in classification — and `P15→G1` is not even in the S19b gate.

**Why this is systemic and not just seven findings:** each lens verified its own link and (reasonably) assumed the device survives the others. P15 exists precisely to guard this target end-to-end — but QA.5/QA.16 show P15 is a *Level-1* persona whose taxonomy half has no gate, whose stated protection mechanism is subtly wrong, and which tests *one* device in isolation, never the gauntlet. **No artifact in the design holds the full survival chain of the primary target.** Recommendation (synthesis A-tier): create an explicit, cross-lens **"primary-target survival budget"** — a single owned document/persona-set tracing a deep-sleep device through all seven links with a pinned value at each (eviction tier, collapse window, drift tolerance, burst gap, MIN_OBSERVATIONS, rebuild window, min-events exception) and a single end-to-end persona (P15 extended, gated at both S16 *and* S19b) that fails if *any* link drops it. This is the one place where the sum of the findings is larger than the parts.

**D.2 · A single tunable (`presence_ratio`) changing value silently re-routes the primary target — and it disagrees across docs.** LEAD.4/RF.4/QA.2: 0.95 vs 0.8 is not cosmetic — at 0.8 a 90-minute-cadence tracker (≈16/24 hours present) fails the presence test and never reaches the low-duty exception. Combined with D.1, this is a *second* silent-loss path through the same knob, and the "authoritative" tuning doc holds the *rejected* value. Emergent because the knob's blast radius (which filter branch the target takes) is only visible when you trace the target through the classifier with both documented values.

**D.3 · The cold-start corpus-poisoning loop.** RF.6 + QA.6 + PARAMETER-TUNING: at first-facility (the moment the operator most needs to trust the tool), 3 of 4 G5 discriminators are unavailable, so benign fixed IoT and planted trackers both score high; the J3 dashboard journey walks the operator into *confirming* them; each confirmation *seeds the corpus* that is supposed to fix the problem — with mislabels. Meanwhile the fixture encodes provisional taxonomy labels (P1→G1) as law (QA.6) before Q-T1–T4 field truth exists. Emergent: the corpus is both the fix for cold-start *and* is poisoned by cold-start, a feedback loop no single lens fully traced (RF saw the discriminators, QA saw the fixture-as-law, PARAMETER-TUNING saw the corpus dependency).

**D.4 · Build bus-factor = run bus-factor = 1, and D-058 tightens it.** OPS.1/7/15 + SEC.2: the design works hard to keep the *running* system alive without the operator (self-draining queues, recovery-by-rebuild), but the *build* has the opposite property — D-058's "no session completes without Guy's independent re-run" plus five un-delegable human-oracle points plus ASK-gates make the operator the strict rate-limiter, and a 6-month gap pauses *building* entirely. Add SEC.2's operator-age-key single point of failure (lose the laptop, lose the ability to re-render any config) and the human is a harder SPOF than the code. Emergent because "single operator" is treated as a *build-scheduling* constraint in the plan and an *availability* constraint in ops, and no doc holds both as the *same* bus-factor.

---

## E. Conflicts to resolve in Phase 2 cross-examination (genuine cross-lens tension) and where DECISION-LOG already answers

These are where reviewers pull in different directions or where a decision's rationale may rebut a finding. Phase 2 targets these (plus stress-testing the convergent blockers):

1. **Halt-on-failure: fail-loud (D-044, ops relies on it) vs per-agent fault-isolation (DE.6/REL.6).** Not truly opposed — resolvable as per-agent quarantine + a non-zero exit that still pages but lets the 59 healthy agents export. Phase 2: confirm the D-044 rationale ("clean memory, pinned image, fewer moving parts") does not *require* whole-run halt, only run-to-completion.
2. **State-change-only alerting (OPS protects) vs repeat-until-acknowledged (OPS.1 wants for the data-loss class).** Resolvable: escalation *only* for the data-loss-imminent class; state-change for all else. Confirm no contradiction.
3. **Un-defer Q-22 (REL.6, DE.4, QA.13 collectively push) vs Guy's explicit deferral.** This is a genuine **dissent for the operator** (Section F): the panel argues you cannot safely set run cadence (Q-19) or the stale-lock threshold without measuring max run time, so a *narrow* single-agent load probe must precede first facility even if the full fleet-scale gate stays deferred. Guy deferred it deliberately — adjudication required.
4. **Pass-2 stub behavior: RF.7 (make the stub a no-op, accept the tolerable false-split) vs RFANALYSIS-SCOPE's suggested population-fold placeholder (which can false-*merge*, the worse error).** QA.3 independently agrees the stub must be honestly marked. These *reinforce* rather than conflict — Phase 2 should confirm and merge them.
5. **Security scoping (SEC.3 per-facility R2 creds + object-lock) vs single-operator simplicity.** Mild tension; the security case (one stolen warehouse PC = whole-fleet confidentiality+integrity compromise) is strong enough that Phase 2 should test whether simplicity is a real rebuttal (it is not — scoped tokens are near-zero added ceremony).
6. **`rationale_addressed` candidates — where DECISION-LOG may already answer a finding:** DE.8 (open/close overhead) is softened by D-023's "open→work→close" being *per-task* where task=chain; REL.9 (LIST consistency) is largely *subsumed* by fixing REL.2; DE.12 (LIST eventual consistency) self-corrects by re-run *if* REL.2's window is fixed. Phase 2 marks these as partially-addressed rather than killing them.

**No finding is proposed for outright killing pre-rebuttal** — the panel's steelman-first discipline already pre-empted the weak ones (each reviewer's COVERAGE STATEMENT lists what it found *sound*). Phase 2's job is to (a) stress the convergent blockers (they should survive and harden), (b) adjudicate the six tensions above, (c) mark rationale_addressed items, and (d) surface the cross-lens conflicts explicitly for the author. Findings not individually rebutted stand as filed (all are grep-cited and steelman-anchored).

---

## F. What Phase 1.5 adds to the synthesis inputs

Beyond the 89 reviewer findings and the 11 lead-consistency findings, Phase 3 must carry these **lead-seam findings** as first-class items:
- **SEAM-1** conservation defeated three ways (RF+DE+QA must be fixed together).
- **SEAM-2** oldest-drop eviction destroys first-seen detection signal (RF consequence of an ops/security/reliability policy).
- **SEAM-3** correlation ships before its time-sync precondition because D-055 moved a dependency across WP-O3's deferral line.
- **SEAM-5** the D-004 topology decision carries four unweighed cross-lens criteria.
- **D.1 THE PRIMARY-TARGET GAUNTLET** — the systemic risk; candidate #1 top risk to project success.
- **D.3** the cold-start corpus-poisoning loop.
- **D.4** build-bus-factor = run-bus-factor = 1, tightened by D-058 and SEC.2's key SPOF.

These are exactly the "failure modes visible only when the whole chain is held at once" the orchestration mandate assigns to the lead — none is reducible to a single lens, and D.1 in particular is larger than the sum of its seven constituent findings.
