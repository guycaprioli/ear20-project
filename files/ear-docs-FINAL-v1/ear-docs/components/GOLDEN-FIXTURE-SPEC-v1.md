# EAR System — Golden Fixture Specification (v1)
### One synthetic dataset that makes every Level 1 contract executable

**Purpose.** The contracts are prose; prose drifts. This fixture is the executable form: a small synthetic input dataset with **known-correct expected outputs** for every rule. It runs in CI and at the end of every build phase. If the fixture passes, the contracts hold; if a change breaks a contract, the fixture — not a human rereading 1,100 lines — catches it. Build it in Phase 2 (ingest personas) and extend in Phase 3 (event personas).

**Also housed here (D-046):** curated **replay datasets** — raw `hackrf_sweep` stdout captures and legacy JSONL sets — for the agent's replay capture-source. Fixture personas prove contracts; replay datasets prove realism.

**Shape.** Synthetic `.jsonl.gz` files under fixture R2-style prefixes (`tst-rf1a/`, `tst-rf1b/`, one unregistered `tst-rf9x/`), covering a 24 h window, low band, with a generator script (parameterized, seeded, deterministic) committed alongside expected-output tables.

---

## Personas (each encodes specific contracts)

| # | Persona | Input construction | Expected output | Contracts proven |
|---|---|---|---|---|
| P1 | **Deep-sleep tracker** (the target) | 3 bursts in a 5-min window, then 6 h silence, repeating. Each burst = one ~10 s transmission captured as 4 sweep hits ~2 s apart. Power jitters ±0.6 dBm around −41. | Exactly **1 event per burst-cluster window** (bursts <600 s apart merge; >600 s gap → new event). Survives noise filter. `detection_count` = sweep hits. | Event=burst group; sweep-collapse; 600 s gap; undersampling handling |
| P2 | **Power-saving beacon** (must NOT be filtered) | 1 short burst **every hour**, ~12 obs/burst → present in ~100 % of hours, total < timeframe×40. | `filter_decision='pass'` with reason `PROTECTED: Low duty cycle`. Events built. **If this persona is filtered as infrastructure, the build FAILS.** | Low-duty-cycle exception (the NEVER-simplify rule) |
| P3 | **Infrastructure chatterer** | Same EARFCN, ≥500 obs/hour, all 24 h. | `filter_decision='infrastructure'`; zero events emitted. | Presence×activity filter |
| P4 | **Curated-override infra** (L4.5) | Same as P3 on a second EARFCN listed in the override rule file. | Upload side emits only summary-heartbeat rows (1/window, correct `observation_count`) **in the context feed (`ctx_`)**; signal feed contains none of it; never zero rows. | L4.5 forced summary, feed split (D-012) |
| P5 | **Boundary straddler** | One device at −40.1 ± 0.4 dBm — readings land on both sides of the −40 bin edge, 50/50. | **One** unified power_bin after Pass 2; one event stream, not two. *(Marked `xfail` while Pass 2 is a stub — the test exists from day one so the stub can never be mistaken for done.)* | Two-pass boundary reconciliation |
| P6 | **Two real devices, same EARFCN** | Device A steady −20, device B steady −44, interleaved in time. | **Two** distinct power_bins, two event streams. Never merged. | False-merge guard (bins far apart must not fuse) |
| P7 | **Cadence trap** (false-CRITICAL regression) | AGGREGATE and FILTER rows arriving at exact 5-min file cadence in the **context feed**, `observation_count`=350 each. | **Zero events** — structurally impossible (event path reads signal feed only); ctx rows land in the context table; presence metrics weighted by `observation_count`. | D-012 feed split; R2 edge semantics; historic false-CRITICAL bug |
| P8 | **Sparse noise** | 3 total observations, scattered. | `filter_decision='noise'`; no events. | Min-obs floor |
| P9 | **Malformed rows** | In one file: a bad timestamp row, a missing-power row, and a row whose internal `agent_id` says `tst-rf1b` inside the `tst-rf1a/` folder. | All land in `raw_landing`; enrichment quarantines each **with reason + count**; folder identity wins and the mismatch is flagged. `observations` row-delta reconciles exactly. **Silent drop = FAIL.** | R1 no-silent-drop; folder-authoritative identity; two-table ingest |
| P10 | **Duplicate re-sync** | Re-deliver an already-ingested file (same name/size). | Ledger skips it; event counts unchanged. | R5 idempotency |
| P11 | **Unregistered agent** | Populated `tst-rf9x/` prefix, no Sensor row. | Surfaced as `pending`; **zero** rows ingested anywhere. | R5.0 onboarding gate |
| P12 | **Fake-gzip file (legacy-corpus robustness)** | One file named `.jsonl.gz` containing uncompressed JSONL (donor-era artifact; v5 agents ship real gzip). | Content-detected; ingested correctly; flagged in run log. | R3/R5.1 content detection |
| P13 | **Config drift** | Batches stamped `config_version=cfgA` then `cfgB` (noise floor change mid-window). | Both versions recorded; events carry the version of their source batch. | L0.5 capture-config provenance |
| P14 | **Heartbeat** | `_health/{agent_id}.json` present and fresh for `tst-rf1a`; absent for `tst-rf1b`; data quiet for both. | Staleness logic distinguishes alive-but-quiet vs sensor-dead; **dead-heartbeat alert fires (D-049)**; data uploads unaffected. | L6.5 / D-015 / D-049 |
| P15 | **Deep-sleep survival (end-to-end)** ⚠ | One device: 4 short bursts across a 10-day window (multi-day gaps), stable power, recent first-seen. | Survives noise filter via **`total ≥ MIN_OBSERVATIONS` + the ELSE branch** — the low-duty *presence* exception does **NOT** apply (a multi-day-silent device is not present in ≥`infra_hours`; finding QA.16/A1); events built; at Level 2 classified **G1 deep-sleep** without being starved by the `min_events` floor (deep-sleep path bypasses it, QA.5). **If red, the system misses 20595-class devices.** **Gate at S16 (L1 survival) AND S19b (`P15→G1`).** | `MIN_OBSERVATIONS` floor (L1, the *real* survival gate) × deep-sleep `min_events` bypass (L2); taxonomy G1 |
| P16 | **Late arrival (store-and-forward reality)** | Run pipeline; THEN deliver a delayed file (older timestamps, e.g. WAN-outage backlog); run again. | Ledger ingests it; the window's events **update deterministically** (full-rebuild semantics); no duplicates, no orphaned partial state. | R5 idempotency × R7 rebuild × D-024 atomic re-export |
| P17 | **Edge-filter conservation (agent-side)** | Known raw input through the edge filter. | raw rows == FORWARD rows + Σ `observation_count` of summaries + Σ L4.5 heartbeat counts. Nothing silently eaten. | R1 volume honesty at the AGENT boundary |
| P18 | **QA isolation** | `tst-rf1b` set to `qa` with plausible tracker-pattern data. | Its data lands in the isolated context; its events appear in NO production candidate, classification, or correlation output. | R5.0 qa semantics × L2 scoping |
| P19 | **Override mismatch flag** | Band partners `tst-rf1a`/`tst-rf1c` with DIFFERENT L4.5 rules for one EARFCN. | Server flags the pair (correlation-skew warning) in run output/dashboard; correlation for that EARFCN marked non-comparable. | L4.5 ENFORCED-BY clause |

