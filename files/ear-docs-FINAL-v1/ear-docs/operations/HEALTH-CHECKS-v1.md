# EAR System — Health Checks (v1)
### The runtime answer to "is everything functional right now?" (D-059). Build-time confirmation is D-058; this is the running system, checked continuously, with one place to look.

**Principle:** health is judged from **real signals of real work** (heartbeats, object arrivals, run outcomes, partition freshness) — not synthetic probes injected into production data. The chain itself is the canary: if capture→upload→ingest→events are all advancing, the system is functional by demonstration.

## Layer 1 — Container liveness (compose healthchecks; WP-S1)
`web` → HTTP `/health/` (DB ping inside) · `postgres` → `pg_isready` · `minio` (dev) → `/minio/health/live` · `proxy` → upstream check. Docker restarts unhealthy containers; restart-looping surfaces in Layer 3.

## Layer 2 — Agent self-check (`ear-agent check`; WP-A5)
On-host command (manual + optional timer), used by the provisioning checklist and any SSH diagnosis:
scanner units active ×4 · expected HackRF serials present (`hackrf_info`) · NTP synchronized · disk vs ceiling · queue/DLQ depths · R2 credential probe (cheap HEAD) · config renders + identity validates. Exit non-zero with a human-readable failure list.

## Layer 3 — System health sweep (`manage.py check_health`; WP-S4, systemd timer ~15 min)
The unifying process. One run evaluates the whole chain and produces a **verdict per check**:

| Check | Functional means | Degraded/Failed when |
|---|---|---|
| DB | query round-trip | unreachable |
| Object store | list on bucket succeeds | unreachable / auth |
| Agent liveness | every **active** sensor's heartbeat fresh (< 3× upload cadence) | stale → **that sensor flagged**; many stale in one facility → WAN suspicion noted |
| Agent condition | heartbeat fields sane: status ok, disk below ceiling, queue not monotonically growing across sweeps, `ntp_offset_ms` within tolerance | any breach flagged per sensor |
| Capture advancing | new `obs_`/`ctx_` objects per active sensor within expected window | none arriving while heartbeat fresh = capture problem (distinct from host-down) |
| Pipeline | last L1 run: recent (< 2× cadence) AND exit 0 | old or failed |
| Handoff advancing | newest Parquet partition per active agent within expected window | stale = pipeline completing but not producing |
| L2 | last run_level2 recent + successful; candidate/classification tables updated | stale |
| Server disk | volumes below thresholds | breach |
| Timers | upload/pipeline/health timers enabled+scheduled (where visible) | disabled |

**Outputs:** (1) a persisted `HealthSweep` row (verdicts JSON) — the dashboard renders the latest as a **system-status banner** (green/amber/red) on every page, satisfying "one place to look"; (2) `notify()` on any new red (state-change alerting — no re-alert spam on persisting known failures, one reminder per configurable period); (3) non-zero exit if red (so the timer's `OnFailure=` is a backstop if `notify()` itself is broken).

**Self-monitoring:** the sweep is itself watched — if no `HealthSweep` row lands within 2× its cadence, the dashboard banner turns amber ("health checker silent") and the web `/health/` endpoint reports it. Watchdog for the watchdog, cheaply.

## Layer 4 — Deep validation (existing, scheduled)
The fixture suite against the deployed stack (weekly timer, dev/staging context) · the BACKUP-DR restore drills (per plan) · UAT journey re-runs at facility onboarding. These confirm *correctness* is still intact, not just liveness.

## Fixture coverage
Persona **P14** already stages heartbeat-fresh/stale; extend at build time with sweep assertions: staged conditions (stale sensor, failed run, old partition) → `check_health` produces the expected verdicts and a single state-change notify. (Folded into WP-S4's gate.)
