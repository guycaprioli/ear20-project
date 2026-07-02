# EAR System — Build-Readiness Plan (from audit → to a buildable dev environment)

**What this is.** The operational bridge between the audit (`synthesis/REPORT.md`) and the buildability assessment (`buildability/BUILDABILITY.md`) and an actually-executable build. It orders every remaining action by dependency and by *when it bites*, and maps each to its owner (doc-fix the audit already wrote · a decision only Guy holds · a new artifact to author · a sanctioned stub · a production-only gate).

**The gate principle.** Nothing here is a rewrite — it is all cheap *now* precisely because no code exists. The **fixture (M1) is the critical path**: it is built first, it *defines truth*, and its double-derivation step is exactly what trips over the unreconciled contradictions. So the sequence is: **Phase A/B/C land before WP-D0 creates repos → then the runsheet S00–S21 executes to a green replay loop → Phase E gates production.**

```
 Phase A (doc reconciliations)  ┐
 Phase B (Guy decisions x2)     ├─► WP-D0 repo creation ─► S00…S21 build (stubs ride) ─► DEV ENV GREEN
 Phase C (author new artifacts) ┘                                                          │
                                                                          Phase E (algorithms · corpus · legal · hardware) ─► PRODUCTION
```

---

## Phase A — Pre-repo doc reconciliations (BLOCK the dev build; the audit already wrote the exact edits)

Apply these **before S00**. All are D-030-class "change-before-repo-creation." Fold each with the project's paired-change discipline (contract + a new/amended decision + the persona in the same commit; version-bump the doc). This is ≈ one focused doc-editing session.

| # | Fix | Finding | Exact change | Docs touched |
|---|---|---|---|---|
| A1 | **Record field names** | LEAD.2/DE.11/QA.8 | R1/L1: `time`→`observed_at`; resolve `direction` (drop, or add to schema); fix `duty_cycle`/`observation_count` capture-vs-edge ownership; fix ENFORCED-BY filename (points at non-existent `observation.schema.json`) | DATA-WORKFLOW-RULES-v3 R1, LOCAL-HOST-DATA-FLOW-v2 L1, D-052 |
| A2 | **Uploader/DLQ cardinality** | LEAD.1/REL.1 | State "1 shared ephemeral uploader + 1 shared DLQ per host"; delete "its own uploader/DLQ"; redraw the diagram; state who authors the 4 heartbeats | LOCAL-HOST-DATA-FLOW-v2 §Deployment, D-016/D-045 |
| A3 | **Heartbeat scope field** (frozen schema) | REL.1/SEAM-8 | Add a `scope`/`host` annotation so host-level `disk_pct` vs per-device `queue_depth` is unambiguous | heartbeat.v1.schema.json |
| A4 | **Conservation identity** | DE.1/RF.5/QA.7 (SEAM-1) | One identity everywhere: `agent.raw == observations + quarantined + suppressed_summarized`; forced-EARFCN counted once; quarantine is exit-0-alert-on-ratio, not a halt | R5.1, EVENT-CONSTRUCTION, GOLDEN-FIXTURE l.39, P17 |
| A5 | **Stage order** | RF.5 | Pin one canonical order (bin → noise-filter over *raw* counts → sweep-collapse → burst); fix the methodology's self-contradicting numbered list; state the activity-count basis | EVENT-CONSTRUCTION §19/§100, R7 |
| A6 | **Determinism = logical** | QA.4 | Replace "byte-identical Parquet" with "identical event set after canonical sort, excluding `run_id`"; reconcile GOLDEN-FIXTURE l.37 ↔ R7 | GOLDEN-FIXTURE, R7, EVENT-DATASET-HANDOFF |
| A7 | **presence_ratio** | LEAD.4/RF.4 | 0.8 → 0.95, layer Analysis → Prep; note "0.8 = rejected earnest-v3 value" | PARAMETER-TUNING l.42 |
| A8 | **P15→G1 in the S19b gate** | QA.5 | Add `P15→G1` to S19b DoD; pin "compute CV anyway for few-gap deep-sleepers" | BUILD-RUNSHEET S19b |
| A9 | **Repo count / Q-01** | LEAD.5 | Q-01 and README list all **six** names incl. `ear-analysis`; fix D-030's "five" rationale | README, OPEN-QUESTIONS Q-01, D-030 |
| A10 | **Stale contract xref** | LEAD.6 | `DATA-WORKFLOW-RULES-v1` → `-v3` (×2); add a doc-lint that links resolve to non-archive | LOCAL-HOST-DATA-FLOW-v2 L9/L281 |
| A11 | **CLAUDE-DECIDED review scope** | LEAD.7 | Q-18 range D-041 → D-030…D-061; foreground D-042/D-050/D-052/D-061 | README, OPEN-QUESTIONS Q-18, MANIFEST |
| A12 | **Capture-advancing liveness** | OPS.10 | One line: liveness = `ctx_`/heartbeat freshness; `obs_` volume is informational/self-baselined | HEALTH-CHECKS-v1 |
| A13 | **Schema↔prose CI check** | A2 mechanism | Add a CI diff so prose field lists can never drift from the JSON schemas again | contracts/schemas + CI |

**Exit A:** the fixture's expected outputs are now *derivable to a single correct answer*; a build agent reading any contract builds the same thing.

---

## Phase B — Decisions only Guy holds (extract before the session that needs them)

