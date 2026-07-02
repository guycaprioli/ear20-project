# Cross-Examination — Cluster: THE PRIMARY-TARGET GAUNTLET (lead-seams §D.1)

**Cross-examiner stance:** adversarial. Each of the seven survival-chain links, plus the systemic no-single-artifact-holds-the-chain claim, is subjected to the strongest available rebuttal drawn from DECISION-LOG rationale, cross-lens priors, and a direct re-read of the authoritative classifier (`contracts/EVENT-CONSTRUCTION-METHODOLOGY-v1.md`). The load-bearing question — is QA.16's claim that the low-duty exception does NOT fire for a multi-day-silent device actually correct — was verified against the classifier source, not accepted on the reviewer's word.

**Key source verification (done first, because most of the cluster hinges on it):**
`EVENT-CONSTRUCTION-METHODOLOGY-v1.md` lines 108–113, the three-way noise classifier:
```
IF present in ≥ infra_hours (≈ every hour) AND total < timeframe×ACTIVITY_PER_HOUR → 'pass'  ← LOW-DUTY EXCEPTION
IF present in ≥ infra_hours (≈ every hour)                                          → 'infrastructure'
IF total activity < MIN_OBSERVATIONS                                                → 'noise'
ELSE                                                                                → 'pass'
```
The low-duty exception's FIRST predicate is `present in ≥ infra_hours (≈ every hour)`, `infra_hours = ceil(effective_hours × 0.95)`. A deep-sleeper beaconing 4× across 10 days is present in ~4 of ~240 hours — three orders of magnitude below infra_hours. **The exception provably cannot fire for it.** Its only survival path is: fail line 111 (not omnipresent), pass line 112 (`total ≥ MIN_OBSERVATIONS`), fall to line 113 ELSE → 'pass'. QA.16's central mechanical claim is CONFIRMED at the source. This single verification hardens QA.16 and REL.7's RF-consequence and simultaneously exposes that D-028's own "low-duty exception is NEVER-remove" framing and D-057's "guards the min-obs/min-events starvation risk" framing both mis-describe the mechanism that actually protects the primary target.

---

## LINK 1 — REL.7 / SEAM-2 · Oldest-drop eviction destroys first-seen signal

**target:** REL.7 (+ SEAM-2 RF-consequence)

**rebuttal (strongest available).** Three DECISION-LOG / doc facts cut at this: (1) the eviction policy is an *explicitly OPEN item* (`LOCAL-HOST-DATA-FLOW` L5 OPEN, "cap size + oldest-drop policy, OR alert-and-hold — Recommend resolving before fleet rollout") — so the finding is attacking a decision the design already flags as undecided and defers past the current pre-code phase; there is no committed oldest-drop policy to refute, only a named fork. (2) The blast radius is bounded and *alerting-first*: disk ceiling checked each upload run, `disk_pct` + `status: ceiling` on every heartbeat (D-045), D-049 disk-ceiling breach → ntfy push. So the operator is not blind. (3) This is a multi-day-WAN-outage-during-arrival-week compound condition — the reviewer stacks (outage) × (disk actually fills, i.e. Q-10 headroom is insufficient, which is unknown) × (oldest-drop chosen) × (tracker arrives in that exact window). Each conjunct is conditional.

