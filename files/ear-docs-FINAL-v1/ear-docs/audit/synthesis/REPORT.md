# EAR System — Independent Steelman Audit · Orchestrator Synthesis (REPORT)

**Panel:** 7 independent expert sub-agents (RF/SIGINT · Data Engineering · Distributed Systems/Reliability · Security & Privacy · Test/QA · SRE/Operations · Software Architecture/PM), isolated in Phase 1; adversarial cross-examination in Phase 2; lead cross-document (pass 0), seam, and synthesis passes throughout. **Corpus:** `ear-docs-FINAL-v1` (41 files, D-001–D-061, Q-01–Q-25 + Q-T1–T4, 19 personas, runsheet S00–S21). **Nothing is built** — every finding below costs one edit now, not a rewrite later.

**Tally.** 89 reviewer findings + 11 lead-consistency + 7 lead-seam findings. Phase 2 cross-examined the 30 most contested/convergent into 37 verdicts: **0 killed, 10 survives, 14 softened, 13 hardened**. Zero findings were defeated by rebuttal — the panel's steelman-first discipline pre-empted the weak ones (each reviewer's coverage statement lists what it found *sound* and why). Cross-examination *sharpened* the survivors and produced several decisive corrections folded in below.

**Headline judgment.** This is an unusually rigorous pre-code design — the contract discipline (R0 change rules, machine-readable schemas as tie-breakers, fixture-as-CI-gate, decision-log-with-rationale, recovery-by-rebuild) is better than most *shipped* systems have, and much of it should be actively protected from build-time churn (§E). The defects are **not bad decisions**; they cluster into three classes: (1) **propagation debt** from three late edits (D-045 ephemeral uploader, D-052 canonical schema, D-055 the sixth repo / un-deferred Level 2) that updated their home docs but left contradictions in the older contracts and index; (2) **the guarantees' arithmetic and failure-granularity** (conservation, determinism, fault-isolation, idempotency keys) being weaker than their prose claims; and (3) **the primary target's survival chain having no owner** — the one finding larger than the sum of its parts. All are one-edit fixes in the current pre-code state; several become fixture-re-derivation-plus-pipeline-rewrite after build, which is the entire argument for fixing them now.

---

## A. TOP RISKS TO PROJECT SUCCESS (ranked; not down-ranked for fix size)

Ranked by threat to the project *succeeding at its mission*, not by finding severity. Each aggregates its backing findings (post-cross-exam verdict in brackets).

### A1 · The primary target can be silently lost, and both artifacts that appear to guard it don't. **(systemic — #1)**
*Backing: QA.16 [hardened, minor→major], REL.2 [hardened], QA.5 [survives], REL.7/SEAM-2 [hardened], the D-028/D-057 mechanism-correctness defect, lead-seams §D.1.*

The deep-sleep, power-saving tracker (G1, multi-hour-to-multi-day beacon — the 20595 class the system exists to find) must survive a chain of Level-1 and Level-2 filters. Cross-examination *verified at the classifier source* (`EVENT-CONSTRUCTION-METHODOLOGY-v1` L108–113) two things the authors cannot self-see:

1. **D-028's low-duty-cycle exception — the "NEVER-remove, CRITICAL" protection — does NOT fire for a multi-day-silent deep-sleeper.** The exception's first clause requires the device be *present in ≥ infra_hours* (≈ every hour). A device that beacons once every several hours-to-days is present in ~4 of 240 hours — nowhere near omnipresent — so it never reaches the exception branch. It survives **only** via `total ≥ MIN_OBSERVATIONS` + the ELSE branch. **D-028 and D-057/P15 both describe the protection mechanism incorrectly** — they credit the low-duty *presence* exception, which actually protects P2-class *hourly* beacons, not the deep-sleeper. This is a mechanism-correctness defect in the two most authoritative artifacts, not a documentation nit.
2. **`MIN_OBSERVATIONS`, the value that actually decides whether the deep-sleeper lives or dies, is unstated** everywhere (`(validated default)` in the methodology and PARAMETER-TUNING). A deep-sleeper's tiny total (≈12–16 observations) sits right at that unspecified floor.

The load-bearing survival links (cross-exam narrowed the filed "seven" to the ~4 that actually drop *this* device):
- **LINK 1 — eviction (REL.7/SEAM-2, hardened):** oldest-drop `pending/` eviction during a WAN outage can push a deep-sleeper's surviving obs-total below `MIN_OBSERVATIONS`, flipping its Level-1 verdict to `noise` — and it destroys the *first-seen* data that is a core scoring factor. A detection-survival choice masquerading as a retention policy.
- **LINK 5 — `MIN_OBSERVATIONS` floor, Level 1 (QA.16, hardened):** value unstated; the ELSE branch is the only thing keeping the deep-sleeper alive at prep.
- **LINK 6 — undefined rebuild window (REL.2, hardened):** "configured window" is defined nowhere; the taxonomy itself demands "long analysis windows," which no window parameter expresses. An ordinary days-scale window starves a multi-day-to-multi-week beacon even with perfect delivery.
- **LINK 7 — `min-events=3` floor, Level 2 (QA.5, survives):** a 4-event stream has 3 gaps, exactly at the floor; the classifier may refuse to compute CV and starve it — and **`P15→G1` is not even in the S19b gate line.**

*(Re-homed by cross-exam: RF.2 sweep-collapse and RF.3 600 s gap survive as real RF findings but are second-order to the obs-total for *this* device; REL.4 clock-drift keeps full force as a cross-sensor-correlation finding — SEAM-3 — not a deep-sleeper burst-split link.)*

**Why #1:** the entire system's value is catching this device, and it can be dropped silently at four independent points with no single artifact holding the chain, while the two artifacts that *look* like they guard it provably don't. **Recommended change:** (a) correct D-028's applicability caveat and P15's contract-proven wording to state the deep-sleeper survives via `MIN_OBSERVATIONS`+ELSE, *not* the presence exception; (b) set `MIN_OBSERVATIONS` to an explicit value derived from the smallest survivable deep-sleep total, with a boundary persona at `MIN_OBS ± 1`; (c) author a single **primary-target survival budget** doc tracing a deep-sleeper through LINKS 1/5/6/7 with a pinned value at each, and a **dual-gated end-to-end persona** (P15 at S16 *and* S19b, with `P15→G1` added to S19b) that fails if any link drops it; (d) define the rebuild window with an explicit long-horizon path for deep-sleepers.

### A2 · The agent emits records the loader rejects 100%, and the fixture would cement the wrong truth. **(pre-repo; cheap; corrosive)**
*Backing: LEAD.2 / DE.11 / QA.8 [all hardened; severity reconciled to major]; LEAD.4 / RF.4 [hardened] (presence_ratio); QA.2 [survives].*

