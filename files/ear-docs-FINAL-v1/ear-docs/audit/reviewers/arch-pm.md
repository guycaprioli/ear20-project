# EAR System — Independent Review · SOFTWARE ARCHITECTURE / PROGRAM MANAGEMENT lens

Reviewer prefix: `FINDING-ARCH`. Scope: is the plan genuinely buildable as written by the described agent sessions?
Primary docs read in full: `README.md`, `REVIEW-BRIEF.md`, `MANIFEST.md`, `plan/GREENFIELD-DEVELOPMENT-PLAN-v1.md`, `plan/BUILD-RUNSHEET-v1.md`, `plan/REVIEW-DISPOSITION-v1.md`, `DECISION-LOG-v1.md` (D-001…D-061), `OPEN-QUESTIONS-v1.md` (Q-01…Q-25, Q-T1…Q-T4), `operations/BUILD-SESSION-BOOTSTRAP-v2.md`, `operations/DEV-ENVIRONMENT-v2.md`, `components/EAR-AGENT-SPEC-v1.md`, `components/RFANALYSIS-SCOPE-v3.md`, `components/GOLDEN-FIXTURE-SPEC-v1.md`, `contracts/DATA-WORKFLOW-RULES-v3.md`, `contracts/LOCAL-HOST-DATA-FLOW-v2.md` (L0–L0.5 + header), `contracts/EVENT-CONSTRUCTION-METHODOLOGY-v1.md`, `contracts/EVENT-DATASET-HANDOFF-v1.md`, `contracts/schemas/observation-signal.v5.schema.json`. (Did NOT read `audit/` per isolation rule.)

---

## 1. SURFACE ENUMERATION

Everything below is in-scope for this lens. Findings that follow reference these by id.

**Repo topology & dependency direction**
- D-030 (six-repo polyrepo; isolation vs single-maintainer overhead)
- D-031 (`ear-` naming), Q-01
- D-032 (fresh Pegasus generation; `bss_earv1` as donor, customizations "believed minimal")
- D-033 (`ear-docs` as contract source of truth; same-cycle contract+code landing)
- D-034 (machine-readable schemas in ear-docs)
- D-035 (legacy-corpus adapter isolated in ear-fixture)
- D-036 / D-061 (all repos private / proprietary), Q-02
- D-042 (ear-analysis as an installed Django app inside ear-server container)
- D-048 (local bare-repo origin + GitHub mirror; push-at-close)
- D-055 (Level 2 un-deferred; ear-analysis created now)
- The stated dependency direction: `ear-docs` ← all; `ear-fixture` ← test-dep of agent+server; `ear-fleet` ← ops-time; agent/server never depend on each other

**Contract-as-source-of-truth & paired-change discipline**
- D-033, R0.4 (paired change: producer+consumer never change in same commit unless the contract test moves too), R0.5 (rules travel with code), R0.3 (every rule has a check), D-037 (fixture-as-CI-gate; new rule ⇒ new persona same change)

**Runsheet realism**
- BUILD-RUNSHEET S00–S21 + S19a–e; the A-track‖S-track parallelism claim (runsheet line 38); question-gate ASK-vs-DEFAULT rule; verification protocol D-058 (gate command, gate report, Guy re-run, human-oracle points, regression floor)
- Milestone/calendar realism (plan §5: M0–M4 ≈ 5–7 weeks part-time single operator; M5 +1–2 wk)
- The critical-path position of Pass-2 (unresolved through S15) vs everything downstream assuming an event dataset

**Model-tier assignment**
- D-053 (Opus for contract-critical, Sonnet for skeletons/plumbing/CRUD); the per-session assignments in the runsheet table

**Decision-log / versioning discipline**
- D-037…D-039 (semver/tags, fixture-first, CI gate), D-052 (canonical v5 fields), `dataset_version` (handoff), the LOCKED-reopen rule ("new evidence, never silence"), the `[CLAUDE-DECIDED]` review gate (Q-18) vs D-042…D-061 which are also CLAUDE-DECIDED but not in Q-18's D-030…D-041 range

**Fresh-Pegasus-generation bet**
- D-032, RFANALYSIS-SCOPE "Disposition of the v1 scaffold" table, the "port-don't-redesign" rule (D-010, bootstrap non-negotiables)

**Pass-2 seam / provisional-correlation buildability**
- D-027 (two-pass LOCKED), Pass-2 calc OPEN (methodology "Pass-2 calc" section; Q-03), P5 xfail, WP-S5/S6, the "swappable stub" claim; D-055 provisional correlation seam (WP-L3), Q-05 / false-split-vs-merge

**Review-disposition as build gate**
- D-060, REVIEW-DISPOSITION-v1 (triage → adjudicate → fold → log → re-review), the "WP-D0 proceeds only after all findings dispositioned" gate

**Fixture-first authoring feasibility**
- D-039 (fixture built first), D-057 (double-derivation), the 19 personas, GOLDEN-FIXTURE-SPEC "Rules" section

**Cross-cutting internal-consistency items surfaced during the read**
- v5 field-name contradiction: R1 (DATA-WORKFLOW-RULES line 43) `time`/`band`/`direction` vs D-052 + signal schema `observed_at` (no `direction`)
- Uploader-count contradiction: LOCAL-HOST-DATA-FLOW line 26 / diagram "4 independent uploaders per host" vs D-016/D-045/spec "ONE shared uploader"

---

## 2. FINDINGS

### FINDING-ARCH.1 · severity: blocker · target: BUILD-RUNSHEET §Parallelism (line 38) + plan §5 calendar / D-053

**steelman.** The plan is honest that this is a single operator orchestrating CLI sessions, and the parallelism claim is deliberately hedged: "the A-track (S06–S10) and S-track (S11–S14) *can* interleave or run in parallel CLI sessions." The dependency graph (plan §5) shows `M2 ∥ M3 possible (different repos/sessions)`, and D-030's whole point is that agent and server are isolated repos depending only on the R2 contract — so there is no code-level coupling forcing serialization. The value being claimed is that *the work is decoupled enough to be reordered freely*, which is true and worth stating.

**failure_scenario.** Guy reads "M0–M4 ≈ 5–7 weeks" and "M2 ∥ M3 possible," and builds a calendar/budget on the two tracks overlapping. But a single human operator running Claude CLI sessions is a **single execution resource**: he cannot drive S07 (Opus, agent scanner) and S13 (Opus, server ingest) simultaneously — each is an interactive session needing his attention (bootstrap paste, mid-session ASK answers, gate re-run per D-058 step 3). The "parallelism" is not throughput parallelism; it is only *ordering freedom*. Worse, both tracks depend on **shared upstream artifacts that are themselves serial and Opus-heavy**: S03/S04 (fixture personas, Opus) and S05 (adapter, Opus) must complete before either track's gates can go green, because D-037/D-039 make the fixture the merge gate for *every* repo. So the real critical path is a long serial Opus chain S03→S04→(fan-out)→S15→S16→S17→S18, and the "5–7 weeks" figure has no visible derivation (no session-count × session-duration × operator-availability model anywhere). When the two tracks turn out to be strictly sequential in wall-clock time, the 5–7 week estimate slips by roughly the duration of whichever track was assumed to be "free," and every dependent commitment (first-facility date, Q-24 budget) slips with it.

