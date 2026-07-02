# Adversarial cross-examination — Cluster: the Q-22 fleet-volume-gate deferral

**Cross-examiner stance:** try to kill each finding. A finding survives only if it withstands the strongest rebuttal I can mount from (a) DECISION-LOG rationale, (b) another lens's priors, (c) an under-weighted steelman, (d) whether the failure actually reproduces.

**Operator's explicit deferral (the thing under challenge):** Q-22 (OPEN-QUESTIONS line 31) — "fleet-volume load gate … Not a concern at this stage; revisit **before a facility depends on run cadence**." Guy deferred deliberately. The build runsheet (BUILD-RUNSHEET line 36, S21) gates first facility on "ASK Q-04/Q-14/Q-15/Q-21 before facility" — Q-22 is pointedly **not** in that pre-facility ASK list. So the operator's position is not "never measure scale" — it is "the *full fleet-scale replay gate* is not a pre-first-facility blocker."

**Load-bearing fact the reviewers under-cite:** WP-S5 (S15 in the runsheet) is a **Pass-2 compute-budget benchmark scheduled during the build**, with a "benchmark report committed" as its gate condition — this happens at S15, well before S20 (live HackRF) and S21 (soak → first facility). So the claim "nobody has measured whether a run can exceed an hour / what a heavy agent's RSS is" is only true if WP-S5 is scoped to Pass-2 microbenchmarking and NOT whole-run wall-clock or RSS. That scoping ambiguity is the fulcrum of this whole cluster.

---

## FINDING-REL.6 — stale lock broken on wall-clock age (= timer period) can murder a live long run at fleet scale; Q-19 cadence cannot be set safely without max run time

**Rebuttal (strongest kill attempt):**

*Rationale angle.* D-044 says "overlap prevented by a run lock … scheduled by systemd timer/cron." The steelman the reviewer *grants* is fatal to the "murder a live run" scenario if you read systemd honestly: the pipeline is `docker compose run --rm pipeline … run_level1` fired by a **systemd `.timer` → oneshot `.service`**. A systemd oneshot service that is still `Type=oneshot`/active when its timer next elapses does **not** spawn a second instance — systemd refuses to re-start an already-active unit; the elapsed trigger is coalesced/skipped. So at the *systemd* layer, a run that outlives its period simply blocks the next tick as a no-op — exactly the reviewer's own "proposed_alternative." The self-stealing thundering herd requires that the auto-break logic run *inside* a fresh process that systemd allowed to start concurrently. Under the default oneshot model, systemd never starts that fresh process while the first is live, so the wall-clock stale-break never fires against a live run.

*Reproduction check.* The scenario also requires two independent triggers to both fire: (1) Guy sets Q-19 to *hourly* (the doc leans "2–4×/day", i.e. 6–12 h period — the reviewer concedes that is safe), AND (2) a legitimate run exceeds one hour. Under 2–4×/day the margin is 6–12× the plausible run time; the failure is only "armed" under an aggressive cadence the design does not choose and the runsheet does not pick before soak. So the failure is conditional on a config choice that is itself Q-19-OPEN and made *after* the WP-S5 benchmark exists.

