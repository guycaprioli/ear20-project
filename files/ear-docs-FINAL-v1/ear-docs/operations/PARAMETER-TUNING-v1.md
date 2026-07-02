# EAR System — Parameter Tuning & Validation (v1)
### Forward-looking requirement: how parameters get fine-tuned against real-world data

**Status.** Design requirement, not yet built. Records the intent to A/B-test and fine-tune parameters as real-world data accumulates, the properties that make such testing *valid*, and the one thing to start collecting from day one so tuning is objective rather than guesswork.

**Why this is captured now.** Parameters like the burst gap (600s), consolidation window, CV thresholds, and the (open) Pass-2 power-boundary threshold are validated defaults, not universal truths. Real-world data across facilities will justify fine-tuning. This document ensures that when tuning happens, it is reproducible, isolated, and measured against ground truth — not "results I like the look of," which is the unfalsifiable drift the whole rules effort exists to prevent.

---

## What already makes A/B testing possible (do not regress these)

- **Rebuild-per-run + config snapshots.** Every `AnalysisRun` stores its resolved `config_snapshot`. A run is fully reproducible and fully attributable to one parameter set — the precondition for comparing A vs B honestly.
- **Persistent raw landing table.** Because raw data persists in the DB (the two-table landing→enrich design), the *same source data* can be re-processed under different parameters without re-fetching from R2. A/B testing needs identical inputs with varied parameters; the landing table guarantees the input is truly identical.
- **Parameters externalized as tunable config** (not hardcoded SQL constants). You cannot A/B a buried literal; you can A/B a config value. This is why the burst gap, CV thresholds, etc. are required to live in config.
- **Prep/analysis layer separation — the key enabler.** Event-*construction* parameters (burst gap, consolidation window, Pass-2 reconciliation) live in the prep layer; *analysis* parameters (CV thresholds, scoring) live downstream. This lets them be tested **independently**: hold the event dataset fixed and vary only analysis parameters, or vary construction and hold analysis fixed. Changing parameters in both layers at once makes results unattributable — the layer boundary is what isolates the variable.

---

## The hard part: what does "better" mean?

RF tracker detection often lacks known ground truth for real-world data. More candidates is not better (could be false positives); fewer is not better (could be misses). A valid A/B framework needs a comparison basis. Three, in priority order:

1. **Labeled ground-truth corpus (gold standard — START COLLECTING NOW).** Every time a device is *physically confirmed* (tracker or benign — e.g. the EARFCN 20595 vehicle tracker from the real threat assessment), record it against its event signature (facility, agent, EARFCN, power, behavior, confirmation outcome). A/B success = "parameter set correctly classifies the known set." Without this corpus, all tuning is subjective. **This is the single most valuable thing to accumulate from day one of real-world data** — it costs almost nothing to record confirmations as they happen, and it is impossible to reconstruct retroactively.

2. **Cross-sensor agreement (proxy when ground truth is absent).** The two same-band sensors of a host see the same physical device. A better parameter set should make them *agree more* about that device (present in `CorrelatedDetection` within `(facility, band_group)`). Agreement is a correctness proxy: correct event construction should make independent antennas converge on the same events.

3. **Known-noise suppression.** Does parameter set B correctly summarize/filter the infrastructure already identified (L4.5 overrides, `InfrastructureSignature`) without eating real signals? A regression here is an immediate fail.

---

## Tunable parameters and their layer (the A/B surface)

| Parameter | Layer | Default | Notes |
|---|---|---|---|
| Noise floor (per-agent) | Capture (agent) — values live in `ear-fleet` per-host configs | per-sensor | Shapes what data exists; test with care — changes the dataset itself |
| Edge-filter thresholds | Capture (agent) | thresholds.json | FORWARD/AGGREGATE/FILTER classification |
| Sweep-collapse window | Prep | ~2s (confirm) | Reconstructs one transmission from undersampled sweeps |
| **Burst gap** | Prep | **600s (validated)** | Defines the event; tune per use case |
| Pass-2 boundary threshold | Prep | **OPEN** | Power-boundary reconciliation (algorithm not yet built) |
| Power bin size | Prep | 2 dBm | Base quantization |
| CV thresholds (tracker/periodic) | Analysis | 0.3 / 0.5 | Pattern classification |
| Presence / infrastructure ratio | Analysis/filter | ~0.8 | Noise filtering |
| Min events for analysis | Analysis | 3 | Below this, no meaningful CV |

---

## Requirements for the future A/B framework (when built)

- **Isolate one variable.** Vary parameters in only one layer per test; hold the other layer's output fixed (re-use the same event dataset when testing analysis params).
- **Same input, snapshotted config.** Run both arms against the same raw-landing data; record both `config_snapshot`s; diff results attributable solely to the changed parameter.
- **Measure against the corpus first, proxies second.** Report labeled-set accuracy where ground truth exists; fall back to cross-sensor agreement and known-noise suppression where it doesn't.
- **Version every change.** A parameter change is a versioned config change (R0.1 discipline) with the run that validated it recorded — never a silent edit.
- **Per-facility awareness.** Optimal parameters may differ by facility (RF environment differs). Test per-facility before assuming a global change; a parameter good for `ral` may not suit `cht`.

---

## Immediate action (costs nothing now, invaluable later)

Start recording a **labeled corpus** the moment real-world confirmations begin: for every physically-verified device, log its facility, agent, EARFCN, power, event signature, and confirmed classification. This corpus is the foundation of all objective tuning and cannot be reconstructed after the fact.
