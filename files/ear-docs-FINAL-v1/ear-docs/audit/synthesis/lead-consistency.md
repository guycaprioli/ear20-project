# Lead Consistency Review — Pass 0 (whole-system vantage)

**Reviewer:** Lead Orchestrator (Fable). **Scope:** the seam no single-lens reviewer owns — inter-document contradictions, decision-vs-contract mismatches, stale/broken cross-references, numbers that disagree across files, and claims in one doc undermined by another. This is a *review*, not orientation: every item below is a defect in the document set as it stands, found by holding all 41 files at once.

**Method.** Read the full authoritative set (contracts, components, plan, operations, schemas, decision log, open questions, both review-prompt variants; archive/ read for cross-reference validity only). Every finding is grep-verified with file:line citations. Findings use the panel schema. Severity is calibrated to *pre-code cost of not fixing it*: a contradiction a build agent would resolve wrongly is `major`; a stale count is `minor`.

**Headline.** The design's *content* is unusually rigorous and internally reasoned. The defects here are almost entirely **alignment debt from three late edits that were not fully propagated**: (1) D-045's shift to an ephemeral single uploader, (2) D-052's canonical v5 schema, and (3) D-055's un-deferral of `ear-analysis` (the 6th repo / Level 2). Each edit updated its home doc and the decision log but left stale language in the older contract/index docs. That pattern is itself the finding: the docs claim "contracts are the law / schemas are the tie-breaker," but the law currently disagrees with itself in ways a build agent (or the operator confirming Q-01/Q-18) would resolve wrongly.

---

FINDING-LEAD.1 · severity: **major** · target: `contracts/LOCAL-HOST-DATA-FLOW-v2.md` §Deployment shape (L11, L26 diagram) vs D-016 / D-045 / `components/EAR-AGENT-SPEC-v1.md`
steelman: The doc was consciously "corrected against code v4.3.0" and D-016 notes "Docs corrected from an earlier 4×2 claim," so the intent to standardize on one shared ephemeral uploader is clear and logged; a reader who trusts the decision log gets the right model.
failure_scenario: A WP-A3/WP-A4 build agent reads the *contract* (the bootstrap tells it to read "the contracts your WP touches"), not just the decision log. LOCAL-HOST-DATA-FLOW-v2 L11 says in one breath "**plus ONE shared EPHEMERAL uploader per host (D-045)**" and "each device has **its own** scanner process, its own local folders, **its own uploader, and its own dead-letter queue**"; the ASCII diagram caption (L26) says "**(4 independent uploaders per host)**" and each pipeline box shows its own `uploader → edge filter → DLQ`. The agent builds four per-device uploader timers + four DLQs — directly contradicting D-016/D-045/EAR-AGENT-SPEC ("1 shared uploader + 1 shared DLQ per host"). This is not cosmetic: the cardinality decides whether a run lock is needed (4 uploaders racing the same host disk-ceiling check), who authors the four `_health/{agent_id}.json` heartbeats (one uploader writing 4 vs four writing 1 each), and how the disk-ceiling/eviction policy is scoped (per-device vs per-host).
evidence: `LOCAL-HOST-DATA-FLOW-v2.md:11` (both claims in one sentence), `:26` ("4 independent uploaders per host"), `:21-23` (per-pipeline uploader/DLQ boxes) vs `DECISION-LOG-v1.md:19` (D-016 "1 shared uploader + 1 shared DLQ") and `:54` (D-045 "1 ephemeral uploader … scans all four devices' folders") and `components/EAR-AGENT-SPEC-v1.md:8` ("1 ephemeral uploader … 1 shared DLQ").
proposed_alternative: Rewrite L11 and the ASCII diagram of LOCAL-HOST-DATA-FLOW-v2 to a single shared ephemeral uploader + single shared DLQ; delete "its own uploader, and its own dead-letter queue"; redraw the diagram as 4 scanners → 1 uploader (fan-in at upload) → R2. Add an explicit line: "one uploader run writes all four sensors' heartbeats." Bump the doc version. Add/extend a fixture assertion that the host emits exactly one uploader-run heartbeat set covering 4 agent_ids.
downstream_impact: D-016, D-045, EAR-AGENT-SPEC, HEALTH-CHECKS (heartbeat-per-active-sensor semantics), HOST-PROVISIONING-CHECKLIST ("_health/{agent_id}.json fresh for all four"), the run-lock question (reliability reviewer), disk-ceiling scoping (SRE reviewer).
confidence: high

