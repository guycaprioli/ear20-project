# EAR System — Independent Review · DATA ENGINEERING lens

**Reviewer question:** Does the pipeline hold at scale and under failure? Every table, every transform, every boundary.
**Method:** Primary documents only (no `audit/` read). Steelman-first per REVIEW-BRIEF; DECISION-LOG cited for WHY.

---

## 1. SURFACE ENUMERATION

The complete set of decisions, contract clauses, schema fields, WPs, personas, and questions I hold in-scope for this lens:

**Decisions (DECISION-LOG-v1):**
- D-002 R2-primary transport (store-and-forward load-bearing)
- D-011 edge filter kept (90% noise)
- D-012 feed split (`obs_`/`ctx_`)
- D-013 L4.5 forced-summary override
- D-021 two-table ingest, quarantine-with-reason, `ignore_errors` banned, folder-authoritative identity, `schema_version=v5` only, no COALESCE
- D-023 per-agent DuckDB files (`l1/{facility}/{agent_id}.duckdb`), open→work→close, tunables = worker concurrency × memory_limit
- D-024 handoff medium: atomic partitioned Parquet, temp-write+rename
- D-025 per-agent chains for bin/consolidation/events; load/enrich may batch
- D-042 DuckDB-computes / Postgres-serves split
- D-044 ephemeral pipeline job (`docker compose run --rm`), run lock, process pool, Celery/Redis removed from L1
- D-045 ephemeral uploader timer, `pending/` = retry queue
- D-048 local git hub + GitHub mirror
- D-052 canonical v5 record fields
- D-055 L2 feature extraction / classification / scoring / correlation

**Contract clauses (DATA-WORKFLOW-RULES-v3):**
- R0.1–R0.5 change discipline
- R4 transport (atomic PUT, prefix scheme, "R5 keys on filename+size/etag")
- R5.0 onboarding gate
- R5.1 fetch+ingest two-table, ledger uniqueness, same-transaction land+ledger, post-load assertion "observations delta == sum ledger row_counts", fake-gzip detection
- R6 analytical column semantics
- R7 pipeline stages (full rebuild per window, per-agent correctness boundary, determinism)
- R8 results persistence

**Handoff contract (EVENT-DATASET-HANDOFF-v1 + handoff-events.v1.columns.json):**
- Partition path `handoff/{facility}/{agent_id}/{date}/`
- Atomic temp-write+rename, orphan cleanup at chain start
- `dataset_version`, immutability, half-written-partition prevention
- `events` + `event_gaps` column specs

**Schemas:** observation-signal.v5, observation-context.v5, heartbeat.v1, handoff-events.v1.columns.

**Ops:** RECOVERY-RETENTION-v1 (recovery-by-rebuild, retention table), BACKUP-DR-v1, PARAMETER-TUNING-v1, DEV-ENVIRONMENT-v2 (pool 4×4 GB, 128 GB box, 2 TB NVMe).

**WPs:** WP-S4 (run_level1, process pool, run lock), WP-S9 (atomic Parquet export), WP-L1 (L2 feature extraction glob), WP-A3 (disk ceiling).

**Personas:** P9 (malformed/quarantine + row-delta reconcile), P10 (duplicate re-sync idempotency), P16 (late arrival × rebuild × atomic re-export), P17 (edge-filter conservation), P5 (boundary straddler xfail).

**Questions:** Q-15 (retention), Q-19 (run cadence), Q-22 (fleet-volume load gate, deferred).

Every item above is addressed either in FINDINGS or in the COVERAGE STATEMENT.

---

## 2. FINDINGS

### FINDING-DE.1 · severity: blocker · target: R5.1 post-load assertion "`observations` row delta == sum of ledger row_counts for the batch"

**steelman.** This assertion is the load-bearing integrity check of the whole ingest design and it is genuinely well-motivated. D-021's rationale is that "every major breakage was a silent contract violation, not a code bug"; R5's ENFORCED-BY makes idempotency real with a runtime equality check rather than trust. The two-table split (R5.1 rationale) deliberately lands raw first so failures are *inspectable and countable* — an assertion that the counts reconcile is exactly the "nothing silently vanished" guarantee (CONSUMER MAY ASSUME, R5.1). It is the right instinct: make conservation checkable at runtime, not just asserted in prose.

**failure_scenario.** A batch of 3 files lands: file A = 10,000 rows, file B = 8,000 rows, file C = 5,000 rows. The ledger records `row_count` at the **landing** step (R5.1: "each file lands exactly once, tracked in an `ingest_ledger` (filename, size/etag, row_count, ingested_at)"), so ledger sum = 23,000. Enrichment then runs: 40 rows in file A have bad timestamps, 12 rows in file B have missing power, 3 rows in file C carry an internal `agent_id` that disagrees with the folder. Per R5.1 and P9 these go to the **quarantine table with reason + count** — they are removed from the `observations` promotion. So `observations` delta = 23,000 − 55 = 22,945. The assertion `observations delta == sum ledger row_counts` = `22,945 == 23,000` → **FALSE**. Every run with even one malformed row fails the assertion. The pipeline is `halt-on-failure via exit code` (D-044), so a non-zero exit fires `notify()` and the *entire per-agent chain aborts* — meaning quarantine (a designed, expected, non-error path per D-021/P9) crashes the pipeline. The operator gets a page every run in which any of 60 sensors emitted one dirty row, which — given the documents' own repeated statements that "raw data is dirty/inconsistent in ways transform-on-read can't handle" (R5.1 rationale) — is essentially every run.

**evidence.** DATA-WORKFLOW-RULES-v3 line 169: "Post-load assertion: `observations` row delta == sum of ledger row_counts for the batch." Line 159: enrichment sends malformed rows to "quarantine table with reason + count." P9 (GOLDEN-FIXTURE-SPEC line 24): "enrichment quarantines each with reason + count … `observations` row-delta reconciles exactly." Note P9 says the row-delta "reconciles exactly" but does **not** state the reconciliation formula — the prose in R5.1 line 169 gives a formula (`== sum ledger row_counts`) that quarantine breaks. The two documents contradict on the arithmetic. What is *absent*: any term for `quarantined_count` in the assertion. The correct invariant is `observations_delta + quarantined_delta == sum(ledger row_counts)`, and this three-term form appears nowhere in the contracts. (The whole-run "Conservation" assertion in the fixture spec line 39 *does* get it right — "input rows = observations + quarantined + suppressed-summarized" — which makes the two-term R5.1 formula demonstrably the wrong one.)

