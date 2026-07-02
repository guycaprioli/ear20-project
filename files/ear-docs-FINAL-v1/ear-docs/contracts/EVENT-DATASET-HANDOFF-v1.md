# EAR System — Event Dataset Handoff Contract (v1)
### The interface between Level 1 (dataset preparation) and Level 2 (analysis)

**Purpose.** This is the single interface that lets Level 2 (analysis) be a separate effort from Level 1 (dataset preparation). It defines *exactly* what the prep layer guarantees to hand off: the event dataset. Level 1 produces it; Level 2 consumes it and may assume nothing beyond what is written here. If this contract holds, either side can be rebuilt independently.

**Position in the chain.** The handoff is the output of Level 1:
```
… power-bin grouping → noise filtering → sweep-collapse → burst grouping (= events)
   → EVENT DATASET  ═══ this contract ═══  → Level 2 analysis (CV, periodicity, scoring, classification, reports)
```

---

## What is handed off

Two related tables constitute the handoff. Level 2 reads these and only these.

**Medium (DECIDED): partitioned Parquet.** Each completed per-agent Level 1 chain **exports** the handoff as immutable Parquet: `handoff/{facility}/{agent_id}/{date}/events.parquet` (+ `event_gaps.parquet`). Level 1 runs in **per-agent DuckDB files** (`l1/{facility}/{agent_id}.duckdb`); Level 2 runs in its **own dedicated DuckDB** that reads the Parquet via glob/ATTACH. Properties this buys: no writer contention when many chains finish at once; the handoff NEVER (no reaching back into raw/pre-filter tables) is **physically enforced** — Level 2's DB does not contain Level 1's tables; retention = partition lifecycle; QA isolation is literal (a `qa` agent's partition is simply not globbed by production Level 2).

### 1. `events` — the primary unit (burst groups)

**One row per event** = one device activity episode (burst group), per `(facility, agent_id, earfcn, power_bin)`. This is the analyzed unit.

| Field | Type | Meaning / guarantee |
|---|---|---|
| `facility` | text | 3-letter code, from agent_id prefix. Correlation/report scope. |
| `agent_id` | text | `{facility}-{host}{device}`, lowercase, validated. The sensor. |
| `band_group` | text | `low` (a/c) or `high` (b/d). Correlation scope. Derivable from device letter. |
| `earfcn` | int | Frequency channel (uplink). |
| `power_bin` | int | **Authoritative, post-boundary-reconciliation** power bin (server-owned; not the agent's advisory bin). |
| `event_start` | timestamptz | First detection in the burst (UTC). |
| `event_end` | timestamptz | Last detection in the burst (UTC). |
| `detection_count` | int | Number of sweep-collapsed detections in this event (the 2–4 undersampled sweeps → 1 transmission, summed across the burst). |
| `avg_power_dbm` | double | Mean power across the event. **Uncalibrated relative** dB. |
| `power_variance` | double | Power spread within the event. |
| `consolidation_confidence` | numeric | 0–1; how confidently the sweeps collapsed into events. |
| `observation_count` | bigint | Total underlying observations (weighted; honors edge-filter AGGREGATE/FILTER — R2). |
| `config_version` | text | Capture config version the source data was produced under (L0.5). |
| `run_id` | text | The `AnalysisRun` that built this dataset (provenance). |

### 2. `event_gaps` (optional/derived) — inter-event gaps within a device stream

Provided if prep computes it; otherwise Level 2 derives it from `events`. One row per consecutive event pair per `(agent_id, earfcn, power_bin)`:
`agent_id, earfcn, power_bin, gap_seconds` (time from one event_end to the next event_start). This is the raw material for CV/periodicity — but the **CV/periodicity computation itself is Level 2**, not prep.

---

## Guarantees Level 1 makes (Level 2 may rely on these)

- **Every event is a burst group** (10-min-gap sessionized), never a raw record and never a single sweep-collapsed detection.
- **Noise and infrastructure are already removed** — only `filter_decision = 'pass'` devices are present, *including* the low-duty-cycle exception (power-saving trackers are preserved, not filtered).
- **Power bin is authoritative** — boundary reconciliation (Pass 2) has been applied; Level 2 does not re-bin.
- **Per-sensor isolation** — events are per `agent_id`; no cross-sensor mixing has occurred. (Cross-sensor correlation is a *later, separate* step operating on this dataset within `(facility, band_group)` — see note.)
- **Volume is honest** — `observation_count` reflects true underlying volume regardless of edge-filter summarization.
- **Provenance is complete** — every event carries `run_id` and `config_version`; the run's resolved config is snapshotted (reproducibility, A/B testing).
- **UTC throughout** — all timestamps UTC; assumes NTP-synced capture (R1).

## What Level 2 must NOT assume

- **No calibrated power.** `avg_power_dbm` is relative; proximity/bearing from it is coarse (R9).
- **No completeness of a transmission.** `detection_count` reflects undersampled sweeps — an event represents *observed* activity, not every moment of a transmission.
- **No tracker verdict.** The dataset contains *candidate* events (survived noise filtering, worth analyzing). "Is this a tracker" is a Level 2 judgment; prep asserts nothing about it.
- **No cross-facility or cross-band meaning.** An `earfcn` in `ral` low-band is unrelated to the same `earfcn` in `cht` or in high-band.

---

## NEVER (contract invariants)

- Level 2 never reaches back into raw observations, power_clustering, or pre-filter tables — it consumes only `events` (+ `event_gaps`). Reaching around the handoff couples the layers and reintroduces the drift this contract prevents.
- Level 1 never emits an event that skipped noise filtering or boundary reconciliation.
- Field meanings never change under the same name (R0.1); evolve by adding a field + `dataset_version`.
- `power_bin` in the handoff is never the agent's advisory bin — always the server-authoritative reconciled bin.

## ENFORCED BY

- A `dataset_version` stamped on the handoff (Parquet metadata / manifest; bumped on any schema change).
- Partition path is contract: `handoff/{facility}/{agent_id}/{date}/` — consumers select by path, never by scanning L1 files.
- **Atomic partition export:** a chain writes to a temp directory and **renames into place**; a re-run replaces the whole partition atomically. Level 2 must never be able to glob a half-written partition. Orphaned temp dirs are cleaned at chain start.
- A contract test: given a known synthetic input, assert the `events` table has the fields/types above, all rows are `pass`, all timestamps UTC, and no row lacks `run_id`/`config_version`.
- FK/provenance: every event → a valid `AnalysisRun` (R8).
- A Level-2-side read test that fails if analysis code queries any table other than `events`/`event_gaps`.

---

## Note: where correlation and analysis sit relative to this handoff

- **Cross-sensor correlation** (same physical device seen by both same-band antennas of a host) operates **on this dataset**, within `(facility, band_group)`, and is part of **Level 2** (or a distinct correlation effort) — not Level 1. It is *out of scope* for the prep layer; the handoff simply guarantees the per-sensor events it needs, tagged with `facility`/`band_group`.
- **CV / periodicity / tracker scoring / threat classification / reports** — all Level 2, all downstream of this contract.

> **Open (Level 2 concerns, deferred):** the correlation *method* (what makes two per-sensor events "the same device" across antennas — time overlap + power-gradient?) is itself an unspecified algorithm, same class as the Pass-2 calc. Flagged here so the Level 2 effort picks it up; not resolved by the prep layer.
