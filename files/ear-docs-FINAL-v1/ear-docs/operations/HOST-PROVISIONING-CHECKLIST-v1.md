# EAR System — Host Provisioning Checklist (v1)
### Template — copy per host into `ear-fleet/hosts/{facility}/{host}/CHECKLIST.md` and check off. A host is not `qa`-eligible until every box is checked.

**Host:** `____-rf__`  ·  **Facility:** `___`  ·  **Date:** ______  ·  **Provisioned by:** ______

## Hardware
- [ ] PC meets baseline (Q-10); disk ≥ ___ GB free; UPS/power noted
- [ ] 4 × HackRF connected via powered hub; antennas mounted; **bearing + offset recorded** per device: a:____ b:____ c:____ d:____
- [ ] `hackrf_info` lists 4 serials; **serials recorded**: A:________ B:________ C:________ D:________

## OS
- [ ] Ubuntu LTS current; NTP active (`timedatectl` → synchronized: yes) — **hard requirement** (R1; gap-based analysis)
- [ ] Service user created; `plugdev` membership; udev rules for HackRF
- [ ] python3.12 + venv; git access to hub/mirror (D-048)

## Agent install (D-051)
- [ ] `ear-agent-update <current tag>` — venv installed, version verified
- [ ] Config rendered from ear-fleet (sops, D-050): `FACILITY=___` `HOST_PREFIX=rf__` `HACKRF_SERIAL_A..D` set (never first-found)
- [ ] **Noise floor set per device** (Q-07, environment-appropriate) and **recorded in ear-fleet**: a:___ b:___ c:___ d:___ dBm
- [ ] `S3_ENDPOINT/BUCKET/credentials` valid; `ALERT_URL` set (D-049)
- [ ] L4.5 override rules file present (may be empty)

## Bring-up verification
- [ ] `systemctl start uplink-scanner@{a..d}` — all four resident, staggered, journal clean; startup identity validation passed (regex + serials)
- [ ] `uplink-upload.timer` enabled; first tick uploaded: R2 shows `{facility}-rf_x/obs_…` **and** `ctx_…` gz files (spot-check: real gzip, `schema_version: v5`)
- [ ] `_health/{agent_id}.json` fresh for all four; `queue_depth` draining; `status: ok`
- [ ] Kill-WAN 10-min test: pending/ buffers, no crash, drains on restore *(first host per facility at minimum)*

## Registration
- [ ] All four sensors appear **pending** on the dashboard; facility/band-group correct
- [ ] ear-fleet committed: checklist, serials, bearings, noise floors, `.env` (sops)
- [ ] Promoted to **qa** → isolation review done → **active** (date: ______)