**proposed_alternative.** Restate R5.1 ENFORCED-BY as: `observations_delta + quarantined_delta == Σ ledger.row_count` for the batch, and record `quarantined_count` (with reason breakdown) on the ledger row or a per-run reconciliation record so the equality is checkable. Decide explicitly that quarantine is a **non-halting** outcome: a run with quarantined rows exits 0 but surfaces the quarantine count in the run report and only alerts if quarantine exceeds a threshold ratio (mirroring the over-filter safeguard's design). Add a fixture whole-run assertion binding the three-term identity to P9's numbers.

**downstream_impact.** R5 ENFORCED-BY row in the enforcement summary table (line 260); P9 expected output wording; D-044 halt-on-failure semantics (needs a "quarantine is not failure" carve-out); RUNBOOK quarantine-review discipline; the alerting policy (D-049) needs a quarantine-ratio emitter.

**confidence.** high

---

### FINDING-DE.2 · severity: blocker · target: R4/R5 ledger uniqueness "keyed on filename + size/etag" under re-upload with changed etag

**steelman.** Keying idempotency on `filename + size/etag` is a defensible, common S3 pattern. R4's NEVER ("an object is never mutated in place; re-upload of changed content uses a new key or is treated as an error") plus R3's write-once guarantee ("a filename is never reused with different content") mean that under the *designed* agent behavior, filename alone is nearly sufficient and size/etag is belt-and-suspenders. D-045's ephemeral uploader makes this cleaner: `pending/` files are write-once and rotated before upload, so a given filename maps to one immutable content by construction. The etag component defends against the one degenerate case where a file is somehow re-PUT with different bytes. The reasoning chain is coherent.

**failure_scenario.** Two independent mechanisms break the "same filename ⇒ same content ⇒ same etag" assumption the ledger relies on:

(1) **Multipart etag instability.** R2 (and S3) compute the etag of a **multipart** upload as a hash-of-part-hashes plus part count — it is *not* the MD5 of the object, and it **changes if the part size changes**. The agent rotates at 50 MB (R3 line 91); gzipped these are usually single-part, but boto3's default multipart threshold is 8 MB, so any object over the threshold gets a multipart etag. If `ear-agent` v5 and a later `ear-agent` v5.1 use different boto3 versions or a different `multipart_chunksize` (D-051 fleet updates change agent versions facility-by-facility), the **same file re-uploaded after a config-render** produces a **different etag for identical bytes**. The R5 anti-join (`list → anti-join ledger → land new only`) sees a "new" object, re-lands it, and every downstream count for that agent inflates — the exact failure R5 exists to prevent ("A file is never ingested twice (would inflate every downstream count)", line 165).

(2) **Re-upload after a partial-visibility window.** D-045 says "`pending/` IS the retry queue … a failed run exits leaving files." Suppose a PUT succeeds at R2 but the *confirmation* is lost (WAN drop after the object is durable, before the 200 is received). The next timer tick re-PUTs the same file. If the object was multipart and the retry uses a different part boundary (different available memory, different chunk config), the re-PUT **overwrites** the object with a *different etag but identical logical content*. If the sync ran between the two PUTs, the ledger holds etag-1; the re-PUT installs etag-2; a later re-sync (or a size-mismatch skip check, R5.1 "skip-if-exists" / "size-mismatch") re-downloads and the anti-join treats it as new.

**evidence.** DATA-WORKFLOW-RULES-v3 line 120: "R5 keys on filename+size/etag." Line 157: "list → anti-join ledger → land new only." Line 116: "Sync downloads only missing/size-mismatched objects (skip-if-exists)." The contract never states which etag semantics R2 provides, never pins the boto3 multipart threshold/chunksize as a provisioning constant, and never says whether the anti-join key is `filename` alone, `filename+size`, or `filename+size+etag` — the three behave differently under the scenarios above. RECOVERY-RETENTION line 11 ("at worst re-ingest is idempotent by file identity") asserts the property but "file identity" is never pinned to a stable definition.

**proposed_alternative.** Make the ledger idempotency key **`(agent_id, filename)` primary**, with size and etag as *change-detection* fields, not part of the uniqueness key. Rule: if a listed object matches an existing ledger `(agent_id, filename)`, it is **already ingested — skip** (write-once per R3 guarantees content identity), and if size/etag *differ*, that is a **contract violation → quarantine + alert** (R4 NEVER: re-upload of changed content "is treated as an error"), never a silent re-land. Pin `multipart_threshold` and `multipart_chunksize` as fleet-wide provisioning constants (ear-fleet) so etag is at least stable across a fixed agent version. Add a fixture persona extending P10: re-deliver an already-ingested file with a *different etag but identical content* → assert skip + flag, not re-ingest.

**downstream_impact.** R4 NEVER clause; R5.1 PRODUCER GUARANTEES (skip-if-exists / size-mismatch wording); IngestLedger model (`RFANALYSIS-SCOPE-v3` line 36: "size/etag"); P10 persona; D-051 fleet-update procedure (boto3 pinning becomes a release-note requirement); the whole downstream count integrity (R6/R7/R8) since a double-land inflates everything.

**confidence.** high

---

### FINDING-DE.3 · severity: major · target: D-023 / D-024 — atomic partition "rename" semantics on the actual filesystem, and the temp-dir/rename correctness boundary

**steelman.** Atomic-rename-into-place is the correct, battle-tested pattern for making a directory of Parquet files appear atomically, and D-024's rationale is sound: "no writer contention; temp-write+rename." EVENT-DATASET-HANDOFF line 78 spells out the discipline precisely — "a chain writes to a temp directory and renames into place; a re-run replaces the whole partition atomically … Orphaned temp dirs are cleaned at chain start." RECOVERY-RETENTION line 10 correctly derives that partial exports are "impossible to observe." On a single POSIX filesystem, `rename(2)` of a file is atomic, and the design's disposability (delete-and-rebuild) means even a botched export is cheaply recoverable. This is a genuinely strong recovery story.

**failure_scenario.** The atomicity claim quietly assumes the wrong `rename` granularity. The partition is a **directory** containing *two* files: `events.parquet` **and** `event_gaps.parquet` (handoff-events.v1.columns.json: "events.parquet (+ event_gaps.parquet)"). POSIX `rename(2)` is atomic **per path**, not across two files. Two failure shapes follow:

(1) **Directory rename over an existing target.** A re-run "replaces the whole partition atomically" (line 78) — but `rename(2)` of a *directory* onto an **existing non-empty directory** fails with `ENOTEMPTY`/`EEXIST` on Linux; it is not an atomic replace. So the implementer must instead delete-then-rename (a non-atomic two-step: a reader globbing mid-window sees the target **gone**, then **half-new**) or rename each file individually (two separate atomic operations, not one). Either way the *directory*-level atomicity the contract promises does not exist as a single syscall. A P16 late-arrival re-export (which "updates deterministically … no orphaned partial state") that runs concurrently with an L2 glob can expose L2 to a partition that has `events.parquet` (new) but `event_gaps.parquet` (old) — a **torn multi-file partition**, exactly the half-written state the contract says is impossible.

(2) **Rename across filesystems is not atomic.** DEV-ENVIRONMENT-v2 puts `/srv/ear/handoff/` and `/srv/ear/l1/` under the same NVMe, but production (Q-11 open: "managed or in-compose," Render/Hetzner) may mount the handoff volume separately, or a container bind-mount may cross a device boundary. If the temp dir is on a different device than the final path, `rename(2)` returns `EXDEV` and the runtime falls back to copy+delete — **non-atomic**, and a reader can glob the target mid-copy. The contract says "renames into place" as if it is always one device; it never states the invariant "temp dir and final partition MUST be on the same filesystem."

**evidence.** EVENT-DATASET-HANDOFF-v1 line 78: "writes to a temp directory and renames into place; a re-run replaces the whole partition atomically. Level 2 must never be able to glob a half-written partition." handoff-events.v1.columns.json line: two files per partition. RECOVERY-RETENTION line 10: "Partial Parquet export: impossible to observe (atomic temp-write+rename)." What is *absent*: (a) any statement that temp and final must share a filesystem; (b) any handling of the two-file torn-read; (c) any reader-side guard (a manifest/marker file the glob checks) — the design relies purely on rename atomicity that does not hold at directory granularity.

**proposed_alternative.** Adopt a **marker-file commit protocol** instead of relying on directory rename: write both Parquet files into the temp dir, then write a `_SUCCESS`/manifest file *last*, then move the whole set; L2's glob selects only partitions containing the marker (this is the Spark/Hive `_SUCCESS` convention and it is robust to torn multi-file reads and to EXDEV). Alternatively, make the partition a **single file** (one Parquet with both event and gap data, or gaps always L2-derived per the handoff contract's "optional/derived" clause) so per-file `rename(2)` *is* the whole partition. Add the invariant to the contract: "temp dir and final partition are on the same filesystem (enforced at job start; abort if not)." Add a fixture assertion: interleave an L2 glob with a P16 re-export and assert L2 never sees a partition with mismatched `events`/`event_gaps` run_ids.