---

FINDING-LEAD.2 · severity: **major** · target: `contracts/DATA-WORKFLOW-RULES-v3.md` R1 (L43) & `contracts/LOCAL-HOST-DATA-FLOW-v2.md` L1 (L93,L102) vs `contracts/schemas/observation-signal.v5.schema.json` (D-052)
steelman: D-052 explicitly declares the schemas "the tie-breaker over prose on field questions," so the *intended* canonical record is the JSON Schema and any conflicting prose is knowingly subordinate; a disciplined agent resolves the conflict correctly by rule.
failure_scenario: The prose contracts and the canonical schema disagree on the record's actual fields in three independent ways, and the prose is what R1's "ENFORCED BY … shared `observation.schema.json`" points a builder at:
  (a) **Field name `time` vs `observed_at`.** R1 L43 "Emits exactly these fields: `time`, …"; L1 L93 "stamps: `time` (ISO-8601 UTC …)". The signal schema requires `observed_at` and D-052 says "Legacy names (`time`, `power`, `agent`) are NOT valid v5." So the two contracts mandate a field name the canonical schema *rejects*.
  (b) **`direction` field.** R1 L43 lists `direction`; L1 L93 stamps `direction: 'ul'`; L1 L102 "checks … direction ∈ {ul,dl}". The signal schema has **no** `direction` property and sets `additionalProperties:false` — a row carrying `direction` fails validation outright.
  (c) Net effect: an agent built to R1/L1 emits `{time, direction, …}` rows; the loader validating against `observation-signal.v5.schema.json` quarantines **100%** of them (wrong name for the timestamp + a forbidden extra property). The fixture (P9 "bad timestamp") and the round-trip personas would encode whichever the generator author happened to pick, cementing the wrong answer as "truth."
evidence: `DATA-WORKFLOW-RULES-v3.md:43`; `LOCAL-HOST-DATA-FLOW-v2.md:93`,`:102`; `contracts/schemas/observation-signal.v5.schema.json:6-21` (required `observed_at`, no `direction`, `additionalProperties:false`); `DECISION-LOG-v1.md:65` (D-052).
proposed_alternative: Make the prose conform to the schema *before* WP-F1/WP-A2: in R1 and L1 rename `time`→`observed_at`; either add `direction` to the schema (if uplink-only, it is a constant and arguably droppable) or delete it from R1/L1 and the `observationSchema.validate()` description; state once, in R1, "the JSON Schema in `contracts/schemas/` is authoritative for field names and presence; this list is descriptive." Add a CI check that the prose field list is generated from / diffed against the schema so they cannot drift again.
downstream_impact: R1, L1, L2, the "shared `observation.schema.json`" enforcement clause, fixture generator (P9, P13, round-trip P17), agent enricher (WP-A2), loader validation (WP-S3), D-052.
confidence: high

---