**why the rebuttal FAILS.** (1) "It's an OPEN item" is not a refutation — it is the finding. REL.7's claim is precisely that this open item must be resolved *for a surveillance-specific reason* (oldest = first-seen = highest-value) that the L5 open item does not itself state, and that the "alert-and-hold" alternative branch has its own unhandled failure (resident scanners writing to a full disk between upload-run ceiling checks — the ceiling is enforced at *upload* time, scanners write *continuously*). (2) The alerting-first defense is exactly what SEAM-2 skewers: `status: ceiling` tells the operator backlog is growing, but by the time oldest-drop runs *the data is already gone* and RUNBOOK offers no recovery ("If eviction ran, note the window in the run log"). Visibility of the loss is not recovery of the loss. (3) The compound-condition objection is the weakest: WAN quality is admitted-unknown and per-site (Q-14), Q-10 headroom is unknown, so "the disk fills during a multi-day outage" is not exotic — it is the exact scenario BACKUP-DR names as the one true loss window. The verified LINK-5 result *sharpens* SEAM-2: the deep-sleeper's survival at Level 1 depends on `total ≥ MIN_OBSERVATIONS`, and oldest-drop during arrival week doesn't just delete "first-seen recency" as a scoring factor — it can push the surviving total *below MIN_OBSERVATIONS*, converting a detectable device into 'noise' at line 112. Eviction is therefore not only a scoring-quality choice (RF) but can be a detection-survival choice (it can flip the Level-1 classifier verdict).

**verdict:** hardened

**residual_claim:** Oldest-drop eviction of a deep-sleeper's `pending/` during its arrival week is not merely a data-retention or scoring-factor loss (SEAM-2 as filed); the verified MIN_OBSERVATIONS survival path (LINK 5) means eviction can drop the surviving observation total below the noise floor and flip the Level-1 classifier from 'pass' to 'noise' — silently converting the primary target into filtered noise. Resolve L5 with tiered eviction (`ctx_` first, `obs_` last, DLQ never) + durable capture-gap record + write-time ceiling enforcement in the scanner.

**downstream:** L5 open item, D-045, D-049, BACKUP-DR loss-window claim, RUNBOOK eviction response, `observation-context` schema (capture_gap record_type), and — newly — EVENT-CONSTRUCTION MIN_OBSERVATIONS classifier (eviction as a noise-floor input). Feeds the survival-budget doc (link 1 pinned value = eviction tier + gap record).

---

## LINK 2 — RF.2 · Sweep-collapse window (2 s vs revisit time)

**target:** RF.2

**rebuttal (strongest available).** (a) D-026 explicitly separates sweep-collapse (seconds scale) from burst-grouping (minutes scale) and marks the collapse window "validated, tunable, never hardcoded"; the 2 s value is not invented — it is earnest-v3's field-used production value (methodology line 159). (b) The doc is honest: line 22 flags "confirm target value, likely tied to sweep revisit time," and Q-06 explicitly asks for the real per-band revisit time. (c) Crucially for THIS cluster: does sweep-collapse even bear on the *deep-sleeper's* survival? The deep-sleeper's signal is defined by *inter-burst* gaps (multi-hour-to-multi-day, the burst-grouping/600 s scale), not by intra-transmission sweep reconstruction. A mis-set collapse window perturbs `detection_count`/`consolidation_confidence` *within* a transmission — a second-scale artifact — which is far from the min-obs / burst-boundary mechanisms that actually decide whether the deep-sleeper is detected.

**why the rebuttal PARTLY holds (softening).** The (c) argument genuinely narrows RF.2's role *in the gauntlet specifically*. The gauntlet's thesis is "deep-sleeper lost." The deep-sleeper's discriminating signal lives at the burst-gap scale (LINK 4), not the collapse scale. A wrong global 2 s window primarily damages *chatty* / high-band devices whose single transmission gets split into 3 (RF.2's own worst example is a 10 s high-band transmission → 3 detections) — that is a false-positive / detection_count-inflation risk, not a deep-sleeper-starvation risk. For the deep-sleeper, whose few wakes are each isolated by hours, over/under-collapse changes the observation *count per wake* but is unlikely to be the link that drops it below MIN_OBSERVATIONS on its own (it would take under-collapse merging genuinely-separate sub-second reports, which reduces count — plausible but second-order relative to eviction/min-obs).

**why it does not fully die.** RF.2 is a real, high-confidence RF finding *on its own terms* (band-and-bin_width-dependent revisit time cannot be honored by one global constant; the docs flag the tie without wiring it). And there is a genuine deep-sleeper coupling the rebuttal understates: `detection_count` per wake feeds `total activity` (the MIN_OBSERVATIONS input). If under-collapse *merges* a deep-sleeper's already-tiny per-wake hits, its total drops toward the noise floor (LINK 5). So RF.2 is not irrelevant to the gauntlet — it is a *contributing* input to the same MIN_OBSERVATIONS margin, just a smaller lever than eviction or the floor value itself.