**downstream_impact.** D-024; EVENT-DATASET-HANDOFF ENFORCED-BY; WP-S9 (atomic export smoke test must cover the torn-read and EXDEV cases, not just happy path); P16 persona (add concurrency); RECOVERY-RETENTION line 10 (weaken the "impossible to observe" claim or back it with the marker protocol); Q-11 deployment (filesystem layout becomes a provisioning constraint).

**confidence.** high

---

### FINDING-DE.4 · severity: major · target: D-023 — "peak footprint = pool size × memory_limit; file count is irrelevant at rest" (RAM ceiling under fleet scale)

**steelman.** The claim is elegant and mostly correct. D-023's core insight is real: because Level 1 has *zero cross-agent operations*, per-agent files convert a single-writer bottleneck into embarrassingly-parallel chains, and open→work→close means only the *actively processing* agents hold memory. RFANALYSIS-SCOPE line 45 states the tunable model crisply: "Peak footprint = pool size × memory_limit (the only two scaling tunables; file count is irrelevant at rest)." This is genuinely better than a shared-DuckDB design where 60 agents contend for one writer, and it makes the recovery/disposability story (delete-and-rebuild one file) trivial. The steelman is strong: the scaling *knobs* are correctly identified.

**failure_scenario.** "File count is irrelevant at rest" is true; "`memory_limit` × pool size is the peak" is **not the actual peak**, and the gap bites exactly at fleet scale — which is deferred (Q-22) with no load gate before the first facility. Three uncounted terms:

(1) **DuckDB `memory_limit` is a soft target for the buffer manager, not a hard RSS cap.** It bounds the buffer pool but not the full process RSS: query operator scratch, the vector-execution intermediate state, the Parquet writer's row-group buffers, and DuckDB's own metadata all live *outside* the `memory_limit` accounting. A pass over a large-window agent that spills can still exceed `memory_limit` in RSS by a meaningful factor. With pool 4 × 4 GB the doc *believes* peak is 16 GB; real peak can be 1.5–2× that under a heavy agent.

(2) **The pipeline is a `docker compose run --rm` job (D-044).** Each pooled worker is a process holding its own DuckDB instance *plus* the Python/Django import footprint (the same image as `web`, gunicorn stack loaded). At 4 workers that overhead is trivial; the doc's own headroom note says "~10 × 8 GB" (DEV-ENVIRONMENT line 23) = 80 GB of `memory_limit` budget on a 128 GB box, at which point the *un-budgeted* per-worker Python + DuckDB-overhead + writer-buffer terms are no longer negligible and the "128 GB is fine, NVMe is the constraint" conclusion (D-047) is unproven.

(3) **The multi-day full-rebuild window (see DE.5) makes each agent's working set grow over time**, so `memory_limit` that was safe at facility-launch quietly becomes the OOM knob months later — and OOM of one worker under `compose run --rm` is a non-zero exit → whole-run halt (D-044) → the whole fleet's run fails because one agent's window got big.

**evidence.** RFANALYSIS-SCOPE-v3 line 45: "Peak footprint = pool size × memory_limit (the only two scaling tunables; file count is irrelevant at rest)." DEV-ENVIRONMENT-v2 line 23: "start 4 × 4 GB, headroom to ~10 × 8 GB." Q-22 (OPEN-QUESTIONS line 31): "fleet-volume load gate … Not a concern at this stage; revisit before a facility depends on run cadence." What is *absent*: any acknowledgment that `memory_limit` ≠ RSS; any per-worker overhead term; any statement of the actual per-agent working-set size as a function of window length × sensor volume. The scaling model is stated as exact ("the only two") when it is a lower bound.

**proposed_alternative.** Restate the ceiling as `peak ≈ pool_size × (memory_limit × spill_factor + python_worker_overhead)` and require the WP-S5/Q-22 benchmark to **measure** RSS-at-`memory_limit` for a representative heavy agent, not assume it. Set `memory_limit` from measured RSS with a safety margin, and add a per-worker `--memory=` cgroup cap on the pool processes so a runaway agent is killed at a known bound (converting a whole-run OOM into a single-agent quarantine-and-continue). Un-defer the *narrow* version of Q-22: a single-agent worst-case memory probe is cheap and must run before first facility even if the full fleet-scale gate stays deferred. Make one heavy agent's OOM **not** halt the other 59 (see DE.6).

