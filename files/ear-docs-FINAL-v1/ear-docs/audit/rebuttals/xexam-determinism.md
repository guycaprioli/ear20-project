# Cross-Examination Rebuttals — Cluster: Determinism / Reproducibility (SEAM-9)

**Cross-examiner role:** Adversarial. Each finding survives only if it withstands a genuine attempt to kill it.
**Cluster:** QA.4, QA.12, DE.9 — the three pillars (determinism test, gate protocol, A/B framework) that all rest on "reproducibility."
**Primary sources verified this pass:** `components/GOLDEN-FIXTURE-SPEC-v1.md` L37; `contracts/DATA-WORKFLOW-RULES-v3.md` R7 (L195–208); `contracts/EVENT-DATASET-HANDOFF-v1.md` L39/L55/L79; `contracts/schemas/handoff-events.v1.columns.json`; `DECISION-LOG-v1.md` D-024/D-025/D-044/D-047; `OPEN-QUESTIONS-v1.md` Q-15/Q-22; `operations/PARAMETER-TUNING-v1.md` L13/L50; `operations/RECOVERY-RETENTION-v1.md` L18–31.

---

## TARGET: FINDING-QA.4 — "byte-identical events Parquet physically unachievable as stated"

**Filed severity:** blocker.

### The strongest rebuttal I can mount

Three attacks, in order of force:

