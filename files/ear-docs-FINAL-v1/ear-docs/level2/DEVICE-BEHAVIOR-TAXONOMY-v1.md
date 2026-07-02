# EAR System — Device Behavior Taxonomy (v1, DRAFT — for Guy's field correction)
### The v1 classification spec (D-055 — Level 2 un-deferred to best-effort build). Rule-based classifier WP-L2 implements the decision sketch below; the labeled corpus tunes it over time. Level 1 hands over events; THIS is what Level 2 sorts them into.

**What Level 2 sees per device stream** (from the handoff dataset, per `(facility, agent_id, earfcn, power_bin)`): event count, inter-event gap stats (mean, stddev, **CV**), burst durations, `detection_count`/`observation_count` volumes, presence ratio across hours/days, **power stability** (variance — stationary vs moving), first-seen/last-seen dates, capture context (`config_version`), and cross-sensor agreement within `(facility, band_group)`.

---

## The activity groups

### G1 — Battery-powered GPS tracker, power-saving mode  ⚠ PRIMARY TARGET
The device the system exists to find. Battery preservation forces a distinctive rhythm: **sleep → wake → brief uplink burst (position report) → sleep.**
- **Signature:** present across many hours/days but **sparse total activity** (low duty cycle — the L1 exception exists precisely to keep these); **highly regular inter-event gaps** (machine timing: CV < ~0.3); short bursts (seconds); **stable power** (planted on a stationary asset/vehicle → fixed multipath); **sudden first-seen date** (it arrived).
- **Sub-patterns:** *interval reporter* (fixed cadence, e.g. every 30/60 min); *deep-sleep* (multi-hour to multi-day intervals — the 20595 class; needs long analysis windows and tolerates very few events); *motion-activated* (bursts cluster when the tagged asset moves — gaps regular within active periods, silent otherwise).
- **Threat logic:** NEW + STATIONARY + LOW-DUTY + PERIODIC = the four-factor high score.

### G2 — Active/real-time tracker
Tracker without aggressive power saving (wired to vehicle power, or cheap hardware).
- **Signature:** frequent regular reports (tens of seconds to minutes), near-continuous presence, low CV, moderate volume — far below infrastructure chatter, far more regular than a phone. Power stable if the vehicle is parked; steps when it moves.

### G3 — Cell phone (human-carried)
- **Signature:** **irregular** bursts (calls, app sync, screen wakes — human-driven: CV high); bursty variable volume; **power varies** (the person moves); presence correlates with **shift hours**; comes and goes across days (workers). Transient appearance at varying power = human, not plant.
- Mostly benign background; a *stationary, 24/7, regular* "phone-like" emitter is not a phone — reclassify.

### G4 — Infrastructure (towers, boosters, fixed radios)
- **Signature:** present essentially every hour, **high total activity** (constant chatter), rock-stable power, permanent (present since data began). **Already removed at Level 1** (presence×activity filter + curated signatures + L4.5); anything infra-like that leaks through gets classified here and feeds the `InfrastructureSignature` curation loop.

### G5 — Fixed IoT / facility telemetry  ⚠ THE HARD CONFUSION CLASS
Mains-powered sensors, meters, alarm panels, fleet-telematics on parked company vehicles — **periodic like a tracker, stationary like a tracker.**
- **Discriminators vs G1:** *first-seen date* (present since deployment vs appeared suddenly); *duty/interval character* (mains power often = shorter intervals, longer sessions, less aggressive sleep); *inventory correlation* (curated facility allowlist — a known asset at a known power); *permanence* (months of unchanging presence).
- **This class is why the labeled corpus matters most** — G1-vs-G5 is the judgment that physical confirmation trains.

### G6 — Cellular modem/router (fixed backhaul)
- **Signature:** heavy sustained data sessions, near-infra volume with session structure, stable power, permanent. Usually caught by L1 filtering; else classified here.

---

## Classification dimensions (the feature set Level 2 computes)
| Dimension | Separates |
|---|---|
| Gap CV (regularity) | machine (G1/G2/G5) vs human (G3) |
| Duty cycle / total activity | battery (G1) vs powered (G2/G5/G6) vs infra (G4) |
| Power variance | stationary (G1/G4/G5/G6) vs mobile (G3, moving G2) |
| Presence ratio + window | persistent vs shift-correlated vs sparse-regular |
| First-seen recency | **arrived** (suspicious) vs always-there (facility) |
| Shift-hours correlation | human rhythm (G3) vs 24/7 machine |
| Inventory/signature match | known facility asset (G5/G4) vs unknown |
| Cross-sensor agreement | physical device confirmation; power gradient → rough position |

## Draft decision sketch (order matters)
1. Infra-like leakage → G4 (feed signature curation).
2. Known inventory/signature match → G5/G6 (benign, retain).
3. Human-pattern (high CV + power mobility + shift correlation) → G3.
4. Regular + stationary + low duty → **G1 candidate**; score by first-seen recency, interval class, window length; deep-sleep sub-path for very-few-event streams.
5. Regular + higher duty → G2 vs G5 on first-seen + inventory.
6. Residual → unclassified queue (human review → corpus label).

**Every physically-confirmed classification — every group, including benign — goes to the labeled corpus (ear-fleet). The corpus is what eventually turns this sketch into tuned, tested classification.**

---

## For Guy's correction (field truth needed)
- Q-T1: Real wake intervals you've observed per sub-pattern (30 min? hourly? the 20595 cadence?) — anchors the interval classes.
- Q-T2: Known confusion cases from the old effort besides G1-vs-G5? (e.g., booster/repeater oddities, motion-activated vs phone?)
- Q-T3: Missing groups? (Asset tags/BLE-cellular hybrids, cargo trackers with different band behavior, jammers/anomalies worth a class?)
- Q-T4: Does shift-hours correlation hold at your facilities (24/7 operations would weaken the G3 discriminator)?