**downstream_impact.** D-023 tunable statement; DEV-ENVIRONMENT sizing; Q-22 (partial un-defer); D-044 halt-on-failure granularity (couples to DE.6); PARAMETER-TUNING (memory_limit becomes a measured, versioned parameter).

**confidence.** high

---

### FINDING-DE.5 · severity: major · target: R7 "each run is a full rebuild over the configured window" × P16 late-arrival (rebuild cost as windows grow)

**steelman.** Full-rebuild-per-run is the single most valuable simplification in the design and its rationale is excellent. R7 line 202 ("Two runs on identical data+config produce identical events") gives determinism for free; PARAMETER-TUNING line 11 shows it is the precondition for honest A/B ("A run is fully reproducible and fully attributable to one parameter set"); RECOVERY-RETENTION line 6 shows it is *why* recovery is delete-and-rebuild. P16 leverages it beautifully: a late file just triggers a re-run and events "update deterministically … no orphaned partial state." There is no incremental-merge bug surface because there is no incremental merge. This is the right instinct and should largely be protected.

**failure_scenario.** "Full rebuild over the configured window" has an unbounded cost term that the documents never bound. Consider the mechanics for one agent as the deployment ages:

- The handoff retention is **indefinite** (RECOVERY-RETENTION line 25: "Handoff Parquet — indefinite — it IS the dataset"), and raw landing is 90 days (line 23).
- P16 says a late file causes the window's events to rebuild. A WAN-outage backlog (L5: "delayed by hours or days") delivers files with **old timestamps**. If the "configured window" is anchored to observation time (it must be, for events to be correct), a file backdated 8 days forces a rebuild spanning **at least** 8 days for that agent.
- Each rebuild is a full pass over that agent's raw landing for the window. At a busy sensor (P3-class infrastructure alone is ≥500 obs/hour before filtering; a real warehouse sensor pre-filter is far higher), a multi-day window is tens of millions of rows. That pass runs **every scheduled run** (Q-19: possibly hourly) for the entire duration the window stays "configured," not just once when the late file lands.
- Because the rebuild is per-agent and the run halts on any agent's failure (D-044), the slowest agent's window sets the whole run's wall-clock, and the run must finish before the next timer tick (Q-19 cadence) or runs overlap (blocked by the run lock, so the *next* run is skipped — cadence silently degrades, and dashboard freshness with it).

The document acknowledges this obliquely — the review brief's own P16 note and Q-22 — but the *cost model* of "how big does the rebuild window get, and what happens when rebuild time exceeds run cadence" is nowhere written.

**evidence.** DATA-WORKFLOW-RULES-v3 line 198: "Each run is a full rebuild over the configured window (deterministic given data + config)." P16 (GOLDEN-FIXTURE line 31): "the window's events update deterministically (full-rebuild semantics)." RECOVERY-RETENTION line 23/25: landing 90 d, handoff indefinite. L5 (LOCAL-HOST-DATA-FLOW line 227): "delayed by hours or days." What is *absent*: (a) a definition of "configured window" (rolling N days? since-last-handoff? anchored to which clock?); (b) any statement of rebuild cost as a function of window × volume; (c) any mechanism to *bound* the rebuild to the affected date-partitions rather than the whole window when a late file lands; (d) the interaction between rebuild wall-clock and run cadence (Q-19).

