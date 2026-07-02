# EAR System — Event-Construction Methodology (v1, DRAFT)

> **SCOPE (this effort):** Covers **Level 1 — dataset preparation**: the local-host capture/upload chain and the server-side steps that build the **event dataset** (power-bin grouping, boundary reconciliation, event construction, noise filtering) up to the analysis handoff. **Level 2 — analysis** is a **separate component** (`ear-analysis`) consuming only the event dataset via the handoff contract; an initial best-effort version (feature extraction, G1–G6 classification, scoring, provisional correlation, dashboard) is IN scope of this build (D-055). The data boundary is unchanged.

### How individual observations become device/event records on the server

**Status.** This is the single most valuable and least-documented algorithm in the system: how the server merges many jittering individual observations into discrete event/device records. It is being reconstructed and pinned so a future rewrite cannot silently change what an "event" means. Parts are **settled** (problem, constraint, design); the **pass-2 calc is OPEN** — it was designed/explored but is not fully present in any reviewed repo and may exist in code not yet seen.

**Where this sits in the chain.** Server-side, after ingest (R5), before candidate scoring (R7). This is the R6→R7 boundary — turning raw observations into the events the detection methodology analyzes. Owned entirely by the server; the agent's `power_bin` is advisory only (see local-host L4).

---

## What an "event" is (SETTLED — the event-defining step)

**An event is a burst group — a device's activity episode — and it is the unit that gets analyzed.** Event construction is dataset preparation; everything downstream (CV, periodicity, tracker classification, threat assessment) is analysis operating on events. The prep/analysis boundary is: *build events (Level 1) → hand off → analyze events (Level 2).*

**Physical grounding (why events are built this way).** The HackRF captures only the **uplink** channel — one side of the two-way device↔tower conversation (device transmits on uplink EARFCN; tower answers on downlink, which is not observed). Because the radio is **sweeping** across a frequency range rather than parked on one EARFCN, it **undersamples** each transmission: a ~10-second uplink broadcast might be caught only 4 of 5 possible times, as the sweep passes that frequency. Gaps within a transmission are the sweep looking elsewhere, not signal absence. A device also transmits in **bursts** — e.g. 3 bursts of activity across a 5-minute window.

**Event construction (Level 1 — dataset prep), in order:**
1. **Power-bin grouping** — establish which device/signal (with Pass-2 boundary reconciliation, see below).
2. **Sweep collapse** — merge the undersampled sweep-hits of a *single transmission* into one detection (the consolidation window; earnest-v3 used 2s — confirm target value, likely tied to sweep revisit time).
3. **Burst grouping = THE EVENT** — group detections separated by less than the burst gap into one burst/activity episode. **This burst group is the event.** A gap greater than the burst threshold ends one event and begins the next.
4. **Noise filtering** — remove infrastructure/known-benign/sub-threshold.
5. Output: the **event dataset** — the handoff to analysis.

**Burst gap threshold (SETTLED — validated default, tunable).**
- **600 seconds (10 minutes)** is the confirmed working value for the primary use case: >10 min of quiet ends an event; detections closer than that belong to the same burst.
- It is **use-case-tuned, not universal** — a device whose normal cadence is minutes-apart could be wrongly merged, a chattier one over-split. Therefore it is exposed as a **tunable analysis parameter** (server-side analysis config, e.g. `rfanalysis.toml`), global-default for cross-sensor comparability with per-use-case override — **never a hardcoded SQL constant** (a silent change would redefine what an event is).

**NEVER.**
- The burst gap is never buried as a hardcoded literal; it is a versioned, tunable parameter (a change to it redefines "event" and must be traceable — R0.1/R6 discipline).
- Analysis never operates on raw records or bare sweep-collapsed detections — only on events (burst groups).
- Sweep-collapse and burst-grouping are never conflated: collapse reconstructs one *transmission* from undersampled sweeps (seconds); burst-grouping assembles one *episode* from transmissions (minutes). Two different scales, two different thresholds.