**verdict:** softened

**residual_claim:** RF.2 survives as a first-order RF correctness finding (per-band/per-bin_width derived collapse window), but its role *within the primary-target gauntlet* narrows to a second-order contributor to the MIN_OBSERVATIONS margin, not an independent deep-sleeper-loss link. In the survival-budget doc it should be listed as "affects the obs-total input to LINK 5," not as a standalone kill point.

**downstream:** D-026, HANDOFF `detection_count`/`consolidation_confidence`, P1/P15 derivations (encode 2 s), Q-06. Contributes to LINK 5's obs-total margin rather than owning a link.

---

## LINK 3 — REL.4 · Clock drift vs 600 s across multi-day beacons

**target:** REL.4

**rebuttal (strongest available).** The design *names* this risk (R1: "NTP-synced clocks become a provisioning requirement… CV… meaningless if clocks drift"), D-057 added `ntp_offset_ms` to the heartbeat as a "continuous clock-drift visibility (time-sync validation hook)," HEALTH-CHECKS Layer-3 checks `ntp_offset_ms` within tolerance, and `ear-agent check` verifies NTP sync. Most importantly, the reviewer's OWN steelman concedes the strongest point: burst grouping is **per-sensor** (within one `agent_id`), and a constant clock *offset* is gap-invariant — it shifts all of one sensor's timestamps uniformly and cannot move a detection across the 600 s boundary relative to that sensor's own other detections. So for the *intra-sensor deep-sleeper burst grouping* that this cluster cares about, a mere skewed clock is provably harmless.

**why the rebuttal PARTLY holds but the residual is real.** The offset-invariance argument fully kills the naive "skewed clock splits the deep-sleeper's events" version. What survives is narrower and correct: 600 s grouping is offset-invariant but NOT *drift*-invariant (rate error, not constant offset). BUT — and this is the adversarial cut the reviewer under-weighted — for the drift to move a *deep-sleeper's* detections across a 600 s boundary, the drift must accumulate to >600 s of error *between two detections that are truly <600 s apart*. A deep-sleeper's whole signature is that its detections are hours-to-days apart (way past 600 s) — those never merge/split at the boundary regardless of drift, because they were never near the boundary. The only deep-sleeper detections near a 600 s boundary are the *sweep-collapsed hits within a single wake* (seconds apart) — and for drift to split those, the RTC would need ~600 s of accumulated error within a single multi-second wake, i.e. an absurd ~100%+ rate error. So the drift-splits-the-deep-sleeper's-own-events mechanism is, on inspection, near-unreachable for the deep-sleeper specifically.

**where REL.4 keeps full force — outside this cluster.** REL.4's genuinely strong half is *cross-sensor correlation* (R8 time-overlap join), which D-055 ships in THIS build (SEAM-3), is drift-AND-offset sensitive, and has no numeric tolerance and no hard gate. That is a real, high-confidence blocker-class finding — but it is a correlation finding, not a primary-target-survival-at-Level-1 link. Within the D.1 gauntlet framing (single-device survival to detection), REL.4's contribution is weaker than filed.

**verdict:** softened

**residual_claim:** REL.4 survives with full force as a cross-sensor-correlation finding (SEAM-3), but its placement as an *independent deep-sleeper burst-splitting link in the gauntlet* is over-stated: per-sensor grouping is offset-invariant, and drift large enough to cross a 600 s boundary between two truly-<600 s-apart detections is unreachable for a device whose detections are hours-to-days apart. The gauntlet should cite REL.4 for correlation, and drop or heavily caveat the "drift splits the deep-sleeper's own multi-day beacons" mechanism — a deep-sleeper's beacons are never near the 600 s boundary to begin with.