## Whole-run assertions (after all personas)
- **Determinism (logical):** run the full chain twice on the fixture → **identical `events` set** after a canonical total sort on `(agent_id, earfcn, power_bin, event_start)`, comparing all columns **except `run_id`**. *(Finding QA.4: byte-identical Parquet is unachievable — the per-row `run_id` plus DuckDB row-ordering and Parquet metadata/compression defeat it; the binding rule R7 already says "identical event set". If byte-identity is also wanted as a CI convenience, additionally pin `ORDER BY` on export + codec/level/row-group + a constant `run_id` in test mode + the DuckDB/Arrow version.)*
- **Handoff contract:** exported Parquet matches `EVENT-DATASET-HANDOFF-v1` (fields, types, all-UTC, every row has `run_id`+`config_version`+`dataset_version`; partition path correct).
- **Conservation:** input rows = observations + quarantined + suppressed-summarized (accounted volume, R1) — nothing unaccounted.
- **Over-filter safeguard:** a variant config that would filter >X% flips to pass-all and flags the run.
- **Isolation:** `tst-rf1a` processing never touches `tst-rf1b`'s L1 file.

## Rules
- Expected outputs are **committed tables**, not recomputed — the fixture defines truth; code conforms to it.
- **Double-derivation rule:** every expected output is independently hand-derived (worked calculation committed alongside, `expected/derivations/`) before any pipeline code exists — the fixture must not be able to encode a wrong answer as law. Guy spot-reviews the derivations for P1/P2/P15 (the target-class personas) at S04.
- Every future contract change adds/updates a persona **in the same commit** (R0.3: every rule has a check).
- P5 stays `xfail` until the real Pass-2 lands; flipping it to pass is the definition of that work being done.