| Decision | Needed by | Blocks dev? | Note |
|---|---|---|---|
| **`MIN_OBSERVATIONS` value** | S04/S16 | **Yes** — can't derive P8/P15 without it | Encodes the deep-sleeper's survival floor (A1/§C2); set it, then add a `MIN_OBS ± 1` boundary persona |
| **Q-05 false-split vs merge** | S15 | **Yes** — sets the Pass-2 stub's bias (lean *merge*, per the logged worse-error) | Blocking for the stub choice, per cross-exam |
| **Q-01 repo names / org** | S00 | Yes (trivially) | Resolved with A9 |
| **Q-04 server topology** | before M3 | No (dev = single box) | Decide with the legal/SPOF/egress/filesystem criteria (SEAM-5) — cheapest now |
| DEFAULT-gated: Q-06 sweep-collapse, Q-07 floor, Q-08 thresholds, Q-19 cadence | their sessions | No | Agent proceeds on the logged provisional and flags — the runsheet's own rule |

---

## Phase C — New artifacts to author (part of "fixing the audit," but authoring not editing)

Fold via D-060. These don't exist yet and the audit recommends creating them:

- **Primary-target survival-budget doc + dual-gated P15** (§A1) — trace a deep-sleeper through eviction → `MIN_OBSERVATIONS` → rebuild-window → `min-events`, one pinned value per link; P15 gated at both S16 and S19b.
- **Three security contracts** (SEC.2/4/7) — access-control (the human-plane analog of R5.0), access/export audit, key-management (escrow + revocation-that-rotates). Author in WP-D1 alongside the schemas so build agents build *against* them.
- **Band-plan appendix** (RF.1) — exact 3GPP uplink/downlink sub-ranges per MHz window; make Q-13 a design question; tie the EARFCN map into `contracts/schemas/`.
- **New personas** — P2b (duty-cycle boundary), P20 (`hours_present` regression = the origin bug), P21 (cross-band-correlation guard), P15-split (L1 + L2 halves), one concurrency-correctness persona (pool ≥ 2 + crash injection). Each with the rule it pins.
- **Heartbeat-semantics ADR** (A2/A3 decision record) + **the conservation identity as a single referenced source** (A4).
- **Rebuild-window definition** (REL.2) — a first-class versioned param with a long-horizon path for deep-sleepers.

---

## Phase D — Then the runsheet executes (S00–S21, as-written, with sanctioned stubs)

After A/B/C, the plan runs. These ride as **clearly-marked stubs** per the project's own seam discipline (P5-xfail / DEFAULT-and-flag) — they do **not** block the dev loop; only *finalizing* them is later work:

| Stub | Marker | Finalized when |
|---|---|---|
| Pass-2 boundary calc (Q-03) | P5 xfail; `pass2=stub` stamped in handoff `dataset_version`; `power_bin` marked provisional (QA.3) | code found or candidate chosen + corpus-validated |
| Correlation method | "loudly provisional" (D-055) | real method chosen (post-corpus) |
| Four-factor score (no formula) | transparent marked-provisional weights, graded vs personas | field truth (Q-T1–T4) + corpus |
| Classifier cutoffs (DRAFT taxonomy) | DEFAULT Q-T1–T4 | field truth |
| Eviction policy (A9/REL.7) | ship alert-and-hold default, tiered | Q-10 hardware + first-facility |
| Fault-granularity (A4/DE.6) | ship naive halt marked; per-agent isolation is a localized WP-S4 refinement | during build |
| L2 dual-write (DE.7) | ship transactional-per-run-replace + `results_complete` flag | during build |

**Exit D:** replay agent → MinIO → pipeline → Parquet → L2 → dashboard, every contract exercised, fixture green, UAT J1–J7 runnable against dev data. **This is a complete dev environment** — producing *provisional* detections.

---

## Phase E — Production / first-facility gates (NOT dev-blocking; the deepest "what's left")

None of these can be shortcut by a document:

- **Solve the two algorithms** — find the missing Pass-2 code or choose+validate; design the real correlation method.
- **Build the labeled corpus** — the ground truth that makes classification *validated* not *provisional*; accrues only from physical confirmations over time; unreconstructable.
- **Field truth Q-T1–Q-T4** — turns the DRAFT taxonomy into a tuned classifier.
- **Legal clearance Q-23** — the hard deployment gate (blocking checkbox per SEC.1).
- **The five human-oracle validations** — esp. S18 (does it find 20595 in the legacy corpus?) and S20 (live RF).
- **The physical/production layer** — HackRF, Strix box, real R2 + scoped credentials + object-lock (SEC.3), facilities/antennas, host FDE, external beacon, DR/rollback drills.

---

## Effort shape (order-of-magnitude, single operator + agents)

| Phase | Effort | Blocks |
|---|---|---|
| A — doc reconciliations | ~1 focused session | dev build |
| B — 2 blocking decisions (+2 pre-M3) | your call | dev build |
| C — author new artifacts | ~1–2 sessions | correctness/regression coverage |
| D — run S00–S21 | the plan's 5–7 weeks part-time | — |
| E — algorithms + corpus + legal + hardware | open-ended (field/R&D/legal) | trustworthy production |

**Bottom line:** Phases A+B+C are a small, well-scoped, mostly-already-written amount of work that takes the design from "would misbuild" to "builds correctly." Then D is the real build. E is the part no document can shortcut. The audit's whole value is collapsing the expensive-later contradictions into a cheap-now A/B/C — do those first, and the runsheet is genuinely executable.
