# EAR System — Operations Runbook (v1)
### Day-2 procedures. Written against contracts + D-044/045/049/050/051; verified/refined during M5.

## Onboard a new host/sensor
1. Provision per `HOST-PROVISIONING-CHECKLIST-v1` (ear-fleet copy). Agent starts uploading → folders appear in R2.
2. Dashboard shows the agent as **pending** (R5.0). Verify: facility/host correct, heartbeat fresh, serials match checklist.
3. Promote to **qa** → next pipeline runs analyze it in isolation. Review: noise floor sane vs siblings, event shapes plausible, quarantine near-zero.
4. Promote to **active**. Log in ear-fleet inventory.

## Retire/replace a sensor
Retire: set `disabled` (stops fetch; history retained). Replace hardware on same identity: update `HACKRF_SERIAL_x` in ear-fleet, re-render, restart scanner unit — identity persists (D-003). Never reuse a `{facility}-{host}` on different physical placement.

## Alert responses (D-049)
- **Dead heartbeat** (fresh data absent + heartbeat stale): host down or network. Check facility WAN first (multiple agents dark = WAN). Single agent: SSH → `systemctl status uplink-scanner@x uplink-upload.timer` → journal. Data loss is bounded to un-uploaded `pending/` only if the disk also fails.
- **Heartbeat `status: ceiling` / disk alert:** backlog growing (WAN outage or R2 auth). Fix connectivity; uploads drain automatically (pending/ = retry queue). If eviction ran, note the window in the run log.
- **Pipeline non-zero exit:** read job log. Contract violations land in quarantine with reasons — review table; a bad file batch is skippable (ledger-mark) after diagnosis. Mid-chain crash: safe to just re-run (per-agent rebuild semantics); if a L1 file is suspect, delete it — next run rebuilds (RECOVERY-RETENTION-v1).
- **Over-filter safeguard tripped:** a threshold or env change made the filter reject too much; run continued pass-all. Diff `config_version`s and recent L4.5 rule changes before touching thresholds.

## Quarantine review (weekly)
Group by reason + agent. Spike on one agent = that host's problem (clock, config, hardware); spike fleet-wide after a release = the release. Never delete quarantine rows without logging disposition.

## L4.5 rule change (tune out a new noisy EARFCN)
Add rule (EARFCN, facility scope, rule_id, rationale) in ear-fleet → commit → re-render host(s). Applies on the next uploader tick (D-045). Verify: signal feed volume drops; ctx heartbeats appear for that EARFCN. Record in the corpus log if it was investigated first.

## Confirmed device (tracker OR benign)
**Always** append to the labeled corpus log (ear-fleet): date, facility, agent(s), EARFCN, power, event signature, physical confirmation outcome. This is the future tuning ground truth (PARAMETER-TUNING-v1) — no exceptions, including negatives.

## Releases
Agent: tag → staged `ear-agent-update <tag>` one facility at a time; verify heartbeats carry the new `agent_version` before the next facility (D-051). Server: GHCR tag bump → `compose pull && up -d`; next pipeline run is on the new image. Any contract-touching release: fixture suite green first (D-037).

## R2 outage playbook
Agents: nothing to do — pending/ buffers, ceiling protects, heartbeats go stale from the server's view (expected). Server: pipeline runs fail/produce nothing new — silence alerts for the window. Recovery: uploads drain on next ticks; run pipeline; verify ledger picked up the backlog; check no agent hit ceiling/eviction.
