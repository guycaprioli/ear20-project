# EAR System — Production Plan (v3)

> **SCOPE (this effort):** Covers **Level 1 — dataset preparation**: the local-host capture/upload chain and the server-side steps that build the **event dataset** (power-bin grouping, boundary reconciliation, event construction, noise filtering) up to the analysis handoff. **Level 2 — analysis** (CV/periodicity, scoring, tracker classification, threat reports, A/B tuning) is a **separate effort** that consumes the event dataset and is out of scope here.

### RF tracker-detection for a 60-sensor warehouse deployment

**Deployment target:** 7–15 monitoring PCs × up to 4 HackRF sensors (≈60 sensors), one LAN, on-site server, **periodic forensic reports** (not real-time alerting).

**Document status:** Plan of record. Supersedes ad-hoc experiment repos. Written to be executed incrementally with Claude Code on the server.

---

## 1. Executive summary

You have **eight repositories** representing ~4 generations of the same system. None is production-ready alone, but together they contain every needed part. This plan consolidates them into **three deployable components**:

1. **Agent** (`uplink-monitor`) — runs on each PC, unchanged in role. Scans HackRFs, edge-filters, uploads.
2. **Transport** — how observations get from PCs to the server. **Recommendation: LAN-primary, R2 as off-site backup** (you're on one site).
3. **Analysis server** (`bss_earv1` Pegasus project + a consolidated `rfanalysis` Django app) — syncs/ingests, runs the DuckDB pipeline, correlates across sensors, and generates forensic reports.

The single most important architectural decision — and the one the experiments kept re-litigating — is **which analysis implementation is the spine.** This plan selects the **Django + embedded DuckDB** lineage (`rfanalysis`, hosted in `bss_earv1`) as the base, and treats the NestJS (`earnest-v3`) and cloud-TS (`eartest`) servers as **parts donors**. Rationale in §3.

---

## 2. Repository inventory & disposition

| Repo | What it actually is | Generation | Disposition |
|---|---|---|---|
| **uplink-monitor** | The agent: HackRF scanner + uploader + edge filter (FILTER/AGGREGATE/FORWARD), file-queue, R2 upload, systemd units | Current | **KEEP** — sole agent. Minor hardening only (§6). |
| **bss_earv1** | SaaS Pegasus Django project: Django 6, Postgres 17, Redis, Celery+beat, DRF, shadcn dashboard, Docker Compose, Makefile | Current | **KEEP** — production host. The app lives here. |
| **rfanalysis** (built this session) | Django + embedded DuckDB port of the full pipeline. Only implementation with all stages **and** the `hours_present` bugfix + admin UI | Current | **KEEP as spine.** Move into `bss_earv1/pegasus/apps/rfanalysis/`. |
| **earnest-v3** | NestJS + Postgres/TimescaleDB server. Most feature-complete: REST API, rf-device-status, tracker-details export, edge_filter_rules table, pattern-analysis | Gen-1 (heaviest) | **HARVEST, then retire.** Donate: device-status logic, export queries, edge-filter awareness, forensic report shapes. Do **not** adopt Timescale or its agent-rotation scheduler. |
| **eartest** / "earcore" | Cloud-native TS: DuckDB native bindings, integrated SyncService, **ForensicReportGenerator**, Dockerfile, real analyst reports. **Missing consolidation + time-clustering stages** | Gen-3 (partial) | **HARVEST, then retire.** Donate: forensic report format, sync concurrency approach, noise-floor realism (−60), real report templates in `analysis/reports/`. |
| **gittest2** / "ear-pipeline" | **Python** refactor of eartest: DuckDB, pydantic-settings, Typer CLI, boto3 R2 sync, resilience (retry/circuit-breaker), Prometheus metrics + Grafana dashboard, health checks, structlog, ~120 unit tests, CI, CLAUDE.md. **Same gap as eartest: no consolidation/time-clustering stages** | Gen-3.5 (partial) | **PRIMARY PYTHON PARTS DONOR, then retire.** Donate: `services/sync.py` (replaces earsyncr2core *and* eartest sync), `core/resilience.py`, `core/metrics.py` + metrics_server, `core/health.py`, structlog setup, test patterns, Grafana dashboard. |
| **earsyncr2core** | Standalone R2→local sync (TS, zod, atomic writes, `verify-sync` integrity audit) | Gen-2 | **ARCHIVE.** Fully superseded by gittest2's Python `sync.py`. `verify-sync` optional keeper. |
| **ear-compared** | Methodology-comparison harness carved out of earnest-v3 (earduck/eargemini/deep_sleep). Where the `hours_present` bug was found | Gen-2 | **REFERENCE ONLY.** Its methodology already ported (corrected) into rfanalysis. Archive. |

**Net result:** 2 live codebases (agent + Pegasus-hosted Django app), everything else archived after harvest. Lineage, for the record: earnest-v3 (NestJS/Timescale) → ear-compared (methodology harness) → eartest/earcore (TS+DuckDB, cloud) → gittest2 (Python re-platform, stalled before detection stages) → rfanalysis (complete Django+DuckDB port). gittest2 and rfanalysis are convergent: same language, same engine — one has the infrastructure, the other has the pipeline. The synthesis merges them.

---

## 3. Why Django+DuckDB is the spine (not NestJS+Timescale)

This decision was made against your stated constraints, not on general preference:

- **On-site + forensic (batch) workload.** DuckDB is an embedded analytical engine ideal for "rebuild the whole analysis per run." TimescaleDB is built for continuous high-ingest time-series with real-time continuous aggregates — power you'd operate but not use in a nightly-report model. Fewer moving parts on a warehouse server matters more than raw ingest ceiling.
- **The host already exists and is Django.** `bss_earv1` is a Pegasus project with Celery, Redis, Postgres, DRF, a dashboard, and health checks already wired. The reporting UI, the scheduler (celery-beat), and the API you'd want are **already in the box**. Adopting NestJS means running a second stack beside it.
- **Correctness is already ahead on this branch.** `rfanalysis` is the only implementation carrying the `hours_present` fix (the bug that made the original tools suppress the high-frequency trackers they were built to find). earnest-v3 and eartest both predate it.
- **Dev/prod parity & reproducibility (your stated priorities).** DuckDB is a single file; the pipeline SQL is in git; config will be a committed TOML. No Timescale extension version-matching between dev and prod.
- **earnest-v3's scheduler is actively wrong for your scale.** It processes **one agent per interval on rotation** — at 60 sensors a sensor is analyzed once per 60 cycles. The Celery model in this plan runs all sensors per batch. Adopting earnest-v3 means rewriting its core scheduler anyway.

**What NestJS/earnest-v3 did better (and we therefore harvest):** a real REST API surface, an `rf-device-status` monitoring module, mature export queries, and an explicit `edge_filter_rules` table proving the FILTER/AGGREGATE/FORWARD contract must be honored server-side. All portable into Django.

---

## 4. Target architecture (on-site, LAN-primary)

```
 WAREHOUSE PC (×7–15)                          ON-SITE SERVER (1)
 ┌───────────────────────────┐                 ┌──────────────────────────────────────────┐
 │ uplink-scanner@a  (HackRF) │                 │  Docker Compose (Pegasus stack)            │
 │ uplink-scanner@b  (HackRF) │                 │                                            │
 │ uplink-scanner@c  (HackRF) │   JSONL.gz      │  ┌─────────┐  ┌─────────┐  ┌────────────┐  │
 │ uplink-scanner@d  (HackRF) │  ──────────►    │  │ web     │  │ celery  │  │ celery-beat│  │
 │        │ (local queue)     │  LAN share /    │  │ (Django │  │ worker  │  │ (schedule) │  │
 │        ▼                   │  R2 sync   │  │  +DRF)  │  │         │  │            │  │
 │ uplink-uploader           │                 │  └────┬────┘  └────┬────┘  └─────┬──────┘  │
 │  ├─ edge filter           │                 │       │            │             │         │
 │  └─ push ─────────────────┼─────────────────┼───────┴──────┬─────┴─────────────┘         │
 │                           │   (R2 = off-site │              ▼                             │
 │                           │    backup only)  │      ┌───────────────┐   ┌─────────────┐   │
 └───────────────────────────┘                 │      │ Postgres 17   │   │  DuckDB file│   │
                                                │      │ (app, runs,   │   │  (raw obs + │   │
                                                │      │  results,     │   │  pipeline)  │   │
                                                │      │  sensors)     │   └─────────────┘   │
                                                │      └───────────────┘                     │
                                                │      report artifacts → /reports volume     │
                                                └──────────────────────────────────────────┘
```

### Transport — DECIDED: R2-primary (central cloud hub)

**Multiple facilities with an often-remote server make a central cloud object store the CORRECT hub, not a compromise.** All facilities' agents push to one Cloudflare R2 bucket under per-agent folders (`{facility}-{host}{device}/…`); server(s) pull by facility prefix, location-independent. An earlier single-site draft recommended MinIO/LAN-primary — **that recommendation is REVERSED and MinIO is dropped** (it only made sense when producer and consumer shared one LAN). A per-facility local cache could return later *only* as an optimization for a chronically-bad-WAN site. Consequence: store-and-forward (L5) is **load-bearing infrastructure**, not a safety net — inter-facility WAN is the weak link.

**Identity (locked):** `agent_id = {facility}-{host}{device}`, lowercase, `^[a-z]{3}-rf\d+[a-d]$` (e.g. `ral-rf2a`); facility = 3-letter code from `FACILITY` env, immutable. Correlation scopes to `(facility, band_group)`.

---

## 5. The consolidated `rfanalysis` app (server)

Lives at `bss_earv1/pegasus/apps/rfanalysis/`. Spine = the app built this session, plus harvested modules.

### 5.1 Data model (Postgres)
- `AnalysisRun`, `TrackerCandidate`, `InfrastructureSignature` — **already built.**
- **NEW `Sensor`** (registry): `agent_id` (unique), `host`, `sensor_index`, `band_group` (LOW/HIGH — see §5.4), `bearing_deg`, `offset_feet`, `config_overrides` (JSON, nullable), `active`. Editable in Django admin.
- **NEW `SensorReading`/`RunSummary`** (optional, for dashboard): per-run rollups so the web tier never needs DuckDB access.
- **NEW `CorrelatedDetection`**: cross-sensor confirmed trackers (see §5.4).

### 5.2 Analytics (embedded DuckDB) — **already built, with fixes to add**
Ported stages: power clustering, signal filtering, event consolidation (gap/bucket), time clustering (lag/lead/long-interval). Config via `PipelineConfig`/`SignalFilterConfig`, all values validated in `sqlsafe`.

### 5.3 Correctness fixes (must-do before production)

**(a) Edge-filter awareness — CORRECTNESS, top priority.**
The agent emits three record types: `FORWARD` (all raw rows), `AGGREGATE` (one median record per 5-min cluster), `FILTER` (one summary record). The current pipeline treats every row as a detection event. An `AGGREGATE`'d signal therefore arrives as one record per 5-min file → the pipeline sees a perfect 300-second period (the *file-rotation cadence*, not the signal) → false CRITICAL tracker. **Fix:** event/CV analysis runs on `filter_action = 'FORWARD'` (or NULL for legacy) rows only; `FILTER`/`AGGREGATE` rows still count toward presence/infrastructure classification, weighted by `observation_count`. earnest-v3's `edge_filter_rules` table confirms this contract was known; honor it in SQL.

**(b) `hours_present` bug — ALREADY FIXED in rfanalysis.** Keep the fix (`COUNT(DISTINCT date_trunc('hour', time_bucket))`). Note in docs that earnest-v3/eartest baselines will differ.

**(c) Noise-floor realism.** Agent scanner floor default is **−60 dBm** (per eartest) / −50 (per uplink constants) — nothing quieter is recorded. Server defaults referencing −110 are dead. Set server noise-floor config to match agent reality and document that sweep dBm are **uncalibrated relative** values (fine for binning, not absolute power claims).

### 5.4 Cross-sensor correlation — the capability that justifies the array

**Topology reality (from agent `DEVICE_RANGES`):** sensors a/c cover **Low bands (662–850 MHz)**, b/d cover **High bands (1709–1911 MHz)**. A given EARFCN appears on **at most two** sensors of a PC — the same-band pair. "3 of 4 antennas agree" is physically impossible; correlate **within band group**.

**What it buys:** a candidate seen on both same-band sensors of a host, in overlapping time windows, is a far stronger detection than any single-sensor score. With `bearing_deg` + `offset_feet` on the `Sensor` rows, the RSSI differential across the ~20-ft-spaced antennas gives coarse direction/proximity — this is exactly how the real analyst report pinned EARFCN 20595 as "on the vehicle, high power." Across a 60-sensor warehouse grid, correlation turns per-sensor candidate lists into **location-aware threat detection**: which trackers appear at multiple points, moving or stationary, and where.

**Implementation:** a pure-Postgres query over `TrackerCandidate` joined on `Sensor.band_group` and time overlap, materialized into `CorrelatedDetection`. No DuckDB access → lives comfortably in the (read-only) web tier and the report generator.

### 5.5 Ingest ledger — required for the batch/timed workflow
Agents upload on a timer; a straggler PC can upload a backlog whose *filenames* fall outside a "last N hours" window. Filename-date filtering would silently skip never-ingested files. **Fix:** an `ingested_files` table in DuckDB (filename, size/etag, rows, ingested_at). Loader = list → anti-join ledger → load only new → record in the **same transaction** as the INSERT. R2 PUTs are atomic, so no half-file risk. Uploads are write-once (5-min/50 MB rotation, pending→processing→completed), so filename keying is safe; add size/etag if agents could ever re-push a grown file.

### 5.6 Reporting (harvest from eartest + earnest-v3)
- **Sync:** adopt gittest2's `services/sync.py` (typed boto3, concurrency, adaptive retry, R2 checksum quirk handled, unit-tested) as the in-app sync module — no TS port needed.
- **Observability:** adopt gittest2's `core/resilience.py` (error taxonomy, retry decorator, circuit breaker), `core/metrics.py` (+/metrics server), and `core/health.py` for the Celery pipeline tasks; its Grafana dashboard JSON seeds the ops view.
- Port `ForensicReportGenerator` output shape (structured, methodology-traceable) from eartest into a Django management command / Celery task.
- Use the real `analysis/reports/*.md` and `FINAL_THREAT_ASSESSMENT.md` as **templates** — they're the proven output format (threat priority, classification, behavior, recommendation).
- Reports written to a `/reports` volume + surfaced in the dashboard; optional nightly off-site copy.

### 5.7 Config file (your Claude-Code-on-server tuning loop)
`rfanalysis.toml` (committed, volume-mounted): `[pipeline] [filtering] [consolidation] [time_clustering] [time_clustering.long_interval]` sections + `[profiles.*]` overrides, mirroring the agent's env-var vocabulary so a Claude Code session reasons about both ends. Promote hardcoded SQL literals (stationary-penalty ratio, CRITICAL cutoffs, duty-cycle multiplier, burst gaps) into config. Add `manage.py check_config` (validate without running). Precedence: dataclass defaults → TOML → per-`Sensor` overrides (noise floor only, where physically justified) → CLI flags. Every run snapshots resolved config to `AnalysisRun.config_snapshot`.

---

## 6. Agent hardening (uplink-monitor)
Minimal — it's the most production-ready piece. Changes:
- **Uploader stays pointed at R2** (decided): per-agent prefixes carry facility; verify `FACILITY` + `R2_PREFIX` per host.
- **Add `sensor_index`/`band_group` to agent metadata** if not already derivable from `agent_id` (it is: `rf{N}{a|b|c|d}`), so the server `Sensor` registry can be auto-seeded.
- **Confirm write-once** file naming (done — 5-min/50 MB rotation) and 24 h `completed/` retention is enough buffer for the server sync cadence.
- **Provisioning:** template the systemd unit install (`setup-agent.sh` exists) for repeatable rollout across 7–15 identical PCs — one script, per-host `DEVICE_NAME`/serials in `.env`.

---

## 7. Scaling math (≈60 sensors)

**Ingest volume.** Post-edge-filter (FILTER/AGGREGATE collapse most rows; only FORWARD is full-fidelity). Even assuming each sensor emits a few MB/day of gzipped JSONL after filtering, 60 sensors is on the order of low-hundreds of MB/day → **single-digit GB/month**. DuckDB handles this comfortably on one file; the on-site server needs a sensible SSD (say 512 GB–1 TB) and 16–32 GB RAM.

**DuckDB sizing.** Set `memory_limit` to ~50–60% of server RAM, `threads` to core count, and enable disk-spill temp dir (eartest already learned this the hard way at 800 MB/1 GB on Railway — you have far more headroom on-site). Rebuild-per-run over a 72–168 h window stays in-memory easily at this volume.

**Report cadence.** Nightly full comparison across all active sensors is the baseline (celery-beat). A 168 h (7-day) window is what earnest-v3 used and suits sleep-cycle trackers (the 20595 case woke every ~3.5 days — a 72 h window would miss it). **Recommend default 168 h, nightly.** On-demand runs via management command / dashboard button.

**Concurrency.** DuckDB is single-writer. One analysis task at a time — enforce with a dedicated Celery queue (concurrency 1) for pipeline tasks. Sync and report tasks can run separately.

---

## 8. Deployment (on-site Docker)
Extend `bss_earv1/docker-compose.yml` (already has db/redis/web):
- **web** (Django+DRF), **celery worker** (pipeline queue, concurrency 1; plus a general queue), **celery-beat** (nightly schedule), **db** (Postgres 17), **redis**.
- **Volumes:** `duckdb_data` (the analytical file), `raw_data` (synced observations), `reports` (artifacts), plus a mounted `rfanalysis.toml`.
- Local `raw_data` volume for synced R2 objects (no MinIO — R2-primary decided).
- **Backups:** nightly `pg_dump` + DuckDB file copy + reports → off-site (R2). Postgres holds results/registry; DuckDB is rebuildable from raw but back it up to skip re-ingest.
- Health checks already present (`django-health-check`); add a pipeline-freshness check (last successful run age).

---

## 9. Phased rollout

**Phase 0 — Consolidate & decide (0.5 wk).** Land this plan in `bss_earv1` as `docs/`. Archive ear-compared, earsyncr2core (keep verify-sync if wanted). Transport decided: R2-primary. Remaining Phase-0 decision: server topology (central vs per-facility).

**Phase 1 — App integration (1 wk).** Move `rfanalysis` into `pegasus/apps/`. INSTALLED_APPS, settings (`RFA_*`), migrations. `Sensor` model + admin. `make init` boots the whole stack. Smoke-test the pipeline against one agent's real data.

**Phase 2 — Correctness (1 wk).** Edge-filter-aware analysis (§5.3a). Ingest ledger (§5.5). Noise-floor realism (§5.3c). Re-verify against known targets (20595 must surface).

**Phase 3 — Correlation + reporting (1–1.5 wk) — NOTE: this is Level 2 work (separate effort per scope banner; kept here for the overall roadmap).** `Sensor` band-group correlation → `CorrelatedDetection`. Port `ForensicReportGenerator` as Celery task. Dashboard: run history, candidate tables, correlation view, report downloads (shadcn components already present).

**Phase 4 — Config + scheduling (0.5 wk).** `rfanalysis.toml` + `check_config` + promoted literals + per-sensor overrides. celery-beat nightly 168 h run.

**Phase 5 — Fleet rollout (1 wk).** Template agent install across all PCs. Seed `Sensor` registry. Point transport at server. Run for a week; validate volume, timing, report quality against manual analyst baseline.

**Phase 6 — Ops hardening (0.5 wk).** Backups, health/freshness checks, restore drill, off-site copy. Document the Claude-Code-on-server tuning loop.

_~6–7 weeks part-time, single operator + Claude Code. Phases 1–2 deliver a correct end-to-end pipeline; 3 delivers the differentiating capability; 4–6 make it operable._

---

## 10. Risks & open decisions

- **Transport.** DECIDED: R2-primary (multi-facility hub). Closed.
- **Server topology.** OPEN: central multi-facility server vs per-facility appliance — decide before Phase 1.
- **Raw-data retention.** Edge filter means full-fidelity raw exists only for FORWARD clusters; AGGREGATE/FILTER are summarized before leaving the PC, and `completed/` holds 24 h. If deep forensic re-analysis of *summarized* signals is ever needed, that fidelity is already gone — decide now whether any sensors should run FORWARD-all (no edge filter) for full capture at higher bandwidth cost.
- **Sweep calibration.** dBm are uncalibrated relative values; proximity/bearing from RSSI is *coarse*. Good for "which sensor is closest," not metric ranging. Manage expectations in reports.
- **Multi-facility (locked).** Design is multi-facility from the start: facility-scoped agent_id, one central R2 bucket, `Sensor.facility`, per-facility reports, correlation scoped `(facility, band_group)`.
- **DuckDB single-writer.** Enforced by Celery queue design; don't bypass it with ad-hoc concurrent runs.