**downstream:** R1, R8, D-055, D-057/`ntp_offset_ms`, HEALTH-CHECKS L3, WP-O3. In the survival budget, drift belongs to the *correlation* link, not the burst-grouping link.

---

## LINK 4 — RF.3 · 600 s burst gap across 3-order cadence spread

**target:** RF.3

**rebuttal (strongest available).** D-026 marks 600 s "validated, tunable, never hardcoded"; methodology line 27 calls it "confirmed working value for the primary use case"; line 28 *pre-states the exact objection* ("a device whose normal cadence is minutes-apart could be wrongly merged, a chattier one over-split") and resolves it by exposing the value as a tunable analysis parameter with per-use-case override. So the design already concedes non-universality and provides the escape hatch. And for the *deep-sleeper* specifically: RF.3's own walk-through concedes the deep-sleeper is the EASY case — multi-hour-to-multi-day wakes are all >600 s apart, so "every wake is its own event — correct." The gap threshold does the right thing for the primary target by construction.

**why the rebuttal LARGELY holds for THIS cluster.** RF.3's genuine bite is on the *motion-activated* and *chatty-G2* sub-patterns (a 15-min-in-motion tracker shatters into singletons; a 5-min G2 merges into one giant event). Those are real. But they are NOT the deep-sleep 20595-class device this cluster is scoped to. For the deep-sleeper, RF.3 explicitly agrees 600 s is correct. So as a *primary-target-gauntlet link*, RF.3 is the weakest of the seven: it does not drop the deep-sleeper; it mis-handles *sibling* sub-patterns.

**why it does not fully die.** Two residuals keep it alive (softened, not killed): (1) 600 s is marked **SETTLED** (methodology headers, line 158) while Q-T1 — the real per-sub-pattern wake intervals needed to justify it — is OPEN. "SETTLED but the data that would validate it is unknown" is a legitimate status-honesty finding. (2) The motion-activated shatter case has NO fixture persona (P1/P15 cover deep-sleep and a 5-min burster; nothing exercises >600 s-in-motion). So RF.3 survives as a *class-coverage* finding, just not as a *deep-sleeper-loss* link.

**verdict:** softened

**residual_claim:** RF.3 survives as (a) a status-honesty finding (600 s marked SETTLED while Q-T1 is open) and (b) a missing-motion-activated-persona finding — but NOT as a link that drops the deep-sleep primary target, which RF.3 itself concedes 600 s handles correctly. Remove RF.3 from the deep-sleeper survival chain; keep it as a G1-sibling-sub-pattern coverage gap.

**downstream:** D-026, taxonomy G1 sub-patterns, Q-T1, PARAMETER-TUNING 600 s A/B, missing motion-activated persona. In the survival budget, LINK 4's pinned value (600 s) is *correct for the deep-sleeper* and needs only a status downgrade from SETTLED to validated-default-pending-Q-T1.

---

## LINK 5 — QA.16 · MIN_OBSERVATIONS Level-1 floor (value unstated; low-duty exception does NOT fire)

**target:** QA.16

**rebuttal (strongest available).** QA.16 is filed as **minor** by its own author, and D-028 makes the low-duty-cycle exception a NEVER-remove, CRITICAL-do-not-simplify invariant — the design's central, most-protected mechanism for exactly this device class. D-057 created P15 specifically to "guard the min-obs/min-events starvation risk," and P8 (3 obs → noise) + P15 (4 bursts over 10 days survives) bracket the floor. So the strongest rebuttal is: the design took this seriously, created a dedicated end-to-end persona, and enshrined the protecting mechanism as NEVER-remove. Surely the primary target is covered.