The prose contract R1 says the scanner "emits exactly these fields: `time` … `direction` …"; the canonical v5 signal schema (D-052, declared the tie-breaker) requires `observed_at`, has no `direction`, and sets `additionalProperties:false`. An agent built to R1 emits `{time, direction, …}` → **every row fails schema validation → 100 % quarantine.** D-052's tie-breaker resolves only the `time`→`observed_at` *name*; it *escalates* the `direction`/`additionalProperties` half and leaves R1's ENFORCED-BY pointing at a filename (`observation.schema.json`) that does not exist. Worse, because the fixture *defines truth* (D-037/D-057), whichever name the S03 author types becomes law — the exact silent-contract-drift class the whole rules effort exists to prevent, now living *inside* the rules documents. Compounding: PARAMETER-TUNING lists `presence_ratio` as **0.8** (earnest-v3's *provably rejected* presence-only value, methodology L118/L138) misfiled into the "Analysis" layer, vs **0.95** everywhere else — and 0.8 vs 0.95 changes which *filter branch* a 90-minute-cadence tracker takes (infra_hours 20 vs 23), a second silent-loss path for the primary target.

**Recommended change:** reconcile R1's field list to the schema in one paired-change commit (R0.4): `time`→`observed_at`, resolve `direction` (drop or add to schema), fix `duty_cycle`/`observation_count` ownership (scanner-time vs edge-filter-time — LEAD.3), fix the ENFORCED-BY filename; add a CI check that the prose field lists are diffed against the JSON schemas so this class cannot recur. Correct PARAMETER-TUNING to 0.95 / prep-layer with a "0.8 was rejected — do not use" note; add persona P2b for the sub-95%-presence tracker.

### A3 · The conservation guarantee is provably wrong and halts every run — and one deterministic dirty row is a fleet-wide handoff outage. **(blocker)**
*Backing: DE.1 [hardened], RF.5 [softened→major], QA.7 [softened]; composed in SEAM-1.*

R5.1's post-load assertion `observations_delta == Σ ledger.row_counts` is under ENFORCED-BY, and D-044 maps a false assertion to a non-zero exit. But the ledger records `row_count` at *landing* (pre-quarantine), so the moment any row quarantines (a designed, expected path per D-021/P9), the delta is short by exactly `quarantined_count` → assertion false → run halts. Cross-exam *hardened* this: a **deterministic** dirty row (an uncastable value the enrichment rejects every time) makes every retry fail identically, so **the fleet produces no handoff at all until a human intervenes** — a data-availability outage, worse than the filed noisy page. The three conservation findings compose into one fix (DE.1's missing quarantine term is structurally required by RF.5's stage-order pin *and* QA.7's agent↔server composition): write one identity, `agent.raw == server.observations + server.quarantined + server.suppressed_summarized`, with an L4.5-forced EARFCN counted once and `quarantined ∩ summarized = ∅`, referenced by R5.1, the methodology, and P17 (extend GOLDEN-FIXTURE l.39). Make quarantine **exit-0-but-alert-on-ratio**, preserving fail-loud for genuine failures.

### A4 · Per-agent isolation is undone at the job level — one disposable file fails or loops the whole fleet. **(blocker)**
*Backing: DE.6 [survives], REL.6 [survives], REL.5 [softened].*