FINDING-LEAD.3 · severity: **major** · target: `contracts/DATA-WORKFLOW-RULES-v3.md` R1 (L43,L48) vs `contracts/LOCAL-HOST-DATA-FLOW-v2.md` L1 (L95) — capture-time vs edge-filter-time field ownership
steelman: The pipeline is a chain; a field can be "guaranteed present by the time the row is a v5 record on the wire" even if it is stamped at a later hop, so R1 (the on-wire contract) legitimately lists fields that L1 (the capture hop) hasn't added yet.
failure_scenario: R1 L43 lists `observation_count` and `duty_cycle` among the fields the **scanner** "Emits exactly," and L48 asserts "`observation_count ≥ 1`; `duty_cycle ∈ [0,1]`" as a *producer (scanner)* guarantee. But L1 L95 says the opposite explicitly: "No `power_bin`, `observation_count`, `duty_cycle`, or `filter_action` yet — those are added later at the edge-filter step (L4/R2)." So one contract says the scanner emits these at capture; the other says they don't exist until two hops later at the uploader. A builder can't tell whether the scanner writes `duty_cycle` or the uploader does — and the conservation persona P17 (raw rows == FORWARD + Σ summaries) depends on knowing exactly which fields the scanner emits vs the uploader adds.
evidence: `DATA-WORKFLOW-RULES-v3.md:43`,`:48`; `LOCAL-HOST-DATA-FLOW-v2.md:95`; signal schema makes `duty_cycle` optional and `observation_count` `const 1` (`observation-signal.v5.schema.json:18-19`).
proposed_alternative: Pick the ownership explicitly and state it in both docs: the scanner emits the L1 core fields only; the uploader's edge filter adds `power_bin`/`observation_count`/`duty_cycle`/`filter_action` at L4. Move `observation_count`/`duty_cycle` out of R1's "scanner emits" list into an "added at R2/L4" note. Reconcile with the schema (which is the *signal-feed* record = post-edge-filter, so `observation_count const 1` and `filter_action const FORWARD` are correct there).
downstream_impact: R1, R2/L4 boundary, P17 conservation math, agent scanner vs uploader split (WP-A2 vs WP-A4).
confidence: high

---

FINDING-LEAD.4 · severity: **major** · target: `operations/PARAMETER-TUNING-v1.md` L42 vs `contracts/EVENT-CONSTRUCTION-METHODOLOGY-v1.md` (L105,L127), `DECISION-LOG-v1.md` D-028, `components/RFANALYSIS-SCOPE-v3.md` L66
steelman: PARAMETER-TUNING is explicitly "forward-looking … not yet built," so its table is illustrative and the methodology doc is the binding spec; a careful reader treats the contract as authoritative.
failure_scenario: The single knob that separates a power-saving tracker from infrastructure is the presence ratio. The methodology, the decision log, the server scope doc, the review brief, and both review prompts all state **0.95** and place noise filtering in **Level 1 (prep)**. PARAMETER-TUNING L42 states "**Presence / infrastructure ratio | Analysis/filter | ~0.8**" — wrong value *and* wrong layer. `0.8` is precisely earnest-v3's **rejected** "≥80% hours ⇒ infrastructure" rule that the methodology (L118, L138) singles out as the version that "would filter out hourly-beaconing trackers." So the one document a future operator opens *specifically to tune parameters* hands them the discarded value for the most safety-critical knob, and misfiles it into the analysis layer where the A/B framework would vary it against a fixed event dataset — but at 0.8 the event dataset itself would already have dropped the trackers.
evidence: `operations/PARAMETER-TUNING-v1.md:42`; `EVENT-CONSTRUCTION-METHODOLOGY-v1.md:105`,`:127`,`:118`,`:138`; `RFANALYSIS-SCOPE-v3.md:66` ("presence 0.95"); `DECISION-LOG-v1.md:31` (D-028).
proposed_alternative: Correct the PARAMETER-TUNING row to `0.95` and layer `Prep (Level 1)`; add a one-line note "0.8 was earnest-v3's rejected presence-only value — do not use." Since PARAMETER-TUNING is the A/B surface, add the low-duty-cycle exception multiplier (40) as its own row so the tracker-protection knob is visible there too.
downstream_impact: PARAMETER-TUNING, the A/B framework design, WP-S7, fixture P2/P3/P15 (the personas that assert the exception), RF-domain + Test/QA reviewers.
confidence: high

---