**why the rebuttal FAILS — and this is the load-bearing verification of the whole cluster.** I read the classifier source (`EVENT-CONSTRUCTION-METHODOLOGY-v1.md` lines 108–113) directly. QA.16's mechanical claim is **correct**: the low-duty exception's first predicate is `present in ≥ infra_hours (≈ every hour)`, `infra_hours = ceil(effective_hours × 0.95)`. A deep-sleeper present in ~4 of ~240 hours is nowhere near infra_hours — **the exception provably does not fire for it.** The device is NOT omnipresent, so it also escapes line 111 (infrastructure). Its ONLY survival path is line 112 not firing (`total ≥ MIN_OBSERVATIONS`) → line 113 ELSE → 'pass'. Therefore:
- D-028's NEVER-remove low-duty exception — the design's flagship protection — **does not protect the primary target at all.** It protects hourly-present beacons (P2), which ARE omnipresent. The multi-day-silent 20595-class device is protected *only* by the MIN_OBSERVATIONS floor + ELSE branch.
- D-057's claim that P15 "guards the min-obs/min-events starvation risk" and P15's own stated mechanism ("low-duty exception × min-obs × min-events interplay") are **both mechanically wrong** about *why* the deep-sleeper survives. The implementer at S16, reading P15's contract-proven list, would believe presence-exception logic protects deep-sleepers. It does not.
- `MIN_OBSERVATIONS` has **no numeric value anywhere** — verified: the only two occurrences are the classifier line and the tunables table, both reading "(validated default)". A value set anywhere near a deep-sleeper's ~12–16-obs total silently drops the primary target as noise, and no persona pins the floor against the smallest survivable deep-sleep total.

So the DECISION-LOG rationale (D-028) that would appear to rebut this actually *mis-describes the mechanism* and thereby proves the finding: the most-protected invariant guards the wrong device. This is worse than a minor documentation nit.

**verdict:** hardened

**residual_claim:** QA.16 is under-severity as filed (minor). It is a mechanism-correctness defect in the two most authoritative artifacts: D-028's NEVER-remove low-duty exception does not fire for the primary target (it protects omnipresent beacons, not multi-day-silent deep-sleepers), and D-057/P15's stated protection mechanism names the wrong branch. The deep-sleeper survives Level 1 *solely* via an unvalued MIN_OBSERVATIONS floor + ELSE. Elevate to major; pin MIN_OBSERVATIONS numerically; add a boundary persona at MIN_OBSERVATIONS±1 against the smallest real deep-sleep total; correct P15's contract-proven wording and D-028's applicability note.

**downstream:** EVENT-CONSTRUCTION classifier lines 108–113; MIN_OBSERVATIONS default (must be set); D-028 (applicability caveat needed); D-057/P15 stated mechanism (correction); P8/P15 boundary persona; taxonomy G1 deep-sleep sub-path. This is the anchor link of the survival budget — the one pinned value (MIN_OBSERVATIONS) that most directly decides deep-sleeper survival and is currently blank.

---

## LINK 6 — REL.2 · Undefined rebuild window

**target:** REL.2

**rebuttal (strongest available).** D-023 (recovery-by-rebuild) and R7 ("each run is a full rebuild over the configured window, deterministic given data + config") plus P16 (late-arrival reprocessing persona, created by D-057) plus the ledger's idempotent skip-if-exists (R5.1) plus atomic partition re-export (D-024) mean late arrival is "solved by construction" — a late file lands, the window's events update deterministically, no duplicates. The rebuild spine is one of the design's strongest, most-praised invariants (five reviewers list it in WHAT-TO-PROTECT).

**why the rebuttal FAILS.** The rebuttal defends the rebuild *mechanism*, which is not what REL.2 attacks. REL.2's claim is that "the configured window" is **never defined anywhere** — verified: it is absent from R7's own param list and from `rfanalysis.toml`'s enumerated params (bin size, sweep-collapse, burst gap, presence, duty multiplier, min-obs, safeguard, Pass-2 threshold — no window length). The rebuild mechanism being correct is irrelevant if the window it rebuilds over is undefined and can be set shorter than the deep-sleeper's multi-day beacon span. For the primary target this is acute: a deep-sleeper's evidence is by definition spread across multi-day-to-multi-week windows; if the rebuild window is "last 24–48 h" (the natural default given Q-19's 2–4×/day cadence), the deep-sleeper's beacons *land* (ledger ingests them) but are *never grouped into events* because they fall outside the rebuild window — they exist in `raw_landing` but never reach the classifier. P16 tests a delayed file but explicitly does not bound its age relative to the window. And the 90-day landing prune vs 13-month R2 retention compounds it: reprocess beyond 90 days is defeated by skip-if-exists. REL.2 is high-confidence and directly deep-sleeper-relevant.

