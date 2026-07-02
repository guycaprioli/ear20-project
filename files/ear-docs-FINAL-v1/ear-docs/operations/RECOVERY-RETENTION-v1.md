# EAR System — Recovery Semantics & Data Retention (v1)
### Pre-writes two of WP-O3's four ops contracts (the decisions that determine them — D-023/D-024/D-044 — are already made). Time-sync and security remain scheduled for WP-O3.

## Recovery semantics (server pipeline)

**The model: per-agent disposability.** L1 DuckDB files are cache (D-023); the R2 bucket + ingest ledger are truth. Therefore recovery is never surgical:

- **Mid-run crash / non-zero exit:** nothing to clean *inside* a DuckDB file that matters — the next run's per-agent stages are full rebuilds over the window. Just re-run. If a file is suspected corrupt (DuckDB open error), **delete it**; the next run rebuilds it from R2 via the ledger.
- **Partial Parquet export:** impossible to observe (atomic temp-write+rename, D-024). Orphaned temp dirs are swept at every job start.
- **Ledger vs landing divergence** (crash between land and ledger write): prevented by same-transaction rule (R5.1); if ever observed, delete the agent's L1 file — rebuild reconciles.
- **Postgres loss:** restore per BACKUP-DR-v1; ledger restores with it; at worst re-ingest is idempotent by file identity.
- **Run overlap:** prevented by the run lock; a stale lock (host crash mid-run) older than the timer period is broken automatically with a `notify()`.

**Invariant:** no recovery procedure ever edits data in place. Recovery = delete + rebuild, or restore + re-run. (This is why the two-table/per-agent/atomic-export decisions were made — recovery falls out of them.)

## Retention & lifecycle (defaults — confirm/adjust via Q-15)

| Store | Retention | Mechanism |
|---|---|---|
| R2 signal feed (`obs_`) | **13 months** | R2 lifecycle rule (covers a full year + reprocessing window) |
| R2 context feed (`ctx_`) | 6 months | R2 lifecycle rule |
| R2 `_health/` | overwritten per tick (latest only) | by construction |
| Raw landing tables (in L1 files) | 90 days rolling within the file | pipeline prunes at run start |
| L1 DuckDB files | last successful run only (cache) | rebuilt per run; deletable anytime |
| Handoff Parquet | **indefinite** — it IS the dataset | archived to R2 `_handoff-archive/` (BACKUP-DR) |
| Postgres: Sensor/ledger/runs | indefinite (small) | — |
| Quarantine tables | 12 months | pruned after review discipline (RUNBOOK) |
| pg_dump backups | 30 d local / 180 d in R2 | nightly timer |
| Agent pending/ | until uploaded (ceiling-bounded) | D-045 |
| Poison DLQ | until manually reviewed — never auto-deleted | RUNBOOK |
| Dev MinIO / sim data | per-simulation, pruned manually | DEV-ENVIRONMENT-v2 |

**Rules:** retention is enforced by *lifecycle policy and job-start pruning*, never by ad-hoc deletion; any retention change is a config change (versioned, R0.1); nothing that feeds an unreviewed quarantine or an unreplayed DLQ is ever purged.