D-023 sells per-agent disposability as fault-containment ("a wedged file affects one agent and is disposable"). D-044's `compose run --rm` halt-on-failure *undoes* it: one pool worker's non-zero exit (a corrupt DuckDB file, an OOM, DE.1's assertion) fails the whole run, and the "next run retries" self-heal never fires for a *deterministic* fault — the fleet never produces a handoff. Cross-exam confirmed **D-044's rationale (clean memory, pinned image, fewer moving parts) contains no premise requiring whole-run halt — only run-to-completion.** Separately, REL.6's stale-lock predicate ("older than the timer period") cannot distinguish a crashed run from a healthy run that legitimately outlives the period at fleet scale; the disposability that makes recovery cheap *enables* a self-stealing rebuild loop. **Recommended change:** make the pool per-agent fault-isolating (quarantine the failing agent, record per-agent status on `AnalysisRun`, export the 59 healthy handoffs, exit non-zero so it still pages); after K consecutive same-agent failures, auto-disable + page (bounds the deterministic-poison loop); add a per-worker cgroup `--memory` cap (converts a whole-run OOM into a single-agent quarantine — DE.4/DE.6 disposition together); make the run-lock **liveness-based** (pid/heartbeat), not wall-clock.

### A5 · The flagship security control gives false assurance; the one gate ranked above all technical risk is un-enforced. **(blocker cluster)**
*Backing: SEC.3 [hardened], SEC.1 [survives], SEC.2 [softened], SEC.11 [softened], SEC.5/6/7.*

- **SEC.3 (hardened):** the onboarding gate (R5.0) authorizes *prefixes* but does **not authenticate object provenance.** Folder-authoritative identity (R5.1, "the R2 prefix wins") means an object *forged under an already-active prefix* — writable by anyone holding a stolen warehouse-PC's R2 credential — is ingested as authentic, satisfying every integrity check (etag/size, ledger, folder-identity). The design's flagship data-plane control gives *false assurance* against exactly this class. Because credential scope is unstated, the path of least resistance is one shared fleet-wide write cred → one stolen unattended warehouse PC compromises confidentiality *and* integrity of the entire multi-facility dataset.
- **SEC.1 (survives, split verdict):** the *legal conclusion* of Q-23 (workplace RF-interception lawfulness — Wiretap Act/ECPA/two-party-consent) is correctly a you-decision (`rationale_addressed`), but the finding's actual ask — *enforce* it with a blocking provisioning checkbox, per the design's own "no rule without a check" discipline — is unaddressed. It is the only deployment precondition the doc ranks above all technical risk, yet it is the one gate with no checkbox.
- **SEC.2 (softened):** the operator age-key has no offline escrow (key *loss* is unrecovered — you cannot re-render any config), and "rotation = re-encrypt + re-render" does not *rotate the underlying* R2/DB/ALERT secrets after a leak.

**Recommended change:** per-facility (or per-agent) scoped R2 tokens + object-lock/WORM on `obs_`/`ctx_` prefixes + read-only server pull cred (bounds a stolen-PC blast radius to one facility and makes R4's "never mutated" an *enforced* control); a producer-side batch-manifest signature (distinguishes authentic agent output from a forgery under a valid prefix); a blocking Q-23 checkbox in HOST-PROVISIONING and an M5 milestone gate; a KEY-MANAGEMENT contract (escrow + revocation-that-rotates-secrets); split WP-O3 so the *structural* security seams (credential model at M2, access/audit contract at M3, topology at M0) land with the surface they shape, leaving only deploy-time config at M5.

### A6 · The record-of-truth doc contradicts itself on the process topology, and the schema is already frozen. **(pre-repo)**
*Backing: LEAD.1 [survives], REL.1 [major], OPS.2/OPS.10 [survive].*

`LOCAL-HOST-DATA-FLOW-v2` says, in normative PRODUCER-GUARANTEE lines (not scene-setting), both "ONE shared uploader per host" and "its own uploader … its own dead-letter queue" (L133 "uploader, one per device"; L221 per-device DLQ), and the diagram says "4 independent uploaders." D-016/D-045/EAR-AGENT-SPEC say one shared. Cross-exam: no logged precedence rule covers a *topology* contradiction inside the build-spec contract (D-052's tie-breaker is scoped to *field* questions), so a builder reading L133 literally ships four uploaders + four DLQs. REL.1 softened from blocker to major (one oneshot *can* author four per-device heartbeats), but the pre-repo residual is real: the **frozen** heartbeat schema (D-052) has no `host`/`scope` field to disambiguate host-level `disk_pct` from per-device `queue_depth`/`dlq_count`, so the server mis-scopes one host event as four sensor events — and the consumer-owning lens (ops) declared the schema "clean," evidence it *reads* as unambiguous when it is not. **Recommended change:** one heartbeat-semantics ADR now (cardinality: one shared uploader; who authors the four heartbeats; per-device vs host-level field scope + a `scope` annotation on the frozen schema — pre-repo since frozen); stage the build-time halves (ctx_-based liveness, facility-rollup alert cardinality).

### A7 · "Determinism" is unachievable as written, contradicts the binding rule, and undermines the gate. **(blocker, doc-level)**
*Backing: QA.4 [hardened], QA.12 [softened], DE.9 [softened]; SEAM-9.*

GOLDEN-FIXTURE l.37 asserts "byte-identical events Parquet" across two runs — defeated by DuckDB row-ordering, Parquet metadata/compression, and the **per-row `run_id` requirement** (a UUID/timestamp per run makes byte-identity impossible *by construction*). Cross-exam hardened it: the impossible assertion sits in the *truth-defining* doc while the *binding* rule R7 (L202/L208) already uses the correct "identical event set" language — the two authoritative docs contradict, and the wrong one becomes law; at S17 the determinism test fails on first honest implementation and gets weakened under gate pressure, which then undermines Guy's independent re-run (QA.12). **Recommended change:** restate as **logical determinism** — identical event set after a canonical total sort, comparing all columns except `run_id`; if byte-identity is still wanted, additionally pin ORDER BY + codec+level+row-group + inject a constant `run_id` in test mode + pin the DuckDB/Arrow version. Reconcile GOLDEN-FIXTURE l.37 with R7 (paired change). *DE.9 (A/B reproducibility horizon) is a **separate** doc-gap, not sequenced behind this.*

### A8 · Pass-2 ships mislabeled "authoritative," and Q-05 — the spec for its stub's bias — is deferred. **(major)**
*Backing: QA.3 [survives], RF.7 [softened], lead-consistency LEAD interplay.*

From S15 through production the pipeline runs with Pass-2 as a stub, yet the handoff contract *unconditionally* labels `power_bin` "authoritative, post-boundary-reconciliation" (HANDOFF L30/L52/L70/L72) — through Guy's S18 human-oracle sign-off on the legacy corpus. P6's merge-guard is unfalsifiable under any stub. Cross-exam corrected RF.7's *prescription*: a no-op stub "because false-split is tolerable" **inverts Q-05**, under which false-split (missed tracker) is the operator's stated *worse* error → Pass-2 should lean *merge*. So the population-fold placeholder's uncontrolled-merge hazard is real (survives), but the fix is **resolve Q-05 first, then a boundary-scoped merge-leaning stub**, not a no-op. **Q-05 is blocking** — it is the spec for the stub's conservatism and cannot be deferred past the stub choice. Ride the honesty on the *data*: stamp `pass2=stub` in the handoff `dataset_version`/run-flag so no consumer (or Guy at S18) mistakes stub-binned data for reconciled; mark `power_bin` provisional while P5 is xfail (mirror the correlation-provisional discipline).

### A9 · Silent data loss of the highest-value data via an undecided eviction policy. **(major)**
*Backing: REL.7 [hardened], OPS.1, SEC.5; SEAM-2.*

The disk-ceiling eviction policy (L5 open item) is undecided ("oldest-drop vs alert-and-hold"). Oldest-drop silently destroys the *earliest* `pending/` observations during a WAN outage — which is both the un-reconstructable field data (reliability/security) *and* the first-seen signal the four-factor score most depends on (RF), *and* can push a deep-sleeper below `MIN_OBSERVATIONS` (gauntlet LINK 1). **Recommended change:** default to **alert-and-hold with scanner backpressure** (a *visible* capture gap beats invisible evidence destruction); if oldest-drop is unavoidable, tier it (drop `ctx_` first, `obs_` last, DLQ never) and emit a durable `capture_gap` context record; enforce the ceiling at *write* time in the scanner, not only at upload-run start; size `pending/` from Q-10 (hardware) × Q-14 (WAN) × the operator-availability window.

### A10 · The bus factor is 1 for both running *and building*, and the design tightens it. **(major, program-level)**
*Backing: OPS.1/4/7/15 [as filed], SEC.2; lead-seams §D.4.*

The architecture works hard to keep the *running* system alive without the operator, but: total-box-death is invisible (every watchdog layer lives *inside* the box — OPS.4); there is no external liveness beacon; there is no cold-return runbook or fleet-state artifact (OPS.7); the operator age-key is a personal-laptop SPOF whose loss is unrecoverable (SEC.2); and D-058's "no session completes without Guy's independent re-run" plus five un-delegable oracle points make the operator the strict *build* rate-limiter, so a gap pauses building entirely (OPS.15). **Recommended change:** an external dead-man's-switch (a Cloudflare Worker reading R2, or healthchecks.io) that alerts on *absence* of the server's ping — the only watchdog that catches total-box-death; a cold-return runbook + a rendered fleet-state artifact (deployed versions per facility, per-sensor baselines, key custody, last drill dates); age-key escrow; a named second contact for the data-loss alert class; honest acknowledgment that operator availability, not Opus throughput, sets the build pace.

### A11 · The core RF premise ("uplink-only") is asserted but not enforced by the chosen bands. **(major, deepest RF finding)**
*Backing: RF.1 [as filed, med confidence].*

The whole event model rests on undersampling an *uplink* channel where "gaps within a transmission are the sweep looking elsewhere, not signal absence." But the stated windows (662–850, 1709–1911 MHz) straddle 3GPP uplink *and* downlink sub-bands (e.g. 1805–1880 is Band 3 downlink). For any downlink carrier caught, the undersampling premise is false (a downlink carrier is genuinely continuous), and the `(facility, band_group, earfcn)` identity key mislabels a downlink EARFCN. The EARFCN table is never shown, so the premise cannot be verified from the docs — which is itself the finding. **Recommended change:** a band-plan appendix listing exact 3GPP uplink/downlink sub-ranges per window; either narrow the sweep to uplink-only blocks or rewrite the premise to handle downlink carriers (classify them G4 by frequency identity); make Q-13 a design question, not a confirm; tie the frequency→EARFCN map into `contracts/schemas/`.

### A12 · Further real risks (condensed; full detail in §D)
- **qa→active promotion folds onboarding-window data into production** (REL.8/DE.10/QA.11) — the rebuild is window-scoped, not status-scoped; add `active_since` and a promotion persona.
- **Idempotency keys on an unpinned etag** (DE.2) — multipart etag instability + non-deterministic gzip (REL.3) can re-ingest identical content; key on `(agent_id, filename)` + pin deterministic gzip + treat same-name-different-bytes as an integrity error.
- **Parquet partition atomicity is claimed at a granularity `rename(2)` doesn't provide** (DE.3) — two files per partition, `ENOTEMPTY`/`EXDEV`; adopt a `_SUCCESS` marker-commit protocol.
- **Cold-start corpus-poisoning loop** (RF.6/QA.6, lead-seams §D.3) — 3 of 4 G5 discriminators unavailable day one; the four-factor score fires high on benign telemetry; J3 walks the operator into confirming it; each confirmation poisons the corpus that would fix it. Add a warm-up doctrine + provisional taxonomy-label marking.
- **L2 result-persistence dual-write hole + no L2 runbook** (DE.7/OPS.9, SEAM-7) — transactional per-run replace + a `results_complete` flag + an L2 recovery clause.
- **Server-topology D-004 carries four unweighed cross-lens criteria** (SEC.8/OPS.8/REL.12/DE.3, SEAM-5) — decide it with the legal map, SPOF posture, egress budget, and filesystem-atomicity in hand.
- **DR/rollback drills never recur; no rollback procedure; mixed-version fleet safety implicit** (OPS.3/OPS.6) — cadence + a HealthSweep drill-freshness check + a rollback drill + a stated v5-uniformity release invariant.
- **Alert channel is a single unauthenticated ntfy topic** (SEC.9/REL.10/OPS.11) — auth tokens + a maintenance-window suppression primitive with auto-expiry.
- **R6 (`hours_present` — the origin bug), R8 (no-cross-band), R9 (uncalibrated formatter) have no *persona*** (QA.10) — their "unit test" ENFORCED-BY sits outside the regression floor that D-037/D-058 actually enforce; add P20/P21.

---

## B. CHANGE-BEFORE-REPO-CREATION (D-030-class) vs change-during-build

The review-disposition process (D-060) and the brief both ask structural pre-code calls to be separated. These are the ones that get *baked in* and are expensive to retrofit — apply them **before WP-D0 creates repos**:

**Pre-repo (structural / frozen-artifact / affects repo topology or the first commit's contracts):**
1. **Field-drift reconciliation** (A2 / LEAD.2·DE.11·QA.8) — the schema is authored in WP-D1 and imported by both suites; fix R1↔schema before S03 encodes a name as truth. *One major, cheap, corrosive-if-missed.*
2. **Heartbeat-semantics ADR + `scope` field** (A6 / LEAD.1·REL.1) — the heartbeat schema is **frozen** (D-052); adding a scope disambiguator is a pre-repo schema edit.
3. **Uploader/DLQ cardinality** (A6 / LEAD.1) — one shared vs four; it decides the run-lock, heartbeat authorship, disk-ceiling scope. D-016-class.
4. **Repo count / Q-01 / D-030 rationale** (LEAD.5) — Q-01 still omits `ear-analysis`; the repo-creation gate must list all six.
5. **Determinism restated as logical** (A7 / QA.4) — it changes the export code's contract and the meaning of P16/the gate; reconcile GOLDEN-FIXTURE l.37 ↔ R7 before the fixture is authored.
6. **Server topology D-004 / Q-04** (A / SEAM-5) — central-vs-per-facility with the legal, SPOF, egress, and filesystem-atomicity criteria; "before M3 deploy shape," cheapest at M0.
7. **CLAUDE-DECIDED review scope** (LEAD.7) — Q-18 must cover D-030–D-061 (esp. D-042/D-050/D-052/D-061), not stop at D-041, or the schema/security decisions never get the operator's pre-repo veto.
8. **Stale contract cross-references** (LEAD.6) — `LOCAL-HOST-DATA-FLOW-v2` points at the *archived* `DATA-WORKFLOW-RULES-v1`; a build agent could code to superseded rules. Add a doc-lint that intra-repo links resolve to non-archive files.
9. **The credential-model decision** (A5 / SEC.3) — per-facility scoped R2 tokens shape how the M2 agent uploader and R2 tokens are built; deciding it post-hoc reworks the upload path.
10. **`MIN_OBSERVATIONS` value + the D-028/P15 mechanism correction** (A1) — the fixture P15 derivation (S04, double-derived, Guy-reviewed) encodes the deep-sleeper's survival math; the mechanism must be right *before* the derivation is committed as law.
11. **Q-05 (false-split vs false-merge)** (A8) — it is the spec for the Pass-2 stub's bias; the stub is chosen at S15 but the fixture P5 and the handoff "authoritative" wording depend on the answer now.
12. **The stage-order pin + conservation identity** (A3 / RF.5·DE.1·QA.7) — the fixture expected-outputs (P2/P3/P8/P15/P17) and the whole-run conservation assertion are all order- and term-dependent; pinning them post-fixture forces a full re-derivation.

**Change-during-build (localized; the fixture is the regression guard):** the fault-model per-agent isolation (A4, WP-S4), the eviction policy (A9, WP-A3), determinism export mechanics (A7 impl, WP-S9), Pass-2 stub honesty flags (A8 impl, WP-S6), idempotency-key/gzip-determinism (A12, WP-A4/S3), Parquet marker-commit (A12, WP-S9), L2 persistence transaction (A12, WP-L2), qa-promotion `active_since` (A12, WP-S2), DR/rollback/alert-channel/beacon (A10, WP-O1/O2/O3), the split of WP-O3 security (A5). Per D-060, fold each accepted item into the contract + a new/amended decision + a new persona *in the same commit*, and re-run only the affected reviewer lens on structural changes.

---

## C. OVER-CONFIDENCE AUDIT (the primary deliverable — where the docs claim more certainty than the evidence supports)

The authors *cannot* self-detect these; they are the places the prose asserts as settled what is actually assumed, provisional, or contradicted.

| # | Claim as written | Reality | Finding |
|---|---|---|---|
| C1 | "The HackRF captures **only** the uplink channel" (methodology, stated as physics) | The chosen MHz windows straddle 3GPP uplink *and* downlink; the premise is unverified and the EARFCN table is never shown | RF.1 |
| C2 | Low-duty-cycle exception is the "CRITICAL, NEVER-remove" protection for the primary target (D-028); P15 "guards the min-obs/min-events starvation" (D-057) | Verified at classifier source: the exception does **not** fire for a multi-day-silent deep-sleeper; it survives only via an *unstated* `MIN_OBSERVATIONS` — the two most authoritative artifacts misdescribe the protection mechanism | QA.16, A1 |
| C3 | Burst gap 600 s is **SETTLED** (methodology l.158) | The per-sub-pattern cadences (Q-T1) needed to defend 600 s are *unknown*; it is validated for one use case, applied across a class spanning 60 s–multi-day | RF.3 |
| C4 | Sweep-collapse 2 s (baked into P1's hard expected-output) | The window "should tie to sweep revisit time" (Q-06, open) and is band/bin_width-dependent; a hard assertion built on an open parameter | RF.2, QA.15 |
| C5 | `power_bin` is "**Authoritative**, post-boundary-reconciliation" (HANDOFF, unconditional) | It is the Pass-1 advisory bin for the entire build while Pass-2 is a stub; labeled authoritative through Guy's S18 sign-off | QA.3 |
| C6 | "Two runs produce **byte-identical** events Parquet" (GOLDEN-FIXTURE, the truth doc) | Physically unachievable; contradicts the per-row `run_id` requirement and the binding R7's own "identical event set" | QA.4 |
| C7 | Post-load assertion `observations_delta == Σ ledger row_counts` (R5.1 ENFORCED-BY) | Arithmetically wrong the instant any row quarantines; halts every dirty-row run | DE.1 |
| C8 | Per-agent isolation: "a wedged file affects one agent and is disposable" (D-023) | Undone by D-044 whole-run halt; one deterministic-poison file = fleet-wide handoff outage | DE.6 |
| C9 | "Peak footprint = pool size × memory_limit (**the only two** scaling tunables)" (RFANALYSIS-SCOPE) | `memory_limit` is a soft buffer target, not an RSS cap; a lower bound, not the peak; per-worker overhead un-budgeted | DE.4 |
| C10 | Atomic partitioned Parquet; "partial export impossible to observe" (D-024/RECOVERY) | `rename(2)` is per-path; two files per partition + `ENOTEMPTY`/`EXDEV` expose torn/half-written partitions | DE.3 |
| C11 | Idempotency keyed on "filename + size/etag"; "re-ingest is idempotent by file identity" | "File identity" is never pinned; multipart etag + non-deterministic gzip perturb it → silent double-ingest | DE.2, REL.3 |
| C12 | Onboarding gate = the access-control story; "a new/misconfigured agent cannot silently enter production" | It authorizes prefixes but does **not authenticate provenance**; forgery under an active prefix is ingested as authentic | SEC.3 |
| C13 | "Everything in ear-docs; the decision log mitigates the single-maintainer bus factor" (risk register) | The log preserves *why*, not the *running state* or *key custody*; and the *build* bus-factor is also 1, tightened by D-058 | OPS.7, OPS.15 |
| C14 | Health self-monitoring: "watchdog for the watchdog, cheaply" (HEALTH-CHECKS) | Every watchdog layer lives *inside* the box; total-box-death is invisible — the failure that most needs an alert generates none | OPS.4 |
| C15 | "The configured window" (R7, referenced as if defined) | Defined nowhere; no `window` param exists; late/multi-day-old data lands but is never grouped | REL.2 |
| C16 | Q-22 fleet-volume deferral is safe ("not a concern at this stage") | The *fleet-scale gate* deferral holds, but three cheap non-scale probes (max run-time, heavy-agent RSS, a concurrency persona) are entangled and silently postponed; the stale-lock and Q-19 cadence are set blind without them | REL.6, DE.4, QA.13 |
| C17 | `presence_ratio` (PARAMETER-TUNING = 0.8, "Analysis" layer) | 0.8 is earnest-v3's *provably rejected* value, misfiled into the wrong layer, on the A/B surface; 0.95 everywhere else | LEAD.4, RF.4 |
| C18 | Cold-start classification ships in the initial dashboard with no degraded caveat (D-055) | 3 of 4 G5 discriminators are unavailable day one; the score fires high on benign telemetry and poisons the seed corpus | RF.6 |
| C19 | Index/front-matter: "60 decisions, D-001..D-059, 22+4 questions, Q-01…Q-18" | Actual: D-061, Q-25 (+Q-T1–4); the legal gate Q-23 and the schema/licensing decisions fall outside the stated ranges | LEAD.7, LEAD.8 |
| C20 | "5-repo topology" / Q-01's "five names" (README, Q-01) after D-055 created the sixth repo | Six repos; the repo-creation gate silently drops `ear-analysis` | LEAD.5 |

---

## D. FULL FINDINGS TABLE (all surviving findings by severity; nothing omitted)

Severity is post-cross-exam where adjudicated. **Verdict key:** H=hardened, S=survives, s=softened, r=rationale_addressed, ⊕=lead finding. Findings without a Phase-2 verdict stand as filed (all are grep-cited and steelman-anchored; each reviewer's coverage statement records what it found sound).

### Blocker
| ID | Target | One-line | V |
|---|---|---|---|
| A1/QA.16 | MIN_OBSERVATIONS + D-028/P15 mechanism | Deep-sleeper survives only via an unstated floor; the two guarding artifacts misdescribe the mechanism (minor→**major/blocker-cluster**) | H |
| DE.1 | R5.1 post-load assertion | Conservation assertion wrong under quarantine → deterministic poison = fleet-wide handoff outage | H |
| DE.2 | R4/R5 ledger etag key | Unpinned multipart-etag/gzip → silent double-ingest inflates all counts | S |
| DE.6 | D-044 vs D-023 | Whole-run halt undoes per-agent isolation; deterministic fault never self-heals | S |
| RF.1 | uplink-only premise vs band plan | Core undersampling premise unenforced by the chosen (uplink+downlink) MHz windows | — |
| RF.2 | sweep-collapse window | One global 2 s window can't fit two bands' revisit times; manufactures false periodicity | s(scope) |
| REL.1 | uploader cardinality vs frozen heartbeat schema | Contradiction; schema has no scope field to disambiguate host vs device (blocker→**major**) | s |
| REL.2 | undefined rebuild window | Starves multi-day/week deep-sleeper evidence even with perfect delivery | H |
| SEC.1 | Q-23 legal gate un-enforced | The one gate above all technical risk has no checkbox (legal conclusion is a you-decision) | S |
| SEC.2 | operator age-key | No escrow (loss unrecovered); rotation doesn't rotate underlying secrets | s |
| SEC.3 | R2 credential scope + provenance | Gate authorizes prefix, not provenance; stolen-PC forgery ingested as authentic | H |
| SEC.4 | dashboard/admin authZ | Access control assumed, never specified; no per-facility human boundary for Q-16 | — |
| QA.1 | P7 feed-split guard | Asserts an absence → proves nothing; the most-protected decision has the weakest test | — |
| QA.4 | byte-identical determinism | Unachievable; contradicts run_id and R7; wrong doc is law | H |
| QA.5 | P15 min-events (L2) | 4-event/3-gap deep-sleeper starved at L2; P15→G1 absent from S19b gate | S |
| OPS.1 | unstaffed on-call | Single part-time human is the sole backstop for data-loss-imminent failures | — |
| LEAD.2⊕ | R1 fields vs v5 schema | Agent emits `time`+`direction` → 100% quarantine; fixture cements wrong truth | H |

### Major
| ID | Target | One-line | V |
|---|---|---|---|
| RF.3 | 600 s burst gap | One value across a 60 s–multi-day cadence class; SETTLED while Q-T1 open | s |
| RF.4 | presence_ratio 0.95/0.8 | Doc disagreement changes the primary target's filter branch | H |
| RF.5 | stage-order | Methodology numbered list contradicts its own prose; noise-count basis unstated → P15 order-dependent | H |
| RF.6 | cold-start G1-vs-G5 | 3/4 discriminators unavailable day one; over-claims; poisons seed corpus | — |
| RF.7 | Pass-2 stub bias | Population-fold placeholder can false-merge; no-op prescription inverts Q-05 → resolve Q-05 first | s |
| DE.3 | Parquet rename atomicity | Directory rename not atomic (ENOTEMPTY/EXDEV); torn multi-file partition | S |
| DE.4 | memory_limit ≠ RSS | "Only two tunables" is a lower bound; couples to DE.6 OOM→whole-run halt | s |
| DE.5 | full-rebuild window cost | Unbounded rebuild as windows grow; late arrival forces multi-day passes every run | — |
| DE.7 | L2 DuckDB→Postgres write | Unguarded dual-write; partial results render as complete; no run-supersession | — |
| REL.3 | agent crash / gzip determinism | PUT-confirmed-then-crash re-PUT with different etag; dedup ambiguous | S |
| REL.4 | clock drift vs 600 s / correlation | Drift corrupts cross-sensor correlation (ships now, D-055) before the time-sync contract exists (SEAM-3) | s(re-homed) |
| REL.5 | uploader poison starvation | Halt-on-failure asymmetry with D-044; one poison file could block the batch | s |
| REL.6 | wall-clock stale-lock | Can steal the lock from a live long run → self-stealing rebuild loop; must be liveness-based | S |
| REL.7 | disk-ceiling eviction | Oldest-drop destroys first-seen + can push deep-sleeper to `noise` (SEAM-2) | H |
| REL.8 | qa→active promotion | Window-scoped rebuild folds onboarding-window data into production | — |
| SEC.5 | warehouse-PC physical compromise | Unencrypted disk + creds + un-uploaded data; no de-provision/revocation | — |
| SEC.6 | at-rest encryption | No FDE stated for dev/prod box holding Postgres+Parquet+key | s |
| SEC.7 | no access/export audit | Lineage audit exists; access audit (who viewed/exported surveillance) does not | — |
| SEC.11 | WP-O3 sequencing | Structural security seams inverted vs the M5 security milestone → split WP-O3 | s |
| QA.2 | duty-cycle boundary persona | No persona at the exception/infra boundary; timeframe-vs-effective_hours inconsistency | S |
| QA.3 | Pass-2 handoff labeling | Stub-binned power_bin unconditionally labeled authoritative through S18 | S |
| QA.7 | conservation composition | Agent/server equations never shown to compose; L4.5-vs-FILTER precedence unpinned | s |
| QA.9 | dataset_version placement | Whole-run assertion says per-row; schema says file-metadata | — |
| QA.10 | R6/R8/R9 no persona | Origin bug + no-cross-band + uncalibrated-formatter guarded only by non-persona unit tests | — |
| OPS.2 | alert cardinality | WAN-outage vs single-sensor indistinguishable at the notification | S |
| OPS.3 | DR drills no cadence/owner | Proven once at build, then rot silently | — |
| OPS.4 | watchdog inside the box | Total-box-death is invisible; no external beacon | — |
| OPS.5 | onboarding load O(sensors) | "Repeat WP-O2 only" hides per-sensor manual promotion at 28–60/facility | — |
| OPS.6 | no rollback / mixed-version | No rollback procedure; version-string canary ≠ behavior; mixed-version safety implicit | — |
| OPS.7 | 6-month-gap state | Log preserves why, not running state or key custody; no cold-return runbook | — |
| OPS.8 | dev-box SPOF | Session-close mirror window; prod-on-dev-box (Q-11) compounds; single NVMe | — |
| OPS.9 | L2 ops surface | Second job/failure class with no L2 recovery runbook or unclassified-queue cadence | — |
| DE.10⊕/REL.8 | qa isolation path | Filter-based vs path-structural isolation ambiguous | s |
| LEAD.1⊕ | uploader cardinality | Doc self-contradiction in normative lines (see A6) | S |
| LEAD.3⊕ | duty_cycle/obs_count ownership | Capture-time (R1) vs edge-time (L1) contradiction | — |
| LEAD.4⊕ | presence_ratio 0.8 | Rejected value in the tuning doc | H |
| LEAD.5⊕ | repo count five/six | Q-01 drops ear-analysis | — |
| LEAD.6⊕ | stale contract xref | Live doc points at archived v1 | — |
| LEAD.7⊕ | CLAUDE-DECIDED scope | Three ranges; Q-18 stops at D-041 | — |
| LEAD.9⊕ | R8/R9 L1/L2 ownership | "Level 1" doc specifies L2 activities; unowned after D-055 | — |
| SEAM-1⊕ | conservation triple-fix | DE.1+RF.5+QA.7 compose into one identity | H |
| SEAM-2⊕ | eviction destroys detection signal | RF consequence of an ops/security policy | H |
| SEAM-3⊕ | correlation ships before time-sync | D-055 moved a dependency across WP-O3's line | — |
| SEAM-5⊕ | D-004 four unweighed criteria | Topology decision more loaded than framed | — |
| D.1⊕ | primary-target gauntlet | No artifact holds the survival chain (systemic) | H |
| D.4⊕ | build+run bus-factor=1 | Tightened by D-058 + key SPOF | — |

### Minor (condensed — all filed, none omitted)
RF.8 (dead −110 refs + floor-comparability), RF.9 (power-variance→stationary over-claim), RF.10 (assumed 10 s transmission duration); DE.8 (open/close per-stage overhead + "parallelism"=pool-size), DE.9 [s] (A/B 90-day horizon), DE.11 [H, minor→major] (field-drift, folded to A2), DE.12 (LIST eventual-consistency); REL.9 (LIST consistency, subsumed by REL.2), REL.10 (single-channel alerting SPOF), REL.11 (`processing/` crash-recovery still OPEN), REL.12 (partial server↔R2 fetch → partial-but-green); SEC.8 [s] (D-004 isolation), SEC.9 (ntfy metadata side-channel), SEC.10 (unsigned update channel/no hash-pinning); QA.11 (P13 config-drift/P18 promotion under-constrained), QA.12 [s] (gate re-run vs determinism), QA.13 [s] (concurrency-correctness persona), QA.14 (double-derivation misses accounting personas), QA.15 (P1 parameter-dependent), QA.16 [H→major, folded to A1]; OPS.10 [S] (capture-advancing on obs_ false-reds quiet target), OPS.11 (no maintenance-window suppression), OPS.12 (secret rotation/scope/revocation), OPS.13 (Q-19 cadence coupling), OPS.14 (kill-WAN first-host-only), OPS.15 (build rate-limiter); LEAD.8 (stale counts), LEAD.10 (retention 13-mo not in Q-15), LEAD.11 (min-events seam, folded to A1); SEAM-4⊕ (disposability = strength + loop, hold both), SEAM-6⊕ (best decision, weakest test), SEAM-7⊕ (L2 persistence two-halves), SEAM-8⊕ (heartbeat over-loaded), SEAM-9⊕ (reproducibility = two work items), D.3⊕ (cold-start poisoning loop). **Arch/PM lens** (arch-pm.md) findings on runsheet parallelism realism, Opus/Sonnet assignment, fresh-Pegasus donor-cost, 6-repo coordination, and calendar realism are filed in full in the reviewer file and disposition alongside OPS.15/D.4.

---

## E. WHAT TO PROTECT (guard against improvement-churn during build)

Every reviewer independently converged on a protect-list; these are the choices strong enough that a well-meaning build-time "simplification" would make the system *worse*. Do not let them drift:

- **The feed split (D-012)** — makes the false-CRITICAL bug *structurally unrepresentable* (`ctx_` rows the event path never reads), not merely forbidden. The single most-praised decision (all 7 lenses). *But make P7 actually prove it — QA.1.*
- **The low-duty-cycle exception (D-028) as a NEVER-remove** — correct for P2-class hourly beacons. *Protect it — but correct its stated applicability: it does not protect deep-sleepers (A1).*
- **Two-table ingest + persistent raw landing (D-021)** — the safety-net/quarantine/reprocess backbone; do not collapse it back to single-pass.
- **Full-rebuild-per-run + config snapshot (R7, PARAMETER-TUNING)** — determinism-by-construction eliminates a whole incremental-merge bug class and enables honest A/B. Protect the *principle*; bound the *cost* (DE.5).
- **Per-agent disposable L1 + physically-enforced L1/L2 handoff boundary (D-023/D-024/D-029)** — recovery falls out of the architecture; L2's DB *cannot* see L1 tables. Protect the isolation and the physical boundary; fix the fault-*granularity* (A4), not the structure.
- **The two-pass *structure* (D-027)** — cheap-coarse + expensive-on-boundary is the correct tractability answer; only the *calc* and *stub bias* are open.
- **Schemas as tie-breaker + fixture-first + fixture-as-CI-gate (D-052/D-039/D-037)** — machine-readable enforcement over prose; the reason every finding here is a one-edit fix. Protect absolutely; use it to auto-catch drift (A2).
- **The onboarding gate (R5.0)** — auto-discovery ≠ auto-production, structurally enforced. Protect the gate; add the *human*-plane analog (SEC.4) and pin qa-path isolation (REL.8).
- **`pending/` as the retry queue; circuit breaker retired (D-045)** — deletes the watcher/breaker bug class. Protect the oneshot-timer model; fix its edges (heartbeat authorship, poison isolation, run-lock).
- **Recovery-by-rebuild ("no recovery ever edits in place"), staged fleet updates, health-from-real-signals, state-change alerting** — all correct instincts; the findings refine their *edges*, never their cores.
- **The seam-honesty discipline (P5 xfail; provisional correlation marking) and the verification protocol (D-058: "agent claims are not evidence, reports are" + regression floor)** — exemplary; *extend* the discipline to the taxonomy labels and Pass-2 data-marking, and repair the determinism dependency.

---

## F. DISSENTS (unresolved; for the operator to adjudicate)

1. **Q-22 fleet-volume deferral — reviewers vs the operator's explicit call.** *Operator's steelman:* the full fleet-scale replay gate (60 sensors × multi-day × speed factor) is legitimately not a pre-first-facility concern; first facility is one small site, D-047 sizes RAM with ~3.5× margin, and building the harness now models load that doesn't exist. *Reviewers' steelman:* three *cheap, non-scale* probes are entangled in the deferral and get silently postponed — max whole-run wall-clock (REL.6: the stale-lock predicate and Q-19 cadence are set blind without it), heavy-agent peak RSS (DE.4), and a fixture-scale concurrency persona (QA.13, needs no scale run at all). **Panel's recommended resolution (operator decides):** keep the fleet-scale *gate* deferred; split the Q-22 wording so the narrow probe set lands before first facility, mostly inside already-scheduled work — widen WP-S5/S15 to emit wall-clock + peak-RSS-at-`memory_limit` for the heaviest agent; add one concurrency persona; and apply the decision-only doc fixes (liveness stale-lock, delete "the only two tunables," per-worker cgroup cap) that need no measurement.
2. **Quarantine halting (DE.1 fix) vs fail-loud (D-044 / SRE reliance on non-zero-exit paging).** Resolvable, but it is a genuine design decision: quarantine should exit 0 and alert on a *ratio* threshold, preserving fail-loud for genuine failures. The operator should confirm the ratio and that quarantine is not a run failure.
3. **Pass-2 stub conservatism depends on Q-05, which is unanswered.** RF.7's no-op prescription inverts the operator's stated worse-error (false-split). The operator must either uphold Q-05 (stub leans merge via a boundary-scoped rule) or consciously invert it (then a split-leaning stub is vindicated). Either way **Q-05 is blocking** for the stub choice.
4. **Security scope vs single-operator simplicity.** SEC.2's per-facility *age keys* vs SEC.3's scoped *R2 tokens* as competing blast-radius controls: the panel adjudicates **SEC.3 wins** (token scoping bounds a leaked key's R2 value without multiplying age keys, preserving D-050's simplicity); SEC.2 should drop its per-facility-key sub-proposal but keep escrow + revocation-that-rotates. The residual dissent for the operator: is scoped-tokens + object-lock + a manifest signature worth the (small) added ceremony? The panel says yes, decisively, given one stolen unattended warehouse PC otherwise compromises the whole fleet.

*No genuine cross-lens dissent exists inside the conservation, determinism, or heartbeat clusters — those reviewers are additive, not opposed. The above four are the only unresolved adjudications.*

---

## G. COVERAGE MATRIX (per reviewer, surface examined — so nothing looks skipped)

| Lens | Surface examined | Findings | Found sound (protected) |
|---|---|---|---|
| **RF/SIGINT** | Undersampling model, band plan, freq→EARFCN, noise floor, two-pass binning + Pass-2 calc, sweep-collapse vs revisit, event=burst/600 s, noise filter (0.95/40×/low-duty/min-obs/safeguard), G1–G6 + cold-start, four-factor score, power/proximity, cross-doc numeric/order; Q-03/05/06/07/13, Q-T1–T4 | 10 (2 blk, 5 maj, 3 min) | undersampling *insight*, two-scale separation, low-duty exception, feed split, calibration discipline, server-owned binning |
| **Data Eng** | Two-table ingest/quarantine/ledger, per-agent DuckDB (open/close, mem×pool), Parquet atomicity/partition/dataset_version, ephemeral job, DuckDB↔Postgres, feature extraction, retention, 4 schemas; P5/9/10/12/16/17; Q-15/19/22 | 12 (2 blk, 5 maj, 5 min) | two-table ingest, full-rebuild+snapshot, per-agent+physical handoff boundary, schemas-as-tie-breaker, ephemeral model, onboarding gate |
| **Reliability** | Store-and-forward, R2-primary, ledger idempotency, recovery-by-rebuild, late/out-of-order, WAN partition, clock skew, run lock, delivery semantics, disk ceiling, heartbeat freshness; D-002/003/015/016/017/023/024/044/045/049/051/052/057/059; Q-09/14/19/22 | 12 (2 blk, 6 maj, 4 min) | D-045 core, D-023/024 disposability+atomic export, same-tx land+ledger, R5.0 gate, feed split, health-from-real-signals |
| **Security** | Surveillance sensitivity, sops/age, R2 credential blast radius, dashboard/admin authZ, audit, at-rest, facility isolation, physical compromise, legal/privacy (Q-23), update integrity; D-002/004/022/036/048/049/050/051/055/056/061; Q-02/12/21/23 | 11 (4 blk, 5 maj, 2 min) | R5.0 gate, private/proprietary posture, sops/age as tool, physical L1/L2 isolation, retention-as-config |
| **Test/QA** | All 19 personas (mechanism-vs-output), whole-run assertions (determinism/conservation/conformance/isolation), double-derivation, gate protocol D-058, UAT J1–J7, P5 xfail seam, Q-22; every R-clause for persona coverage | 16 (4 blk, 5 maj, 7 min) | P9/P11 (mechanism-pinning), seam-honesty discipline, D-058 + regression floor, fixture-first, feed split, physical L1/L2 boundary |
| **SRE/Ops** | Health checks (4 layers/10-check sweep/self-monitoring), alerting (state-change/ntfy), runbook completeness, backup/DR + drills, fleet updates (staged), provisioning, single-operator load, 6-month-gap; D-015/044/045/048/049/050/051/059; Q-09/10/14/15/19/21/22/23/24 | 15 (1 blk, 8 maj, 6 min) | recovery-by-rebuild, health-from-real-work, pending-as-retry-queue, state-change alerting, feed split, onboarding gate, sops/age, DR-drill design |
| **Arch/PM** | 6-repo topology, contract-as-source-of-truth + paired-change, runsheet realism (S00–S21 + sub-sessions, parallelism, ASK/DEFAULT gates), Opus/Sonnet assignment, decision-log/versioning discipline, fresh-Pegasus bet (D-032), D-042/D-060, buildability of the seams | 13 (as filed) | contract-source-of-truth (D-033), fixture-first, decision-log discipline, seam-swappability, disposition process (D-060) |
| **Lead (pass 0)** | All 41 files for inter-document consistency: field sets, uploader cardinality, presence-ratio, repo count, contract xrefs, CLAUDE-DECIDED range, counts, retention, R8/R9 ownership | 11 (7 maj, 4 min) | identity/regex, 600 s, feed-split namespace, schema_version=v5, two-pass lock (all consistent) |
| **Lead (seams §B–D)** | Cross-domain contradictions, uncovered seams, emergent/systemic risks, panel meta-review | 7 seams + 4 systemic | — |

**Panel integrity:** all seven exhausted their surface (no re-dispatch; each ≥7.5k words with an explicit coverage statement). Isolation held (one immaterial 2-line incidental grep-preview, self-reported, unused). Every REVIEW-BRIEF-flagged weak spot was extended, not merely re-raised. Cross-examination defeated **zero** findings and hardened 13 — the surviving finding set is high-confidence.

---

*Prepared for the D-060 review-disposition process. Reviewer files: `audit/reviewers/*.md`. Rebuttals: `audit/rebuttals/*.md`. Lead passes: `audit/synthesis/lead-consistency.md`, `lead-seams.md`, `load-bearing-decisions.md`. Every finding carries severity, steelman, failure_scenario, proposed_alternative, downstream_impact, and (where cross-examined) a verdict — ready to triage into ACCEPT / ACCEPT-DEFER / REJECT / NEEDS-INFO per D-060.*
