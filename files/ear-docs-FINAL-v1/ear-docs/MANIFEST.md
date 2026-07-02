# EAR System — Document Manifest (final v1 · pre-code · for expert review)

**Start:** `README.md` (index) → `REVIEW-BRIEF.md` (reviewers) → by area below.
**State:** 62 decisions (`DECISION-LOG-v1`, D-001…D-062), 25+4 open questions (`OPEN-QUESTIONS-v1`, Q-01…Q-25 + Q-T1…Q-T4), 19 fixture personas, runsheet S00–S21 (+S19a–e). Nothing built.

## Tree
```
README.md                     index / entry point
REVIEW-BRIEF.md               for the 7-expert panel
MANIFEST.md                   this file
DECISION-LOG-v1.md            D-001..D-062 + rationale ([CLAUDE-DECIDED] = review)
OPEN-QUESTIONS-v1.md          Q-01..Q-25, Q-T1..Q-T4

contracts/                    THE LAW (implemented, not redesigned)
  DATA-WORKFLOW-RULES-v3.md         R0..R9 chain
  LOCAL-HOST-DATA-FLOW-v2.md        L0..L6.5 agent chain
  EVENT-CONSTRUCTION-METHODOLOGY-v1.md   event=burst-group, two-pass, noise filter
  EVENT-DATASET-HANDOFF-v1.md       L1/L2 interface (Parquet)
  schemas/                          machine-readable: signal, context, heartbeat, handoff

components/                   BUILD SPECS
  EAR-AGENT-SPEC-v1.md              agent (Python)
  RFANALYSIS-SCOPE-v3.md            L1 server
  GOLDEN-FIXTURE-SPEC-v1.md         19 personas + assertions
  DASHBOARD-DESIGN-v1.md            screens + UAT journeys J1-J7

level2/
  DEVICE-BEHAVIOR-TAXONOMY-v1.md    G1-G6 classifier spec

plan/
  GREENFIELD-DEVELOPMENT-PLAN-v1.md master plan, 6 repos, milestones M0-M5
  BUILD-RUNSHEET-v1.md              S00-S21, Opus/Sonnet, gates, verification protocol

operations/
  BUILD-SESSION-BOOTSTRAP-v2.md     paste per Opus session
  DEV-ENVIRONMENT-v2.md             Strix Halo single-box deploy + local git hub
  HEALTH-CHECKS-v1.md               runtime functional confirmation
  RUNBOOK-v1.md                     day-2 ops
  BACKUP-DR-v1.md                   backup + disaster recovery
  RECOVERY-RETENTION-v1.md          pipeline recovery + retention
  HOST-PROVISIONING-CHECKLIST-v1.md per-host template
  PARAMETER-TUNING-v1.md            A/B framework + tunables

archive/                      superseded versions (history; not for review)
```

## Reading paths by reviewer (see REVIEW-BRIEF for assignments)
- **RF/domain:** EVENT-CONSTRUCTION-METHODOLOGY, DEVICE-BEHAVIOR-TAXONOMY, LOCAL-HOST-DATA-FLOW (L0-L4.5)
- **Data eng:** RFANALYSIS-SCOPE, EVENT-DATASET-HANDOFF, RECOVERY-RETENTION, DATA-WORKFLOW-RULES (R5-R9)
- **Distributed/reliability:** LOCAL-HOST-DATA-FLOW (L5), DATA-WORKFLOW-RULES (R0-R4), HEALTH-CHECKS, BACKUP-DR
- **Security:** DECISION-LOG (D-050/D-036/D-002), DEV-ENVIRONMENT, + the acknowledged WP-O3 gap
- **Test/QA:** GOLDEN-FIXTURE-SPEC, BUILD-RUNSHEET (verification protocol), DASHBOARD-DESIGN (UAT)
- **SRE/ops:** RUNBOOK, HEALTH-CHECKS, BACKUP-DR, HOST-PROVISIONING, PARAMETER-TUNING
- **Arch/PM:** GREENFIELD-DEVELOPMENT-PLAN, BUILD-RUNSHEET, DECISION-LOG (structure D-030..D-062)