**why it HARDENS in this cluster.** The gauntlet framing surfaces something REL.2 alone did not: this is not just "late arrival" — a deep-sleeper that beacons steadily but *sparsely* over 3 weeks, with a rebuild window of days, has *most of its evidence permanently outside every rebuild*, even with perfect delivery and no outage. The window-vs-cadence mismatch starves the deep-sleeper in the ordinary case, not only the WAN-outage case. The deep-sleeper needs `rebuild_window ≥ its analysis window`, and the taxonomy itself says deep-sleep "needs long analysis windows" (line 13) — a requirement no window param currently expresses.

**verdict:** hardened

**residual_claim:** REL.2 survives and hardens: beyond late-arrival, an undefined/short rebuild window starves the deep-sleeper in the *normal* case, because its multi-day-to-multi-week evidence spans more than any days-scale window, yet the taxonomy explicitly requires "long analysis windows" for this class. Pin `window` as a first-class versioned param with invariant `rebuild_window ≥ deep_sleep_analysis_window ≥ raw_landing_retention ≥ max_backlog_age`, and add P16b (file older than routine window but within retention still groups).

**downstream:** R7, R5.1 (skip-if-exists), D-023/D-024, RECOVERY-RETENTION (90-day prune vs 13-month R2), P16/P16b, Q-14, Q-19, taxonomy "needs long analysis windows." Survival-budget link 6 pinned value = the rebuild window length, tied explicitly to the deep-sleep analysis window.

---

## LINK 7 — QA.5 · min-events=3 Level-2 floor; P15→G1 not in S19b gate

**target:** QA.5

**rebuttal (strongest available).** D-057 makes P15 the crown-jewel end-to-end persona in the double-derivation set, explicitly guarding min-obs/min-events starvation; D-055 un-defers Level 2 with a rule-based classifier and the taxonomy has a stated "deep-sleep sub-path for very-few-event streams" (line 53) that "tolerates very few events" (line 13). So the design anticipated few-event deep-sleepers and gave the classifier a sub-path for them. And min-events=3 is a *reasonable* floor — you genuinely cannot compute a meaningful CV on <3 gaps.

**why the rebuttal FAILS.** Two verified facts defeat it. (1) The S19b gate line reads (verified, BUILD-RUNSHEET line 31): `P1→G1, P2→G1, P3→G4 assertions green` — **P15→G1 is not listed.** So P15's *taxonomy/classification* half has no Level-2 gate assertion at all; P15 is gated only at S16 (Level 1 events built). QA.5's claim that "the starvation risk is assigned to a layer that has no persona" is literally true on the page. (2) The starvation the persona claims to guard ("min-events rules must not erase it") is a Level-2 Analysis parameter (PARAMETER-TUNING "Min events for analysis | Analysis | 3"), but P15 is a Level-1 fixture persona — so P15 can be GREEN at S16 (4 events built, survived noise filter) while the device is STILL starved at S19b, because the 4-event/3-gap stream sits exactly at the min-events=3 floor and the `≥` vs `>` operator on that boundary is untested. The taxonomy's "deep-sleep sub-path" (line 53) is a *decision sketch*, not an assertion that the min-events floor is waived on that path — there is no gated check that a 3-gap deep-sleeper reaches G1 without the CV computation the floor forbids.

**cross-lens adjudication vs LINK 5 (QA.16).** QA.5 and QA.16 pull *toward the same conclusion from different layers* — not a conflict. QA.16 owns the Level-1 killer (MIN_OBSERVATIONS + the low-duty exception mis-attribution); QA.5 owns the Level-2 killer (min-events + missing S19b gate). Together they establish that the deep-sleeper has TWO independent starvation floors, one per layer, and P15 pins neither at its boundary (Level-1 boundary unvalued; Level-2 gate absent). This is mutually reinforcing, and it strengthens the systemic claim.