> **Code reality:** only rfanalysis's long-interval stage implements burst grouping (fixed 600s gap, `SUM(is_new_burst) OVER (…)` sessionization). earnest-v3 collapses sweeps (2s) then jumps straight to hourly-bucket CV — it has **no burst-as-event grouping**. eartest/gittest2 have no time/event grouping at all. So "event = burst group" is under-represented in surviving code and must be built deliberately as the event-defining step.

---

## The problem (SETTLED — confirmed, discovered in practice)

RF power readings jitter. The same physical emitter reads −13.8, −14.2, −13.9 across sweeps — radio nature, multipath, receiver variance. Nothing falls into exact numeric ranges.

Fixed power buckets (`FLOOR(power_dbm/2)*2`, 2 dBm) have **hard edges**. A single emitter whose power sits **near a bucket boundary** scatters its readings across *both* adjacent buckets purely because of where the arbitrary cut fell — e.g. readings at −13.8 and −14.2 land in different buckets though they are the same device sitting on the −14 fence. The device didn't move; the bucketing split it.

**Consequence if unhandled:** one physical device fragments into multiple `(earfcn, power_bin)` streams, diluting event counts and weakening CV/periodicity scores — the tool under-detects the very devices it exists to find. **This is a confirmed defect, not a theoretical concern.**

---

## The constraint (SETTLED — the reason for two passes)

The accurate grouping methods that correctly handle jitter/boundaries are **computationally expensive** — too slow to run across the full observation volume (millions of rows, 60 sensors, multi-day windows). Running an accurate-but-slow method over *every* observation is not tractable at production scale.

**Therefore the design must not** run the expensive reconciliation over all observations. This constraint is load-bearing — it is *why* the two-pass structure exists.

---

## The design (SETTLED — two-pass split)

A cheap coarse pass over everything, then an expensive precise pass over only the ambiguous minority:

**Pass 1 — fast coarse bucketing (over ALL observations).**
- `power_bin = FLOOR(power_dbm / 2) * 2` — trivial O(n) arithmetic.
- Groups the bulk of observations instantly; most readings sit comfortably inside a bucket, nowhere near an edge, and are correctly grouped by this pass alone.
- This is the pass all surviving implementations have.

**Pass 2 — edge-case reconciliation (over ONLY boundary observations).**
- **Cheap selector first:** identify the small subset of observations sitting within a boundary zone at the top/bottom edge of their bucket (distance-to-boundary ≤ threshold). This keeps the expensive step narrow.
- **Expensive calc second:** for only those edge observations, apply a mathematical test to decide whether they belong with the neighboring bucket's group, and reassign them to unify the device split by the boundary.
- You pay the costly calc only for the fraction of observations that actually straddle a boundary — not the whole dataset.

**LOCK (D-027, Guy):** the two-pass structure is a locked decision. The WP-S5 benchmark sizes the **Pass-2 compute budget** (how expensive the edge calc may be per agent) — it never questions the structure.

**NEVER (invariants).**
- Never replace the two-pass split with a single accurate-but-slow pass over all observations — the split is load-bearing for processing time; collapsing it reintroduces the performance problem the design solves.
- Never treat the fast Pass-1 bucketing alone as the final grouping — it leaves boundary-straddling devices fragmented (the confirmed defect above).
- Never let the meaning of "event" or "power_bin grouping" change silently under the same names (R0.1).

---

## Pass-2 calc (OPEN — pending code that may exist elsewhere)

The specific mathematics of the edge-reconciliation calc is **not fully present in any reviewed repo**. It was designed/explored (performance-motivated, as above) and may exist in code not yet reviewed — a branch, scratch script, or a build outside the eight repos examined.

