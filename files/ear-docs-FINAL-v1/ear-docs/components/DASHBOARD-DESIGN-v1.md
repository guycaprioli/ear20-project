# EAR System — Dashboard Design & UAT Journeys (v1)
### Information architecture + design direction for the Django dashboard (ops + analysis), and the journey stories that ARE the UAT scripts. Implemented by WP-O1 (ops) + WP-L4 (analysis); UAT gate = session S19e.

**User:** one primary persona — **the operator (Guy)**: technical, terse, wants signal over chrome, checks in briefly and drills deep only when something's off. Secondary persona (facility staff, read-only reports) deferred to Q-16.

**Design direction:** use the Pegasus/shadcn dashboard shell as-is — no custom design system. Dense tables over cards; **status color discipline** (green ok / amber degraded-pending / red dead-critical / grey disabled) used consistently everywhere; every list row click-through to a detail page; every page filterable by facility; timestamps shown as relative ("14 min ago") with absolute on hover; dark mode default. No charts for chart's sake — the two visualizations that earn their place: the **event timeline** (device detail) and the **classification breakdown bar** (facility).

## Information architecture (nav)
```
FLEET        Sensors board · Onboarding queue · Heartbeats
PIPELINE     Runs · Quarantine · Suppression (L4.5 heartbeat volumes)
ANALYSIS     Facility summary · Classifications (G1–G6) · Candidates · Unclassified queue
DEVICE       (detail page, reached from anywhere: identity, capture context, event timeline,
              classification + features, correlation partners, corpus-log action)
ADMIN        Django admin (Sensor registry, InfrastructureSignature, users)
```

## Key screens (build order = value order)
1. **Sensors board** — one row per sensor: agent_id, facility, band, status pill, heartbeat age, queue_depth, disk %, last data. Sort by "worst first." *The 30-second fleet check.*
2. **Onboarding queue** — pending prefixes w/ first-seen + volume; qa sensors w/ isolation-run results; promote/disable actions.
3. **Runs** — L1/L2 job history: status, duration, rows in/quarantined/suppressed, events out, config_version; failure rows link to the log excerpt.
4. **Quarantine** — grouped by reason × agent, sparkline by day, drill to raw rows; disposition note field.
5. **Candidates** — TrackerCandidate table: score (four-factor breakdown on hover), classification, facility/agent/EARFCN, first-seen, status (new/reviewed/confirmed/dismissed); actions: confirm→corpus, dismiss→corpus.
6. **Device detail** — the workhorse: identity block, capture context (config_version, floor), **event timeline** (events as marks on time axis, burst durations, gaps labeled), feature panel (CV, duty, presence, power variance, first/last seen), classification + why (rule path), correlation partners, one-click corpus entry.
7. **Facility summary** — classification breakdown, new-arrivals list (first-seen < N days), open candidates, sensor health rollup. (Shape refined by Q-16.)

## UAT journey stories (the acceptance scripts — run verbatim at S19e)
Each journey passes only if completable **without touching the DB shell or logs** (except J7's log link).

- **J1 · Morning check (2 min):** open dashboard → Sensors board sorted worst-first → identify the one stale heartbeat (fixture stages one) → row click → see queue_depth history → conclude WAN vs host. ✅ Reached conclusion in ≤4 clicks.
- **J2 · Onboard a host:** new prefix appears (staged) → Onboarding queue shows pending w/ volume → promote to qa → next run's isolation results visible on the sensor page → promote to active → sensor joins the board green. ✅ No step required the admin shell.
- **J3 · Investigate a candidate:** Candidates → top score → device detail → timeline shows the sparse-regular pattern → features confirm (CV low, duty low, first-seen recent) → correlation partner agrees → mark **confirmed** → corpus entry form pre-filled → submitted. ✅ Corpus row exists; candidate status updated.
- **J4 · Tune out noisy infra:** Suppression page shows a high-volume EARFCN not yet overridden → (rule added in ear-fleet, re-rendered — outside dashboard, by design) → after next uploader tick, Suppression shows the heartbeat volume and signal-feed volume drop for that EARFCN. ✅ Before/after visible on one screen.
- **J5 · Weekly quarantine review:** Quarantine grouped view → one agent spikes → drill to reasons → disposition note saved. ✅ Spike identifiable in ≤2 clicks.
- **J6 · Facility review:** Facility summary → new-arrivals shows a G5-classified device → device detail → rule path explains why not G1 → agree/reclassify → corpus if physically checked. ✅ "Why this classification" is answerable on-screen.
- **J7 · Pipeline failure:** run fails (staged non-zero) → alert received (ntfy) → Runs shows red row → log excerpt link → cause identifiable. ✅ Alert→cause in ≤3 minutes.

## UAT gate (runsheet S19e, before M5)
All seven journeys executed by Guy against dev data (fixture + legacy corpus), each ✅ or a filed fix. **M5 does not start with an open red journey.** Journeys re-run once on real data during first-facility week (WP-O2).