**(a) Fixable-not-fatal (the operator's brief challenge).** Every one of QA.4's four defeaters has a named, cheap, standard remedy: (1) row order → pin a total `ORDER BY (agent_id, earfcn, power_bin, event_start)` on every export — the pipeline already declares a fixed *stage* order (R7 "Fixed order") and per-agent correctness boundaries (D-025), so a total sort at export is a one-line addition, not an architecture change; (2) Parquet metadata (writer version, column stats) → pinnable via a fixed Arrow/DuckDB version (D-047 already pins *the same GHCR image per run* and calls the pipeline image "pinned image per run"); (3) compression → pin codec+level+row-group size in export config; (4) `run_id` → in **determinism-test mode**, inject a constant `run_id` so the provenance column is held equal across the two runs. Under all four pins, byte-identity is achievable. So the finding as literally worded ("physically unachievable") is **false** — it is achievable, just not *for free* and not *as currently under-specified*.

**(b) The DECISION-LOG / contract rationale the finding under-engaged — and it cuts BOTH ways.** The single most important thing QA.4 under-weighted is that **the authoritative rule already says the softer thing.** R7 CONSUMER-MAY-ASSUME (DATA-WORKFLOW-RULES L202) reads "Two runs on identical data+config produce **identical events**"; R7 ENFORCED-BY (L208) reads "Determinism test: same fixture twice → **identical event set**." *"Identical event set" is logical determinism.* The byte-identity overreach lives **only** in the fixture spec (GOLDEN-FIXTURE L37), a downstream operationalization of R7. So this is not "the design claims the impossible" — it is a **doc-to-doc inconsistency** where the binding contract is already correct and one derived test doc overstated it. That materially reduces the blast radius: no core decision is wrong; one line in one test spec is.

**(c) The run_id "contradiction" is not a contradiction of the contract, only of the fixture wording.** The finding's sharpest claim — "run_id per row + byte-identity is contradictory by construction" — is real *arithmetic* (a per-run provenance token cannot be equal across two runs while also being present in the bytes) but it refutes only L37, not the system. The handoff requires `run_id` per row for provenance (L39, L55); R7 requires an identical event *set*. These two are perfectly consistent the moment "determinism" means logical-set-equality-excluding-provenance. So the contradiction QA.4 surfaces is genuine but it indicts the fixture author's word choice, not the provenance design.

### Why the rebuttal nonetheless FAILS to kill it

The rebuttal defeats the *literal* claim ("physically unachievable") and relocates the fault ("it's just one line in the fixture spec"), but it does not defeat the finding's operative danger, which QA.4 stated explicitly and which the rebuttal actually *sharpens*:

- **The fixture defines truth (D-037/D-039, GOLDEN-FIXTURE L44: "the fixture defines truth; code conforms to it").** In this project a wrong expected-output in the fixture spec **becomes law**. So "it's only the fixture, not the contract" is not mitigation — the fixture is the *most* load-bearing of the two. L37 is precisely the artifact that gets encoded as a CI assertion. An impossible assertion in the truth-defining document is worse, not better, than the same overreach in prose.
- **The "just pin it" remedy is unwritten, and the finding's real prediction is about what happens at S17 under gate pressure.** QA.4's failure_scenario is not "byte-identity is impossible in principle" — it is "the determinism test fails on first honest implementation and gets weakened undocumented, under gate pressure, so 'determinism' silently comes to mean something softer than the contract claims." That scenario reproduces exactly *because* no doc currently pins ORDER BY, codec, row-group, version, or the constant-run_id test-mode injection. The remedy existing in my head does not stop the S17 scramble; only the remedy existing in the doc does.
- **The rebuttal (b)/(c) is itself a *second* independent defect**: R7 and the fixture spec disagree on what "determinism" means (set-identity vs byte-identity). That is an R0.4 paired-change violation waiting to happen and a live source inconsistency — surfacing it strengthens the case for a coordinated fix rather than weakening the finding.

### Verdict: **hardened**

Survives, and the cross-exam surfaced an additional reason it is worse than filed: the byte-identity overreach sits in the **fixture spec — the truth-defining document (D-037/D-039)** — while the binding contract R7 already uses the correct "identical event set" language. So the two authoritative docs *already contradict each other* on the definition of determinism, and the wrong one is the one that becomes law. QA.4 filed this as "assertion is unachievable"; it is really "the truth-oracle and the contract disagree, and the truth-oracle holds the impossible version."

### Residual claim (what the author must fix)

1. Change GOLDEN-FIXTURE L37 to logical determinism: "run the full chain twice → **identical event set** after canonical total sort on `(agent_id, earfcn, power_bin, event_start)`, comparing all columns **except `run_id`**" — matching R7's already-correct wording.
2. If byte-identity is *additionally* wanted as a CI nicety, pin all four in the export/test config **and write them down**: total `ORDER BY` on every Parquet export; fixed codec+level+row-group size; pinned DuckDB/Arrow version (version-scope the assertion); constant `run_id` injected in determinism-test mode. Add to the handoff ENFORCED-BY list.
3. Reconcile R7 (L202/L208) and GOLDEN-FIXTURE L37 as a **paired change (R0.4)** so "determinism" means one thing in all docs.

### Downstream

Propagates to QA.12 (gate re-run must compare the logical result, not artifact bytes) and to any P16 assertion ("update deterministically"). Touches R7 ENFORCED-BY, EVENT-DATASET-HANDOFF ENFORCED-BY, D-044 parallel export, S17 gate, PARAMETER-TUNING A/B precondition.

---

## TARGET: FINDING-QA.12 — "Guy's independent gate re-run (D-058) presumes determinism"

**Filed severity:** minor.

### The strongest rebuttal I can mount

**(a) D-047 dev/prod parity is a direct, logged answer to the version-pinning half.** The operator's brief asks precisely this. D-047 makes the dev host run "the same container-per-service topology as production … same GHCR images, same compose," and D-044 calls the pipeline a "pinned image per run." Guy's independent re-run happens *on that same box, against that same pinned image*. So the sub-claim that "no doc pins the DuckDB/Arrow version across agent and Guy's environments" is **substantially rebutted by D-047**: the image is the version pin, and Guy re-runs inside it. The finding even concedes this ("DEV-ENVIRONMENT-v2 parity (D-047) helps for the server").

**(b) The whole finding is derivative of QA.4 and inherits QA.4's over-statement.** QA.12's failure_scenario is entirely conditional on "per QA.4, byte-identical Parquet is not achievable." If QA.4's fix is logical determinism (which it is), then D-058's "re-runs the same command … sees green" was never about byte-matching artifacts in the first place — the gate report records "pass/fail, fixture release version, xfail list" (BUILD-RUNSHEET L42), which are **logical** outcomes. A green gate is a green *result*, not a byte-hash of Parquet. So the protocol as intended already survives.

**(c) The residual is a doc clarification, not a protocol defect.** Even granting the finding, its own proposed_alternative is one sentence added to D-058 ("the gate report records the logical result and tool versions; Guy's re-run must reproduce the logical result, not the artifact bytes"). That is a clarification, consistent with the minor severity filed.

### Why the rebuttal only PARTLY succeeds

- D-047 pins the **server/dev box**, but QA.12 explicitly names the **CI** path ("the gate also runs in CI"). If `make gate` also runs in CI on a different base, the version pin must be asserted there too. D-047 does not cover CI. So (a) closes most, not all, of the gap.
- The core logical point stands: **nothing currently states that Guy's re-run compares the *logical* result rather than artifact bytes.** As long as GOLDEN-FIXTURE L37 says "byte-identical," a diligent re-runner *could* diff bytes and see a (harmless, run_id-driven) mismatch and mis-read it as a gate failure — the "opposite of the protocol's intent" the finding names. The remedy is real but it is downstream of fixing QA.4.

### Verdict: **softened**

Reduced claim: this is **not an independent defect** — it is the QA.4 fix's obligation to propagate into D-058. D-047 already answers the version-pinning concern for the dev/server path (quote: D-047 "the same container-per-service topology as production … same GHCR images, same compose"), leaving only the CI environment un-pinned. Once QA.4 is restated as logical determinism, D-058 needs one clarifying sentence (gate report = logical result + tool versions; re-run reproduces the logical result, not artifact bytes) and a note that CI must run the pinned image too. Severity stays minor; it is a propagation task, not a standalone finding.

### Residual claim

Add to D-058: gate report records the **logical** pass/fail plus pinned tool versions; Guy's re-run must reproduce the **logical** result, not artifact bytes. Ensure `make gate` in **CI** runs the same pinned image D-047 pins for the dev box.

### Downstream

Strictly gated behind QA.4. If QA.4 is fixed as logical determinism, QA.12 is discharged by a one-line D-058 edit + CI image pin. If QA.4 were (wrongly) "fixed" by forcing byte-identity, QA.12 becomes a live blocker (Guy could see false-red). So QA.12's fate is a *function* of how QA.4 is resolved — the author should treat them as one work item.

---

## TARGET: FINDING-DE.9 — "A/B re-process-same-landing-without-re-fetch holds only 90 days"

**Filed severity:** minor.

### The strongest rebuttal I can mount

**(a) The 90-day landing prune is still an OPEN proposal (Q-15), not a locked decision — so the finding may be attacking a number the operator hasn't committed to.** RECOVERY-RETENTION L23 says landing = 90 d, but Q-15 reads: "Retention targets: raw landing (**proposal: 90 d**) … Confirm or adjust." The operator has an explicit open lever. DE.9's residual ("reproducibility beyond the R2 window is *lost*, which should be a conscious Q-15 decision, not implicit") is therefore **already the state of the world** — Q-15 is exactly that conscious decision, still open. This is close to rationale_addressed: the finding asks for a Q-15 tradeoff to be surfaced, and Q-15 is the vehicle.

**(b) The reproducibility guarantee is not actually broken — it degrades to a slower path, and the doc says so.** A handoff partition older than 90 days is still rebuildable: R2 `obs_` is retained **13 months** (RECOVERY-RETENTION L21), so "delete L1 → re-sync from R2 → re-land → re-enrich" works for over a year. PARAMETER-TUNING L13's exact claim is narrower than DE.9 implies — it says the landing table lets you re-process "**without re-fetching from R2**," i.e. it is an *optimization* (skip the fetch), not the *only* reproducibility path. So "reproducibility is lost at 90 days" overstates it: only the *fast, fetch-free* path is lost at 90 days; full reproducibility is lost only past **13 months** (R2 obs retention), which is a far larger and clearly-stated horizon.

**(c) A/B testing against 6-month-old events is arguably out of the design's intended use.** PARAMETER-TUNING is "not yet built" (L4) and A/B's whole point (L23) is a *labeled ground-truth corpus accumulated forward from day one*. A/B is a forward-looking tuning loop on recent data, not a demand to re-run against half-year-old landing snapshots. So the 90-day window may be entirely adequate for the actual A/B workflow, making the "silent" gap benign.

**(d) Provenance-dangling (arm 2) is defended by the retention table.** DE.9's second arm (indefinite handoff outliving finite `AnalysisRun` provenance) is directly contradicted by RECOVERY-RETENTION L26: "Postgres: Sensor/ledger/**runs** | indefinite (small)." Runs are indefinite. So the FK ("every event → a valid AnalysisRun," handoff L79) holds by the retention policy. DE.9 concedes this is "consistent only if runs are truly never pruned" — and the table says they are never pruned.

### Where the rebuttal FAILS

Two arms behave differently under cross-exam:

- **Arm 1 (A/B window) survives, narrowed.** Even granting (a)-(c), the concrete defect stands: **PARAMETER-TUNING L13 sells "the *same source data* … without re-fetching from R2" as an unqualified capability, and it is silently time-bounded.** A reader of PARAMETER-TUNING has no way to know the landing optimization expires at 90 days; they must cross-reference RECOVERY-RETENTION to discover it. That is a real doc gap regardless of whether 90 d is the final number. The rebuttal reduces the *stakes* (fetch-free is an optimization, full reproducibility survives to 13 months) but does not erase the un-stated qualifier. Worse for the design (a hardening detail I found): RECOVERY-RETENTION says raw landing lives **inside the L1 DuckDB file** ("90 days rolling *within the file*"), yet the same table says L1 files are "**last successful run only (cache) … rebuilt per run; deletable anytime.**" These two rows *collide*: if L1 files are last-run-only and deletable anytime, the "90 days rolling within the file" landing history is **not a durable 90-day store at all** — it is only as durable as the current L1 cache file. The A/B "re-process without re-fetch" promise may hold for far **less** than 90 days if L1 files are pruned to last-run. The 90-day figure and the last-run-only figure are in tension.

- **Arm 2 (dangling provenance) is killed** by (d): runs are indefinite (RECOVERY-RETENTION L26), so the FK holds. DE.9's own hedge (the pg_dump 30d/180d restore horizon) is a BACKUP-DR restore-correctness concern, not a retention-policy gap — and it belongs to the DR review, not here.

### Verdict: **softened**

Arm 1 survives narrowed; arm 2 killed. Reduced claim: this is a **documentation gap in PARAMETER-TUNING**, not a reproducibility failure — the "re-process without re-fetch" promise (L13) must be stated as a bounded capability. The binding horizon is *not* cleanly "90 days": it is `min(landing retention, L1-file lifetime)` on the fast path and **13 months** (R2 obs) on the full-rebuild path. The provenance-dangling arm is answered by RECOVERY-RETENTION L26 (runs indefinite). Severity stays minor; largely a Q-15 surfacing task.

### Residual claim

1. PARAMETER-TUNING L13: qualify the promise — "re-process without re-fetch" holds only while the source is still in the L1 landing window (proposed 90 d, and **only as long as the L1 file persists**); older A/B requires an R2 re-fetch, bounded by the 13-month R2 `obs_` retention, past which reproducibility is genuinely lost.
2. Resolve the RECOVERY-RETENTION internal tension: "raw landing 90 d rolling within the file" vs "L1 files = last successful run only, deletable anytime." State whether landing history durably survives across L1 rebuilds or is bounded by the L1 cache lifetime. **(new, this pass)**
3. Fold the reproducibility-horizon tradeoff explicitly into Q-15 so the retention numbers are chosen with the A/B window in view.

### Downstream

Q-15 (retention confirmation), PARAMETER-TUNING A/B window, RECOVERY-RETENTION table self-consistency. Not coupled to the QA.4/QA.12 logical-determinism fix — this is a retention/reproducibility-horizon issue, orthogonal to run-to-run determinism.

---

## Cross-finding adjudication (SEAM-9 as a whole)

The lead framed SEAM-9 as "reproducibility underpins three pillars; the fix must be applied to all three or 'reproducible' keeps meaning three different things." After cross-exam:

- **The three pillars do NOT share one fix.** QA.4 + QA.12 are **one** work item: logical determinism (canonical total sort, exclude `run_id`, pin codec+version) at the export, propagated into the gate protocol. DE.9 is a **separate** work item: a retention-horizon documentation gap in the A/B framework, unrelated to run-to-run determinism. So the seam is really **two** issues wearing one word ("reproducibility"), not one issue in three places. The author should not wait to fix DE.9 behind the determinism fix, nor vice versa.
- **Logical-determinism is the correct restatement, and it must propagate to exactly two of the three pillars** (the determinism test QA.4 and the gate re-run QA.12). It does **not** propagate to A/B/DE.9, whose problem is *how long the input survives*, not *whether two runs on the same input agree*.
- **No genuine cross-lens dissent in this cluster.** QA (test) and DE (data) approach reproducibility from different angles but do not contradict each other; they are additive, not opposed. There is nothing here for the author to adjudicate as a values conflict — only work to sequence.

### One challenge to flag for the author (not a dissent, a sequencing risk)

QA.4 and QA.12 are so tightly coupled that fixing QA.4 the **wrong** way (forcing literal byte-identity instead of logical determinism) would convert QA.12 from minor to a live blocker (Guy's re-run could see harmless run_id-driven byte mismatches and read them as gate failures). The author must fix QA.4 as **logical** determinism specifically; the byte-identity path is a trap that damages the gate protocol.
