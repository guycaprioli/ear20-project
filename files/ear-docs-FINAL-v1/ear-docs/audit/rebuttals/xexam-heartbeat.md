# Cross-Examination Rebuttals — Cluster: Uploader cardinality & heartbeat/health semantics (SEAM-8)

**Cross-examiner:** Phase 2 adversarial (heartbeat cluster). **Findings under exam:** LEAD.1, REL.1, OPS.2, OPS.10.
**Mandate:** try to KILL each. A finding survives only if it withstands the strongest rebuttal drawn from the DECISION-LOG rationale, another lens's priors, an under-weighted steelman, and a reproduction test.

**Documents read:** `contracts/LOCAL-HOST-DATA-FLOW-v2.md` (L3, L11, L26, L71, L89, L133), `contracts/schemas/heartbeat.v1.schema.json` (full), `operations/HEALTH-CHECKS-v1.md` (full), `DECISION-LOG-v1.md` (D-012, D-015, D-016, D-028, D-034, D-045, D-049, D-052, D-057, D-059), `OPEN-QUESTIONS-v1.md` (Q-09). Reviewer files for LEAD.1 (lead-consistency L11-15), REL.1 (reliability L26-34), OPS.2 (sre-ops L69-81), OPS.10 (sre-ops L197-209).

---

## The decisive fact that governs the whole cluster

Before adjudicating individual findings I establish the load-bearing fact, because it decides whether the DECISION-LOG "already answers this."

**D-016 and D-045 resolve the *cardinality* (one shared uploader). They do NOT resolve the *heartbeat-authorship consequence*, and the freeze makes the consequence live.**

- D-016: "Process model: 4 scanner processes + **1 shared uploader** + 1 shared DLQ per host. (Docs corrected from an earlier 4×2 claim.)" — cardinality decided: ONE.
- D-045: "ephemeral = `uplink-upload.timer` → oneshot ... scan `pending/` → edge filter + feed split → gzip → PUT → **write heartbeat** → exit" — ONE process writes "the heartbeat" (singular).
- BUT `heartbeat.v1.schema.json` L18-21 pins `agent_id` to `^[a-z]{3}-rf[0-9]+[a-d]$` — a **mandatory device letter** — with `additionalProperties:false` (L16) and **no** `host`, `scope`, or `facility` field in the property set. The key template (schema `$id` title, D-015) is `_health/{agent_id}.json`.
- Per HEALTH-CHECKS L20, Agent-liveness checks **"every active sensor's heartbeat fresh ... stale → that sensor flagged"** — the consumer reads **per-device** heartbeats.

So the decided cardinality (one process) and the frozen schema (per-device-lettered key, no host scope) are in **direct structural tension**: one process must author four device-lettered objects, and the two candidate ways to do that are each independently broken as-written (see REL.1). D-016/D-045 answered "how many uploaders" but **left "who authors the four device-lettered heartbeats, and are `queue_depth`/`disk_pct` per-device or host-level" unstated** — and the schema that would carry the disambiguator is frozen (D-034: "schemas ... imported by both component test suites"; D-052 line: "Schemas are the tie-breaker over prose"). This is why the cluster is **not** merely stale prose that a decision already fixes.

I also confirm the freeze is real for *this* schema specifically: heartbeat.v1 is one of the three D-034 machine-readable schemas ("JSON Schema for signal/context/**heartbeat** records ... live in `ear-docs/contracts/schemas/`, imported by both component test suites"), it is enumerated in D-052's regime ("Context records ... `l45_heartbeat`" and the tie-breaker rule), and it was last touched by D-057 (added `ntp_offset_ms`). It is a governed, imported, tie-breaking artifact — not a scratch draft. A schema change here ripples into both test suites and the server consumer. That is exactly the D-030-class "cheaper before repo creation" property REL.1 claims.

---

## FINDING-LEAD.1 — LOCAL-HOST-DATA-FLOW-v2 says ONE shared uploader AND "its own uploader" AND "4 independent uploaders per host"