**What the surviving repos actually do (NONE is this method — do not canonize):**
- **earduck, eargemini:** no adjacency merging at all (`adjacency_merging: false` per the 2026-01-06 methodology comparison).
- **earnest-v3:** merges only *same-timestamp* simultaneous readings (`MAX(power_bin)`, `AVG(power_dbm)`) — a different problem (sweep hitting a signal multiple times), not boundary reconciliation over time.
- **eartest / gittest2 / rfanalysis:** single-pass `bin_mappings` fold — remaps a lower-count bin into its highest-count adjacent neighbor (`arg_max`/`ROW_NUMBER() ORDER BY obs_count DESC`, adjacency `ABS(source−target) ≤ bin_size`). This triggers on **bin population**, not **proximity to a boundary**, and is a whole-bin remap, not an edge-observation reconciliation. It is a later addition, not part of the validated original methodology, and it is **not** the intended two-pass method.

**Candidate calcs for Pass 2 (design decision, to confirm against the missing code):**
1. **Boundary-zone reassignment** — an edge observation within X dBm of a boundary is reassigned to the adjacent bucket if that bucket's population/centroid indicates it's the true home. Simplest; threshold X is the tunable.
2. **Centroid-nearest** — compute each bucket's core mean power; reassign an edge observation to whichever adjacent bucket *centroid* it is closest to (so −14.2 joins the −12 group if that group's true center is −13 while the −14 bucket centers at −16).
3. **Valley/gap test** — if adjacent buckets' populations form one continuous hump straddling the boundary (no real valley), merge them; if genuinely bimodal, keep separate. Most robust against fusing two real nearby devices; most expensive.

**Signature to look for when hunting the missing code:** a *second* SQL/processing step after the initial `FLOOR(power_dbm/2)*2` that (a) cheaply selects observations by **distance to a bucket boundary**, then (b) applies a heavier calc (centroid, mean, valley, or tolerance test) to only that subset. The two-part shape — cheap edge-selector feeding an expensive reconciler — is the fingerprint.

---

## Noise filtering (SETTLED — the infrastructure/noise removal step)

**Where it sits.** After power-bin grouping, **before** event construction — so infrastructure and noise are removed before event-building effort is spent on them. The decision is materialized per `(agent_id, earfcn, power_bin)` in a `filter_decisions` table; downstream stages `INNER JOIN … WHERE filter_decision = 'pass'`, so filtered devices simply don't flow forward. Part of Level 1 (dataset prep).

**The metric (hourly presence × total activity).** Computed from the hourly rollup (`hourly_stats`) per device:
- **hours present** = `COUNT(DISTINCT hour_bucket)` — how many distinct hours the device appeared in.
- **total activity** = `SUM(observation_count)` — total transmission volume, **weighted by `observation_count`** so edge-filtered AGGREGATE/FILTER rows count correctly (honors R2 — a summarized row represents many observations, not one).
- **presence threshold** `infra_hours = ceil(effective_hours × presence_ratio)`, `presence_ratio` default **0.95**.

**Classification (three-way):**
```
IF present in ≥ infra_hours (≈ every hour)
   AND total activity < timeframe × ACTIVITY_PER_HOUR   → 'pass'   ← LOW-DUTY-CYCLE EXCEPTION
IF present in ≥ infra_hours (≈ every hour)               → 'infrastructure'   (filtered)
IF total activity < MIN_OBSERVATIONS                     → 'noise'            (filtered)
ELSE                                                     → 'pass'
```

**The low-duty-cycle exception (CRITICAL — do not simplify away).** A signal present in nearly every hour *looks* like infrastructure by presence alone. But if its **total activity is low** (below `timeframe × ACTIVITY_PER_HOUR`, default multiplier **40** ⇒ averaging < ~40 obs/hour), it is **present-but-sparse**: it appears regularly across many hours but only briefly each time, with long quiet gaps between. **That is the signature of a battery-powered device in power-saving mode waking periodically to beacon — not a constantly-chattering tower.** The exception keeps it (`pass`) instead of filtering it as infrastructure.

