# EAR System — Open Questions (v1)
### Clarifications wanted from Guy, ordered by when they block. Answer inline and commit, or answer in-session.

## Blocks M0 (repo creation / first sessions)
- **Q-01** Repo naming: accept the `ear-` prefix and the five names (`ear-docs`, `ear-agent`, `ear-server`, `ear-fixture`, `ear-fleet`)? Any GitHub org vs personal account preference?
- **Q-02** Confirm all repos **private** (D-036).
- **Q-03** Did the Pass-2 boundary-reconciliation code turn up anywhere (branches, scratch dirs, a ninth repo)? If not found by M4, which candidate: boundary-zone / centroid-nearest / valley-test? (My provisional lean: centroid-nearest — best accuracy-per-compute; the WP-S5 benchmark will confirm affordability.)
- **Q-04** Server topology (D-004): central multi-facility server, or per-facility appliance? Affects M3 deploy shape and where the L2 DB will eventually live.
- **Q-05** False-split vs false-merge: confirm my working assumption that a false split (missed tracker) is worse than a false merge, so Pass-2 leans toward merging boundary cases.

## Blocks M2 (agent build)
- **Q-06** Sweep-collapse window: is 2 s right, or should it derive from the actual sweep revisit time? What IS the sweep revisit time per band with 4 devices?
- **Q-07** Noise floor(s): the old constants said −50, docs said −60, and it's per-agent anyway — what are the real starting values per environment type (open warehouse vs racked areas)?
- **Q-08** `thresholds.json` edge-filter values from v4.3 — carry over as v5 defaults, or re-derive during the test-host soak?
- **Q-09** ~~Heartbeat interval~~ answered by D-045 (heartbeat = per upload run, ~5 min). Remaining: DLQ (poison-file) retention policy — proposal: keep + alert, manual review, never auto-delete?
- **Q-10** Hardware baseline for monitoring PCs (CPU/RAM/disk) — sizes the disk ceiling default and confirms 4 scanners fit comfortably.

## Blocks M3–M4 (server build / validation)
- **Q-11** Where does `ear-server` deploy — a facility/central box you own, or a host (Render? Hetzner? other)? Containerized either way (D), but ops differ. Postgres: managed or in-compose?
- **Q-12** Fresh R2 bucket name/account for production, and does the **old bucket data still exist and remain accessible** for the legacy corpus (WP-S10)? Roughly how much history survives?
- **Q-13** Confirm the EARFCN band plan (a/c = 662–850 MHz; b/d = 1709–1911 MHz) is still the target for the redeploy, or is the band plan being revisited?
- **Q-14** Facilities at launch: which codes (ral? cht? dal?), how many hosts each, and WAN quality per site (sizes SAF/DLQ expectations)?
- **Q-15** Retention targets: raw landing (proposal: 90 d), L1 DuckDB files (cache — rebuildable, keep last run only), handoff Parquet (proposal: indefinite — it IS the dataset), context feed (proposal: 180 d). Confirm or adjust.

- **Q-19** Pipeline run cadence (D-044): how often should the ephemeral Level 1 job run — hourly, or a few times daily? (Forensic cadence suggests 2–4×/day is plenty; affects timer config and dashboard freshness.)

- **Q-20** Do raw `hackrf_sweep` stdout captures exist from the old effort, or only enriched JSONL? (Raw gives the new parser real-data coverage on day one; otherwise first coverage comes from the first field `--record`.)

- **Q-21** Off-R2 disaster appetite (BACKUP-DR): is a periodic second-provider snapshot of the R2 bucket wanted (cost vs total-history-loss risk), or accept fleet-repopulates-forward if R2 is ever lost?

- **Q-22** (DEFERRED by Guy, pre-first-facility at earliest): fleet-volume load gate — replay at fleet scale (speed factor), assert pipeline completes within time/disk budget. Not a concern at this stage; revisit before a facility depends on run cadence.

## Level 2 charter (later, no urgency)
- **Q-16** When Level 2 is chartered: single consolidated report format like the old ForensicReportGenerator, or per-facility scheduled reports? Who is the report audience (you only, or facility staff)?
- **Q-17** Any cross-facility analysis ever wanted (same tracker appearing at two sites), or is correlation strictly within-facility forever? (Affects whether L2 central DB must read all partitions.)

## Taxonomy field truth (level2/DEVICE-BEHAVIOR-TAXONOMY-v1 — answer when convenient)
- **Q-T1…Q-T4** — observed wake intervals per tracker sub-pattern; known confusion cases; missing groups; whether shift-hour correlation holds at your facilities.

## Deployment-gating (not architecture — but gate real deployment)
- **Q-23** LEGAL/PRIVACY: has workplace RF monitoring at your facilities been cleared as lawful (employee privacy, wiretap/ECPA-style statutes on signal interception, jurisdiction-specific rules)? This gates deployment harder than any technical risk — the security reviewer will raise it, but the clearance is a you-decision, not a design one.
- **Q-24** Build-phase API cost: is the cumulative spend (dozens of Opus/Sonnet build sessions + the ~$10k review) budgeted/acceptable? Unstated so far.
- **Q-25** Move Q-16/Q-17 (Level 2 report format/audience) into the early-answer set — they now block S19d.

## Post-effort review
- **Q-18** Review `DECISION-LOG-v1.md` items **D-030…D-041** `[CLAUDE-DECIDED]` — veto or amend any before repos are created (cheapest moment to change them).