**verdict:** survives

**residual_claim:** Stands as filed. P15's taxonomy half has no S19b gate (`P15→G1` absent from the gate line — verified), and the min-events=3 floor can starve a 4-event/3-gap deep-sleeper at Level 2 while P15 is green at Level 1. Split P15 into P15-L1 (S16) and P15-L2 (S19b), add `P15→G1` to the S19b gate, and add a gated assertion that the deep-sleep sub-path reaches G1 without requiring min-events≥3.

**downstream:** D-057, PARAMETER-TUNING min-events row, taxonomy G1 deep-sleep sub-path, S19b gate line, double-derivation set, four-factor PERIODIC factor. Survival-budget link 7 pinned value = the min-events exception for the deep-sleep path.

---

## SYSTEMIC CLAIM — "No single artifact holds the full survival chain" + proposed survival-budget doc + end-to-end persona

**target:** D.1 systemic finding

**rebuttal (strongest available).** The design DOES have an end-to-end guardian for the primary target: P15 is explicitly "deep-sleep end-to-end survival — the primary target's hardest case" (D-057), in the double-derivation set, and D-055's un-deferral means Level 2 now has a gated classifier. D-028's NEVER-remove low-duty exception is a dedicated, protected mechanism for this exact device class. So the operator could argue the chain IS held — by P15 end-to-end plus D-028's flagship protection — and the "seven independent links" are just the normal decomposition of one persona's concerns.

**why the rebuttal FAILS — and the systemic claim HARDENS.** The cross-examination *strengthened* the systemic finding rather than weakening it, because the two defenses the rebuttal leans on are precisely the two that this cluster proved hollow for the deep-sleeper:
- **D-028's flagship low-duty exception does not fire for the primary target** (LINK 5, verified at classifier source). The design's single most-protected, NEVER-remove mechanism guards omnipresent beacons, not multi-day-silent deep-sleepers. The one artifact that looks like it holds the chain protects the wrong device.
- **P15 does not hold the chain end-to-end**: its Level-2 half has no gate (LINK 7, `P15→G1` absent — verified), its stated protection mechanism is mechanically wrong (LINK 5), and it tests one device in isolation, never the gauntlet. P16 (rebuild) tests a *different* device. No persona traces a deep-sleeper through eviction → obs-total → rebuild window → MIN_OBSERVATIONS → min-events as one chain.

Adjudicating the seven links after cross-exam refines but does not dissolve the gauntlet: LINKS 5, 6, and 1 (REL.7/SEAM-2) are the load-bearing deep-sleeper-loss links and all three HARDENED. LINKS 2, 3, 4 SOFTENED *as gauntlet links* (they are real findings but bear on the deep-sleeper only second-order, or on sibling sub-patterns / correlation) — which actually makes the systemic claim *cleaner*, not weaker: the gauntlet is really a **3–4 load-bearing-link chain** (obs-total inputs → MIN_OBSERVATIONS floor → rebuild-window inclusion → min-events Level-2 floor), each owned by a different lens (RF/REL/QA), each with an unset pinned value, and NO single artifact tracing it. The proposed cross-lens **survival budget** (one owned doc with a pinned value at each link) and **extended end-to-end persona** (P15 gated at BOTH S16 and S19b, failing if any link drops the device) are the correct fix and are, if anything, *under-scoped* as filed — they should center on the verified critical links (MIN_OBSERVATIONS value, rebuild-window length, min-events deep-sleep exception, eviction tier) rather than treating all seven as equal.

The one honest caveat for the author: the gauntlet as *narrated* (seven equal independent kill points) over-counts. After adversarial pruning, LINK 3 (drift) and LINK 4 (600 s) do not independently drop the deep-sleeper, and LINK 2 (collapse) is a second-order input. The systemic *risk* is real and hardened; the *count* should be corrected to the load-bearing subset so the survival-budget doc pins effort where it matters.

