# EAR System — Dev-Environment Buildability Assessment

**Question:** Is there enough in the documents for a Claude agent team (Fable lead · Opus for contract-critical logic · Sonnet for skeletons/plumbing) to build this solution **completely into a working dev environment** from the docs alone, following the runsheet?

**Method:** 6 isolated per-track assessors (dev-env/infra · agent · server-L1 · fixture · Level 2 · ops/dashboard), each judging against the *dev-environment* bar (MinIO substitutes for R2; a `replay` capture-source substitutes for HackRF — no hardware, no cloud, no field truth needed) and factoring in the audit's already-found contradictions. Full per-track detail in `audit/buildability/{track}.md`.

## Verdict: **Yes — but not by starting build sessions today.** ~76% buildable as-written; the rest is *cheap pre-written doc-fixes + one number from Guy + sanctioned stubs*, not missing design.

| Track | Verdict | %-as-written |
|---|---|---|
| Dev environment / infra (M0) | buildable after cheap doc-fixes | 80% |
| ear-agent (M2) | buildable after cheap doc-fixes | 80% |
| ear-server L1 (M3–M4) | buildable with marked stubs | 80% |
| **ear-fixture (M1 — built FIRST)** | **blocked — needs the pre-repo fixes + 1 Guy answer** | **55%** |
| ear-analysis L2 (M4.5) | buildable with marked stubs | 75% |
| ops / health / dashboard (M5) | buildable with marked stubs | 85% |

The architecture, contracts, schemas, compose topology, runsheet (S00–S21 with per-session model tier, ASK/DEFAULT gates, one-command `make gate`), and the dev harness itself are **genuinely execution-ready**. The design is buildable *by construction* for a dev loop — the replay→MinIO→pipeline→Parquet path needs no hardware or cloud. This is a strong pre-code corpus.

## The one thing that changes the answer from "yes" to "yes, after these fixes"

The blockers are **not** open algorithms and **not** missing design. They are ~a dozen contradictions the audit already found, and their danger is specific: **an agent building literally to the contract would pass its own green gate having built the wrong thing** — the worst failure mode, because the fixture *defines truth* and M1 is built first, so a wrong answer propagates fleet-wide into every downstream repo's regression floor.

**The fixture (M1) is the gating dependency.** Its defining operation — double-derivation of exact expected values, Guy-reviewed at S04 — is *precisely* what surfaces the contradictions: you cannot hand-compute a number the docs leave unset or self-contradictory. So S04 hits a wall on first attempt until these are pinned.

### 17 hard dev-blockers — but they resolve cheaply
| Blocker | Resolver | Where the fix already exists |
|---|---|---|
| Field-drift: R1/L1 emit `time`+`direction`; frozen v5 schema requires `observed_at`, forbids `direction` → **100% quarantine** | doc-fix | REPORT §A2 / §B1 |
| Uploader cardinality: 1-shared vs 4-independent contradicted in normative lines; frozen heartbeat schema has no `scope` field | doc-fix (+ pre-repo schema `scope`) | §A6 / §B2–3 |
| Conservation assertion halts every run on the first quarantined row (deterministic dirty row → no handoff ever) | doc-fix | §A3 |
| Determinism = "byte-identical Parquet" unachievable (contradicts per-row `run_id`; R7 already says "identical event set") | doc-fix | §A7 |
| Stage-order self-contradiction flips P15's "survives" verdict | doc-fix | §A3 (RF.5) |
| **`MIN_OBSERVATIONS` unset** — the floor deciding if the deep-sleeper (≈12–16 obs) lives; can't derive P8/P15 without it | **guy-answer** | §A1 |
| `presence_ratio` 0.8 (rejected value) vs 0.95 in the tuning doc | doc-fix | §A2 (LEAD.4) |
| `P15→G1` absent from the S19b gate → agent passes green while starving the primary target | doc-fix | §A1 (QA.5) |
| Repo count 5-vs-6 at the S00 Q-01 gate; stale xref to archived `DATA-WORKFLOW-RULES-v1` | doc-fix | §B4, §B8 |

**Net:** 15 of 17 hard blockers are one-edit doc reconciliations the audit *already wrote the exact edits for* (REPORT §B). Exactly **one** needs a decision only Guy holds: `MIN_OBSERVATIONS` (it encodes the deep-sleeper's survival math). One more (Q-05, false-split-vs-merge) sets the Pass-2 stub's bias and should be answered before S15.

### What correctly does NOT block the dev build
- **Open algorithms ship as marked stubs — by design.** The project's own seam discipline (P5-xfail, "provisional correlation loudly marked", DEFAULT-and-flag) is a *sanctioned* path past the Pass-2 boundary calc (Q-03), the correlation method, the four-factor score (which has no formula — only "NEW+STATIONARY+LOW-DUTY+PERIODIC"), and the DRAFT classifier cutoffs (pending Q-T1–T4). The dev stack stands up with these stubbed; only *finalizing* the algorithms needs field truth + the corpus.
- **DEFAULT-gated values** (sweep-collapse 2 s / Q-06, noise floor / Q-07, `thresholds.json` / Q-08, run cadence / Q-19) — the agent proceeds on the logged provisional and flags it, per the runsheet's own rule.
- **Production-only** (real R2 credential model, HackRF, 48 h soak, legal clearance Q-23, at-rest FDE, external beacon) — the dev environment substitutes MinIO + replay, so none of these gate a dev build.

## Bottom line
There is **enough** — the design is complete and coherent enough that Fable + Opus + Sonnet could build the whole system to a working, every-contract-exercising dev environment following the runsheet. But "enough to build" is not "ready to build today": start build sessions on the docs as-written and the agents would cement contradictions as law at M1. **Apply the audit's Section-B change-before-repo-creation set (≈1 focused doc-editing session) + get `MIN_OBSERVATIONS` and Q-05 from Guy + accept the sanctioned stubs — then the runsheet executes to a green replay loop.** The buildability assessment and the audit agree on the same critical path: the pre-code fixes are the gate, and they are cheap *now* precisely because nothing is built yet.