**proposed_alternative.** Define "configured window" precisely and make it a **rolling bounded window** (e.g. last N days, N tied to the 90-day landing retention and the burst-gap horizon), so rebuild cost is bounded regardless of deployment age. For late arrivals, exploit the **date partitioning already in the handoff path** (`handoff/{facility}/{agent_id}/{date}/`): rebuild only the affected date partition(s) plus a burst-gap-sized boundary margin, not the whole window — the events for `date=D` depend only on observations in `[D − burst_gap, D + burst_gap]`, so a late file for day D re-exports at most 1–2 partitions atomically (which is exactly what D-024's per-date partition is *for*). Add a run-report metric: rebuild rows and wall-clock per agent, alert if a single agent's chain exceeds a fraction of the run cadence (early warning that the window model is outgrowing the schedule). Make WP-S5/Q-22 measure rebuild time at a realistic window, not just Pass-2 budget.

**downstream_impact.** R7 PRODUCER GUARANTEES (window definition); D-024 partition granularity (now load-bearing for incremental rebuild, not just handoff medium); P16 (must assert only-affected-partition re-export, not whole-window); Q-19 cadence choice; Q-22 load gate; the run-lock skip behavior (RECOVERY-RETENTION line 12).

**confidence.** high

---

### FINDING-DE.6 · severity: major · target: D-044 halt-on-failure granularity vs D-023 per-agent isolation (one agent's failure halts all 60)

**steelman.** Halt-on-failure via exit code is the correct default for a batch job with a single operator: D-044's rationale ("run-to-completion gives clean memory, pinned image per run, fewer moving parts") is exactly right for a periodic forensic pipeline, and mapping failure to a non-zero exit that pages via `notify()` (D-049) is the minimum-moving-parts choice. Fail-loud beats fail-silent, which is the entire ethos of the contracts (R0.3). For a first-facility deployment with a handful of sensors, halting the whole run on any error is a perfectly reasonable, debuggable posture.

**failure_scenario.** D-023 sells per-agent isolation as delivering fault-containment: "a wedged/corrupt file affects one agent and is disposable" (D-023). But D-044's execution model **undoes** that containment at the job level: "Non-zero exit = alert; next scheduled run retries" and per-agent stages parallelized in a process pool with "halt-on-failure via exit code." So the actual behavior of one agent's chain throwing (a corrupt DuckDB file, an OOM per DE.4, a quarantine-triggered assertion per DE.1, one unreadable Parquet during export) is: the pool worker exits non-zero → the job exits non-zero → **the whole run is a failure** → the next timer tick re-runs **all 60 agents** from the top. Concretely: `cht-rf3b`'s L1 file corrupts. The physical-isolation promise says only `cht-rf3b` is affected. The execution reality says the entire fleet's run fails, no handoff is produced for the 59 healthy agents that run, and the operator is paged for a fleet-wide outage caused by one disposable file. Worse, if that one file corrupts *deterministically* (e.g. a poison input row that the enrichment can't cast and the assertion halts on), **every** retry fails at the same agent, and the fleet never produces a handoff until a human intervenes — the "next run retries" self-heal never fires.

**evidence.** D-023 (DECISION-LOG line 26): "a wedged/corrupt file affects one agent and is disposable." D-044 (line 52): "Non-zero exit = alert; next scheduled run retries … completion-gating is function order." RFANALYSIS-SCOPE line 58: "halt-on-failure via exit code." RECOVERY-RETENTION line 8: "the next run's per-agent stages are full rebuilds … Just re-run" — assumes re-run fixes it, silent on the deterministic-poison case. What is *absent*: any statement of whether one agent's failure aborts the pool or is contained; any per-agent success/failure accounting in the run; any "produce handoff for the agents that succeeded" partial-success mode. The two decisions state opposite fault models and are never reconciled.

**proposed_alternative.** Make the pool **per-agent fault-isolating**: a worker failure quarantines *that agent's* run (records agent-level status on `AnalysisRun`, skips its handoff export, alerts with the agent id) and the job continues the other agents; the job's exit code reflects "N of 60 agents failed" (non-zero so it still pages, but the 59 healthy agents still export their handoff and the dashboard still updates). Add a **per-agent failure counter**: after K consecutive failures for the same agent, auto-disable it (Sensor status) and page — this converts the deterministic-poison infinite-fail into a bounded, self-quarantining outcome. Keep whole-run halt only for *shared-resource* failures (Postgres down, R2 unreachable, disk full), which genuinely affect everyone. Reconcile the wording in D-023 and D-044 so the fault model is stated once.

**downstream_impact.** D-023 and D-044 (reconcile fault model); RFANALYSIS-SCOPE run_level1 spec; AnalysisRun model (needs per-agent status, not just run status); D-049 alert emitters (per-agent vs whole-run); RECOVERY-RETENTION (deterministic-poison case); HEALTH-CHECKS run-success check (WP-S4) must distinguish "run partially succeeded" from "run failed."

**confidence.** high

---

### FINDING-DE.7 · severity: major · target: D-042 "DuckDB computes, Postgres serves" — the results-persistence hop (R8) is unspecified as a data-engineering transform

**steelman.** The split is architecturally clean and single-operator-friendly: D-042's rationale ("Pegasus already supplies auth/admin/Celery/UI; L2 is batch; a second service duplicates everything for nothing") is sound, and the read-path discipline ("Django dashboard renders exclusively from Postgres, never queries DuckDB for a page") is exactly right — it keeps page latency off the analytical store and gives the dashboard a stable, indexed serving layer. R8's FK/provenance guarantees (every candidate → a run + config) are strong. The *shape* of the split is correct.

**failure_scenario.** The **write** side of the split — how DuckDB-computed results get *into* Postgres — is nowhere specified, and it is a classic dual-write consistency hole. D-055 says L2 "persists to `DeviceClassification`/`TrackerCandidate`/`CorrelatedDetection`" in Postgres, computed in the L2 DuckDB. That is a cross-store copy: read Parquet → compute in DuckDB → write rows to Postgres. Failure modes the contracts never address: (1) the L2 job computes 5,000 candidates, writes 3,000 to Postgres, then crashes — Postgres now holds a **partial** result set with no marker that it is partial, and the dashboard (which "renders exclusively from Postgres") shows a truncated analysis as if complete. (2) A re-run of L2 over the same handoff must **replace** the prior run's Postgres rows, but R8 only guarantees FK linkage to `AnalysisRun` — it never says the old run's candidates are superseded/deleted, so stale candidates from run N coexist with run N+1 unless something (unspecified) reconciles them. (3) There is no transaction spanning "L2 DuckDB computed" and "Postgres written," so a crash between them leaves DuckDB with the truth and Postgres serving nothing or stale — the opposite of the "Postgres serves" guarantee. RECOVERY-RETENTION addresses L1 recovery in detail but says nothing about L2 result recovery beyond "Postgres restore per BACKUP-DR."

**evidence.** D-042 (DECISION-LOG line 48): "persists only results … to Postgres; Django dashboard renders exclusively from Postgres." D-055 line 71: "(5) persistence to `DeviceClassification`/`TrackerCandidate`/`CorrelatedDetection`." R8 (DATA-WORKFLOW-RULES line 216): "Every candidate is linked to its `AnalysisRun`." What is *absent*: the write transaction boundary; the run-supersession/upsert semantics (does a new run delete prior candidates for the same window?); a "results complete" marker distinguishing a partial write from a finished one; any idempotency key for L2 result rows (analogous to the L1 ledger). The entire L2 result-persistence hop has no ENFORCED-BY of the kind R5–R8 demand of L1.

**proposed_alternative.** Specify the L2 write as **transactional per-run replace**: L2 writes all results for a run inside one Postgres transaction that (a) inserts the new `AnalysisRun`/results, (b) marks the prior run for the same `(facility, window)` superseded, (c) commits atomically — the dashboard filters to `AnalysisRun.status='current'` so it never renders a partial or stale set. Add a "results_complete" flag on `AnalysisRun` set only in the final commit. Give L2 results an idempotency key `(run_id, agent_id, earfcn, power_bin)` so a re-run is a clean upsert. Extend R8 ENFORCED-BY with these. Add an L2 recovery clause to RECOVERY-RETENTION: L2 results are rebuildable from the handoff Parquet (which is sacred/indefinite) — so an L2 crash is delete-current-run + re-run, symmetric with L1 disposability.

**downstream_impact.** D-042; D-055 persistence scope; R8 PRODUCER GUARANTEES + ENFORCED-BY; AnalysisRun model (status/completeness); RECOVERY-RETENTION (add L2 row); DASHBOARD-DESIGN (queries filter to current run); the dashboard's honesty guarantee (a partial analysis must never look complete).

**confidence.** med

---

### FINDING-DE.8 · severity: minor · target: D-023 open→work→close claim of "parallelism" vs open/close overhead at high pool counts

**steelman.** Open→work→close (no long-lived cached connections) is the right discipline and D-023 correctly identifies why: it bounds memory to *active* agents and prevents connection-state leakage across tasks — enforced in the stage base class (RFANALYSIS-SCOPE line 44). For a forensic batch pipeline the per-file open cost is amortized across a multi-stage chain, so the overhead is genuinely small relative to the work. The isolation win (one file per agent, disposable) is real and worth a modest open/close tax.

**failure_scenario.** The design opens and closes a DuckDB file **per task, per agent** across a 7-stage chain (`sync → land → enrich → bin+reconcile → filter → events → export`, RFANALYSIS-SCOPE line 60), and RFANALYSIS-SCOPE line 43 says "each task: open its file → work → close (no long-lived cached connections across tasks)." If "task" = stage, that is up to 7 open/close cycles per agent per run × 60 agents = 420 DuckDB file-open operations per run, each of which for DuckDB involves reading the file header, WAL replay check, and catalog load. At small scale this is noise; but combined with DE.5 (large windows) and per-agent files that grow, the fixed open cost per stage stops being free — and the claimed benefit (parallelism) is capped by pool size (4), so 60 agents are processed in **15 serial waves** regardless. The "parallel per-agent chains" language (RFANALYSIS-SCOPE line 41) can be misread as 60-way parallelism when it is really 4-way with 56 agents queued. This is a documentation-honesty issue more than a performance blocker, but it feeds the scale-optimism that DE.4/DE.5 flag.

**evidence.** RFANALYSIS-SCOPE-v3 line 43–44: "each task: open its file → work → close … enforce in the stage base class." Line 41: "parallel per-agent pipeline chains (process pool, D-044)." Line 60: 7-stage sequence. D-044: pool size is the throttle. What is *absent*: whether "task" is stage-granular or agent-granular (i.e. does the file open once per agent-chain and close at chain end, or open/close per stage?), and any statement that effective parallelism = pool size, not agent count.

**proposed_alternative.** Clarify: open the agent's DuckDB **once per agent-chain**, hold it across the 7 stages, close at chain end — one open/close per agent per run (60 opens, not 420). This preserves the "no long-lived connection *across tasks*" intent (a task = one agent's whole chain, not one stage) while eliminating per-stage reopen overhead. State explicitly in D-023/RFANALYSIS-SCOPE that effective concurrency = pool size and agents beyond that queue, so the "parallel chains" language is not read as unbounded parallelism.

**downstream_impact.** RFANALYSIS-SCOPE stage base class spec; D-023 wording; PARAMETER-TUNING (open cost is negligible only under the once-per-chain reading).

**confidence.** med

---

### FINDING-DE.9 · severity: minor · target: Retention interaction — raw landing 90 d vs handoff indefinite vs full-rebuild reproducibility

**steelman.** The retention table (RECOVERY-RETENTION lines 18–31) is thoughtfully tiered and each choice has a clear rationale: L1 files disposable (cache), handoff indefinite (it IS the dataset), raw landing 90 d rolling (bounded storage). PARAMETER-TUNING line 11 correctly leans on the landing table for A/B ("the same source data can be re-processed under different parameters without re-fetching"). The tiering is coherent and storage-conscious.

**failure_scenario.** Two retention horizons quietly conflict with the reproducibility/rebuild guarantees. (1) **Raw landing is pruned at 90 days (RECOVERY-RETENTION line 23), but R2 signal feed is retained 13 months and handoff is indefinite.** So a handoff partition older than 90 days can **no longer be rebuilt from the landing table** — its rebuild path is "delete L1 file → re-sync from R2 → re-land → re-enrich" (still possible, since R2 obs is 13 months), but PARAMETER-TUNING's promise that A/B can re-process "the same source data … without re-fetching from R2" (line 11) **only holds for 90 days**. An A/B test comparing a parameter change against 6-month-old events must silently re-fetch from R2 (slower, and only if R2's 13-month window still covers it) — the landing-table optimization the doc sells is time-bounded in a way the A/B framework doc doesn't state. (2) **Handoff is indefinite but the config that produced it is snapshotted on `AnalysisRun` in Postgres** (R8) — if a handoff partition outlives its `AnalysisRun` provenance (e.g. a Postgres restore that loses runs older than the pg_dump retention, or a run pruned), the handoff Parquet carries `run_id`/`config_version` strings that point to nothing, breaking the "every event → a valid AnalysisRun" FK guarantee (EVENT-DATASET-HANDOFF line 80). Indefinite data + finite provenance = dangling provenance.

