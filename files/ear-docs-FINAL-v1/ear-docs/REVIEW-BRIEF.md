# EAR System — Independent Review Brief (for the 7-expert steelman panel)
### Read `README.md` first (the index). This brief tells you where to push hardest and hands you the weak spots I already suspect, so your time goes to finding what I couldn't see — not rediscovering what I flag here.

## What you're reviewing
A complete **pre-code** architecture + build plan for a multi-facility RF tracker-detection system: Python agents (HackRF capture, edge feed-prep, R2 upload) → containerized Django/DuckDB Level 1 server (gated ingest → event construction → Parquet handoff) → Level 2 analysis (feature extraction → G1–G6 classification → scoring → dashboard). 60 decisions logged with rationale; 19-persona executable fixture; session-by-session runsheet with Opus/Sonnet assignment. Single operator, greenfield rebuild of a system that ran for ~6 months.

## Ground rules for the review
- **Steelman first, then break.** Assume each decision had a reason (it's in `DECISION-LOG`); argue the strongest case FOR it, *then* attack that. A critique that ignores the logged rationale is weaker than the design.
- **Everything is reversible now** — nothing is built. The cost of a found gap here is one edit; post-build it's a rewrite. Bias toward flagging.
- Land findings as: **severity (blocker/major/minor) · which decision or doc · the failure scenario · a proposed alternative.**

## Suggested reviewer assignments (7)
1. **RF/SIGINT domain** — the physics & methodology: undersampling model, two-pass binning, event=burst-group, the G1–G6 taxonomy, the 600 s / 40× / 0.95 tunables. *Is the science right?*
2. **Data engineering** — two-table ingest, per-agent DuckDB, Parquet handoff, ephemeral-job model, retention. *Does the pipeline hold at scale and under failure?*
3. **Distributed systems / reliability** — store-and-forward, ledger idempotency, recovery-by-rebuild, late arrival, WAN-partition behavior, the run lock. *What breaks under partial failure?*
4. **Security** — surveillance data sensitivity, sops/age secrets, R2 credential blast radius, access control (the thinnest area — see below).
5. **Test/QA** — the fixture-as-contract model, 19 personas, double-derivation, gate protocol, UAT journeys. *What's untested? Can a persona pass while the contract is violated?*
6. **SRE/operations** — health checks, alerting, runbook, backup/DR, fleet updates, single-operator load. *Can one person actually run this?*
7. **Software architecture / PM** — the 6-repo split, contract-as-source-of-truth, the runsheet realism, Opus/Sonnet assignment, decision-log discipline. *Is the plan buildable as written?*

## Weak spots I already suspect — start here, then go past them
- **Pass-2 boundary calc is unresolved** (open algorithm; a stub with an xfail persona). If the real method never surfaces, is the fallback (centroid-nearest, provisional) good enough, and is the seam honest?
- **Correlation method is provisional** (time-overlap only). Same question — is the placeholder marked clearly enough that it won't be mistaken for validated?
- **G1-vs-G5 discrimination** (battery tracker vs fixed IoT telemetry) leans hard on the labeled corpus that doesn't exist yet. Is the cold-start classification (before corpus) acceptable, or does it over-claim?
- **Security is the least-developed area** — WP-O3 has it as a stub; D-050 covers secrets but not access control, audit, or the legal/privacy dimension of warehouse RF surveillance. Push here.
- **Single-operator bus factor** — the whole thing assumes one person + AI sessions. Is the decision-log/fixture/doc discipline actually enough to survive a 6-month gap or a handoff?
- **Volume/scale is deferred** (Q-22, my call) — no fleet-scale load gate before first facility. Is that deferral safe?
- **The fixture defines truth** — if a persona's expected output is wrong, the error becomes law. Double-derivation (D-057) is the mitigation; is it sufficient?
- **Time-sync** — gap-based analysis assumes synced clocks; drift detection is a heartbeat field + WP-O3, not yet a contract. Under-specified?

## Questions already open (don't re-raise; extend if you have a view)
`OPEN-QUESTIONS-v1.md` holds Q-01…Q-22 + Q-T1…Q-T4 (field truth) + the D-030…D-059 `[CLAUDE-DECIDED]` items flagged for the operator's own review. If your finding overlaps one, reference it.

## What would most help
Rank-order the top 5 risks to *project success* (not just correctness). Name anything that should change **before** repo creation (D-030-class structural calls) separately from what can change during build. And flag any place the documents claim more confidence than the evidence supports — over-claiming is the failure mode I can't self-detect.