**target:** `contracts/LOCAL-HOST-DATA-FLOW-v2.md` L11 / L26 / L133 vs D-016 / D-045

**Filed claim:** the contract self-contradicts on uploader cardinality; a build agent reading the *contract* (as the bootstrap instructs) rather than the decision log builds four per-device uploader timers + four DLQs, and the cardinality decides run-lock need, heartbeat authorship, and eviction scoping. Severity: **major**.

**Strongest rebuttal I can mount:**

1. *Rationale-addressed attempt.* D-016 explicitly says "**Docs corrected from an earlier 4×2 claim**" and LEAD.1's own steelman concedes "the intent to standardize on one shared ephemeral uploader is clear and logged; a reader who trusts the decision log gets the right model." So the *decision* is unambiguous — this is stale prose the log already overrides, and D-052's own principle ("schemas/decisions are the tie-breaker over prose") means a disciplined reader resolves it correctly. Downgrade to doc-hygiene.

2. *Under-weighted steelman.* The contradiction is confined to a single legacy paragraph (L11) plus a donor-era diagram; the authoritative D-045 mechanics (timer oneshot, `pending/` = retry queue) are stated correctly everywhere the *build* work-packages (WP-A3/A4) actually key off. A competent agent cross-references the decision log the brief tells it to read.

**Why the rebuttal FAILS (finding survives):**

- The "trust the decision log" defense is defeated by the design's *own stated contract-precedence rule*. The bootstrap tells a WP agent to read "the contracts your WP touches." LOCAL-HOST-DATA-FLOW-v2 **is** that contract, and it is labelled the build spec: L3 — "the contracts here ARE the build spec." D-052's tie-breaker is scoped to **field questions** ("schemas are the tie-breaker over prose **on field questions**"), *not* to process-topology/cardinality questions. There is **no** logged precedence rule that a decision-log entry overrides a contradiction *inside a contract body* on a topology question. So the disciplined reader has no rule that resolves it; the contract genuinely says three different things (L11 "ONE shared" AND "its own uploader ... its own dead-letter queue"; L26 "(4 independent uploaders per host)"; L133 "PRODUCER GUARANTEES (uploader, **one per device**)").

- The "single legacy paragraph" minimization is factually wrong: I verified the contradiction is replicated in **four** places, including the normative **PRODUCER GUARANTEES** block at L133 ("uploader, one per device") and the per-device circuit-breaker/DLQ block at L221 — i.e. it is embedded in the parts of the contract a builder treats as authoritative interface promises, not just prose scene-setting.

- The failure reproduces concretely and cheaply: a builder implementing L133's "uploader, one per device" ships four `uplink-upload@{a..d}.timer` units + four DLQs, contradicting D-016/D-045's single unit, and (critically) creating exactly the four-uploaders-race-the-disk-ceiling and run-lock problems REL.1/REL.5 warn about. This is not speculative — it is the literal reading of a PRODUCER GUARANTEES line.

**Verdict:** **survives.** The decision-log-overrides defense fails because no logged precedence rule covers a topology contradiction *inside the build-spec contract*, and the contradiction lives in normative PRODUCER GUARANTEES lines (L133), not just scene-setting prose.

**residual_claim:** unchanged as filed — major, pre-repo doc fix, but see the cluster consolidation note: LEAD.1 is the *documentation* half of a single defect whose *schema/semantic* half is REL.1.

**downstream:** WP-A3/WP-A4 topology; run-lock scoping (REL.5); eviction scoping (REL.7); merges with REL.1 for the heartbeat consequence.

---

## FINDING-REL.1 — a shared uploader cannot author four device-lettered `_health/{agent_id}.json` heartbeats; the schema is frozen

**target:** D-016 / D-045 / L-chain "Deployment shape" / `heartbeat.v1.schema.json` `agent_id`