**evidence.** RECOVERY-RETENTION line 23 (landing 90 d), line 21 (R2 obs 13 months), line 25 (handoff indefinite), line 26 (Postgres runs "indefinite (small)"). PARAMETER-TUNING line 11: "re-processed … without re-fetching from R2." EVENT-DATASET-HANDOFF line 80: "every event → a valid AnalysisRun." What is *absent*: a statement that landing-table A/B is a 90-day capability; any guarantee that `AnalysisRun` provenance outlives the handoff it describes (they have *different* retention: handoff indefinite, runs "indefinite (small)" — consistent only if runs are truly never pruned, which the pg_dump 30 d/180 d backup horizons don't guarantee under restore).

**proposed_alternative.** State the landing-table A/B window explicitly (90 d) and note that older A/B requires R2 re-fetch (bounded by the 13-month R2 obs retention — beyond which reproducibility is *lost*, which should be a conscious Q-15 decision, not implicit). Guarantee `AnalysisRun` provenance is retained **at least as long as any handoff partition it is referenced by** — i.e. runs are truly never pruned while their handoff exists, and this is enforced (not just "indefinite (small)"). Consider stamping the *resolved config* into the handoff Parquet metadata itself (not only Postgres), so the dataset is self-describing and survives Postgres loss — the handoff already carries `config_version`; carrying the full config snapshot makes it self-contained.

**downstream_impact.** RECOVERY-RETENTION retention table; Q-15 (retention confirmation should surface the reproducibility-horizon tradeoff); PARAMETER-TUNING A/B window; EVENT-DATASET-HANDOFF provenance guarantee; BACKUP-DR (Postgres restore must not orphan handoff provenance).

**confidence.** med

---

### FINDING-DE.10 · severity: minor · target: R5.0 onboarding gate — `qa` isolation is "an isolated staging area" but the per-agent-file/handoff-glob model makes isolation structural only if paths differ

**steelman.** The onboarding gate is a genuine strength and closes a real, evidenced gap ("this gate did not exist as code … a new folder would auto-enter production," R5.0 INTENT vs ACTUAL). The structural-isolation argument is elegant: EVENT-DATASET-HANDOFF line 18 notes "QA isolation is literal (a `qa` agent's partition is simply not globbed by production Level 2)." P18 tests it. The `pending→qa→active→disabled` lifecycle is clean.

**failure_scenario.** The isolation is only as structural as the **glob pattern discipline**, and the docs describe two different isolation mechanisms that must agree: (1) R5.0 says qa runs "in isolation (staging context)" / "a separate `qa` path to an isolated staging area" (line 147) — implying a *separate directory*; (2) EVENT-DATASET-HANDOFF says qa isolation = "not globbed by production Level 2" — implying qa partitions live in the **same** `handoff/{facility}/{agent_id}/{date}/` tree but L2's glob filters them out by consulting the Sensor registry. These are different: under (2), the isolation is a **runtime filter**, not a structural path separation — if L2's glob is `handoff/{facility}/*/{date}/events.parquet` (glob by facility, per WP-L1 line 69 "glob by facility") and *then* filters agent_ids by Sensor.status='active', a bug or race (agent flips qa→active→qa during a run; or the filter is forgotten in one query path) leaks qa data into production analysis. The handoff export itself (RFANALYSIS-SCOPE line 63: "qa agents run in isolation, excluded from production correlation/reports") doesn't say qa handoff goes to a *different path* — so the "literal" structural isolation depends on an unstated path convention.

**evidence.** DATA-WORKFLOW-RULES line 147: "a separate `qa` path to an isolated staging area." EVENT-DATASET-HANDOFF line 18: "not globbed by production Level 2." WP-L1 (GREENFIELD line 69): "glob by facility." RFANALYSIS-SCOPE line 63: qa "excluded from production correlation/reports." What is *absent*: a single authoritative statement of *where* qa handoff partitions physically live — same tree filtered by status, or a separate `handoff-qa/` tree. The two contracts imply different mechanisms and P18 tests the outcome without pinning the mechanism.

**proposed_alternative.** Make qa isolation **path-structural**, not filter-based: qa agents export to `handoff-qa/{facility}/{agent_id}/{date}/` (or `handoff/{facility}/_qa/…`), and production L2's glob physically cannot reach it (`handoff/{facility}/{agent_id}/…` excludes the qa tree by pattern, not by a status lookup). This makes "not globbed by production" true by path, matching the "literal/structural" claim, and removes the qa→active→qa race. Pin the path convention in EVENT-DATASET-HANDOFF and R5.0. Keep P18 but assert the *path*, not just the absence in output.

**downstream_impact.** R5.0; EVENT-DATASET-HANDOFF partition-path contract; WP-L1 glob spec; P18 assertion; the Sensor lifecycle transition handling (qa↔active must move/republish partitions if path-structural).

**confidence.** med

---

### FINDING-DE.11 · severity: minor · target: observation-signal.v5.schema.json vs D-052 / R1 field-name drift (`observed_at` vs `time`)

**steelman.** Pinning canonical field names in JSON Schema and making the schema "the tie-breaker over prose" (D-052) is exactly the right anti-drift discipline — it is the machine-readable enforcement R0.3 demands, and it directly attacks the `time_bucket`-class bug that motivated the whole rules effort. The schemas are well-formed and `additionalProperties:false` on the signal record correctly rejects unknown fields.

**failure_scenario.** The contracts and the machine-readable schema **disagree on the timestamp field name**, and D-052 declares the schema wins — which silently invalidates prose the build will follow. R1 PRODUCER GUARANTEES (DATA-WORKFLOW-RULES line 43) says the scanner "emits exactly these fields: `time`, `agent_id`, …" and R1 NEVER (line 53) says "`time` is never the write time." But observation-signal.v5.schema.json requires **`observed_at`**, not `time`, and D-052 says legacy names including `time` "are NOT valid v5." So R1's own field list uses a name the v5 schema rejects. An implementer following R1's prose ("emits exactly these fields: `time`…") writes an agent that emits `time`; the schema (`additionalProperties:false`, requires `observed_at`) **rejects every row to quarantine**. The tie-breaker rule (D-052) means the schema is right and R1's prose is wrong — but R1 is the contract an agent author reads first (it is *the* producer guarantee for the record). Same drift exists for `power`/`power_dbm` and `agent`/`agent_id` but those happen to align; the timestamp one does not. Additionally, R1 lists `direction` and `duty_cycle` as required-emitted fields, but the signal schema makes `duty_cycle` optional and omits `direction` entirely — three field-set disagreements between the prose contract and the authoritative schema.

**evidence.** DATA-WORKFLOW-RULES line 43: "Emits exactly these fields: `time`, `agent_id`, `band`, `earfcn`, `frequency_mhz`, `power_dbm`, `direction`, `observation_count`, `filter_action`, `duty_cycle`, `schema_version`." observation-signal.v5.schema.json `required`: `["schema_version","observed_at","agent_id","earfcn","frequency_mhz","power_dbm","band","filter_action","observation_count","config_version"]` — has `observed_at` (not `time`), lacks `direction`, adds `config_version`, makes `duty_cycle` optional. D-052 (DECISION-LOG line 65): "Legacy names (`time`, `power`, `agent`) are NOT valid v5 … Schemas are the tie-breaker over prose." R1 line 43 uses `time`. Direct contradiction.

**proposed_alternative.** Reconcile R1's field list to the schema in the same edit: `time` → `observed_at`, add `config_version` to the required list, resolve `direction` (either add to schema or drop from R1), align `duty_cycle` optionality. Since D-052 makes the schema authoritative, the fix is to correct the *prose* — but the contradiction must be closed before an agent author builds from R1. Add a CI check that the field lists in the prose contracts (R1) and the JSON Schemas are diffed for drift (the schemas already exist; a test that R1's enumerated fields ⊆ schema properties would have caught this).