**verdict:** hardened

**residual_claim:** The systemic finding survives and hardens: no single artifact holds the deep-sleeper's survival chain, and the two artifacts that appear to (D-028's low-duty exception, P15 end-to-end) were verified NOT to protect the multi-day-silent primary target — the exception fires only for omnipresent beacons, and P15's Level-2 half is ungated with a mis-stated mechanism. Correction for the author: the chain's load-bearing links are ~4 (obs-total inputs → MIN_OBSERVATIONS → rebuild-window inclusion → min-events Level-2 floor + eviction as an obs-total input), not seven equal ones; drift and 600 s-gap do not independently drop the deep-sleeper. Build the survival-budget doc + P15-extended (gated S16 AND S19b, fails if any load-bearing link drops the device), centered on the verified critical links with their currently-blank values pinned.

**downstream:** A-tier synthesis item. Owns: MIN_OBSERVATIONS value, rebuild-window length + invariant, min-events deep-sleep exception, eviction tier + capture-gap record, D-028 applicability caveat, P15 dual-gate. Cross-references LINKS 1/5/6/7 (hardened/survives) as the chain; LINKS 2/3/4 as secondary inputs.

---

## Cluster summary (verdicts)

| Link | Finding | Verdict | One-line |
|---|---|---|---|
| 1 | REL.7 / SEAM-2 | hardened | Verified: eviction can push surviving obs-total below MIN_OBSERVATIONS, flipping the Level-1 verdict to 'noise' — a detection-survival choice, not just retention. |
| 2 | RF.2 | softened | Real RF finding, but within the gauntlet it is only a second-order input to the MIN_OBSERVATIONS margin, not an independent deep-sleeper kill point. |
| 3 | REL.4 | softened | Per-sensor grouping is offset-invariant and a deep-sleeper's multi-day beacons are never near the 600 s boundary; REL.4 keeps full force as a *correlation* finding, not a deep-sleeper burst-split link. |
| 4 | RF.3 | softened | RF.3 itself concedes 600 s is correct for the deep-sleeper; survives as SETTLED-vs-open-Q-T1 status honesty + missing motion-activated persona, not as a deep-sleeper link. |
| 5 | QA.16 | hardened | Verified at classifier source: D-028's NEVER-remove low-duty exception does NOT fire for the primary target; it survives only via an unvalued MIN_OBSERVATIONS + ELSE. Under-severity as filed (minor→major). |
| 6 | REL.2 | hardened | Window undefined; an ordinary days-scale window starves the deep-sleeper's multi-day evidence even with perfect delivery — taxonomy itself demands "long analysis windows." |
| 7 | QA.5 | survives | Verified: `P15→G1` absent from S19b gate; min-events=3 can starve a 4-event/3-gap deep-sleeper at Level 2 while P15 is green at Level 1. |
| — | Systemic D.1 | hardened | The two artifacts that appear to hold the chain (D-028 exception, P15) verified NOT to protect the primary target; but the narrated seven-equal-links over-counts — load-bearing chain is ~4. Survival budget + dual-gated P15 warranted. |

**Cross-domain conflicts surfaced for the author:**
- **None are true dissents within this cluster** — QA.5 (Level-2) and QA.16 (Level-1) reinforce rather than conflict (two starvation floors, one per layer).
- **Author adjudication required (not a lens conflict but a narration correction):** the D.1 gauntlet is filed as seven co-equal independent kill points; cross-exam finds only ~4 are load-bearing for the deep-sleeper (LINKS 1, 5, 6, 7), while LINKS 2/3/4 are secondary inputs or bear on sibling sub-patterns/correlation. The systemic risk hardens, but the survival-budget doc should be scoped to the load-bearing subset, and REL.4/RF.3 should be re-homed (correlation / G1-sibling coverage) rather than listed as deep-sleeper links.