*Cross-lens.* DE.7 (same author's polyrepo/pool sibling) establishes the pool is bounded at 4 and 60 agents run in ~15 serial waves — which *lengthens* wall-clock. That cuts *for* REL.6, not against it: it makes "run outlives period" more plausible, so I cannot use DE.7 to kill it.

**Why the rebuttal does not fully land:** The kill hinges on an *unstated implementation assumption*. The spec nowhere says the lock IS systemd's oneshot-active guard; it describes an **explicit application-level lock file broken on wall-clock age** ("a stale lock … older than the timer period is broken automatically with a `notify()`"). That is a *second, redundant, and dangerous* mechanism layered on top of whatever systemd does. If the app-level break logic runs at the *start of each run_level1 invocation* and systemd's own guard is what gates invocations, then the app break never sees a live run — the kill works. But if the break logic runs from any path that can execute while a run is live (a manual `run_level1`, a second timer, `check_health`-triggered remediation, or a container that systemd doesn't know is still alive because `compose run --rm` detached), the wall-clock rule *can* break a live lock. The spec does not pin which, and "broken automatically" reads as an in-band rule, not "systemd handles it." The reviewer's core structural point — **staleness must be liveness (pid/heartbeat), not wall-clock age** — is correct regardless of which layer holds the lock, because "older than the timer period" is simply the wrong predicate for "the holder is dead" the moment a legitimate run can exceed the period. At fleet scale a legitimate run *can* exceed the period (DE.7 waves, DE.4 heavy agents, REL.2 backlog windows), so the predicate is unsound as written.

**Verdict: survives.**

**Residual claim (narrowed from filed):** The catastrophic self-stealing-thundering-herd *loop* is speculative and likely blocked by systemd's oneshot semantics at the default cadence — drop that from "major" framing. What survives at full strength is the **decision-half**: the stale-lock predicate "older than the timer period" is unsound because a legitimate run can outlive the period; it must be **liveness-based (pid alive / heartbeat mtime), and the doc must state that the lock is held by systemd's oneshot guard OR carries a pid+heartbeat** — and Q-19 cadence must not be set below measured P95 run duration. This is a cheap **decision-before-repo** fix (the reviewer's own D-030-class recommendation), independent of Q-22.

**Downstream:** D-044 (lock semantics — add "liveness, not wall-clock"; state the lock mechanism explicitly), RECOVERY-RETENTION line 12 (rewrite the stale-lock rule), Q-19 (cadence ≥ measured run time), HEALTH-CHECKS Pipeline check (a "run outran cadence" state, not just old/failed). **Does NOT require un-deferring Q-22's full gate** — it requires (i) a doc fix now and (ii) *a* max-run-time number, which WP-S5 can produce if its scope includes whole-run wall-clock.

---

## FINDING-DE.4 — `memory_limit` ≠ RSS; per-worker Python/DuckDB overhead un-budgeted; "peak = pool × memory_limit" is a lower bound, and the OOM knob is a whole-run halt

**Rebuttal (strongest kill attempt):**

*Rationale angle.* D-047 is explicit and cuts hard against the practical severity: "pipeline pool 4×4 GB start (headroom to ~10×8 GB); **2 TB+ NVMe is the binding constraint, not RAM**." The sizing starts at 16 GB of `memory_limit` on a **128 GB** box. Even granting every one of DE.4's three uncounted terms — a 2× RSS-over-`memory_limit` spill factor AND a full gunicorn/Django per-worker footprint (~0.5–1 GB) — pool 4 lands at roughly `4 × (4 GB × 2 + 1 GB) = 36 GB` against 128 GB. That is a 3.5× margin. The "128 GB is fine" conclusion is not "unproven" at the *starting* config; it is comfortable by a wide margin. The un-budgeted terms only bite at the *headroom* config (10×8 GB = 80 GB nominal → ~180 GB real), which is a config Guy would only reach deliberately and which DEV-ENVIRONMENT already flags as the pressure case.

*Reproduction check.* Term (1), "memory_limit is a soft target not a hard RSS cap," is **true and well-known** for DuckDB — this is not speculative. Term (2), per-worker Python overhead, is **real and genuinely un-budgeted** in the doc. Term (3), windows growing over months → OOM-knob-drift, is real but is a slow-motion, observable failure (RSS trends are visible; the box is 128 GB). So DE.4 is not speculative — its *mechanism* reproduces trivially (any DuckDB spill exceeds memory_limit in RSS). What is speculative is the *magnitude* mattering at the starting config.

*The genuinely surviving core.* DE.4's real payload is not "you'll OOM at 4×4 GB tomorrow." It is two things the DECISION-LOG does NOT address: **(a) the scaling model is stated as exact — "the only two scaling tunables" — when it is a lower bound**, which is a documentation-honesty defect that seeds false confidence into the deferred Q-22 gate; and **(b) coupled with DE.6, a single worker's OOM is a non-zero exit = whole-run halt (D-044), so the memory risk is not contained to one agent even though D-023 sells per-agent isolation.** The DECISION-LOG nowhere reconciles D-023's isolation promise with D-044's whole-run-halt-on-any-worker-exit. That coupling is real and unaddressed.

**Why the rebuttal only softens:** I can kill the "128 GB is unproven / you'll OOM" alarm at the starting config — D-047's 3.5× margin is real. But I cannot kill (a) the exactness overclaim or (b) the isolation-vs-halt coupling. The reviewer's *narrow* ask is also cheap and correct: "a single-agent worst-case memory probe … must run before first facility even if the full fleet-scale gate stays deferred." That probe is nearly free and, critically, WP-S5 (S15) is *already going to load a representative heavy agent through Pass-2* — measuring peak RSS during that run is a one-line addition, not an un-deferral of Q-22.

**Verdict: softened.**

**Residual claim (reduced severity, minor→moderate):** Drop the "128 GB is unproven / whole-run OOM at 4×4 GB" alarm — D-047's margin covers the starting config. What survives: (1) **restate the ceiling as `peak ≈ pool × (memory_limit × spill_factor + worker_overhead)` and delete "the only two scaling tunables"** (honesty fix, cheap, do now); (2) **during WP-S5's existing heavy-agent run, record peak RSS-at-memory_limit** and set `memory_limit` from the measurement with margin (folds into an already-scheduled session — this is the reviewer's "narrow probe" and it does NOT require un-deferring Q-22); (3) add a **per-worker cgroup `--memory=` cap** so a runaway agent is killed at a known bound → converts a whole-run OOM into a single-agent quarantine (this is really DE.6's fix and is the load-bearing one).

**Downstream:** D-023 tunable statement (delete "only two"), DEV-ENVIRONMENT sizing (spill+overhead terms), Q-22 (the *narrow* RSS probe folds into WP-S5 — do NOT need the full gate), D-044 halt-granularity (couples to DE.6 — this is where the real severity lives), PARAMETER-TUNING (`memory_limit` becomes measured/versioned).

---

## FINDING-QA.13 — the volume deferral silently ALSO defers concurrency-CORRECTNESS (parallel-pool export race / run-lock / mid-run-crash orphan), which is distinct and testable at fixture scale

**Rebuttal (strongest kill attempt):**

*Rationale angle.* D-024 is designed *precisely* for the concurrent-finish case: atomic partitioned Parquet, "temp-write+rename," "no writer contention when many chains finish at once." And D-023 gives **physical per-agent isolation** — rf1a and rf1b write to *different files and different partition directories* (`handoff/{facility}/{agent_id}/{date}/`). So the "concurrent-export race" the reviewer worries about largely **cannot exist by construction**: there is no shared write target for two agents to race on. Atomic rename into disjoint per-agent directories is not a race. The isolation is *structural*, not a runtime invariant that concurrency could violate.

*Reproduction check.* The reviewer even concedes the framing: severity **minor**, and "the volume deferral is fine but silently also defers parallel-execution CORRECTNESS." The proposed fix is "one small persona." So this is not a claim that a bug *exists* — it is a claim that a *guard is missing* for a class that *could* harbor bugs (a `--memory=` OOM mid-export, an orphaned temp dir from a crash between rf1a-export and rf1b-export, the pool actually honoring per-agent file isolation under real parallelism rather than in the static GOLDEN-FIXTURE Isolation assertion).

*Where the kill fails.* The structural-isolation rebuttal covers the *export target* race but NOT the other two members of the class the reviewer names: (i) **mid-run crash leaving one agent's partition exported and another's temp-dir orphaned** — EVENT-DATASET-HANDOFF line 78 says "orphaned temp dirs are cleaned at chain start," which is a *behavior that must be tested*, not a structural guarantee; and (ii) the **run-lock itself** (the very mechanism REL.6 attacks) has no persona. Crucially, the GOLDEN-FIXTURE "Isolation" assertion (line 41) is **static** ("rf1a never touches rf1b's file"), verified without ever running the pool at size ≥ 2 concurrently. So the design's central claim — *per-agent parallelism is safe* — is asserted but never *exercised under actual concurrency* at any scale. That is a real coverage hole, and it is **orthogonal to volume**: you can run pool-size-2 over the 2-agent fixture today. The reviewer's key insight is sound: **concurrency-correctness ≠ volume-scale, and only the latter is what Guy deferred.** Deferring Q-22 does not license deferring the former, and nothing in the DECISION-LOG says the pool is exercised concurrently in any gate.

**Why the rebuttal only softens rather than killing:** The export-race sub-claim is killed by structural isolation (D-024 + disjoint per-agent dirs). But the crash-orphan-cleanup path and the run-lock have genuinely zero concurrent coverage, and the fix is cheap and precise (pool≥2 over the existing 2-agent fixture + a crash-injection re-run asserting orphan cleanup + determinism, tying to P16). That is a legitimate, low-cost, pre-first-facility guard.

**Verdict: softened.** (Remains minor; scope narrows to the two sub-claims that survive.)

**Residual claim:** Keep Q-22's *volume* deferral. Kill the "concurrent-export race" sub-claim (D-024 + disjoint per-agent directories make it a non-race). What survives: add **one fixture-scale concurrency persona** — run rf1a+rf1b through the *actual* process pool at size ≥ 2 (not the static isolation check), inject a crash after rf1a exports but before rf1b, re-run, assert (a) rf1b's orphaned temp dir is cleaned at chain start, (b) final state is byte-identical to a clean run (P16 determinism), (c) the run-lock behaves. This exercises the correctness the whole per-agent design leans on, costs one small persona, and requires **no fleet-scale volume**.

**Downstream:** GOLDEN-FIXTURE Isolation assertion (upgrade static → concurrent), EVENT-DATASET-HANDOFF line 78 (orphan-cleanup now has a guard), P16 (determinism under concurrent + crash), D-044 run-lock (the persona also touches REL.6's lock — see cross-finding note), Q-22 (concurrency-correctness carved OUT of the volume deferral).

---

## Cross-finding adjudication and the author-dissent

**The three findings converge on one operator decision:** is Q-22's deferral safe? Adjudicating across them:

- **REL.6** needs *a max-run-time number* (to set the stale-lock threshold and Q-19 cadence).
- **DE.4** needs *a single heavy-agent peak-RSS number* (to set `memory_limit` from measurement, not assumption).
- **QA.13** needs *a concurrency-correctness persona at fixture scale* (independent of volume entirely).

None of the three actually requires the **full fleet-scale replay gate** Guy deferred. Two of them (REL.6's number, DE.4's RSS) can be **harvested from WP-S5 (S15)**, a benchmark session *already scheduled in the build before first facility* — if WP-S5's scope is widened from "Pass-2 compute budget" to also emit whole-run wall-clock and peak RSS for the heaviest representative agent. The third (QA.13) is a fixture-scale persona needing no scale run at all.

**This is the cross-lens agreement, not a conflict:** RF/DE/QA/REL all independently arrive at "carve a *narrow* probe out of the deferral, keep the *fleet-scale gate* deferred." There is no dissent *among the reviewers*. There is no cross-domain conflict to adjudicate — the three findings are complementary, each naming a different cheap probe.

**The genuine author-dissent is reviewers-vs-operator, and it is narrow:**

- **Operator's side (steelmanned):** Q-22 is the *full fleet-scale replay gate* (60 sensors × multi-day × speed factor, assert time/disk budget). That is legitimately not a pre-first-facility concern: (1) the *first* facility is a single site, not the fleet — run cadence there is trivially satisfiable and "a facility depends on run cadence" is exactly the trigger Guy named for revisiting; (2) DEV-ENVIRONMENT already flags months×60-sensor sims as *the* pressure case and mitigates with L1 disposability + per-sim pruning; (3) D-047 sizes RAM with a 3.5× margin at the starting config; (4) building the full fleet-scale replay harness now is real effort spent modeling a load that does not exist until several facilities are live. The deferral of the *gate* is disciplined scope control, not negligence.

- **Reviewers' side (steelmanned):** Three *specific, cheap, non-fleet-scale* measurements are entangled inside the deferral and get silently postponed with it: (a) a single number — max whole-run wall-clock — without which the stale-lock predicate and Q-19 cadence are set blind (REL.6); (b) a single number — heavy-agent peak RSS — without which `memory_limit` is an assumption and the "only two tunables" claim is false (DE.4); (c) a fixture-scale concurrency-correctness persona that needs no scale run at all (QA.13). None of these is the fleet-scale gate. The risk of the deferral as *worded* is that "revisit before a facility depends on run cadence" reads as "measure nothing until then," which also postpones the cheap probes.

**Adjudication / recommendation for the author:** The deferral of the **full fleet-scale replay gate holds** — that part of Guy's call is sound and should stand. But the deferral note is **too coarse**: it should be split so that three narrow probes are explicitly carved OUT and land before first facility, most of them inside already-scheduled work:

1. **WP-S5 scope-widen (do at S15, already scheduled):** emit whole-run wall-clock (→ REL.6's cadence/lock threshold) and peak RSS-at-`memory_limit` for the heaviest representative agent (→ DE.4's measured `memory_limit`). Near-zero marginal cost.
2. **QA.13 concurrency persona (fixture scale, no volume):** pool≥2 over the 2-agent fixture + crash-injection + determinism re-run.
3. **Decision-only doc fixes independent of any measurement:** liveness-based stale-lock predicate (REL.6), delete "the only two scaling tunables" (DE.4), per-worker cgroup cap (DE.4/DE.6).

Everything else in Q-22 (the fleet-scale replay-at-speed-factor time/disk budget) stays deferred exactly as Guy filed it.

**Net:** Guy's deferral of the *gate* holds; the *narrow probe set* must precede first facility. State to the author as author-dissent on wording, with the reviewers and operator largely reconcilable.