**downstream_impact.** R1 PRODUCER GUARANTEES; observation-signal.v5.schema.json (or R1, whichever is authoritative — D-052 says schema); EAR-AGENT-SPEC-v1 (agent emits the schema fields); P9 malformed-row persona (a row using `time` should now quarantine — is that intended?); the enforcement-summary table R1 row.

**confidence.** high

---

### FINDING-DE.12 · severity: minor · target: R4 atomicity relied upon but list-after-PUT consistency not addressed (sync anti-join race)

**steelman.** Relying on S3/R2 PUT atomicity is correct and documented (R4 ENFORCED-BY: "Atomicity is a property of S3/R2 PUT — relied upon, documented"). R2 in particular offers strong read-after-write consistency, so this is on solid ground for the single-object case. The sync integrity check (size match on download) is a good belt-and-suspenders.

**failure_scenario.** The sync's newness test is `list prefix → anti-join ledger → land new` (R5.1). List operations on object stores are **eventually consistent for enumeration** even where GET is strongly consistent — a freshly-PUT object may not appear in a `LIST` for a short window. Sequence: agent PUTs `ral-rf2a/obs_…_000042.gz` at T; the pipeline's sync lists the prefix at T+ε and the new key is not yet enumerated; the anti-join sees nothing new; the run completes without that file. This is *self-correcting* (next run lists it) — but it interacts badly with DE.5's window/cadence: if that file carried the only observations that would have kept a P15-class deep-sleep tracker above the min-events floor for the current window, the run produces a handoff **missing the primary target** for one cycle, and nothing flags that the handoff is incomplete-as-of-list-time. The contract treats "listing a prefix yields only complete objects" (R4 CONSUMER MAY ASSUME) as if listing is also *complete*, which is a stronger guarantee than list provides.