FINDING-LEAD.5 · severity: **major** · target: repo-count drift after D-055 — `README.md` (L12 vs L24), `DECISION-LOG-v1.md` D-030 (L35), `OPEN-QUESTIONS-v1.md` Q-01 (L5)
steelman: D-055 is recent and its intent (create `ear-analysis` now, 6 repos) is unambiguous where it is stated; the stale "five" instances are clearly residue, not a live disagreement about how many repos to make.
failure_scenario: D-055 un-deferred `ear-analysis` into a real, created-now 6th repo, but the count wasn't propagated: **D-030's own rationale** still reads "five balances isolation against single-maintainer overhead" (the decision title says "six repos," the justification says "five"); **README L12** calls it a "5-repo topology" while **README L24** lists all six; and — most consequentially — **Q-01**, the question the operator answers at S00 to authorize repo creation, asks to accept "the **five** names (`ear-docs`, `ear-agent`, `ear-server`, `ear-fixture`, `ear-fleet`)" and simply omits `ear-analysis`. An operator who "confirms Q-01" as written authorizes a five-repo creation that silently drops the analysis repo the plan (M4.5, WP-L1–L4) depends on. WP-D0 "create the 6 GitHub mirrors" then conflicts with the just-confirmed five-name list.
evidence: `DECISION-LOG-v1.md:35`; `README.md:12`,`:24`; `OPEN-QUESTIONS-v1.md:5`; contrast `MANIFEST.md:31` and `GREENFIELD-DEVELOPMENT-PLAN-v1.md:21` (correctly six) and `plan/GREENFIELD-DEVELOPMENT-PLAN-v1.md:38` ("create the 6 GitHub mirrors (incl. ear-analysis, D-055)").
proposed_alternative: Global replace the stale "five/5-repo" with six; fix D-030's rationale sentence; rewrite Q-01 to list all six names including `ear-analysis`. Because Q-01 is a pre-repo structural gate, this is a change-before-repo-creation item.
downstream_impact: D-030, Q-01, README, WP-D0, the whole M4.5 track; the operator's Q-18 pre-repo veto.
confidence: high

---

FINDING-LEAD.6 · severity: **major** · target: `contracts/LOCAL-HOST-DATA-FLOW-v2.md` L9 & L281 — stale cross-reference to an archived contract
steelman: The rule numbering (L1–L6 → R5) is stable across versions, so a reader who lands on the archived v1 still sees roughly the right R-rules; the reference is "morally" correct even if the filename is stale.
failure_scenario: LOCAL-HOST-DATA-FLOW-**v2** (a live contract) tells the reader twice that "server-side sync/ingest/pipeline (R5–R9) is covered in `DATA-WORKFLOW-RULES-**v1**.md`." But v1 is **archived/superseded**; the live document is `DATA-WORKFLOW-RULES-v3.md`. A build agent handed WP-A4 (which spans the L6/R4 seam) follows the cross-reference, opens `archive/DATA-WORKFLOW-RULES-v1.md` (or v2), and builds the agent side against superseded R-rules — the exact "storage paths and bucket prefixes drifted between builds" failure the R0 discipline exists to prevent, reintroduced by a dangling pointer inside the contract that preaches against it.
evidence: `LOCAL-HOST-DATA-FLOW-v2.md:9`,`:281`; the live file is `contracts/DATA-WORKFLOW-RULES-v3.md`; `archive/DATA-WORKFLOW-RULES-v1.md` and `-v2.md` exist.
proposed_alternative: Update both references to `DATA-WORKFLOW-RULES-v3.md`. Add a CI/doc-lint check that every intra-repo doc link resolves to a non-archive file (or is explicitly an archive citation). This is cheap and would have caught this class.
downstream_impact: Every build session that reads the L-chain (WP-A2–A5); the "rules travel with the code" R0.5 claim.
confidence: high

---

