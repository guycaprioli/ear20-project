# EAR System — Agent Component Spec (`ear-agent` v5, spec v1)
### Build spec for the Python capture/upload agent. Implements the L-chain exactly. Port-don't-redesign.

> **SCOPE:** the on-host component only: capture → prepare → upload. No analysis, no binning authority (agent `power_bin` is advisory), no event logic — those are server-owned. Contracts implemented: `contracts/LOCAL-HOST-DATA-FLOW-v2.md` (L0–L6) + relevant R-rules. Reference implementation: `uplink-monitor` v4.3 (Node) — behavior donor, not code base.

## Runtime & process model (D-010, D-016)
- Python 3.12+, pydantic-settings config, structlog, boto3, pytest. No web framework, no DB.
- **Per host (D-045):** 4 **resident** scanner processes (systemd template `uplink-scanner@{a..d}`, per-device env, staggered start 0/3/6/9 s, Restart=always) + **1 ephemeral uploader** (`uplink-upload.timer` → oneshot, ~5-min cadence): scan pending → filter/split → gzip → PUT → heartbeat → exit. 1 shared DLQ (poison files only). systemd hardening/caps/plugdev carried from v4.3.

## Package layout (two entrypoints)
```
ear_agent/
  identity.py      # facility+host from env (never hostname); regex-validated at startup (L0)
  config.py        # pydantic-settings: capture tuning (L0.5), paths, R2, thresholds
  scanner/         # CaptureSource seam (D-046): hackrf (subprocess) | replay (recorded captures, time-shift, speed factor); parser, enricher, buffered writer, rotation (L1–L2); --record tee mode
  queue/           # pending→processing→completed lifecycle, crash re-queue (L3)
  edge/            # edge filter FORWARD/AGGREGATE/FILTER + FEED SPLIT writer (L4) + L4.5 override engine
  saf/             # pending/ = retry queue (D-045); poison-file DLQ + R2-only replay; DISK CEILING check per run (L5)
  upload/          # gzip, R2 PUT (identity-derived keys), heartbeat (L6)
cli: ear-scanner / ear-uploader
```

## Contract-critical behaviors (the port's acceptance surface)
1. **Identity (L0):** `agent_id={facility}-{host}{device}` from env; startup fails loudly on regex miss or missing HackRF serial; device↔radio bound by `HACKRF_SERIAL_A..D`.
2. **Capture config (L0.5):** every batch stamped with capture config + `config_version` + `schema_version=v5`.
3. **Write/rotation (L2):** JSONL, 5 min / 50 MB rotation, atomic rename into `pending/`.
4. **Queue (L3):** crash recovery re-queues `processing/`; no orphan states.
5. **Edge filter + feed split (L4, D-011/D-012):** classify per cluster; **signal feed `obs_…` = FORWARD rows only; context feed `ctx_…` = AGGREGATE/FILTER summaries + L4.5 heartbeats + config stamp.** Noise ≈90% of raw traffic — this stage is load-bearing; never removed.
6. **L4.5 override:** per-EARFCN forced-summary from versioned rule file (`ear-fleet`); always emits heartbeat; never silent-drops.
7. **SAF (L5, D-045):** `pending/` is the retry queue by construction (failed run exits, timer retries); no circuit breaker; DLQ = poison files only, replay targets R2 ONLY (D-017); **disk ceiling** checked at run start, eviction policy + alert.
8. **Upload (L6):** real gzip (verified, D-021's fake-gzip class killed); key `{agent_id}/{filename}` derived from identity config — never filename regex; atomic PUT; retries → DLQ.
9. **Heartbeat (D-015/D-045):** written at the end of every upload run to `_health/{agent_id}.json`: status, disk %, **queue depth** (backlog growth visible per tick), DLQ count, config_version.

## Testing
- Unit + agent-level golden-fixture personas (P4 override, P7 edge semantics, P12 gzip, rotation/idempotency) from `ear-fixture` releases; CI gate (D-037).
- Exit soak (WP-A5): 48 h on test host; kill-WAN drill = bounded disk + clean replay; fresh bucket shows facility-prefixed, gzipped, stamped v5 feeds.

## Explicitly OUT
Statistical analysis, authoritative binning, event construction, correlation, any Postgres, any second upload destination.
