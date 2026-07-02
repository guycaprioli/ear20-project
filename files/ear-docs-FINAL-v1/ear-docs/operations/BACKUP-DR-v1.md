# EAR System — Backup & Disaster Recovery (v1)

## Sacred vs rebuildable (the whole strategy in one table)
| Data | Class | Backup | Restore |
|---|---|---|---|
| R2 bucket (obs/ctx/health) | **SACRED — primary source of truth** | R2 lifecycle/versioning; optional periodic `rclone` snapshot to a second provider | It *is* the recovery source for everything analytical |
| Postgres (Sensor, ledger, runs, InfraSignatures, L2 results later) | **SACRED** | Nightly `pg_dump` to `/srv/ear/backups/` + push to R2 `_backups/` prefix | `pg_restore`; then re-run pipeline for anything mid-flight |
| Handoff Parquet | Sacred-ish (the dataset) | Sync to R2 `_handoff-archive/` after each run | Or rebuild: re-run pipeline over the window |
| `ear-fleet` (configs, L4.5 rules, corpus log) | **SACRED** | Git → GitHub mirror (D-048) + sops-encrypted secrets already committed | Clone + re-render |
| Git bare repos (`/srv/git`) | Sacred | GitHub mirrors, pushed at session close (D-048) | Re-clone from GitHub |
| L1 DuckDB files | Rebuildable | none | Delete → next pipeline run rebuilds from R2 + ledger |
| MinIO (dev) | Disposable | none | Re-seed from replay datasets |
| Agent `pending/`/DLQ | Transient (un-uploaded field data!) | none — protected by disk ceiling + heartbeat visibility | Uploads on recovery; poison files reviewed |

## Loss scenarios
- **Dev box dies:** GitHub mirrors + R2 hold everything sacred. New box: run DEV-ENVIRONMENT-v2 step 0, clone, `pg_restore`, done.
- **Server host dies (prod):** containers are stateless but Postgres volume — restore last `pg_dump`, redeploy compose, pipeline resumes from R2 (ledger in the dump prevents re-ingest confusion; worst case: delete L1 files, full rebuild).
- **Warehouse host dies:** its un-uploaded `pending/` is the only true loss window (bounded by upload cadence ≈5 min + any outage backlog). Replacement host = provisioning checklist + same `.env` from ear-fleet; new HackRF serials updated; sensor identity persists.
- **R2 catastrophe:** the reason for the optional second-provider snapshot; without it, history is lost but the fleet repopulates going forward. Decide appetite in Q-21.

## Drills (WP-O2 exit requires both)
1. **Postgres restore drill:** restore last dump to a scratch container; dashboard renders; ledger intact.
2. **Server rebuild drill:** fresh compose from images + restored dump + empty `l1/` → one pipeline run → handoff Parquet matches pre-drill run (determinism check doubles as restore validation).

## Automation (built in WP-O2)
Nightly host timer: `pg_dump` → `/srv/ear/backups/` → push to R2 `_backups/` → prune >30 d local, >180 d remote → `notify()` on failure (D-049).