FINDING-LEAD.7 · severity: **major** · target: `[CLAUDE-DECIDED]` review-scope boundary — `README.md` L15,L28, `OPEN-QUESTIONS-v1.md` Q-18 (L46) vs actual tag range (D-042…D-061) vs `REVIEW-BRIEF.md` L32 / `MANIFEST.md` L54
steelman: The earliest CLAUDE-DECIDED block (D-030–D-041) is the structural core the operator most needs to veto pre-repo, so foregrounding it is a reasonable prioritization, not an omission.
failure_scenario: Three documents state three different ranges for "the unilateral decisions the operator must review before repos are created": README L15/L28 and **Q-18** say "**D-030…D-041**"; REVIEW-BRIEF L32 and MANIFEST L54 say "**D-030…D-059**"; the actual `[CLAUDE-DECIDED]` tags extend to **D-061**. The narrowest and most operative statement — Q-18, the literal gate the operator acts on — stops at D-041, leaving ~18 unilateral decisions never explicitly surfaced for veto, including several structural ones the review-disposition process (D-060) assumes were gated: **D-042** (Level 2 integration shape), **D-050** (secrets model), **D-052** (the canonical schema that is the tie-breaker for every field question), **D-061** (licensing). So the schema whose authority Finding-LEAD.2 turns on, and the security model the brief calls least-developed, sit outside the operator's stated pre-repo review window.
evidence: `README.md:15`,`:28`; `OPEN-QUESTIONS-v1.md:46`; `REVIEW-BRIEF.md:32`; `MANIFEST.md:54`; `DECISION-LOG-v1.md` tags at lines 48,56,62,63,65,67,69,73,75,77,79,81,82 (D-042…D-061).
proposed_alternative: Set one range everywhere: "D-030…D-061 (all `[CLAUDE-DECIDED]`)"; rewrite Q-18 to that range and split it into structural (must-veto-pre-repo: D-030–D-042, D-050, D-052, D-061) vs refinable-during-build. Reconcile README/REVIEW-BRIEF/MANIFEST to the same statement.
downstream_impact: Q-18, D-060 disposition process, the operator's pre-repo gate, the security and schema decisions specifically.
confidence: high

---

FINDING-LEAD.8 · severity: **minor** · target: stale corpus-size counts — `README.md` L3,L16, `MANIFEST.md` L4,L11,L12, `REVIEW-BRIEF.md` L5,L32, `REVIEW-PROMPT.md` L38
steelman: These are front-matter summaries, not the content itself; the content (the decision log, open-questions file) is complete and correct, and a reviewer works from the content.
failure_scenario: README/REVIEW-BRIEF/REVIEW-PROMPT/MANIFEST all say "60 decisions" / "D-001..D-059"; the log actually runs to **D-061** (61 decisions). README L16 says "Q-01…Q-18"; MANIFEST says "22+4" / "Q-01..Q-22"; the open-questions file actually runs to **Q-25** (+ Q-T1..T4). The index undercounts decisions by 2 and questions by ≥3. Individually cosmetic; combined with Finding-LEAD.7 it means the navigational front-matter systematically under-represents the corpus, so a reviewer trusting the index could treat D-060/D-061 or Q-23–Q-25 (which include the **legal/privacy deployment gate** Q-23 and **build-cost** Q-24) as out of scope.
evidence: `README.md:3`,`:16`; `MANIFEST.md:4`,`:11`,`:12`; `REVIEW-BRIEF.md:5`,`:32`; `REVIEW-PROMPT.md:38`; actual maxima D-061 / Q-25 confirmed by grep.
proposed_alternative: Regenerate the counts (61 decisions, 25+4 questions, 19 personas, S00–S21 w/ sub-sessions) or, better, stop hardcoding counts in prose and derive them. At minimum fix the four index docs so Q-23 (legal) isn't implicitly excluded.
downstream_impact: README, MANIFEST, REVIEW-BRIEF, both review prompts; reviewer coverage of the highest-numbered (and legally most important) questions.
confidence: high

---