**evidence.** DATA-WORKFLOW-RULES line 115: "Listing a prefix yields only complete objects." Line 157: "list → anti-join ledger → land new only." R4 ENFORCED-BY line 120: atomicity of PUT relied upon. What is *absent*: any statement that LIST may lag PUT (enumeration eventual consistency), and any consequence-handling for a run that lists an incomplete view. The design is robust to this by re-run, but the *handoff for the lagged cycle* is silently short.

**proposed_alternative.** Document that LIST is eventually complete and a single run's handoff reflects "objects visible at list time," not "all objects uploaded" — so the handoff freshness guarantee is *as-of-list*, not *as-of-upload*. Since this is inherent, the mitigation is downstream: the heartbeat carries `queue_depth` (heartbeat schema), so the pipeline can cross-check "agent reported queue_depth=0 and last upload at T, but I listed 3 fewer files than its heartbeat implies" and defer finalizing that agent's handoff / re-list. At minimum, add it to the known-behaviors so it is not mistaken for a bug during first-facility debugging.

**downstream_impact.** R4 CONSUMER MAY ASSUME wording; sync spec; the heartbeat-vs-listed cross-check (a cheap consistency guard using the already-present `queue_depth` field); HEALTH-CHECKS handoff-freshness check (WP-S4) interpretation.

**confidence.** low

---

## 3. WHAT TO PROTECT

These choices are strong and should be guarded against improvement-churn during build:

- **Two-table ingest (D-021) — landing → enrich, persistent raw landing.** The rationale (R5.1: "raw data is dirty in ways transform-on-read can't handle safely … safety net, quarantine point, reprocess-without-refetch, audit") is correct and hard-won (the eartest/gittest2 collapse into a temp table is cited as the regression). This is the backbone that makes recovery, A/B, and quarantine all work. Do not let a future "simplification" collapse it back to single-pass. (Fix DE.1's assertion arithmetic *within* this design; the design itself is right.)

- **Full-rebuild-per-run + config snapshot (R7, PARAMETER-TUNING).** Determinism-by-construction eliminates an entire class of incremental-merge bugs and is the precondition for honest A/B. Protect the *principle*; bound the *cost* (DE.5) via partition-scoped rebuild, but never reintroduce stateful incremental event merging.

- **Per-agent files + handoff-through-Parquet, never open two L1 files together (D-023/D-024).** Zero-cross-agent-ops → embarrassingly parallel + disposable is genuinely the right structure for this workload, and physically enforcing the L1/L2 NEVER (L2's DB cannot see L1 tables) is elegant. Protect the isolation and the physical handoff boundary. (Fix the fault-model contradiction DE.6 and the rename mechanics DE.3 — the *structure* is sound.)

- **Schemas as tie-breaker (D-052) + fixture-as-CI-gate (D-037/GOLDEN-FIXTURE).** Machine-readable enforcement over prose is the correct anti-drift posture and directly targets the historic silent-contract-violation failure mode. Protect it — and use it to catch drift like DE.11 automatically.

- **Ephemeral job model (D-044) — resident web + ephemeral pipeline, clean memory per run.** For a single-operator periodic batch, run-to-completion with a pinned image is the right minimum-moving-parts choice. Protect it; refine only the failure *granularity* (DE.6), not the model.

- **Onboarding gate (R5.0).** Closing the auto-discover-into-production gap is a real, evidenced improvement. Protect the `pending→qa→active→disabled` allowlist; only tighten qa *path* isolation (DE.10).

---

## 4. COVERAGE STATEMENT

**Examined in full and found sound (no finding):**

- **D-011 edge filter kept / D-012 feed split.** The structural argument that a summary row *cannot* enter event construction because it physically travels in `ctx_` files the event path never reads (R2/L4/D-012) is genuinely robust — it converts a forbidden operation into an unrepresentable one. P7 (cadence trap) tests it correctly. Held.
- **D-045 ephemeral uploader / `pending/` as retry queue.** The reframe (circuit breaker retired, `pending/` = store-and-forward by construction) is clean and eliminates a bug class. The only related gap (disk-space ceiling) is already tracked as WP-A3 / L5 OPEN, so I did not re-raise it. Held.
- **R6 analytical column semantics.** The `time_bucket`/`hours_present` contract with the `hours_present ≤ timeframe` regression test is the right fix for the originating bug and needs nothing added. Held.
- **R3 fake-gzip content-detection / P12.** Content-detect-don't-trust-extension is correct and tested; the legacy-adapter isolation (D-035) keeps it out of production ingest. Held.
- **Determinism whole-run assertion (byte-identical events Parquet twice, GOLDEN-FIXTURE line 37).** Strong and correctly placed — though note it interacts with DE.3: byte-identical requires deterministic Parquet writing (row order, compression), which the fixture must pin; I flag this as a *note* on the assertion rather than a separate finding since the assertion itself is right and will surface any nondeterminism.
- **BACKUP-DR sacred-vs-rebuildable table + drills.** The classification is correct (R2 + Postgres + handoff sacred; L1 disposable) and the two drills (restore + rebuild-determinism) are well-chosen. The L2-result recovery gap is captured in DE.7; the L1 story here is sound.

**Examined and folded into findings rather than separately:** Q-19 (run cadence) — its risk is the rebuild-vs-cadence interaction (DE.5). Q-22 (fleet-volume gate) — partially un-defer per DE.4. Q-15 (retention) — the reproducibility-horizon tradeoff per DE.9.

**Not examined (out of lens, deferred to other reviewers):** the Pass-2 boundary calc *mathematics* (RF/SIGINT lens — I only touched its data-shape seam via P5); WAN-partition/store-and-forward *distributed-systems* behavior (reliability lens — I touched only the ledger/etag data-integrity slice, DE.2/DE.12); security of R2 credentials and sops/age (security lens); the G1–G6 classifier correctness (RF lens). The run lock's *distributed*-correctness (stale-lock break, RECOVERY-RETENTION line 12) I treated only where it couples to cadence/rebuild (DE.5); its concurrency-correctness proper belongs to the reliability lens.

**Net:** the pipeline's *structure* is strong and largely protectable. The concentrated risk is in the **arithmetic and failure-granularity of the guarantees** — the post-load conservation assertion is provably wrong under the design's own quarantine path (DE.1, blocker), ledger idempotency rests on an unpinned etag definition (DE.2, blocker), partition atomicity is claimed at a granularity `rename(2)` does not provide (DE.3, major), and per-agent isolation is undone at the job level by whole-run halt (DE.6, major). None require a redesign; each is one contract edit now versus a production data-integrity incident later.