Without this exception (as in earnest-v3's presence-only rule), a tracker that beacons hourly would be present in ~100% of hours → classified `infrastructure` → discarded — silently filtering away *the exact device the system hunts*. This exception is the whole reason the filter is presence-**and-activity**, not presence-only.

**Signature filtering (complementary, human-curated).** Alongside the statistical filter, known infrastructure is rejected by identity via the curated `InfrastructureSignature` table (EARFCN/bin) and the upload-side L4.5 overrides. Statistical filtering catches *unknown* infrastructure by behavior; signature filtering catches *known* infrastructure by identity. Both feed `filter_decision`.

**Over-filtering safeguard.** If the filter would reject more than a configured fraction of devices, it reverts to **pass-all** and flags the run, rather than silently emptying the dataset (guards against a bad threshold nuking everything).

**Tunable defaults (config, not hardcoded — R6/parameter-tuning discipline):**
| Parameter | Default | Meaning |
|---|---|---|
| `presence_ratio` | 0.95 | Fraction of hours ⇒ "present nearly always" |
| `ACTIVITY_PER_HOUR` (duty-cycle multiplier) | **40** | Below `timeframe×40` total obs ⇒ sparse ⇒ battery-tracker exception. **The single most important knob — separates power-saving tracker from infrastructure.** |
| `MIN_OBSERVATIONS` | (validated default) | Below this ⇒ `noise` |
| over-filter safeguard ratio | (validated default) | Revert to pass-all if exceeded |

**NEVER.**
- The low-duty-cycle exception is never removed or "simplified" to presence-only — doing so silently filters out power-saving trackers (the primary targets).
- Presence/activity is never measured by raw row count when `observation_count` weighting applies (R2) — a summarized row counts for its full volume.
- Filtering never runs without the over-filter safeguard.
- Thresholds are never hardcoded literals — tunable config, versioned (R0.1).

> **Code reality:** the refined presence-**and**-activity filter with the low-duty-cycle exception and safeguard exists in **rfanalysis** (`signal_filtering.py`). earnest-v3 has only the crude presence-only version (`≥80% hours ⇒ infrastructure`, `<5 obs ⇒ noise`) which would filter out hourly-beaconing trackers. rfanalysis is the version to keep.

---

## Downstream dependency (why this must be right)

Event consolidation (R7) groups these power-grouped observations into events within a consolidation window, then time-clustering computes CV/periodicity per event stream. If Pass 2 is wrong or absent:
- **False split** (device fragmented across buckets) → diluted counts, weak scores, **missed trackers** (the primary failure — under-detection).
- **False merge** (two real nearby devices fused) → one event stream conflating two emitters, corrupted periodicity.

**Open decision for the contract:** which error is worse operationally — false split (miss a tracker) or false merge (conflate two)? This sets how conservative the Pass-2 threshold should be. *(Pending your input; my assumption is false-split/under-detection is worse for a tracker-hunting system, arguing for a Pass-2 that leans toward merging boundary cases — but this must be confirmed, not assumed.)*

---

## To resolve (open items)

| Item | Status |
|---|---|
| Pass-1 fast bucketing (`FLOOR(power_dbm/2)*2`) | SETTLED |
| Event = burst group (the analyzed unit) | SETTLED |
| Burst gap threshold (600s default, tunable) | SETTLED |
| Sweep-collapse window (2s in earnest-v3) | confirm target value (tie to sweep revisit) |
| Noise filter: presence×activity + low-duty-cycle exception | SETTLED (rfanalysis version) |
| Duty-cycle multiplier (timeframe×40) | SETTLED default, tunable |
| Two-pass split (perf constraint) | SETTLED |
| Boundary-straddle problem statement | SETTLED |
| Pass-2 calc (boundary-zone vs centroid vs valley) | **OPEN — find missing code, else decide** |
| Edge-selector threshold (distance-to-boundary) | **OPEN** |
| False-split vs false-merge cost priority | **OPEN — sets threshold conservatism** |
| Transitive vs one-hop reconciliation (does an edge obs chain across >1 boundary?) | **OPEN** |