FINDING-LEAD.9 · severity: **minor→major** · target: `contracts/DATA-WORKFLOW-RULES-v3.md` scope (L3) & R7 (L191-193) vs R8/R9 (L212-246) vs D-029 / D-055 / D-042
steelman: "Antenna to report" is a deliberately end-to-end framing that helps a reader see the whole chain; R8/R9 describe the *destination* the Level-1 dataset ultimately serves, which is legitimate context even if another component implements it.
failure_scenario: The doc's SCOPE (L3) says it "Covers Level 1 — dataset preparation … up to the analysis handoff," and R7 states flatly "This chain ends at the handoff. Time-clustering / CV / scoring / candidate selection are Level 2 … NOT stages of this pipeline." Yet R8 ("candidates → Postgres," "cross-sensor correlation **is computed**," writes `TrackerCandidate`/`CorrelatedDetection`) and R9 ("forensic report") specify Level-2 work as producer-guaranteed rules of *this* chain. After D-055, Level 2 is `ear-analysis`, which D-042/D-055 bind to "reads ONLY the Parquet handoff; writes ONLY its own Postgres models … **never Level 1 tables**." So R8's "runner writes candidates + correlation" now has no owner in the Level-1 doc's scope — it's either an orphaned rule or a boundary violation, and a builder can't tell whether the ear-server pipeline or ear-analysis owns R8/R9's enforcement (`FK candidate→run`, `no cross-band correlation` test, `report pulls only from Postgres`).
evidence: `DATA-WORKFLOW-RULES-v3.md:3`,`:191-193`,`:212-246`; `DECISION-LOG-v1.md:32` (D-029), `:48` (D-042), `:71` (D-055).
proposed_alternative: Either (a) relabel R8/R9 as "Level 2 rules, restated here for chain continuity — owned by `ear-analysis`," and move their ENFORCED-BY into the ear-analysis spec, or (b) split the doc at R7/R8 with an explicit ownership banner. Make the L1/L2 owner of each R-rule unambiguous now that Level 2 is a real component.
downstream_impact: DATA-WORKFLOW-RULES-v3, RFANALYSIS-SCOPE (which correctly says "correlation … separate, later — not in this job"), ear-analysis spec, R8/R9 enforcement tests, Data-eng + Arch reviewers.
confidence: high

---

FINDING-LEAD.10 · severity: **minor** · target: `operations/RECOVERY-RETENTION-v1.md` L21-22 vs `OPEN-QUESTIONS-v1.md` Q-15 (L23)
steelman: Retention numbers are explicitly "defaults — confirm/adjust via Q-15," so any specific value is provisional by construction and no contradiction is binding.
failure_scenario: RECOVERY-RETENTION presents R2 signal feed (`obs_`) retention as "**13 months**" and context feed (`ctx_`) as "**6 months**" in a table of decided defaults with mechanisms ("R2 lifecycle rule"). But Q-15 — the question that *is* the confirm/adjust channel — proposes only "raw landing 90 d, handoff indefinite, **context feed 180 d**" and never mentions a 13-month `obs_` rule at all. 6 months ≈ 180 d (consistent), but the 13-month signal-feed number appears as "decided" in ops while being absent from the operator's confirmation question, so "confirm Q-15" leaves the most important retention value (the raw signal the whole system is rebuildable from) unconfirmed yet treated as set.
evidence: `operations/RECOVERY-RETENTION-v1.md:21-22`; `OPEN-QUESTIONS-v1.md:23`.
proposed_alternative: Add the 13-month `obs_` retention to Q-15's proposal so the operator explicitly confirms it; or footnote the RECOVERY-RETENTION row "pending Q-15." Align the 6-month/180-d phrasing.
downstream_impact: Q-15, BACKUP-DR (which relies on R2 as SACRED source of truth), retention lifecycle config.
confidence: high

---