**evidence.** Runsheet line 38: "the A-track (S06–S10) and S-track (S11–S14) can interleave or run in parallel CLI sessions." Plan §5 line 86: "M2 ∥ M3 possible (different repos/sessions)"; line 88: "M0–M4 ≈ 5–7 weeks." No document contains a session-duration estimate, an operator hours/week assumption, or a count of the ~24 sessions (S00–S21 + S19a–e + S20 + S21+) against a calendar. D-058 step 3 adds an *independent Guy re-run* to every session — pure serial operator time not modeled anywhere. The word "parallel" without the qualifier "of operator attention" is the over-claim.

**proposed_alternative.** (1) Replace "parallel" with "reorderable" everywhere and state explicitly: *one operator = one session at a time; the tracks give scheduling flexibility, not concurrency.* (2) Add a one-line estimation basis to plan §5: N sessions × (build + Guy-rerun + fold) hours ÷ operator hours/week = weeks, with the assumption stated. (3) Move the true critical path forward explicitly: it is the Opus fixture→event chain, and S15's `ASK Q-03` (Pass-2) sits on it — see ARCH.2. Removing the parallelism illusion is a doc edit now; discovering it as a 3-week slip mid-build is not recoverable.

**downstream_impact.** Q-24 (build-phase API + time budget), the M5 first-facility date, REVIEW-DISPOSITION's "pre-repo structural changes applied before WP-D0" gate (if calendar is wrong the gate timing is wrong), and any external commitment keyed to "Level 1 done = M5."

**confidence.** high

---

### FINDING-ARCH.2 · severity: blocker · target: Pass-2 seam on the critical path — D-027 / methodology "Pass-2 calc (OPEN)" / S15 ASK Q-03 / WP-S6 / P5 xfail

**steelman.** The design's handling of the unresolved Pass-2 calc is genuinely sophisticated and is the strongest risk-management move in the plan. D-027 locks only the two-pass *structure* (load-bearing for processing time per the methodology's "constraint" section), never the calc. The calc is isolated behind "a clear seam" (RFANALYSIS-SCOPE open-items), gated by P5 which ships `xfail` from day one (fixture rule: "flipping it to pass is the definition of that work being done" — GOLDEN-FIXTURE line 47), and the risk register's top row explicitly names "Pass-2 calc never found & wrong candidate chosen" with the seam + xfail + corpus validation (WP-S10) as mitigations. The regression floor (D-058) and determinism assertions keep the stub from silently becoming canonical. This is the right *shape* for an open algorithm.

**failure_scenario.** The concern is not the seam — it is where the seam sits on the critical path and what "swappable stub" actually means downstream. Sequence: S15 (`ASK Q-03` — the only hard-ASK gate on Pass-2) runs; Guy has not found the missing code (README line 29 and the methodology "Pass-2 calc OPEN" section both admit it may not exist in any repo). Per the ASK rule ("Unanswered `ASK` = do a different session"), S15 *cannot proceed* until Guy either finds the code or picks a candidate. But S16 (events), S17 (handoff export), S18 (corpus validation), and the entire M4.5 Level-2 track (S19a–d classification, which asserts P1→G1, P2→G1, P3→G4) all consume `power_bin` as an **authoritative, post-reconciliation** field — the handoff contract states this as a hard guarantee ("Power bin is authoritative — boundary reconciliation (Pass 2) has been applied; Level 2 does not re-bin," HANDOFF lines 30/52). So the plan runs S16–S19d on a *placeholder* power_bin while the contract downstream declares that bin authoritative. P5 is xfail, so the fixture does not catch this — but P1, P3, P6, P15 all assert specific power_bin groupings/event counts that a wrong stub can silently satisfy for the fixture's clean synthetic data while being wrong on real boundary-straddling devices (the exact defect the methodology calls "confirmed, not theoretical"). The corpus validation (S18) is the only real check, and it is a *human* oracle against 20595 — which detects gross failure but not subtle under-detection of *unknown* trackers the corpus doesn't contain. Net: the system can reach M4 "green" and M4.5 "classifies sensibly" with a Pass-2 that quietly fragments real devices, and nothing before first-facility catches it.

**evidence.** README line 29: "Pass-2 boundary calc (seam + `xfail` until resolved)." Methodology "Pass-2 calc (OPEN)" — "not fully present in any reviewed repo… may exist in code not yet seen." Runsheet S15: "**ASK Q-03** (hunt result / candidate choice)"; S15 gate: "P5 still `xfail`, seam swappable." HANDOFF line 52: "Power bin is authoritative — boundary reconciliation (Pass 2) has been applied." No contract test asserts that *a still-stubbed Pass-2 makes the handoff's authoritative-bin guarantee false* — i.e. there is no gate that fails when the handoff is exported while Pass-2 is a placeholder. The "swappable stub" is asserted (RFANALYSIS-SCOPE open-items) but no doc specifies the seam's *interface signature* (inputs/outputs of the Pass-2 step), so "swappable" is a claim, not a demonstrated contract.