**Filed claim:** ONE shared uploader per host must produce heartbeats whose key `_health/{agent_id}.json` carries a device letter for four distinct `agent_id`s, but there is one run with one `queue_depth`/`disk_pct`/`dlq_count`. Either the uploader writes one heartbeat (→ three of four sensors permanently stale → dead-heartbeat alert fires forever, D-049/P14) or four copies of a host-aggregate body presented as per-sensor (→ `queue_depth` is host-level but consumed per-sensor, and can disagree with the per-device `obs_` cross-check). "Either way the freshness contract is unsatisfiable as written." Severity: **blocker**; flagged D-030-class (schema frozen → fix before repo).

**Strongest rebuttal I can mount:**

1. *Rationale-addressed / trivial-fix attempt.* The one-process-writes-four-objects pattern is completely ordinary. The oneshot scans four device folders (it already does — D-045: "scans all four devices' folders"); for each folder it emits `_health/{that agent_id}.json` with **per-folder** `queue_depth`/`dlq_count` (it is *counting files in that folder* — trivially per-device) and the shared `disk_pct` (label it host-level). D-045 already says the heartbeat "carries queue depth + disk %"; nothing says the single process may write only one object. So "a single shared process cannot author a device-lettered heartbeat" is simply false — a single process writes N files all the time. The "permanently stale for 3 of 4" scenario assumes the naive implementation (one object) that no competent builder ships. This is an implementation note, not a blocker, and not a schema problem at all.

2. *Another lens's prior (ops).* The sre-ops COVERAGE STATEMENT (sre-ops L323) explicitly clears the heartbeat *schema*: "Well-chosen required fields ... No finding; this is a clean contract. (Note: the *interpretation* of heartbeat freshness is where OPS.2/OPS.10 land, not the schema itself.)" Ops — the lens that owns the consumer — says the schema is fine and only the interpretation needs a doc note. That cuts against REL.1's "schema is frozen / D-030 blocker" framing.

**Why the rebuttal PARTLY succeeds and PARTLY fails (softened):**

The rebuttal **kills REL.1's strongest phrasing** ("a single shared process *cannot* author a device-lettered heartbeat" and "the freshness contract is unsatisfiable as written"). It plainly *can*: one oneshot iterating four folders and writing four `_health/{agent_id}.json` objects with per-folder queue/dlq counts is a five-line loop, and D-045 already has the process visiting all four folders. `disk_pct` being shared is cosmetic (a host number in every sensor's heartbeat — semantically fine; a WAN/disk-ceiling event is host-wide anyway). REL.1's proposed_alternative even offers this exact resolution ("keep four keys but have the one uploader emit four per-device heartbeats"), which concedes the "unsatisfiable" framing is too strong. The "blocker" label rests on "unsatisfiable," and that word does not survive.

But the rebuttal **fails to fully kill it**, because a genuine residual defect remains that is NOT just implementation-notes:

