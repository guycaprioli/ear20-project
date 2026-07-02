# EAR System — Dev Environment (v1)
### Target: AMD Strix Halo · 128 GB · Ubuntu Linux (native) · container-per-service, production parity (D-047)

**Principle (D-047):** the dev stack runs the **same container topology as production** — one container per server service, same images (GHCR), same compose file. Dev differs by a compose **profile** and env values only. The single intentional dev-only addition is MinIO standing in for R2.

## Container topology (parity table)

| Service | Production | Dev (Strix Halo) | Notes |
|---|---|---|---|
| Object store | **Cloudflare R2** (external) | `minio` container (+ bucket-init) | The only substitution; `S3_ENDPOINT_URL` is the switch. Dev-profile only. |
| Reverse proxy / TLS | `proxy` (Caddy) | `proxy` (same, self-signed/HTTP) | Kept in dev for parity even though optional. |
| Web dashboard | `web` (gunicorn, GHCR image) | `web` (same image) | Resident (D-044). |
| Pipeline job | `pipeline` — **same image as web**, `compose run --rm` via systemd timer | identical | Ephemeral (D-044); non-zero exit = alert. |
| Database | `postgres` container **or managed (Q-11)** | `postgres` container | Volumes for persistence. |

**NOT containerized (both environments):** the agent — replay/real scanners + ephemeral uploader run as bare venv + systemd on the host (D-045), exactly as on warehouse PCs. Native Ubuntu on the dev box keeps HackRF USB trivial.

## Networking
- One compose bridge network; in-network DNS by service name (`http://minio:9000`, `postgres:5432`).
- **Two-address rule (pin in `.env` samples):** host-side agents → `localhost:9000`; in-network pipeline → `http://minio:9000`. Same bucket, two addresses — expected, not a bug.

## Sizing (validated for this box)
Resident stack < 4 GB. Pipeline peak = pool size × DuckDB `memory_limit` — start **4 × 4 GB**, headroom to ~10 × 8 GB with 128 GB. **Provision 2 TB+ NVMe** (landing + L1 files + Parquet + MinIO + replay datasets is the real constraint, not RAM).

## Bring-up order (materialized by WP-A6 / WP-S1)
1. Docker + compose plugin; clone repos; `docker compose --profile dev up -d` (proxy, web, postgres, minio).
2. Agent venv on host; seed `Sensor` registry (a `tst-` facility) via admin; place replay dataset.
3. Start replay scanner(s) (`--source replay`, time-shift on) → uploader timer ticks → MinIO fills.
4. `docker compose run --rm pipeline manage.py run_level1` → watch landing → events → Parquet partitions appear.
5. Fixture suite green against the same stack.

## Test progression on this box
1. **Replay loop** (above) — all contracts, no hardware, no cloud.
2. **Live-capture step:** plug HackRF into the Strix box → run one real scanner (`--source hackrf`, `--record` on) alongside replay agents → first real RF through the v5 parser; recorded capture joins the replay datasets in `ear-fixture`.
3. **Cloud step:** flip one agent's `S3_ENDPOINT_URL` to the real (fresh) R2 bucket → verify identical behavior → test-host soak (WP-A5) follows on real hardware.