**proposed_alternative.** (1) Pull the Pass-2 hunt out of S15 and into **WP-D0 / M0** — README already lists "hunt for Pass-2 calc code" as a Phase-0 item; make finding-or-deciding the calc a *precondition of starting M4*, not a gate mid-M4. The candidate choice (centroid-nearest is Q-03's provisional lean) is a doc decision that costs nothing now and de-risks the whole downstream. (2) Add a **handoff-honesty gate**: while Pass-2 is stubbed, the export must stamp `dataset_version` with a `-provisional-pass2` suffix (or a manifest flag), and a conformance test asserts Level 2 sees that flag — so "authoritative bin" is never silently claimed over a placeholder. (3) Specify the Pass-2 seam interface in `EVENT-CONSTRUCTION-METHODOLOGY` (the function contract: takes edge-observation set + bucket stats, returns reassignments) so "swappable" is verifiable, and add a persona (P5-variant) on *real* boundary data from the legacy corpus, not only synthetic 50/50.

**downstream_impact.** D-024/D-029 (L1/L2 boundary — the authoritative-bin guarantee is the whole point of the handoff), the entire M4.5 track (classification asserts on power_bin), Q-03/Q-05, WP-S5 benchmark (benchmarks a method that may not be chosen yet), the risk register's top row.

**confidence.** high

---

### FINDING-ARCH.3 · severity: major · target: R0.4 paired-change vs D-033 same-cycle landing — buildability of the discipline across 6 repos, one operator

**steelman.** R0.4 and D-033 encode the single hardest-won lesson in the doc set: "Every major breakage in this system's history was a silent contract violation" (DATA-WORKFLOW-RULES purpose). Making the contract source-of-truth (D-033) and forbidding producer/consumer drift (R0.4) is exactly the right instinct, and D-037's "new rule ⇒ new persona in the same change" gives it teeth via the fixture. For a single maintainer this is *more* important than for a team, because there is no second reviewer to catch drift.

**failure_scenario.** The discipline is stated as a single-commit / single-cycle rule, but the repo topology makes it physically impossible to honor atomically. Concretely: a contract change to the observation schema touches (a) `ear-docs/contracts/schemas/observation-signal.v5.schema.json`, (b) `ear-agent` (producer), (c) `ear-server` (consumer), and (d) `ear-fixture` (the contract test / personas). These are **four separate git repos** (D-030), so "the same commit" is impossible — the best achievable is four commits that a human must remember to make together, pushed to four local origins and then four GitHub mirrors (D-048), with the fixture repo needing a *version bump + tag* (D-038 semver) before the agent and server can pin the new fixture release. R0.4 says "producer and consumer never change in the same commit unless the contract test between them changes in that commit too" — but the contract test lives in a *third* repo (ear-fixture) and is consumed as a *versioned release* (plan §2: "Versioned releases consumed by agent + server CI"). So the atomic unit R0.4 assumes (producer+consumer+test in one commit) cannot exist in this topology. Under a six-month gap or a tired late-night session, the operator bumps the schema in ear-docs, updates the agent, and forgets to cut a new ear-fixture release — CI in ear-server still pins the old fixture tag, stays green, and the drift R0.4 exists to prevent ships anyway.

**evidence.** R0.4 (DATA-WORKFLOW-RULES line 33): "A producer and its consumer are never changed in the same commit unless the contract test between them changes in that commit too." D-033: "contract changes land there in the same change-cycle as code." D-030: fixture is "a versioned test dependency of two components." Plan §2 ear-fixture row: "Versioned releases consumed by agent + server CI." No document defines the **cross-repo change protocol** — the ordered checklist (bump schema → cut fixture release Y → pin Y in agent+server → land all) that would make R0.4/D-033 executable across 4 repos. "Same change-cycle" is asserted but the mechanism (a meta-commit? a release script? a `make contract-change`?) is absent.

**proposed_alternative.** (1) Write an explicit **cross-repo contract-change runbook** (one page in `operations/` or appended to R0): the exact ordered steps, including "cut ear-fixture release, pin its tag in both consumers, verify both CIs red-then-green against the new tag." (2) Reinterpret R0.4 for a polyrepo as a *fixture-release atomicity* rule: the contract test's atomic unit is the **fixture release tag**, and no consumer may pin a fixture tag whose contract test it hasn't gone red-then-green against. (3) Add a `make contract-drift-check` that fails if `ear-docs` schema hash ≠ the schema hash embedded in the currently-pinned fixture release — the one automated check that catches "forgot to re-release the fixture." This is the missing enforcement that turns R0.3 ("a rule with no check is not a rule") back on R0.4 itself.

**downstream_impact.** D-034 (schemas imported by both suites — needs the pin mechanism), D-037/D-039 (fixture-as-gate presumes a fixture-release cadence), D-048 (mirror push discipline), the bus-factor mitigation (a drift that ships silently defeats "fixture encodes intent executably").

**confidence.** high

---

### FINDING-ARCH.4 · severity: major · target: v5 record field-name contradiction — DATA-WORKFLOW-RULES R1 (line 43) vs D-052 + observation-signal.v5.schema.json

**steelman.** D-052 is explicit that "Schemas are the tie-breaker over prose on field questions," and the bootstrap non-negotiables tell every build agent "Ambiguity → check `contracts/schemas/` first." So the design *anticipates* prose/schema divergence and names the schema as authoritative — meaning a divergence is arguably self-healing: an Opus agent following the bootstrap will use the schema and ignore the stale prose.

**failure_scenario.** R1 (the *contract*, not a sketch) lists the producer-guaranteed fields as "`time`, `agent_id`, `band`, `earfcn`, `frequency_mhz`, `power_dbm`, `direction`, `observation_count`, `filter_action`, `duty_cycle`, `schema_version`" and states "`time` is ISO-8601 UTC… represents the observation moment." But D-052 pins the canonical field as `observed_at`, the signal schema `required` list is `["schema_version","observed_at","agent_id","earfcn","frequency_mhz","power_dbm","band","filter_action","observation_count","config_version"]`, and D-052 declares "Legacy names (`time`, `power`, `agent`) are NOT valid v5." So R1 — the document whose entire thesis is that field-meaning drift is the #1 historical failure — *itself uses a field name (`time`) that its own decision log declares invalid v5*, includes `direction` (absent from the schema entirely), and omits `config_version` (schema-required). An Opus agent building the parser (S07) who reads R1's PRODUCER GUARANTEES literally emits `time` and `direction`; the schema-validation gate then rejects those rows — but only if the fixture's expected outputs were derived from the schema, not from R1. If the fixture author (S03/S04, before parser code exists) reads R1 for the record shape, the fixture encodes `time`, and now the *wrong field name is law* (the exact GOLDEN-FIXTURE risk: "the fixture must not be able to encode a wrong answer as law"). The tie-breaker rule saves the parser agent but not the fixture author, because the fixture is derived *first* (D-039) and the personas' input construction references field semantics from the prose contracts.

**evidence.** DATA-WORKFLOW-RULES line 43: "Emits exactly these fields: `time`, `agent_id`, `band`, `earfcn`, `frequency_mhz`, `power_dbm`, `direction`, `observation_count`, `filter_action`, `duty_cycle`, `schema_version`." Line 46: "`time` is ISO-8601 UTC…". D-052: canonical `observed_at`; "Legacy names (`time`, `power`, `agent`) are NOT valid v5." Schema `required` list (observation-signal.v5.schema.json line 6) has `observed_at`, `config_version`, no `direction`, no `time`. R6 (line 178) also references `time_bucket` "holds a raw observation timestamp" — consistent with an internal timestamp column, but the wire-field name mismatch at R1 is unambiguous. This is a live contradiction inside the "law," version-stamped v3, post-D-052.

**proposed_alternative.** Edit R1 to match D-052/the schema verbatim: rename `time`→`observed_at`, drop `direction` (or add it to the schema if it is truly required), add `config_version`, and add a one-line note "field names are authoritative in `contracts/schemas/`; this list is illustrative." Do the same sweep for `power`/`agent` anywhere in prose. This is a five-minute edit now; as ARCH.4's scenario shows, caught after S04 it means re-deriving fixture expected outputs.

**downstream_impact.** D-052, D-034, the signal/context schemas, S03/S04 fixture derivation, S07 parser, R6 column dictionary, every persona whose input construction names a field.

**confidence.** high

---

### FINDING-ARCH.5 · severity: major · target: D-042 — ear-analysis as an installed Django app inside ear-server, vs D-030 six-repo isolation and the L1/L2 boundary

**steelman.** D-042's rationale is strong and single-operator-appropriate: "single operator; Pegasus already supplies auth/admin/Celery/UI; L2 is batch; a second service duplicates everything for nothing." The *data* boundary is preserved rigorously — ear-analysis reads only the Parquet handoff into a dedicated L2 DuckDB and writes only its own Postgres models, physically enforced because "Level 2's DB does not contain Level 1's tables" (HANDOFF line 18). D-055 reaffirms the boundary is unchanged. So the coupling is deployment-only, not data — arguably the best of both worlds.

**failure_scenario.** The code/deploy coupling quietly violates two other decisions the plan relies on. First, D-030 lists ear-analysis as a *separate private repo* "for isolation," but D-042 makes it "an installable Django app deployed inside the ear-server container (one web service, not two)" installed "via git, no pip-publishing ceremony" (D-055). "Installed via git into the same container" means ear-server's image build now depends on the ear-analysis repo at build time — so the two repos are *not* isolated: a change to ear-analysis models requires an ear-server image rebuild+redeploy (D-051: "GHCR image tag bump"), and the two share a Django settings module, one migration history, one Postgres. The claimed isolation (D-030) is nominal (separate git repos) but not real (one deployable, coupled migrations). Second, the runsheet builds ear-server's dashboard (S19, ops) and ear-analysis's dashboard (S19d) as *separate sessions in different repos* yet both render from the same Postgres and same Pegasus shell — so a model or migration authored in S19b (ear-analysis) can break S19's ops dashboard, but the fixture regression floor (D-058) is *per-repo* ("Every repo's CI runs its slice"), and there is no cross-repo integration gate that runs ear-server + ear-analysis together until S19e UAT. The first time the combined web service is exercised as one deployable is UAT, late in the plan, against a human oracle.

**evidence.** D-030: ear-analysis is a distinct repo, rationale "isolation." D-042: "installable Django app deployed inside the ear-server container (one web service, not two)… Django dashboard renders exclusively from Postgres." D-055: "installed into the ear-server deployment via git, no pip-publishing ceremony." D-051: server updates = "GHCR image tag bump." No document specifies whose migrations own the shared Postgres, how the two repos' Django apps compose into one settings/INSTALLED_APPS, or a combined-image CI that builds ear-server *with* ear-analysis before S19e. The runsheet has no "integrate ear-analysis into ear-server image" session — S19a–d build in `ear-analysis` repo; the image composition is implicit.

**proposed_alternative.** (1) Add an explicit **integration WP** (call it WP-S0-integrate or fold into WP-S1): define how ear-analysis is pulled into the ear-server image (git submodule? pip-from-git with a pinned SHA? vendored?), whose `manage.py`/settings own migrations, and a combined-image CI gate that builds ear-server *including* ear-analysis and runs both fixture slices. (2) State in D-042 that isolation is *repo-level only*; the deployable and Postgres are shared, and therefore migrations across the two repos must be coordinated (a real cross-repo dependency the R0.4 runbook from ARCH.3 must cover). (3) Move the first combined-service smoke earlier than S19e (a Sonnet session after S19a) so the one-web-service claim is exercised before the human-oracle UAT.

**downstream_impact.** D-030 (isolation claim), D-051 (update mechanism), D-058 (per-repo regression floor has a gap at the integration seam), S11/S19/S19a–e sequencing, the bus-factor story (shared migration history is a coupling a returning operator must relearn).

**confidence.** med

---

### FINDING-ARCH.6 · severity: major · target: D-032 fresh-Pegasus bet + "port-don't-redesign" — the two rules are in tension

**steelman.** D-032's logic is sound on its face: "6-month-old boilerplate is stale against current Pegasus/Django; regeneration is Pegasus's supported path," cost is "re-applying any bss_earv1 customizations (believed minimal)." Greenfield was Guy's mandate; regenerating from a current SaaS Pegasus generation avoids inheriting 6 months of framework drift, and the RFANALYSIS-SCOPE disposition table is admirably explicit about what carries forward as *logic fragments* vs what rebuilds as *structure* ("carry forward the known-good logic fragments; rebuild the structure fresh"). This is a defensible, well-documented call.

**failure_scenario.** Two of the plan's load-bearing rules pull against each other and the plan never reconciles them. The bootstrap non-negotiable is "**Port/build to contract, don't redesign**" (BUILD-SESSION-BOOTSTRAP; D-010 for the agent). But D-032 says the server is a *fresh Pegasus generation* where "`bss_earv1` becomes a reference donor, not the base." You cannot "port" from a donor you are not using as the base — porting implies structural continuity; a fresh generation has none. The RFANALYSIS-SCOPE table resolves this for logic (carry fragments) but not for the two places it actually bites: (a) the customizations D-032 calls "believed minimal" are un-inventoried — no document lists *what* bss_earv1 customized (auth? admin? the Sensor model? Celery config?), so "believed minimal" is an unverified assumption sitting on the critical path at S11 (WP-S1, a *Sonnet* session — see ARCH.8); (b) the harvest list in RFANALYSIS-SCOPE ("harvest gittest2 sync.py, resilience.py… earnest-v3 rf-device-status… eartest ForensicReportGenerator") assumes those donor modules graft cleanly onto a fresh Pegasus generation whose Django/Pegasus version may have moved APIs under them. When S11's Sonnet agent regenerates Pegasus and finds bss_earv1's customizations were *not* minimal (e.g. a custom user model, or Celery wiring the new gen structures differently), the "believed minimal" assumption fails mid-session, and a Sonnet agent — assigned precisely because WP-S1 was judged skeleton/plumbing — is the one holding a non-trivial framework-reconciliation problem with no contract to port against.

**evidence.** D-032: "re-applying any bss_earv1 customizations (believed minimal)" — the only quantifier is "believed," and no inventory exists. D-010 conditions: "port-don't-redesign; systemd model kept verbatim; harvest gittest2 patterns." BUILD-SESSION-BOOTSTRAP non-negotiable: "Port/build to contract, don't redesign." RFANALYSIS-SCOPE "Bottom line: carry forward the known-good logic fragments; rebuild the structure fresh" — resolves logic vs structure but not "port" vs "fresh-gen." Runsheet S11 is **Sonnet**: "fresh Pegasus gen, rfanalysis app, models+admin, compose per parity table, GHCR." No WP or question inventories bss_earv1's customizations before S11.

**proposed_alternative.** (1) Add a **WP-D0 phase-0 task**: inventory bss_earv1's customizations against a stock fresh Pegasus generation and record the delta in a short doc — this converts "believed minimal" into a fact before S11, and it is a Guy/investigation task, cheap now. (2) If the delta is non-trivial, re-tier S11 to Opus (or split: Sonnet for stock-gen skeleton, Opus for the customization reconciliation). (3) State in D-032 that "port-don't-redesign" applies to *logic and contracts*, not framework scaffolding — the scaffolding is *regenerated*, and name that explicitly so no agent tries to "port" structure that doesn't exist.

**downstream_impact.** D-053 model-tiering (S11 tier), D-010, RFANALYSIS-SCOPE harvest list, the whole M3 timeline (if S11 blows up, M3 and everything after slips — ARCH.1's calendar).

**confidence.** med

---

### FINDING-ARCH.7 · severity: major · target: Fixture-first authoring (D-039) vs P16/P18/P19 encoding server semantics the contracts leave open

**steelman.** D-039 (fixture built first) is test-first at project scale and is genuinely the plan's best structural bet: "every subsequent Opus session has a red/green target from its first commit." D-057's double-derivation (hand-derive expected outputs before any pipeline code exists, Guy spot-reviews P1/P2/P15) directly guards the "fixture encodes a wrong answer as law" failure mode. Authoring the fixture first is correct.

**failure_scenario.** Some personas' *expected outputs* depend on server behaviors that are still OPEN or under-specified at fixture-authoring time (M1, S04), which means the fixture author must invent the answer — and per D-039 the invented answer becomes law. Concretely: **P16 (late arrival)** asserts "the window's events *update deterministically* (full-rebuild semantics); no duplicates, no orphaned partial state" — but full-rebuild-over-window semantics for a *late file that changes an already-exported partition* is exactly the atomic re-export behavior (D-024) whose interaction with the ledger (R5) is not pinned anywhere with enough precision to hand-derive a byte-exact expected output. **P18 (qa isolation)** asserts a qa agent's events "appear in NO production candidate, classification, or correlation output" — but classification/correlation are Level-2 (ear-analysis, S19b/c), built *after* the fixture; the fixture author at S04 is deriving expected L2 outputs for a classifier whose decision order is still a "draft sketch" (D-054, "Guy to correct"). **P19 (override mismatch)** asserts the server "flags the pair (correlation-skew warning)" — but the correlation method is *provisional/open* (D-055, WP-L3), so what exactly gets flagged is not yet defined. So three of the eight D-057 gap-closing personas encode expected outputs for behavior that is open at authoring time. Either the author guesses (and locks a possibly-wrong answer as law, defeating double-derivation which only covers P1/P2/P15), or the personas can't actually be authored at M1 and D-039's "red/green target from commit one" is false for the parts of the system that are still open.

**evidence.** GOLDEN-FIXTURE P16: "events **update deterministically** (full-rebuild semantics)." P18: "its events appear in NO production candidate, classification, or correlation output." P19: "Server flags the pair (correlation-skew warning)… correlation for that EARFCN marked non-comparable." Runsheet: P15–P19 authored at **S04** (M1, Opus); classification built at S19b, correlation at S19c (M4.5). Double-derivation (D-057) spot-review covers only "P1/P2/P15." D-054 taxonomy is "draft, Guy to correct"; D-055 correlation is "PROVISIONAL… real method remains an open algorithm." The temporal gap between authoring (S04) and the behavior being defined (S19b/c) is unaddressed.

**proposed_alternative.** (1) Split the personas by *what they can pin now*: P16/P18/P19's **structural** assertions (no-duplicate, no-cross-contamination, pair-flagged-boolean) are authorable at M1; their **value** assertions that depend on open behavior (exact re-exported event set, exact classifier verdict) should be marked `xfail`-until-defined exactly like P5, so the fixture doesn't lock a guessed answer. (2) Extend double-derivation's Guy-spot-review set beyond P1/P2/P15 to include P16 (the late-arrival/rebuild interaction is subtle and easy to get wrong). (3) Add a runsheet note: personas whose expected outputs depend on S19b/c behavior get their value-assertions filled *at* S19b/c under the same double-derivation discipline, not guessed at S04.

**downstream_impact.** D-039, D-057, D-054/D-055 (open L2 behavior), S04 vs S19b/c sequencing, the GOLDEN-FIXTURE "fixture must not encode a wrong answer as law" rule.

**confidence.** med

---

### FINDING-ARCH.8 · severity: minor · target: D-053 model-tiering — specific mis-assignments (S11, S14) against the "contract-critical ⇒ Opus" rule

**steelman.** D-053's rule is well-calibrated: "Opus for contract-critical logic (edge filter, ingest, event construction, fixture truth)… Sonnet for skeletons/plumbing/CRUD; when in doubt, Opus." The bias-to-Opus tiebreak is the right default for a system where "a subtle semantic error corrupts data silently." Most assignments look correct (all fixture/event/ingest/filter sessions are Opus).

**failure_scenario.** Two Sonnet assignments carry more contract-critical weight than the "plumbing" label implies. **S11 (WP-S1, Sonnet)** is where the D-032 fresh-Pegasus regeneration happens and where the `Sensor` model — "the onboarding gate and the identity source of truth" (RFANALYSIS-SCOPE) — is defined with its `^[a-z]{3}-rf\d+[a-d]$` regex and status lifecycle. Getting the Sensor model's identity/regex/status semantics wrong is a contract error (D-001, R5.0), yet it is tiered Sonnet as "models+admin." Combined with ARCH.6 (bss_earv1 customization reconciliation also lands in S11), this is the highest-risk Sonnet session in the plan. **S14 (WP-S4, Sonnet)** builds `run_level1` orchestration including the **run lock against overlap** and **halt-on-failure exit-code mapping** (D-044) — concurrency/locking correctness is precisely the "subtle error corrupts silently" class (a broken run lock lets two pipeline runs race on the same L1 file), yet it is Sonnet as "job… timers, notify()." The gate for S14 does test "staged stale-sensor/failed-run → correct sweep verdicts," but not concurrent-overlap correctness of the lock.

**evidence.** Runsheet S11 Model: **Sonnet**, scope includes "models+admin" (the Sensor registry = identity source of truth per RFANALYSIS-SCOPE line 34). S14 Model: **Sonnet**, scope "run_level1, pool, lock, timers" (D-044: "overlap prevented by a run lock"). D-053 names "ingest" as Opus-worthy; the Sensor gate is the entry to ingest. The "when in doubt, Opus" tiebreak is not applied to either.

**proposed_alternative.** (1) Re-tier the Sensor-model definition portion of S11 to Opus (or split S11: Sonnet stock-gen skeleton + compose; Opus for Sensor model + regex + status lifecycle + the bss_earv1 reconciliation from ARCH.6). (2) Keep S14's timers/notify as Sonnet but carve the **run-lock + exit-code halt-on-failure** into an Opus sub-task, and add a concurrency test to its gate (two overlapping `run_level1` invocations → second blocks/no-op, no shared-file corruption). Both are cheap re-tiers now.

**downstream_impact.** D-001/R5.0 (identity gate correctness), D-044 (run-lock correctness), Q-24 (Opus is costlier — but the rule already says "when in doubt, Opus"), ARCH.6 (S11 risk compounds).

**confidence.** med

---

### FINDING-ARCH.9 · severity: minor · target: `[CLAUDE-DECIDED]` review gate scoped to D-030…D-041 (Q-18) but D-042…D-061 are also CLAUDE-DECIDED and structural

**steelman.** Q-18 correctly flags the CLAUDE-DECIDED items for Guy's veto "before repos are created (cheapest moment to change them)," and REVIEW-DISPOSITION (D-060) provides the general landing path for the audit so nothing is dropped. The intent to get human sign-off on unilateral decisions is present and well-structured.

**failure_scenario.** Q-18 names only "**D-030…D-041**" for the pre-repo veto review, but the decision log shows D-042 through D-061 are *also* tagged `[CLAUDE-DECIDED]` and several are as structural as anything in the D-030…D-041 range: D-042 (ear-analysis deployment shape — see ARCH.5), D-044/D-045 (the ephemeral-job execution model — load-bearing for the whole runtime), D-048 (local-origin git topology), D-052 (canonical field names — see ARCH.4), D-055 (Level 2 un-deferred, which *created a whole repo and milestone*), D-060/D-061. If Guy reads Q-18 literally, he reviews D-030…D-041 and repo creation (WP-D0) proceeds, having never explicitly vetted D-042/D-055 (the two decisions that most changed the project's *shape* — a sixth repo and an entire M4.5 milestone). D-060's disposition process covers the *external audit's* findings, but the *self-flagged* CLAUDE-DECIDED items from D-042 onward have no equivalent "review before repos" gate — they slipped past Q-18's range because Q-18 was written when the log ended at D-041.

**evidence.** Q-18: "Review `DECISION-LOG-v1.md` items **D-030…D-041** `[CLAUDE-DECIDED]` — veto or amend any before repos are created." Decision log: D-042, D-043, D-044, D-045, D-046, D-047, D-048, D-049, D-050, D-051, D-052, D-053, D-054, D-055, D-056, D-057, D-058, D-059, D-060, D-061 all carry `[CLAUDE-DECIDED]` (or `[CLAUDE-DECIDED — Guy confirm]`). README line 15 also says "`[CLAUDE-DECIDED]` D-030…D-041 await Guy's post-effort review" — the same truncated range. D-055 explicitly restructured scope ("Level 2 un-deferred… Repo `ear-analysis` is created NOW"). The range in Q-18 is stale relative to the log it points at.

**proposed_alternative.** Update Q-18 (and README line 15) to "**D-030…D-061** `[CLAUDE-DECIDED]`," and in REVIEW-DISPOSITION's exit criteria add: "all self-flagged CLAUDE-DECIDED decisions confirmed or amended by Guy" alongside the audit-finding disposition, so the pre-WP-D0 gate covers the operator's own unilateral calls, not just the panel's. One-line edit; prevents the two most structural decisions (D-042, D-055) from reaching repo creation un-vetted.

**downstream_impact.** REVIEW-DISPOSITION exit gate, D-060, every structural CLAUDE-DECIDED item (esp. D-042/ARCH.5, D-052/ARCH.4, D-055).

**confidence.** high

---

### FINDING-ARCH.10 · severity: minor · target: Uploader-count contradiction inside LOCAL-HOST-DATA-FLOW-v2 (line 26 "4 independent uploaders per host" vs header/D-016/D-045 "ONE shared uploader")

**steelman.** The prose is mid-migration from the old 4×2 process model to the corrected D-016 model (D-016 note: "Docs corrected from an earlier 4×2 claim"), and the *authoritative* statements (the deployment-shape paragraph line 11, D-016, D-045, EAR-AGENT-SPEC) consistently say **one shared ephemeral uploader per host**. So a careful reader who trusts the decision log gets the right answer.

**failure_scenario.** The ASCII diagram in LOCAL-HOST-DATA-FLOW (lines 14–30) shows an `uploader → edge filter → PUT` box *inside each per-device pipeline* (A, B, …C,D), and line 26 labels the fan-out "(4 independent uploaders per host)" — directly contradicting line 11 of the same document ("**plus ONE shared EPHEMERAL uploader per host (D-045)**") and D-016 ("1 shared uploader"). An Opus agent at S09 (WP-A4, uploader) reading the L-chain contract sees both "one shared" (prose) and "4 independent" (diagram + line 26) in the *same authoritative contract doc*. Since the uploader owns edge-filter/feed-split/heartbeat/disk-ceiling (all contract-critical), building it as 4-per-host vs 1-shared changes the disk-ceiling semantics (per-device vs host-wide), the heartbeat model (D-015 is per-agent_id — 4 uploaders vs 1 writing 4 heartbeats), and DLQ sharing. The contradiction is exactly the "storage paths / semantics drifted between builds" class the doc set exists to prevent, sitting inside the contract itself.

**evidence.** LOCAL-HOST-DATA-FLOW line 11: "plus ONE shared EPHEMERAL uploader per host (D-045)." Line 26 (diagram caption): "(4 independent uploaders per host)." Lines 18–24 diagram: each per-device box contains its own `uploader → edge filter → PUT`. D-016: "1 shared uploader + 1 shared DLQ per host." D-045: "resident = 4 scanner processes… ephemeral = `uplink-upload.timer` → oneshot… scans all four devices' folders." EAR-AGENT-SPEC line 8: "1 ephemeral uploader… scan pending → filter/split." The prose and the log agree; the diagram is stale.

**proposed_alternative.** Redraw the LOCAL-HOST-DATA-FLOW diagram to show 4 scanners feeding *one shared uploader box* (scanning all four `pending/` dirs), and change line 26 to "(1 shared ephemeral uploader per host, D-045)." Pure doc fix, but it is inside the "law" and feeds S09 (Opus) — worth fixing before a build agent trusts the diagram.

**downstream_impact.** D-016/D-045, EAR-AGENT-SPEC, S09 (WP-A4), heartbeat model (D-015, P14), disk-ceiling semantics (L5), DLQ (shared vs per-device).

**confidence.** high

---

### FINDING-ARCH.11 · severity: minor · target: D-048 local-origin + mirror-push discipline — the "never sole copy" guarantee depends on a manual per-session step

**steelman.** D-048 is a pragmatic single-maintainer choice: bare repos as origin give fast offline builds, GitHub as a mirror pushed at session close means "the box is never the sole copy." Zero-maintenance bare repos avoid running Gitea/GitLab. The persistence warning (README line 32) shows the risk is understood. For one operator this is reasonable.

**failure_scenario.** The "never sole copy" property rests entirely on the operator remembering to run `git push github --all --tags` at every session close, across 6 repos, as a manual standing rule (runsheet "Standing session rules"). There is no hook, no CI, no automation enforcing it — and D-058's gate report / commit discipline does not include a push-verification. The single realistic disaster: the operator drives several sessions in a burst (the plan's own mode), forgets the mirror push on the agent+server+fixture repos for a week, the NVMe fails (the plan notes "2 TB+ NVMe is the binding constraint" and treats it as a single disk), and a week of build work + un-mirrored fixture releases is gone. The bus-factor mitigation ("everything in ear-docs; GitHub is the copy") is only as good as the last successful mirror push, which is un-enforced.

**evidence.** D-048: "session-close rule: `git push github --all --tags` so the box is never the sole copy." DEV-ENVIRONMENT line 31: same, stated as a rule not an automation. Runsheet "Standing session rules": "push local origin **and** GitHub mirror at close (D-048)." Disk layout (DEV-ENVIRONMENT) shows a single 2 TB NVMe; no RAID, no second local disk mentioned. No `post-commit`/`post-receive` hook or scheduled mirror job specified anywhere. D-058's verification protocol verifies *gate green*, not *mirrored*.

**proposed_alternative.** (1) Add a `post-receive` hook on each bare repo that mirror-pushes to GitHub on every push to origin — makes "never sole copy" automatic, not remembered, and costs one hook script at WP-D1/S02. (2) Or a systemd timer that mirrors all `/srv/git/*.git` hourly. (3) At minimum, add mirror-freshness to the health sweep (D-059 already runs a ~15-min system sweep) so a stale mirror ambers the dashboard banner. Any of these converts a discipline into a check (R0.3 applied to the backup itself).

**downstream_impact.** D-048, D-059 (health sweep could carry the check), BACKUP-DR-v1 (should own this), the single-maintainer bus-factor risk (plan §6 last row).

**confidence.** med

---

### FINDING-ARCH.12 · severity: minor · target: REVIEW-DISPOSITION re-review trigger + WP-D0 gate — the gate can deadlock or be bypassed under structural change

**steelman.** D-060 closes a real gap ("review has no landing path") and the process is disciplined: every finding bucketed, dissents adjudicated as new decisions, structural changes applied *before* WP-D0, targeted re-review only on structural change. Making the review a gate on build rather than a parallel nicety is exactly right, and the disposition log gives an audit trail.

**failure_scenario.** Two edge cases are unhandled. First, **the re-review can recurse without a termination rule**: Step 5 says structural changes trigger re-running "the affected reviewer lens on the changed sections only," and the affected lens can produce *new* structural findings, which trigger *another* fold + re-review, indefinitely. For a single operator paying per Opus review pass (Q-24), there is no "convergence" or "max N rounds / diminishing-returns" stop condition — the gate could churn. Second, **the exit criterion is all-or-nothing against a soft input**: "WP-D0 proceeds only after all findings are dispositioned" — but findings bucketed NEEDS-INFO "become a question or a small investigation task," and several of the highest-leverage questions (Q-03 Pass-2, Q-04 topology, Q-23 legal clearance) may not be answerable quickly. If a structural finding lands in NEEDS-INFO pending Q-23 (legal), the literal gate blocks all repo creation on a legal answer — which may be correct for *deployment* but is over-broad for *building code*. The process has no notion of "gate repo-creation on structural/pre-repo findings only; let non-structural build proceed" — it couples the whole build start to the slowest disposition.

**evidence.** REVIEW-DISPOSITION Step 5: "re-run the affected reviewer lens on the changed sections only" — no round cap. Exit: "Every finding is in a bucket… all pre-repo structural changes applied. THEN WP-D0 proceeds." NEEDS-INFO: "becomes a question or a small investigation task" (Q-23 legal is deployment-gating per OPEN-QUESTIONS; Q-03 Pass-2 may not resolve — ARCH.2). No distinction between "gates repo creation" and "gates deployment" in the exit criterion.

**proposed_alternative.** (1) Add a re-review termination rule: "re-review is one round per structural change; a second round only if the change introduced a *new* structural surface, capped at N, else adjudicate and proceed." (2) Split the exit gate: WP-D0 (repo creation) gates on *pre-repo structural* dispositions only; deployment gates on the deployment-gating set (Q-23 legal, Q-11 host). This lets code building start while legal/topology answers mature, matching the plan's own "everything is reversible pre-code" ethos. (3) NEEDS-INFO items that block a structural decision should carry an explicit "blocks WP-D0? y/n" flag.

**downstream_impact.** D-060, Q-03/Q-04/Q-23, WP-D0 timing, ARCH.1 (calendar — a churning gate is unbudgeted time), Q-24 (re-review cost).

**confidence.** med

---

### FINDING-ARCH.13 · severity: minor · target: D-030 dependency-direction claim "agent and server never depend on each other — only on the R2 contract" vs the shared fixture + shared schemas

**steelman.** The claim is architecturally clean and true at *runtime*: agent and server communicate only through R2 objects, never call each other, and can be rebuilt independently (the R-chain/L-chain split enforces this). This decoupling is the plan's strongest isolation property and is worth protecting.

**failure_scenario.** At *build/test* time the two repos are more coupled than "only on the R2 contract" states: both depend on `ear-fixture` as a versioned test-dep (D-030 itself), both import the same JSON Schemas from `ear-docs` (D-034), and both are gated by the same personas (P4/P7/P12 are agent-side; P9–P13 server-side; but the *whole-run assertions* and conservation checks span the boundary). So a fixture-release change ripples to *both* CIs simultaneously — a coupling the "never depend on each other" phrasing hides. This matters because it interacts with ARCH.3: the mechanism that keeps agent and server *consistent* is the shared fixture release, and if that release cadence isn't managed (ARCH.3), the "independently rebuildable" claim is only true when the fixture version happens to be aligned. The over-claim is minor but it is the kind of confidence-beyond-evidence the REVIEW-BRIEF asked to flag.

**evidence.** D-030: "Agent and server never depend on each other — only on the R2 contract." Same decision: fixture is "a versioned test dependency of two components." D-034: schemas "imported by both component test suites." GOLDEN-FIXTURE whole-run assertions and conservation span agent-produced and server-consumed rows. So a shared build-time dependency (fixture + schemas) exists on top of the runtime independence.

**proposed_alternative.** Amend D-030's phrasing to: "Agent and server never depend on each other *at runtime* — they communicate only through the R2 contract. At build time both depend on `ear-fixture` (versioned test-dep) and `ear-docs` schemas; consistency between them is maintained by pinning the same fixture release (see the contract-change runbook, ARCH.3)." Names the real coupling so no one plans around a runtime-only truth.

**downstream_impact.** D-030, D-034, ARCH.3 (the fixture-release-cadence mechanism this depends on), the "independently rebuildable" claim.

**confidence.** med

---

## 3. WHAT TO PROTECT

- **D-039 fixture-first + D-037 fixture-as-CI-gate + D-058 verification protocol.** This is the single best structural decision in the plan. Test-first at project scale with a per-session gate command, committed gate reports as evidence ("agent claims are not evidence"), a human re-run, and a full-suite regression floor is exactly how you keep a swarm of AI sessions from drifting. Guard against any "streamline the gate" churn during build. (My ARCH.7 refines *what* the fixture can assert at authoring time; it does not weaken the fixture-first principle.)

- **The two-pass Pass-2 *structure* lock (D-027) with the calc left open behind a seam + P5 xfail.** Locking the load-bearing structure while explicitly *not* locking the unknown calc, and encoding the unknown as a day-one xfail that can never be silently marked done, is textbook handling of an open algorithm. Protect the xfail-until-real discipline. (ARCH.2 attacks the *critical-path position*, not the seam design.)

- **D-044/D-045 ephemeral-job execution model (resident only what must be; ephemeral everything periodic).** Removing Celery/Redis from the Level-1 critical path, making `pending/` the retry queue by construction, and mapping halt-on-failure to exit codes is a large, correct simplification for a single operator — it deletes whole bug classes (the file-watcher, the circuit breaker). Guard against a well-meaning "add a proper queue" regression.

- **The handoff contract's physical enforcement (D-024/D-029/HANDOFF).** Making the L1/L2 boundary a *physical* fact (Level 2's DuckDB literally does not contain Level 1's tables) rather than a convention is the right way to enforce a NEVER. This survives even D-042's deployment coupling (ARCH.5) because the *data* boundary is separate from the *deploy* boundary. Protect it.

- **R0 change-discipline as a meta-rule with per-rule checks (R0.1–R0.5, R0.3 "a rule with no check is not a rule").** The instinct — every historical breakage was a silent contract violation, so make contracts enforced, not documented — is correct and hard-won. ARCH.3 asks that this instinct be turned back on R0.4 itself (which currently lacks a cross-repo check); that strengthens the principle rather than weakening it.

- **D-036/D-061 private-by-default for a surveillance-adjacent system.** Correct default; no churn warranted.

---

## 4. COVERAGE STATEMENT

I enumerated and examined the full arch/PM surface listed in §1. Summary of what I examined and, where I found it sound, why it held:

- **Repo topology (D-030…D-036, D-042, D-048, D-055).** Examined isolation-vs-overhead, dependency direction, fixture-as-test-dep, fleet-holds-secrets. Sound *as runtime isolation* (protected above). Findings ARCH.5 (D-042 deploy coupling), ARCH.13 (build-time coupling over-claim), ARCH.11 (D-048 mirror discipline) target the seams where the isolation claim outruns the mechanism. The `ear-fleet` = state/secrets, no-code split (D-030/D-050) is clean and I found no issue with it; sops+age in fleet, plaintext rendered only at provision time, is a reasonable single-operator secrets model (security lens owns the depth here).

- **Contract-as-source-of-truth + paired change (D-033, R0.3/R0.4/R0.5, D-037).** The *principle* is the plan's best idea and is protected. ARCH.3 shows the *mechanism* for honoring it across a 4-repo change is unspecified — the one place the discipline is stated but not made executable. ARCH.4 shows the law contradicts itself on field names, which is the discipline failing on its own home turf.

- **Runsheet realism (S00–S21, S19a–e; parallelism; ASK/DEFAULT; D-058).** The session decomposition is genuinely good — each session is scoped, gated by one command, and evidence-committed. The ASK/DEFAULT gating is a real mechanism. Findings ARCH.1 (parallelism = ordering-freedom not concurrency; unmodeled calendar) and ARCH.2 (Pass-2 ASK on the critical path) are the two places the runsheet's realism claim exceeds its basis. The verification protocol (D-058) itself is sound and protected.

- **Model-tiering (D-053).** The rule is well-calibrated and *most* assignments are correct (every fixture/event/ingest/filter session is Opus, as the rule demands). ARCH.8 flags only two Sonnet sessions (S11 Sensor model, S14 run-lock) that carry contract-critical weight the "plumbing" label hides. The rule held everywhere else.

- **Decision-log/versioning discipline (D-037/D-038/D-052, dataset_version, LOCKED-reopen).** Semver+tags, `schema_version=v5`, `dataset_version` on handoff, and "reopen requires new evidence, never silence" are all sound. ARCH.9 flags the *review-gate scope* (Q-18's stale D-030…D-041 range) — a process gap, not a discipline flaw. The LOCKED-reopen rule is good and I found no issue.

- **Fresh-Pegasus bet + port-don't-redesign (D-032, D-010, RFANALYSIS-SCOPE disposition table).** The disposition table (logic-fragments carry, structure rebuilds) is excellent and mostly resolves the tension. ARCH.6 flags the residual: "believed minimal" is un-inventoried and sits on a Sonnet session. The bet itself (regenerate rather than inherit stale boilerplate) is defensible.

- **D-042 ear-analysis-as-installed-app + D-060 review-disposition-as-gate.** ARCH.5 (integration seam untested until UAT) and ARCH.12 (re-review termination + gate coupling) target real process gaps. The *data* boundary within D-042 is protected; D-060's core (review is a gate, not a nicety) is right and protected.

- **Pass-2 / provisional-correlation buildability as swappable stubs.** The seam+xfail *design* is protected. ARCH.2 targets critical-path position and the missing "handoff-honesty while stubbed" gate; ARCH.7 targets the fixture-authoring-vs-open-behavior timing for P16/P18/P19. Q-05 (false-split vs false-merge) is correctly left open and correctly identified as setting Pass-2 conservatism — I found no flaw in *how* it's tracked, only in *when* Q-03 is forced (ARCH.2).

- **Fixture-first authoring feasibility (D-039, D-057 double-derivation, 19 personas).** Protected as a principle; ARCH.7 is the one refinement (some personas can't fully pin expected outputs at M1). Double-derivation is a strong guard; ARCH.7 asks only to extend its spot-review set and xfail the value-assertions that depend on open behavior.

- **Internal-consistency sweep.** Found two live contradictions inside the "law": ARCH.4 (v5 field names, R1 vs D-052/schema) and ARCH.10 (uploader count, LOCAL-HOST diagram vs D-016/D-045). Both are cheap doc edits now and expensive if a build agent trusts the stale text.

Areas I deliberately did **not** claim on (out of lens or owned by another reviewer): the RF/physics correctness of the two-pass/600s/40× tunables (RF lens); the data-engineering scale behavior of per-agent DuckDB (data-eng lens); security depth of sops/age and Q-23 legal (security lens); SRE single-operator ops load beyond the build phase (SRE lens). I referenced Q-23/Q-24 only where they intersect the build gate (ARCH.12) and calendar (ARCH.1). No existing OPEN-QUESTION was re-raised as new; ARCH.2 extends Q-03, ARCH.7 extends Q-05/D-054/D-055, ARCH.9 extends Q-18, ARCH.12 extends Q-23/Q-03/Q-04.