- **The frozen schema cannot label which numbers are per-device vs host-level.** `queue_depth` (schema L39-42: "files awaiting upload across all feeds") and `disk_pct` are, in the one-process design, some per-device (queue/dlq, if counted per-folder) and one host-wide (`disk_pct`), yet `additionalProperties:false` and the absence of any `host`/`scope` field mean the server consumer **cannot distinguish** a per-device queue number from a host-aggregate one. HEALTH-CHECKS L21 "Agent condition" checks "queue not monotonically growing **across sweeps**" per sensor and "disk below ceiling" per sensor — if `disk_pct` is silently host-level, all four sensors red or amber together on one full disk, which is *correct by luck* but means "per sensor flagged" (L20) over-counts a host event as four sensor events (which is exactly OPS.2's cardinality problem, re-entering here). The schema being frozen (D-034 imported by both suites) means adding the `host`/`scope` disambiguator REL.1 recommends is a **pre-repo** edit, not a build-time one. That specific claim — "the disambiguation belongs in the frozen schema, so do it before the freeze bites" — survives intact.

- Ops's clearance of the schema (rebuttal #2) is about *field hygiene* ("additionalProperties:false prevents drift"), not about *whether the field semantics are host- or device-scoped*. Ops did not examine the one-process/four-device authorship question — REL owns that seam. So the ops COVERAGE STATEMENT does not actually rebut REL.1's residual; the two lenses looked at different properties of the same schema.

**Verdict:** **softened.** Reduced claim: the heartbeat is authorable by the single uploader (writing four per-device objects) — so this is **not a blocker and not "unsatisfiable."** The surviving defect is (a) LEAD.1's contract contradiction must be resolved to the one-uploader model *and paired with an explicit "one uploader emits four per-device heartbeats" statement*, and (b) the frozen schema needs a `host`/`scope` disambiguator (or an explicit "`disk_pct` is host-level; `queue_depth`/`dlq_count` are per-device") so the server does not mis-scope a host event as four sensor events. Severity drops from **blocker → major**. The D-030-class "fix before repo" property survives *for the schema-disambiguator half only* (D-034 freeze), which is why this must still be resolved pre-repo even though it is no longer a blocker.

**residual_claim:** (1) Adopt one shared uploader emitting four per-device `_health/{agent_id}.json` heartbeats; state it in D-045/L11. (2) Add a `host`/`scope` field OR document `disk_pct`=host-level / `queue_depth`+`dlq_count`=per-device, in the frozen schema, pre-repo. (3) Delete the "permanently stale 3-of-4" catastrophe framing — it is the naive-implementation strawman, not the contract.

**downstream:** heartbeat.v1 schema (pre-repo); HEALTH-CHECKS L20/L21 per-sensor vs host verdict scoping (feeds OPS.2); LEAD.1 (shared documentation fix).

---

## FINDING-OPS.2 — facility-WAN outage turns 48 heartbeats stale at once; alert cardinality 1-vs-48 is undefined

**target:** D-049 / HEALTH-CHECKS-v1 — one ntfy topic cannot distinguish facility-WAN from noise; the correlation logic is only advisory.

**Filed claim:** 48 sensors stale in one sweep → either 48 pushes (fatigue, buries the next real single-sensor failure) or one push identical to a single-sensor failure (can't triage from the notification). "WAN suspicion noted" is a dashboard annotation, not an alert-routing rule; the intelligence is on the wrong side of the notification boundary. Severity: **major**.

**Strongest rebuttal I can mount:**

1. *Rationale-addressed attempt.* HEALTH-CHECKS L29 already specifies **state-change-only** alerting with **"no re-alert spam on persisting known failures, one reminder per configurable period."** And L20 already encodes the facility rollup input: "many stale in one facility → WAN suspicion noted." D-049 is deliberately *one channel, no alerting stack to run* for a single operator — adding PagerDuty-style routing is the operational own-goal the design explicitly rejects. The RUNBOOK already leads the human with "Check facility WAN first (multiple agents dark = WAN)." So the design *thought about this*; it is a doc-precision nit, not a real gap.

2. *Reproduction challenge.* "48 pushes" only happens under one specific (naive) implementation of "notify() on any new red." The doc says the sweep "produces a **verdict per check**" (L14) — Agent-liveness is **one** check. If notify() fires per *check* verdict (not per sub-verdict), a whole-facility outage is naturally *one* red on the Agent-liveness check → *one* push. So the catastrophic 48-push case is not what the doc says; it is a misreading of "verdict per check."

**Why the rebuttal PARTLY succeeds (softened):**

Rebuttal #2 lands a real hit on the *severity* of the worst branch. Re-reading L14 ("a **verdict per check**") and L29 ("notify() on **any new red**"), the most natural reading is that notify fires on the *check-level* verdict transitioning to red — and Agent-liveness is a single check. Under that reading a facility-WAN outage produces **one** red (Agent-liveness) → one push. So "48 phone pushes" is the *less* natural reading, not the default. The pure alert-storm scenario is softened.

But the rebuttal **does not kill the finding**, because OPS.2's *core* claim is the other horn, and it is real and un-addressed:

- If notify fires once (the natural reading), then **a facility-WAN outage and a single dead sensor produce an identical single "Agent-liveness red" push** — the operator cannot triage from the notification and must open the dashboard every time (OPS.2's second horn, verbatim). The RUNBOOK's "multiple agents dark = WAN" is, exactly as filed, a *human diagnostic step performed after opening the dashboard*, not information carried in the alert. So the finding's central complaint — *the WAN-vs-single-sensor discrimination is on the wrong side of the notification boundary* — stands whether the cardinality is 1 or 48.

- "WAN suspicion noted" (L20) is verifiably *passive* — I confirmed there is **no rule** anywhere that says "if ≥N same-facility sensors go stale in one sweep, emit ONE facility-level alert with the root cause named." So the notify() payload cannot say "facility ral: WAN, 48 dark" vs "sensor ral-rf2b dark" — the operator gets the same string. That is a genuine, cheap-to-fix gap in D-049/D-059, not a doc nit.

- The "verdict per check" reading that saves us from 48 pushes actually *reintroduces* REL.1's scoping problem: if Agent-liveness is one check producing one verdict, then "that sensor flagged" (L20) and per-sensor flagging in "Agent condition" (L21) are *sub*-verdicts under a single check — so the doc is itself ambiguous about whether the unit of a "red" is the check or the sensor. That ambiguity **is** OPS.2's evidence ("cardinality of 'a new red' is undefined — per-check? per-sensor? per-sweep?"), and my attempt to resolve it in the rebuttal only proved it is undefined.

**Verdict:** **survives** (severity holds at major, one horn softened). The alert-storm horn is softened (the natural reading is one push, not 48), but the *triage-ambiguity* horn — the operator cannot distinguish facility-WAN from single-sensor from the notification, and there is no facility-rollup-before-per-sensor-alert rule — survives fully, and the cross-exam *hardened* the evidence by showing "verdict per check" is itself ambiguous about the unit of a red.

**residual_claim:** Specify in D-059: (1) the sweep computes a **facility-level WAN verdict before per-sensor verdicts**; ≥N same-facility stale in one sweep → exactly one `notify()` naming the root cause ("facility X: probable WAN, K/M dark"), suppressing the per-sensor reds until it clears; (2) define the notify() cardinality contract = at-most-one push per root-cause-group per sweep (root-cause ∈ {facility-WAN, facility-R2-auth, single-sensor, pipeline, server-infra}); (3) P14 sweep-cardinality persona. This is the SAME facility-rollup machinery REL.1's `disk_pct`=host-level and OPS.11's maintenance-window need — one facility-scoping fix.

**downstream:** D-049/D-059 notify de-dup grouping key; HEALTH-CHECKS L20 facility verdict; fixture P14; ties to OPS.11 (maintenance-window) and REL.1 (host-scope in schema).

---

## FINDING-OPS.10 — "capture advancing" keys on `obs_` arrival, which is correctly near-zero for the quiet primary target → false-reds healthy quiet sensors

**target:** HEALTH-CHECKS Layer 3 "capture advancing" + "handoff advancing" — "expected window" per active sensor undefined and per-band variable.

**Filed claim:** HEALTH-CHECKS L22 keys "capture advancing" on "new `obs_`/`ctx_` objects ... within expected window." But `obs_` is FORWARD-only (D-012) and near-zero is *expected and correct* for the primary target's quiet periods (D-028 low-duty exception; P15 deep-sleep). So the check either (a) false-reds a healthy quiet sensor (fatigue) or (b) is loose enough to also accept a genuinely broken silent scanner (a miss). Basing liveness on `obs_` conflates "sensor working" with "sensor seeing signal" — the exact split D-012 exists to separate. Severity: **minor**. Confidence: med.

**Strongest rebuttal I can mount:**

1. *Rationale-addressed / already-handled attempt.* The check text is **`obs_`/`ctx_`** — it already includes `ctx_`. And `ctx_` "arrives *every uploader tick regardless of signal*" (OPS.10's own evidence, L203). So a healthy quiet sensor **still emits `ctx_` every ~5 min** (heartbeat + config stamp ride the context feed, D-012/D-045). If "capture advancing" is satisfied by `ctx_` arrival — which the doc *literally lists* — then a deep-sleeping-target sensor with zero `obs_` but fresh `ctx_` is **green**, not red. The finding assumes the check requires `obs_`, but the doc says `obs_` *or* `ctx_`. False-red does not reproduce. Kill it.

2. *Redundancy attempt.* Agent-liveness (L20, heartbeat freshness) already covers "is the sensor alive," and the heartbeat rides `ctx_`. So "capture advancing" keying partly on `obs_` is *additive* signal, not the sole liveness gate — a quiet sensor stays green on Agent-liveness regardless. Minor at most, arguably no-op.

3. *Confidence.* The reviewer self-rated **med** confidence, acknowledging the reading is uncertain.

**Why the rebuttal PARTLY succeeds but the finding survives (softened):**

Rebuttal #1 correctly identifies that the doc lists `obs_`/`ctx_` disjunctively and that `ctx_` arrives every tick — so under the *charitable* reading, a quiet-but-alive sensor is green and there is no false-red. This softens OPS.10's horn (a): the catastrophe ("false-reds healthy quiet sensors, training the operator to ignore reds") is not forced by the text.

But the rebuttal **exposes and sharpens horn (b), which is the more dangerous one**, and this is where the cross-exam makes the finding *worse*, not better:

- The check as written lumps `obs_` and `ctx_` with an OR and one undefined "expected window." My rebuttal's own logic — "`ctx_` arrives every tick, so `ctx_`-freshness keeps a quiet sensor green" — means the *only* thing distinguishing this check from Layer-3 Agent-liveness (heartbeat freshness, also `ctx_`-borne) is the `obs_` term. Strip the `obs_` contribution to avoid horn (a) and **"capture advancing" collapses into a duplicate of Agent-liveness** — it stops catching the one failure it exists for: **a scanner that has silently died while the uploader still ticks** (heartbeat + `ctx_` fresh, `obs_` correctly zero because the radio is producing nothing). That is precisely the "alive and heartbeating but scanner stopped" case the steelman (L199) praises the check for catching. So the OR-with-undefined-window formulation means the check is **either a false-red machine (if it demands `obs_`) or a no-op duplicate of Agent-liveness (if `ctx_` alone satisfies it)** — it cannot be both a quiet-tolerant *and* a broken-scanner-detecting check without the per-sensor self-baselining OPS.10 asks for. The finding is *correct that the check is mis-specified*, and the cross-exam shows the mis-specification is a genuine dilemma, not a wording slip.

- The "expected window" term is verifiably undefined and the doc itself says nothing per-band, while D-028/P15 and PARAMETER-TUNING establish per-facility/per-band variance — so a single global window is provably wrong for a fleet spanning quiet-racked-high-band and busy-open-low-band sensors. That half is unrebutted.

- Rebuttal #2 (redundancy with Agent-liveness) actually *confirms* the finding's fix rather than killing it: OPS.10's proposed_alternative is exactly "base liveness on `ctx_`, treat `obs_` volume as separate informational, self-baseline the window" — i.e. de-conflate the two so "capture advancing" becomes a real scanner-alive check distinct from heartbeat freshness.

**Verdict:** **survives** (stays minor; one horn softened, the other hardened). The false-red horn is softened by the `obs_`-OR-`ctx_` reading, but the cross-exam *hardens* the core: the OR-with-undefined-window formulation forces a real dilemma (false-red vs no-op duplicate of Agent-liveness) that means the check cannot do the job its steelman credits it with. The fix (split `obs_`-signal-volume from `ctx_`-liveness; self-baseline the window per sensor; state that zero-`obs_` is not a red for the primary target) is required for the check to be meaningful. Severity stays **minor** (it is a health-check semantics nit, not a data-loss path), but it is a *real* semantics defect, not stale prose.

**residual_claim:** Reconcile "capture advancing" with D-028/P15 in HEALTH-CHECKS: liveness = `ctx_`/heartbeat freshness; `obs_` volume = separate baseline-relative informational signal (never a hard red on its own); "expected window" = per-sensor self-baselined from that sensor's recent cadence, not a fleet-global constant. Add a P14/P15 persona: quiet-but-healthy sensor → not red; scanner-dead-while-heartbeating → red.

**downstream:** HEALTH-CHECKS L22/L24; fixture P14/P15; DASHBOARD "last data" column (last `obs_` vs last `ctx_` — they differ); PARAMETER-TUNING per-facility window baselines.

---

## Cluster adjudication — are these four ONE heartbeat-semantics fix?

**Yes — with one caveat.** SEAM-8's thesis (the heartbeat is over-loaded as identity artifact + liveness/signal discriminator + WAN-rollup input, contested from three directions, schema frozen) is **confirmed** by this cross-exam, and the four findings collapse into **one coherent heartbeat-and-health-semantics contract edit** with four clauses:

1. **Cardinality (LEAD.1 + REL.1):** resolve LOCAL-HOST-DATA-FLOW-v2 to the single-shared-uploader model *everywhere* (L11, L26, L133, L221), and add the explicit sentence "one uploader run emits four per-device `_health/{agent_id}.json` heartbeats." This is one edit satisfying both the documentation half (LEAD.1) and the authorship half (REL.1).

2. **Schema scope disambiguator (REL.1):** in the frozen `heartbeat.v1.schema.json`, mark `disk_pct` host-level and `queue_depth`/`dlq_count` per-device (or add a `host`/`scope` field). **Pre-repo**, because the schema is D-034-frozen and imported by both test suites.

3. **Liveness basis (OPS.10):** base "capture advancing" and Agent-liveness on `ctx_`/heartbeat freshness (per-tick by construction), and treat `obs_` volume as separate, self-baselined, informational — so a quiet primary target is green and a silently-dead scanner is red.

4. **Facility rollup / alert cardinality (OPS.2):** compute a facility-level verdict *before* per-sensor verdicts; one `notify()` per root-cause-group per sweep. This same facility-scoping machinery is what clause 2 needs (host-level `disk_pct` → host event, not four sensor events) and what OPS.11's maintenance-window needs.

**The caveat / the one genuine dissent for the author:** the four are one *semantic* contract, but they split across **two edit windows** with different urgency:
- Clauses 1 and 2 touch **frozen artifacts** (the build-spec contract's PRODUCER GUARANTEES and the D-034 schema) → **pre-repo (D-030-class)**, non-negotiable timing.
- Clauses 3 and 4 touch HEALTH-CHECKS/D-059 semantics (not a frozen data schema) → can land at build time (WP-S4), though cheapest to pin the *decision* now.

So "one fix" is right for **coherence** (all four must be decided together or the heartbeat means different things to different consumers) but **not** for **scheduling** — forcing clauses 3/4 into the pre-repo gate over-scopes the blocker, and treating clauses 1/2 as build-time under-scopes it. **Author to adjudicate: bundle the *decision* now (one heartbeat-semantics ADR), but stage the *edits* — schema + contract pre-repo, health-check semantics at WP-S4.**

**Severity re-grade for the bundle:** REL.1 drops blocker→major (authorship is possible; only the schema disambiguator is pre-repo-forced). LEAD.1 stays major. OPS.2 stays major. OPS.10 stays minor. The *bundle* is a **major pre-repo item** on the strength of clauses 1-2 (frozen artifacts), carrying clauses 3-4 as build-time riders.

**Conflict surfaced for the author (cross-lens):** REL.1 filed this **blocker** and put it #2 in its top-3 ("cheapest to fix now, expensive after"); sre-ops's COVERAGE STATEMENT (L323) explicitly declared the heartbeat **schema** clean ("No finding; this is a clean contract"). These are reconcilable, not a true dissent — ops cleared *field hygiene*, REL attacked *field scoping/authorship* — but the author should note that the lens owning the *consumer* (ops) did not see the authorship/scope problem the lens owning *delivery* (reliability) did, which is itself evidence the schema needs the explicit scope annotation (if the consumer-owner assumed the fields were unambiguous and they are not, that assumption ships a bug).