FINDING-LEAD.11 · severity: **minor** (flagged to Phase 1.5 as a cross-lens seam) · target: `operations/PARAMETER-TUNING-v1.md` L43 ("Min events for analysis | 3") vs `components/GOLDEN-FIXTURE-SPEC-v1.md` P15/P1 & taxonomy G1 deep-sleep
steelman: `min_events=3` is an analysis-layer guard against computing CV on statistically meaningless streams (you cannot get a stable coefficient of variation from 1–2 gaps), so a floor is defensible and lives in the right layer.
failure_scenario: The taxonomy's G1 deep-sleep sub-pattern is "multi-hour to multi-day intervals — the 20595 class; needs long analysis windows and **tolerates very few events**," and fixture P15 stages "4 short bursts across a 10-day window" with the assertion "min-events rules **must not erase it**." A hard `min_events=3` analysis floor would drop a real deep-sleep tracker that produced only 2 events in a window — the exact device the system exists to catch — and P15 would (correctly) go red, but only if the fixture window guarantees ≥3 events; a slightly longer real-world sleep interval silently falls under the floor in production where no persona guards it. This is a value that must be reconciled against the deep-sleep guarantee, not set independently in the tuning table.
evidence: `operations/PARAMETER-TUNING-v1.md:43`; `GOLDEN-FIXTURE-SPEC-v1.md:30` (P15), `:16` (P1); `level2/DEVICE-BEHAVIOR-TAXONOMY-v1.md:13`.
proposed_alternative: Not a pure-consistency fix — hand to RF-domain + Test/QA reviewers: define the deep-sleep path's min-events behavior explicitly (e.g. deep-sleep candidates bypass the CV floor and are scored on recency+regularity-of-the-few-gaps), and add a persona whose window yields exactly 2 events to prove a deep-sleep device is not erased.
downstream_impact: PARAMETER-TUNING, WP-L1/WP-L2 (feature extraction + classifier), P15, taxonomy G1 deep-sleep sub-path.
confidence: med

---

## Also noted (sub-threshold; folded into the reviewer briefs rather than raised as findings)

- **Band field naming.** The wire record uses `band ∈ {low,high}` (`observation-signal.v5.schema.json:15`) while the handoff uses `band_group ∈ {low,high}` (`handoff-events.v1.columns.json`) — two names for the same low/high concept across the wire→handoff boundary. Harmless if documented; a trap if a reader assumes `band` == EARFCN-band vs device-letter-group. Flag to Data-eng.
- **Sub-session count vs "S00–S21."** The runsheet advertises "S00–S21" but contains S19a–S19e, so there are ~26 sessions; the human-oracle set and parallelism notes are correct, only the headline range under-counts. Flag to Arch/PM.
- **`agent_version` in heartbeat.** The heartbeat schema *requires* `agent_version` (`heartbeat.v1.schema.json:14`) but EAR-AGENT-SPEC's heartbeat field list (L32) enumerates "status, disk %, queue depth, DLQ count, config_version" without `agent_version`; RUNBOOK relies on `agent_version` for staged releases. Minor omission in the spec's field list. Flag to Data-eng/SRE.

## Coverage statement (pass 0)

Examined for cross-document consistency: all 4 contracts + 4 schemas; all 4 component specs; taxonomy; plan (2) + disposition; all 8 operations docs; decision log (D-001–D-061); open questions (Q-01–Q-25, Q-T1–T4); README/MANIFEST/REVIEW-BRIEF; both REVIEW-PROMPT variants. Archive/ read only to validate cross-references. Verified by grep: uploader cardinality, `time`/`observed_at`/`direction` field set, presence-ratio value/layer, repo count, contract cross-refs, CLAUDE-DECIDED range, decision/question maxima, retention numbers. **11 findings** (0 blocker, 7 major, 4 minor), plus 3 sub-threshold notes. No contradiction was found in: the identity/`agent_id` regex (consistent across D-001, R1, L0, all three schemas), the 600 s burst gap (consistent everywhere), the two-pass-binning lock and Pass-2-open framing (consistent), the feed-split `obs_`/`ctx_`/`_health/` namespace (consistent), `schema_version=v5` (consistent), and the sweep-collapse 2 s "confirm" status (consistently marked open). The defects cluster entirely around three un-propagated late edits (D-045 uploader model, D-052 schema, D-055 sixth repo) — a propagation-discipline problem, not a design-coherence one.
