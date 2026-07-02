# EAR System — Independent Review: SECURITY & PRIVACY lens

**Reviewer role:** Independent expert, Security & Privacy. Steelman-first discipline.
**Method:** Read README → REVIEW-BRIEF → MANIFEST, then the full security surface (DECISION-LOG, OPEN-QUESTIONS, contracts L/R chains, schemas, DASHBOARD-DESIGN, DEV-ENVIRONMENT, HOST-PROVISIONING, BACKUP-DR, RECOVERY-RETENTION, RUNBOOK, HEALTH-CHECKS, EAR-AGENT-SPEC, GREENFIELD-PLAN WP-O3). `audit/` deliberately NOT read (isolation rule).
**Isolation note:** during a grep I saw two incidental preview lines from `audit/synthesis/load-bearing-decisions.md`. I did not open that file and reached every conclusion below independently from the primary documents.

The authors themselves flag this as the least-developed area (REVIEW-BRIEF:25, WP-O3 "security/access basics" a stub in GREENFIELD-PLAN:78). This review treats that flag as an invitation, not an excuse: a system this rigorous everywhere else has a genuinely load-bearing hole here, and the hole is cheapest to close now.

---

## 1. SURFACE ENUMERATION

Everything in scope for this lens, so nothing looks skipped.

### Decisions examined
- **D-002** — R2-primary transport, central Cloudflare hub, per-agent folders (credential scope, blast radius).
- **D-004** — server topology central vs per-facility (OPEN) — tenant/facility isolation, data residency.
- **D-022 / R5.0** — onboarding gate `pending→qa→active→disabled` (the one access-control-shaped contract that DOES exist — but it gates *ingest*, not *humans*).
- **D-036** — all repos private ("surveillance-adjacent, default closed"). The stated extent of the threat-model framing.
- **D-048** — local git hub on dev box; GitHub mirror at session close (where sacred data lives, blast radius of the dev box).
- **D-049** — alerting via `ntfy` topic over `ALERT_URL` (an unauthenticated pub/sub channel — surveillance-signal metadata leak surface).
- **D-050** — secrets via **sops + age**: one operator age key + one per server host; secrets committed to `ear-fleet`; rotation = re-encrypt + re-render. **The core of my mandate.**
- **D-051** — fleet updates: `ear-agent-update <tag>` fetch → pip install → render config → restart (supply-chain / update-channel integrity).
- **D-055 / D-042** — Level 2 in scope; `ear-analysis` renders forensic candidate/classification data in the Django dashboard (who can view surveillance conclusions).
- **D-056** — dashboard single-persona (operator); ADMIN = Django admin exposing Sensor registry + users; **"Pegasus auth" assumed, never specified.**
- **D-061** — licensing/IP: private, proprietary, all-rights-reserved (aligns D-036); `.gitignore` excludes secrets/.env/DuckDB.

### Contract clauses examined
- **R1** (observation record) — no field-level sensitivity classification; `power_dbm`, `earfcn`, `frequency_mhz`, `observed_at`, `agent_id` — the surveillance payload itself.
- **R4 / L6** (transport) — object keys `{agent_id}/obs_…`, `{agent_id}/ctx_…`, `_health/{agent_id}.json`. No mention of bucket encryption, TLS enforcement, credential scope, or object-level ACL.
- **R5.0** (agent authorization gate) — data-plane allowlist only.
- **R8 / R9** (results / forensic report) — persisted surveillance conclusions in Postgres + `/reports` volume. No access control, no export audit.
- **L0.5 / L4.5** — capture config + override rules git-tracked in `ear-fleet` (the same repo that holds secrets).
- **Schemas** — `observation-signal.v5`, `heartbeat.v1` (fields checked for PII/location leakage; `latitude/longitude` optional-and-unpopulated per L1 note).

### Operations docs examined
- **HOST-PROVISIONING-CHECKLIST-v1** — `S3_ENDPOINT/BUCKET/credentials valid` (one line; no scope/rotation), no disk-encryption box, no physical-security box, no de-provisioning/credential-revocation procedure.
- **BACKUP-DR-v1** — `pg_dump` pushed to R2 `_backups/`; agent `pending/`/DLQ classed "Transient (un-uploaded field data!)" with "none" backup — the un-encrypted surveillance-data-at-rest window.
- **RECOVERY-RETENTION-v1** — retention windows (13 mo signal, indefinite handoff/Postgres). No retention-as-privacy-minimization framing; no data-subject deletion path.
- **RUNBOOK-v1** — "SSH → systemctl" for host diagnosis (implies standing SSH access to warehouse PCs; no key management).
- **HEALTH-CHECKS-v1** — `web → HTTP /health/` endpoint (is `/health/` authenticated? DB-ping leak?); `R2 credential probe (cheap HEAD)`.
- **DEV-ENVIRONMENT-v2** — proxy/TLS "self-signed/HTTP", "optional even in prod" framing; MinIO in dev.

### Open questions examined / extended
- **Q-02** repos private (confirmed-shaped). **Q-12** R2 bucket / credential scope + legacy-bucket access. **Q-21** off-R2 DR snapshot. **Q-23** LEGAL/PRIVACY workplace RF monitoring lawfulness — **the hard gate**. Q-04 (topology) and Q-11 (deploy host, managed vs in-compose Postgres) touch isolation/at-rest.

### Absences I treat as first-class findings (nothing to cite because the doc does not exist)
- No **access-control contract** (an "A-chain" with the R-rule discipline) governing *human* read/write/export of surveillance data.
- No **audit-trail contract** (who viewed/exported which facility's candidates/reports, when).
- No **data-at-rest encryption** statement for Postgres, DuckDB (`l1/`, L2), handoff Parquet, R2 objects, or warehouse `pending/`.
- No **physical-compromise threat model** for an unattended warehouse PC holding R2 write credentials + un-uploaded surveillance data.
- No **credential blast-radius / revocation** design (what a stolen warehouse-PC cred can do; how to kill it).
- No **operator-key-loss / key-leak** blast-radius analysis for D-050.

---

## 2. FINDINGS

---

### FINDING-SEC.1 · severity: blocker · target: Q-23 (LEGAL/PRIVACY) + D-036 threat-model framing

**steelman.** The design is internally consistent about this being out of its lane. Q-23 (OPEN-QUESTIONS:41) explicitly names the exact statutes a competent lawyer would name — "employee privacy, wiretap/ECPA-style statutes on signal interception, jurisdiction-specific rules" — and states plainly "This gates deployment harder than any technical risk … the clearance is a you-decision, not a design one." That is honest: a design document cannot render a legal opinion, the operator owns the facilities and the liability, and D-036/D-061 already set the whole system "default closed / surveillance-adjacent." One can argue the correct separation of concerns is exactly this: engineering says "here is the gate," the principal clears it. Deferring a legal question to the human who can actually obtain counsel is not negligence; it is scope discipline.

**failure_scenario.** Guy, working part-time and momentum-driven through a 5–7 week build (GREENFIELD-PLAN:88), treats Q-23 the way the doc invites — as a "you-decision" to resolve "when convenient" — and never converts it into a hard, *blocking* checklist item. M0–M5 complete. WP-O2 rolls out the first facility (GREENFIELD-PLAN:77) because every *technical* gate (UAT green, soak passed, drills passed) is green and nothing in the runsheet, the provisioning checklist, or the milestone-exit criteria requires a signed legal clearance before an antenna goes live. The system now continuously intercepts and records LTE **uplink** transmissions from personal devices carried by employees (and visitors) inside a workplace — in a US jurisdiction that is the fact pattern the Wiretap Act (18 U.S.C. §2511) and state two-party-consent statutes are built around. A single disgruntled employee, a union, or a state AG surfaces it. The exposure is not a bug ticket; it is potential criminal liability, civil class exposure, and the destruction of the project — and it is discovered *after* six months of recorded interception exists in an indefinitely-retained R2 bucket (RECOVERY-RETENTION:20, handoff "indefinite"), which is itself now discoverable evidence.

**evidence.** OPEN-QUESTIONS:41 frames Q-23 as advisory ("the security reviewer will raise it … a you-decision"). Nowhere is it a **build gate**: GREENFIELD-PLAN:80 milestone exits are purely technical ("first facility live + corpus collection running"); HOST-PROVISIONING-CHECKLIST-v1 has 30+ checkboxes covering serials, bearings, noise floors, NTP — and **zero** covering legal clearance; BUILD-RUNSHEET question-gates (S15 "ASK Q-03", etc.) gate on *technical* answers, never Q-23. D-036 (DECISION-LOG:41) is the entire threat-model statement — "Surveillance-adjacent system; default closed" — which governs *repo visibility*, not lawful-basis. The system captures LTE **uplink** specifically (LOCAL-HOST-DATA-FLOW:45, "a = low band 662–850; b/d = high 1709–1911", "direction: 'ul'" L1) — i.e. the phone-to-tower direction, the transmissions of people's personal devices, which is materially different (and legally more fraught) than passively logging tower broadcasts. The taxonomy's stated purpose (DEVICE-BEHAVIOR-TAXONOMY:11, G3 = "cell phone") confirms the system detects and classifies employees' phones as a routine output, not an incidental one.

**proposed_alternative.** Elevate Q-23 from "deployment-gating question" to a **hard, documented gate with the same enforcement discipline the data pipeline gets** ("a rule with no check is not a rule," R0.3):
1. Add a **first-line, blocking** checkbox to HOST-PROVISIONING-CHECKLIST-v1: `[ ] Written legal clearance on file for THIS facility/jurisdiction (counsel memo ref: ____; date: ____). A host is not pending-eligible until checked.` Make it structurally analogous to the NTP "hard requirement" already there.
2. Add a milestone-exit gate to GREENFIELD-PLAN M5: "WP-O2 does not begin at any facility without recorded legal clearance for that facility."
3. Split Q-23 into per-jurisdiction clearance (facilities may span states with different two-party-consent rules — Q-14 already asks which facility codes; the legal answer is *not* uniform across `ral`/`cht`/`dal`).
4. Add a short **DATA-HANDLING / LAWFUL-BASIS note** to the docs stating the intended lawful basis (employer-owned-premises monitoring? posted notice? consent-in-employment-agreement?) so the technical retention/access decisions can be justified against it. This is not legal advice; it is making the *assumption* explicit so it can be checked — exactly the discipline R0 applies to schemas.
5. Consider whether the corpus log (RUNBOOK:25, physical-confirmation of specific devices → specific people/assets) needs a heightened handling classification, since it deliberately links an RF signature to a real-world identity.

**downstream_impact.** Touches HOST-PROVISIONING-CHECKLIST-v1, GREENFIELD-PLAN M5/WP-O2, RECOVERY-RETENTION (retention windows become a compliance parameter, not just an ops one), the corpus-log discipline (RUNBOOK), and D-036 (whose threat-model framing should be widened from "repo visibility" to "lawful-basis + repo visibility"). It also interacts with FINDING-SEC.9 (audit trail) — legal defensibility usually *requires* an access/export audit.

**confidence.** high (that it is under-gated as written). The legal *conclusion* is genuinely not mine to reach — but the *framing* is demonstrably too soft for a design this otherwise-rigorous, and "elevate it to a blocking gate" is a design fix I can stand behind fully.

---

### FINDING-SEC.2 · severity: blocker · target: D-050 (sops+age) — operator-key blast radius + no key-management contract

**steelman.** sops+age is a genuinely good, low-ceremony choice for a single operator (D-050, DECISION-LOG:63). It keeps plaintext `.env` off machines except at provision/update time; it keeps secrets out of component repos (only `ear-fleet` holds ciphertext); it is git-friendly (encrypted blobs diff and version, giving the audit trail L0.5 wants); and "one operator key + one per host, rotation = re-encrypt + re-render" is the textbook minimal keyset. For one maintainer this beats a secrets-manager service he'd have to run and secure himself. The blast-radius question the reviewer is asked to push is real, but the *tool* is defensible.

**failure_scenario.** The single **operator age key** is the master key: by construction it can decrypt **every** secret for **every** host and **every** facility in `ear-fleet` (that is what "one operator age key" means — it is a recipient on every encrypted file, else the operator couldn't re-render). It lives on the dev box (D-048, the Strix Halo) and, realistically, in the operator's `~/.config/sops/age/keys.txt` or `~/.age`. That box also hosts the local git hub with all repos, the Postgres volume, and MinIO. It is a developer workstation, not a hardened HSM. Scenario: the dev box is stolen, or malware exfiltrates `~/.config/sops/age/keys.txt`, or the key is accidentally committed / synced to a cloud backup. The attacker now holds the one key that decrypts **R2 credentials for the entire fleet, all facility `.env`s, the `ALERT_URL`, and every host secret** — and, because rotation is "re-encrypt + re-render" performed *from that same key*, there is no break-glass path that doesn't assume the operator still controls the compromised key. Meanwhile D-050 says nothing about: where the age key is stored, whether it is itself passphrase-protected, whether there is an offline escrow copy, or what the revocation procedure is when it leaks. The whole scheme's security reduces to one file on one Linux laptop, undocumented.

**evidence.** D-050 (DECISION-LOG:63) in full: "Keys: one operator age key + one per server host. Rotation = re-encrypt + re-render." That is the entire key-management spec. There is **no** statement of: age-key storage/passphrase, escrow/backup of the operator key, revocation-on-leak procedure, or blast-radius acknowledgment. BACKUP-DR (BACKUP-DR:9) lists `ear-fleet` as SACRED and notes "sops-encrypted secrets already committed" but does **not** back up or escrow the age *private* key anywhere — meaning key **loss** (not leak) is an unrecovered failure mode too: lose the dev box's age key with no escrow and you cannot re-render any host config or decrypt your own R2 creds. `.gitignore` excludes `.env` (D-061) but nothing addresses the age key file. HOST-PROVISIONING-CHECKLIST:18 renders config "from ear-fleet (sops, D-050)" — so every provisioning event requires the operator key to be present and usable, concentrating its exposure.

**proposed_alternative.** Write a short **KEY-MANAGEMENT contract** (call it the missing half of WP-O3) with the same NEVER-list discipline as the R-chain:
1. **Operator key storage:** age key MUST be passphrase-encrypted at rest (age supports this) and MUST NOT live in any synced/cloud-backed directory. State the path convention and a `.gitignore`/backup-exclusion for it.
2. **Escrow:** one **offline** copy of the operator private key (e.g. paper/hardware token in physical safe) — because BACKUP-DR currently makes key *loss* catastrophic and un-drilled.
3. **Blast-radius reduction:** consider whether the operator needs to be a recipient on *every* file, or whether per-facility age keys (recipient = operator + that facility's host keys) would bound a leak to one facility. At minimum, R2 credentials should be scoped per-facility/per-agent (see FINDING-SEC.3) so the master-key leak's *R2* blast radius is bounded even if the key isn't.
4. **Revocation runbook:** explicit "operator key suspected leaked" procedure — generate new operator key, re-key every sops file, **rotate every secret the old key could read** (R2 creds, DB creds, ALERT_URL), redeploy. Today "rotation = re-encrypt + re-render" only re-encrypts; it does not say *rotate the underlying secrets*, which is the step that actually matters after a leak.
5. Add a provisioning-checklist and a BACKUP-DR row for the age key escrow so it's drilled, not assumed.

**downstream_impact.** Interlocks with D-002/Q-12 (FINDING-SEC.3): the master key's *value* to an attacker is exactly the scope of the R2 credentials it decrypts. Touches BACKUP-DR (add escrow row + a key-loss drill), HOST-PROVISIONING-CHECKLIST, RUNBOOK (add the revocation playbook next to the R2-outage playbook), and WP-O3. If per-facility keys are adopted it lightly touches D-050's "one operator key" phrasing.

**confidence.** high.

---

### FINDING-SEC.3 · severity: blocker · target: D-002 / R4 / Q-12 — R2 credential scope & compromised-warehouse-PC blast radius

**steelman.** R2-primary is well-reasoned (D-002, DECISION-LOG:8): multiple facilities + an often-remote server make a cloud rendezvous correct, and per-agent folders give clean prefix-based isolation. The design already refuses to trust R2 *listing* as authority (R5.0 onboarding gate: "never raw R2 discovery"), so a rogue folder can't auto-enter production. Q-12 (OPEN-QUESTIONS:20) does ask for "Fresh R2 bucket name/account for production," so the credential question is at least on the radar. And per-agent *folders* plus atomic write-once objects (R4) mean the storage layout is already segmented. One can argue credential *scoping* is a deployment detail, correctly deferred to when the real bucket exists.

**failure_scenario.** The documents never state whether the fleet shares **one R2 credential** or uses **per-agent scoped credentials**, and the path of least resistance — which a build agent following the docs will take — is one shared access-key/secret rendered onto every warehouse PC from `ear-fleet` (HOST-PROVISIONING-CHECKLIST:20, "`S3_ENDPOINT/BUCKET/credentials` valid" — singular, unscoped). A warehouse monitoring PC is, by definition, an unattended box in a semi-public industrial space (LOCAL-HOST-DATA-FLOW:14, "ONE WAREHOUSE PC ×7–15 hosts"), physically accessible to staff, cleaners, contractors. Someone pulls the SSD or reads the rendered `.env`. They now hold R2 credentials. If those credentials are the **shared fleet write cred with no prefix restriction**, the attacker can: (a) **read every facility's entire surveillance history** in the bucket — obs/ctx/health for `ral`, `cht`, `dal`, all of it, since S3 creds without a bucket-policy prefix condition list and GET everything; (b) **overwrite or delete** other facilities' objects (R4 says objects are "never mutated in place" as a *producer discipline* — it is not an *enforced* bucket ACL, so a holder of write creds can absolutely PUT over a key or DELETE); (c) **inject poison/forged objects** under any `agent_id/` prefix — the R5.0 gate stops a *new* prefix from auto-entering production, but an attacker can forge objects under an **already-`active`** agent's prefix, and the server will ingest them as authentic (folder-authoritative identity, R5.1: "the R2 prefix wins"). One stolen warehouse PC thus compromises confidentiality *and* integrity of the entire multi-facility dataset.

**evidence.** D-002 (DECISION-LOG:8) and R4 (DATA-WORKFLOW-RULES:104–121) describe the transport but say **nothing** about credential scope, bucket policy, per-agent IAM/R2 tokens, or object-lock. HOST-PROVISIONING-CHECKLIST:20 treats credentials as a single opaque valid/invalid item. R5.1 (DATA-WORKFLOW-RULES:159) makes **folder-authoritative identity** the rule — "a row whose internal agent_id disagrees is flagged … the R2 prefix wins" — which means forged objects under a valid active prefix are trusted; the only integrity check is size/etag match on download (R4 ENFORCED BY), which a forger trivially satisfies since they wrote the object. Nothing in the docs enables R2 **object versioning as tamper-evidence** for the signal feed (BACKUP-DR:6 mentions "R2 lifecycle/versioning" for the bucket generally, but not as an anti-tamper control, and versioning doesn't help if the attacker's creds can also delete versions). Q-12 asks the bucket *name/account* but not the *credential model*.

**proposed_alternative.**
1. **Per-agent (or at minimum per-facility) scoped write credentials.** Cloudflare R2 supports scoped API tokens; issue each host a token permitting `PutObject`/`GetObject`/`ListObjects` **only under its own `{agent_id}/` (or `{facility}-*`) prefix** and the `_health/` key for its agents. A stolen host cred then reads/writes only its own facility, and cannot touch other facilities at all.
2. **Deny delete/overwrite on the write path.** Agents only ever create new immutable keys (R3 write-once) — the host token should not carry `DeleteObject`, and the bucket should enable **object-lock / immutability (WORM)** on the `obs_`/`ctx_` prefixes so even a valid cred cannot rewrite history. This makes R4's "never mutated in place" an *enforced* control, not a producer promise, and gives tamper-evidence.
3. **Server pull credential is read-only**, separate from any write cred.
4. Add credential-**scope** to Q-12 and to the provisioning checklist as an explicit item ("host token scoped to own prefix: [ ]").
5. Because forged-under-active-prefix is the residual integrity risk, consider a lightweight **producer-side signature** (agent signs a batch manifest hash with a per-host key) so the server can distinguish authentic agent output from an object forged with stolen storage creds — this is the integrity analog of the confidentiality scoping.

**downstream_impact.** Touches D-002, R4, R5.0/R5.1 (the folder-authoritative-identity rule gains an integrity caveat), Q-12, HOST-PROVISIONING-CHECKLIST, BACKUP-DR (object-lock changes the retention/versioning story), and FINDING-SEC.2 (scoped creds bound the master-key leak's R2 blast radius). If per-facility server appliances are chosen (D-004/Q-04), read-cred scoping falls out naturally.

**confidence.** high.

---

### FINDING-SEC.4 · severity: blocker · target: D-056 / ADMIN nav — dashboard & admin access control is assumed, never specified

**steelman.** Pegasus (the SaaS starter D-032 builds on) ships real, batteries-included authentication — django-allauth, login, teams, admin — so leaning on "Pegasus auth" (implied throughout DASHBOARD-DESIGN) is not naive; it's reusing a mature auth stack rather than rolling one. The dashboard is explicitly single-persona (DASHBOARD-DESIGN:4, "one primary persona — the operator (Guy)"), so for the initial build there is exactly one human who should have access, and Django's default "login required + is_staff for admin" already covers a one-user system. Deferring multi-user/read-only-facility-staff access to Q-16 is reasonable scope control. For a single operator on a private deployment, "the framework handles it" is a legitimate v1 posture.

**failure_scenario.** Because the access-control model is *assumed* rather than *specified*, no one ever writes down the invariants, so nothing enforces them. Two concrete paths: **(a)** WP-O1/WP-L4 build the dashboard against DASHBOARD-DESIGN, which is entirely about *information architecture* (screens, tables, journeys) and says nothing about which views require auth. A build agent wires up the Facility summary / Candidates / Device-detail pages and — following the doc — makes them reachable; whether every analysis view sits behind `@login_required` and whether the ntfy/health endpoints (`/health/`, HEALTH-CHECKS:7) are public is left to chance. The `/health/` endpoint "DB ping inside" is a classic unauthenticated info-leak/DoS surface. **(b)** The moment Q-16 is answered "yes, facility staff get read-only reports," there is **no designed authorization boundary** to hang that on — no notion that facility-staff user X may see facility `ral` but not `cht`. Multi-tenant data (all facilities commingled in one Postgres/one dashboard, the "central server" arm of D-004) meets a role model that was never designed, and the natural implementation — "give them a read-only login" — exposes every facility's tracker candidates to staff of one facility. Surveillance *conclusions about employees' devices* (candidate lists, confirmed-device corpus with physical-identity links) are the single most sensitive output, and they render in a dashboard whose authZ is undesigned.

**evidence.** DASHBOARD-DESIGN-v1 mentions auth **nowhere**; "ADMIN → Django admin (Sensor registry, InfrastructureSignature, users)" (DASHBOARD-DESIGN:15) is the only reference to `users` and it's a nav label, not an access model. The secondary persona (facility staff, read-only) is deferred to Q-16 (DASHBOARD-DESIGN:4) with no authorization design attached. No document contains the words role, permission, RBAC, `@login_required`, `is_staff`, or per-facility authorization for humans (confirmed by grep across all live docs). WP-O1 (GREENFIELD-PLAN:76) and WP-L4 (GREENFIELD-PLAN:72) cite only DASHBOARD-DESIGN for acceptance (UAT journeys J1–J7), none of which test an access boundary — J1–J7 are all single-operator flows. The onboarding gate (R5.0) is the *only* access-control-shaped contract and it governs **data ingest**, not **human access** — a category the docs conflate by omission.

**proposed_alternative.** Add a short **ACCESS-CONTROL contract** (the human-facing complement to R5.0), even for the single-operator v1:
1. State the invariant explicitly: **every** dashboard/analysis/admin view requires authentication; `/health/` and any status endpoint are either authenticated or return no data-leaking detail unauthenticated.
2. Define the **role skeleton now**, before Q-16 forces it later at higher cost: `operator` (full), and a stub `facility_viewer` role **scoped to a facility set**, even if unused in v1 — so the per-facility authorization boundary exists in the data model (a `facility` scope on the user) rather than being retrofitted onto commingled data.
3. Make it a **fixture/UAT persona**: an access-control journey ("user scoped to `ral` cannot load a `cht` device-detail page → 403") with the same "green before merge" discipline as every other contract. This converts the assumption into an enforced rule (R0.3).
4. Tie the role decision to D-004: if central multi-facility (one commingled DB), the per-facility authorization boundary is **mandatory** the moment any second human exists; if per-facility appliance, it's naturally bounded — so resolve Q-04 partly on this security ground.

**downstream_impact.** Touches DASHBOARD-DESIGN (add an access section + an access-control UAT journey), GREENFIELD-PLAN WP-O1/WP-L4 (acceptance now includes an authZ persona), Q-16 (its answer lands on a designed boundary instead of a greenfield gap), D-004/Q-04 (isolation posture), and FINDING-SEC.9 (audit) since access control and audit are the same contract's two halves.

**confidence.** high.

---

### FINDING-SEC.5 · severity: major · target: physical-compromise threat model of the unattended warehouse PC (D-045 / L5 / BACKUP-DR)

**steelman.** The design already reasons carefully about the warehouse PC as a *reliability* actor: `pending/` is the store-and-forward buffer (D-045), the disk ceiling bounds unbounded growth (L5 OPEN → WP-A3), heartbeats make backlog visible, and BACKUP-DR:13 honestly labels agent `pending/`/DLQ as "Transient (un-uploaded field data!)" whose only true loss window is bounded by the ~5-min upload cadence plus outage backlog. So the box's *availability* and *data-loss* properties are modeled. And physically securing a PC in someone else's warehouse is partly outside software's control — you can't software your way out of a stolen SSD.

**failure_scenario.** The warehouse PC is treated purely as a reliability actor and never as a **confidentiality/integrity** actor, yet it is the softest target in the system and it holds two crown jewels simultaneously: (1) **R2 credentials** (FINDING-SEC.3) and (2) **un-uploaded raw surveillance data** in `pending/`, on an **unencrypted disk** (no doc specifies FDE). A cleaner, a contractor, a departing employee, or a thief with five minutes and a screwdriver removes the SSD. On it: plaintext `.env` with R2 creds (rendered by sops at provision time, D-050 — plaintext once rendered), plus up to a multi-day backlog of raw uplink observations (during any WAN outage `pending/` deliberately accumulates, L5). No FDE means the data and creds are readable directly off the pulled disk with zero credentials. There is **no de-provisioning / credential-revocation procedure** for a compromised or decommissioned host anywhere in RUNBOOK ("Retire/replace a sensor" RUNBOOK:10 covers *identity* continuity and HackRF serials — it never mentions rotating the R2 credential or wiping the disk). So even after the theft is noticed, the stolen creds keep working until the operator manually realizes and manually rotates the shared cred fleet-wide (which D-050 doesn't have a runbook for either — see FINDING-SEC.2).

**evidence.** HOST-PROVISIONING-CHECKLIST-v1 has hardware/OS/agent sections but **no disk-encryption checkbox and no physical-security note** (contrast: it meticulously records antenna bearings and HackRF serials). BACKUP-DR:13 acknowledges the un-uploaded data but only as a *loss* risk ("protected by disk ceiling + heartbeat visibility"), never as a *disclosure* risk. RUNBOOK "Retire/replace a sensor" (RUNBOOK:10–11) has no credential-revocation or disk-wipe step. D-045 / EAR-AGENT-SPEC:8 confirm the box holds `pending/`, DLQ, and the rendered config on local disk. No document specifies LUKS/FDE for warehouse hosts. `.gitignore` protects the *repo* from secrets (D-061) but says nothing about the *host disk*.

**proposed_alternative.**
1. Add to HOST-PROVISIONING-CHECKLIST: `[ ] Full-disk encryption enabled (LUKS)` and `[ ] Physical placement noted; enclosure locked where feasible`. FDE is the single cheapest control that neutralizes an SSD-pull for both data and creds.
2. Add a **host-compromise / decommission playbook** to RUNBOOK: on suspected theft or retirement → **revoke/rotate that host's R2 credential immediately** (trivial if per-agent scoped, FINDING-SEC.3; painful and fleet-wide if shared — another reason to scope), set the sensor `disabled`, and treat any objects newly written under its prefix after the compromise timestamp as suspect.
3. Bound the confidentiality window explicitly: the disk ceiling already bounds `pending/` size; state that as a deliberate *blast-radius* bound on stolen-disk data, not just a disk-space bound.
4. Prefer per-agent scoped creds (FINDING-SEC.3) precisely so that a warehouse-PC compromise is a **single-facility** incident, not a fleet incident.

**downstream_impact.** HOST-PROVISIONING-CHECKLIST, RUNBOOK (new playbook), BACKUP-DR (reframe agent-data row to include disclosure risk), D-045/L5 (disk ceiling gains a security rationale), and tight coupling to FINDING-SEC.2 and FINDING-SEC.3 (revocation depends on credential scoping and key management).

**confidence.** high.

---

### FINDING-SEC.6 · severity: major · target: data-at-rest encryption — Postgres, DuckDB (l1/, L2), handoff Parquet, R2, backups

**steelman.** For a single-operator system on infrastructure the operator controls, "encrypt the transport, private repos, private bucket" is a defensible baseline, and R2/Cloudflare provides server-side encryption of objects at rest by default without the operator doing anything — so the *cloud* leg is arguably covered by the provider. DuckDB L1 files are explicitly *disposable* cache (D-023, rebuildable), so their at-rest sensitivity is lower than the source data. Not every store needs bespoke encryption when the provider and the private-by-default posture already raise the bar. Deferring to the platform is a reasonable v1 stance.

**failure_scenario.** Nothing in the design *states* which stores are encrypted at rest, so it's left to whatever each build agent / deploy target defaults to — and several legs are self-hosted with no default encryption. The **dev box** (D-047, D-048) concentrates Postgres (`/srv/ear/postgres/`), L1 DuckDB files, handoff Parquet (`/srv/ear/handoff/`), the git hub with all repos, **and** the sops age key — all on one NVMe (DEV-ENVIRONMENT:34–44). No FDE is specified for it either. If the deploy target for prod is "a facility/central box you own" (Q-11 offers that as an option), the Postgres volume and handoff Parquet — which together *are* the surveillance dataset and its conclusions — sit on a self-hosted disk with no stated encryption. The handoff Parquet is retained **indefinitely** (RECOVERY-RETENTION:20) and pg_dumps are pushed to R2 `_backups/` (BACKUP-DR:26). A stolen dev box or prod box = the entire analytical dataset + all results + the master key, in the clear. The Parquet "event dataset" is more sensitive than raw obs in one sense: it is the *distilled* per-device behavioral record the whole system exists to produce.

**evidence.** No document contains an at-rest-encryption statement for any store (grep for "encrypt"/"at rest" across live docs returns only sops/age for *secrets* and R2 lifecycle, never volume/DB/Parquet encryption). DEV-ENVIRONMENT:34–44 lays out unencrypted `/srv/ear/postgres`, `/srv/ear/l1`, `/srv/ear/handoff` on one NVMe with no FDE mention. Q-11 (OPEN-QUESTIONS) leaves the prod deploy target and "Postgres: managed or in-compose" open — the security posture of each option differs sharply and isn't discussed. BACKUP-DR:7,26 pushes pg_dumps to R2 without stating they're encrypted client-side before upload (they'd rely on R2's server-side encryption, i.e. Cloudflare holds the keys).

**proposed_alternative.**
1. Add a one-line **at-rest posture** to DEV-ENVIRONMENT and to the (missing) deploy doc: **FDE (LUKS) on the dev box and any self-hosted prod box** — this covers Postgres, DuckDB, and Parquet volumes in one stroke and is near-zero effort at provision time.
2. State explicitly that **R2 server-side encryption is relied upon** for objects (so the assumption is recorded and checkable), and decide whether pg_dumps pushed to R2 should be **client-side encrypted** (e.g. `age`-encrypt the dump before PUT) so the operator, not Cloudflare, holds the key to the most concentrated artifact.
3. Note the handoff Parquet's indefinite retention makes it the highest-value at-rest target and justify FDE on that ground.
4. Fold this into the same WP-O3 security-basics deliverable as FINDING-SEC.4/9.

**downstream_impact.** DEV-ENVIRONMENT, the not-yet-written prod-deploy doc, BACKUP-DR (client-side encryption of dumps), Q-11 (deploy target now carries a security rubric), RECOVERY-RETENTION (indefinite Parquet → FDE justification). Low build cost, high risk reduction.

**confidence.** med (med only because provider-default encryption partially covers the R2 leg; the self-hosted legs are high-confidence gaps).

---

### FINDING-SEC.7 · severity: major · target: no audit trail for viewing/exporting surveillance data (R8/R9, dashboard)

**steelman.** For a single operator, "who looked at the data" is trivially answerable — it's always him — so an access/export audit log looks like ceremony with an audience of one. R8/R9 already give strong **data-lineage** audit: every candidate traces to its run + config (R8 "no orphan results"), every report figure traces to source rows (R9). That is a real audit trail *of the computation*. Django admin also logs model changes via `LogEntry` for free. So "there's no audit" overstates it.

**failure_scenario.** The audit that exists is *lineage* (how a conclusion was computed), not *access* (who read or exported a conclusion). These are different controls and the second is the one that matters for a surveillance system with legal exposure (FINDING-SEC.1). The moment Q-16 adds any second human (facility staff read-only), or an incident/legal-discovery event asks "prove who accessed the `cht` facility's tracker candidates and whether anyone exported the corpus linking device signatures to named individuals," there is **no record**. Reports are written to a `/reports` volume (R9, DATA-WORKFLOW-RULES:233) and rendered in the dashboard; nothing logs a report download or a candidate-list export. The corpus log (RUNBOOK:25) deliberately links RF signatures to physically-confirmed real-world devices/assets — the most identity-laden artifact in the system — and there is no access record on it. In a legal challenge, *inability to show* access controls and access logs is itself damaging (it undercuts any "we handled this responsibly" defense).

**evidence.** R8/R9 (DATA-WORKFLOW-RULES:212–246) specify lineage audit only; no access/export logging. DASHBOARD-DESIGN mentions no audit view and no export logging (grep: no "audit trail"/"access log" in any live doc except the R-chain lineage sense). BACKUP-DR and RECOVERY-RETENTION cover data retention but not *access* retention. The `/reports` volume (R9) and one-click corpus entry (DASHBOARD-DESIGN:24, J3) have no logged actor/timestamp requirement.

**proposed_alternative.**
1. Add to the ACCESS-CONTROL/audit contract (pairs with FINDING-SEC.4): **log actor + timestamp + facility scope on every access to and export of surveillance conclusions** — candidate-list views/exports, report downloads, corpus reads/writes, device-detail views. Even single-operator, this is cheap Django middleware and it's the record legal defensibility needs.
2. Make **corpus writes** (physical-confirmation entries, the identity-linking action) mandatorily attributed (actor, timestamp) — they already capture date/facility/outcome (RUNBOOK:26); add actor.
3. Retain the access log with a defined retention (tie to Q-15's retention discipline).
4. One UAT/fixture assertion: an export produces an audit row.

**downstream_impact.** DASHBOARD-DESIGN, R8/R9 (add an access-audit clause distinct from lineage), the corpus-log discipline (RUNBOOK), Q-15 (retention of the audit log), FINDING-SEC.1 (legal defensibility) and FINDING-SEC.4 (same contract). 

**confidence.** high.

---

### FINDING-SEC.8 · severity: major · target: D-004 / Q-04 — tenant/facility isolation is undecided and has a security dimension the docs don't weigh

**steelman.** Genuinely leaving D-004 open (DECISION-LOG:11, "App built to run either way") is disciplined: the app is written topology-agnostic, the decision is deferred to when deploy realities are known (Q-04), and correlation is already scoped `(facility, band_group)` (D-001) so cross-facility data doesn't accidentally mingle in *analysis*. LOCAL-HOST-DATA-FLOW:261 lays out the tradeoff crisply (central = cross-facility visibility + single point of failure + commingled data; per-facility = isolation + data residency + N deployments). The neutrality is a feature.

**failure_scenario.** The tradeoff is framed almost entirely in **reliability/visibility** terms; its **security and legal** dimension — which may be decisive — is under-weighted. If Q-23 (FINDING-SEC.1) comes back with *different legal answers per jurisdiction* (facility A cleared, facility B not, or facility B requires data residency in-state), the **central-server** arm becomes a liability: one commingled Postgres holds all facilities' surveillance data, so a subpoena or breach at the central box exposes *every* facility including ones with stricter local rules, and there's no per-facility authorization boundary (FINDING-SEC.4) to contain access. Conversely if the operator picks central for convenience *before* the per-facility authorization model exists, retrofitting isolation onto commingled data after facility staff get logins (Q-16) is a painful migration. The docs let this structural, hard-to-reverse call (Q-04 "before M3 deploy shape") be made on convenience grounds without the security/legal input.

**evidence.** LOCAL-HOST-DATA-FLOW:261 and D-004 (DECISION-LOG:11) frame the choice as isolation/residency/SPOF/visibility — the words "breach blast radius," "per-facility authorization," and "jurisdictional data residency as a legal requirement" don't appear in the weighing. Q-04 (OPEN-QUESTIONS:8) asks the topology but ties it to "M3 deploy shape and where the L2 DB will eventually live," not to legal/isolation posture. Q-23's per-jurisdiction nature (FINDING-SEC.1) is not connected to Q-04 anywhere.

**proposed_alternative.**
1. Add the security/legal criterion to the D-004 decision inputs: **"If facilities span jurisdictions with differing legal clearance or residency rules, prefer the per-facility appliance (isolation by construction); central multi-tenant requires the per-facility human-authorization boundary (FINDING-SEC.4) as a precondition, not a follow-up."**
2. Explicitly link Q-04 and Q-23 and Q-14 (which facilities/jurisdictions) so the topology call is made with the legal map in hand.
3. If central is chosen, make the per-facility authorization boundary (FINDING-SEC.4) a **hard prerequisite** before the second human or second facility.

**downstream_impact.** D-004/Q-04, Q-23, Q-14, FINDING-SEC.4 (authorization), FINDING-SEC.6 (at-rest — central concentrates the at-rest target), D-042 (L2 DB location).

**confidence.** med.

---

### FINDING-SEC.9 · severity: minor · target: D-049 — ntfy `ALERT_URL` is an unauthenticated metadata side-channel

**steelman.** D-049 (DECISION-LOG:62) is a deliberately minimal, single-operator-appropriate choice: one `notify()` utility, one env var, one channel, "no alerting stack to run." ntfy topics are the standard zero-infra push mechanism, and the alert *content* is operational (disk ceiling, pipeline exit, dead heartbeat), not surveillance results. Simplicity has real value for a solo operator who must actually run this.

**failure_scenario.** An ntfy topic is a **public pub/sub channel secured only by the secrecy of the topic name** (the default free ntfy.sh model). The `ALERT_URL` is rendered onto every warehouse PC (HOST-PROVISIONING-CHECKLIST:20) — the same softest-target boxes as FINDING-SEC.5 — so a stolen host disk leaks the topic. Anyone who learns the topic can then **subscribe** and passively watch the fleet's operational telemetry: which facilities exist (alert text names agents/facilities — RUNBOOK alert examples reference agent ids), when sensors go dark, when hosts are down (a physical-security tell: "this warehouse's monitoring just died"), and can also **publish** forged alerts to bury real ones or trigger operator fatigue. For a covert surveillance system, leaking *"a monitoring sensor exists at facility X and just went offline"* is a meaningful intelligence and physical-security disclosure, not merely ops noise.

**evidence.** D-049 (DECISION-LOG:62): "POSTing to `ALERT_URL` (an ntfy topic — phone push)"; no mention of ntfy access tokens/auth or self-hosting. `ALERT_URL` rendered per host (HOST-PROVISIONING-CHECKLIST:20). RUNBOOK alert examples (RUNBOOK:14–17) show alerts naming agents/facilities/states. No document treats the alert channel as a confidentiality surface.

**proposed_alternative.**
1. Use ntfy **access tokens / auth** (self-hosted ntfy or ntfy.sh reserved topic with auth) so subscribe/publish require a credential, not just topic-name knowledge.
2. Keep alert *text* facility-anonymized where feasible (agent hash or opaque id in the push, full detail only in the authenticated dashboard).
3. Treat `ALERT_URL`/token as a rotate-on-host-compromise secret (folds into FINDING-SEC.5's revocation playbook).

**downstream_impact.** D-049, HOST-PROVISIONING-CHECKLIST, RUNBOOK, FINDING-SEC.5 (revocation). Low effort.

**confidence.** med.

---

### FINDING-SEC.10 · severity: minor · target: D-051 — fleet update channel integrity (supply-chain to warehouse hosts)

**steelman.** D-051 (DECISION-LOG:64) is sensible and cautious: git-tag releases, staged facility-by-facility (never fleet-wide at once), verify heartbeat after each. Pulling from the operator's own git hub/mirror (D-048) over presumably-authenticated git remotes is a small, controlled supply chain — not pip-from-PyPI-arbitrary. `ear-agent-update <tag>` is a bounded, reviewable action.

**failure_scenario.** `ear-agent-update <tag>` does "fetch tag from the git hub/mirror → **pip install into venv** → render config → restart" (D-051). Nothing states the tag is **signed/verified** or that pip installs from a **pinned, hash-checked** set (no lockfile/hash-pinning mentioned). Two exposures: (a) if the GitHub mirror account or the git-hub box is compromised, a malicious tag pushed there flows straight onto every warehouse host on the next staged update — the update channel *is* remote code execution on the fleet by design, with no signature gate; (b) `pip install` at update time, if it resolves any dependency from PyPI without hash pinning, inherits the entire PyPI supply-chain surface onto boxes that capture surveillance data. Staged rollout limits *blast radius per wave* but doesn't *detect* a malicious tag — the operator verifies the heartbeat *works*, not that the code is *authentic*.

**evidence.** D-051 (DECISION-LOG:64) describes fetch→pip install→render→restart with no signing/verification/pinning. D-061 (DECISION-LOG:82) commits to a permissive dep stack but says nothing about lockfiles/hash-pinning. No document mentions signed tags, `pip --require-hashes`, or a pinned lockfile for the agent venv. GitHub is "a mirror remote" (D-048) — if it's the fetch source for updates, its account security is in the TCB and unstated.

**proposed_alternative.**
1. **Signed git tags** for releases (`git tag -s`), and `ear-agent-update` verifies the signature before installing — the authenticity check the staged rollout lacks.
2. **Hash-pinned lockfile** for the agent venv (`pip install --require-hashes` from a committed lock) so update-time dependency resolution can't pull a tampered/typosquatted package.
3. Prefer the **local git hub** (D-048) as the authoritative fetch source over the public GitHub mirror for the update path, and document the git-hub box as part of the trusted computing base.
4. One provisioning/update-checklist line: "tag signature verified before install."

**downstream_impact.** D-051, D-048 (git-hub box enters the TCB explicitly), D-061 (add lockfile/hash-pinning to the dep policy), HOST-PROVISIONING/RUNBOOK release procedures.

**confidence.** med.

---

### FINDING-SEC.11 · severity: minor · target: WP-O3 sequencing — security is scheduled last, after the system already handles real data

**steelman.** Putting security "basics" in WP-O3 at M5 (GREENFIELD-PLAN:78) is defensible: you can't finalize deploy-specific security (TLS, host hardening, prod credential scoping) until you know the deploy target (Q-11), and much of it (dashboard auth via Pegasus) comes "for free" earlier. Sequencing security config near go-live is common and not per se wrong.

**failure_scenario.** Several of the findings above are **not** deploy-time config; they are **contract and structural** decisions that get baked in *early* and are expensive to retrofit: credential scoping (FINDING-SEC.3) shapes how the agent uploader and R2 tokens are built in M2; the human-access-control boundary and audit hooks (FINDING-SEC.4/7) shape the Postgres data model and dashboard views built in M3/M4.5; per-facility isolation (FINDING-SEC.8) is a topology call due *before M3* (Q-04). If all of this waits for WP-O3 at M5, the agent, server, and dashboard are already built without the seams, and "add security" becomes "rework the data model, the upload creds, and the views" — precisely the pre-code-vs-post-code cost asymmetry the whole review exists to avoid. Meanwhile the corpus and real data start flowing at M5 (WP-O2, "corpus logging live from day one") *concurrently* with security still being written.

**evidence.** GREENFIELD-PLAN:78 puts "security/access basics" as a WP-O3 remainder at M5. But R5.0's data-plane gate is built in M3 (WP-S2) with no human-plane analog; the dashboard (WP-O1/L4) and Postgres models (WP-S1) are built M3–M4.5 with no access/audit contract to build against (FINDING-SEC.4/7); Q-04 topology is due "before M3 deploy shape" (OPEN-QUESTIONS:8) — earlier than WP-O3. The security-relevant *structural* decisions are due *before* the milestone where security is scheduled.

**proposed_alternative.**
1. **Split WP-O3.** Pull the *structural* security items forward to the milestones that build the affected surface: credential-scoping design → M0/M2 (with the agent uploader); access-control + audit contract → M3 (before the dashboard/models exist); topology/isolation → M0 (with Q-04, already due before M3). Leave only genuinely deploy-time config (TLS certs, host hardening, prod token issuance) in WP-O3 at M5.
2. Add the access-control and key-management contracts (FINDING-SEC.2/4/7) to `ear-docs/contracts/` in WP-D1 alongside the schemas, so build agents build *against* them from the first commit — the same "contract-first" discipline the data pipeline enjoys.

**downstream_impact.** GREENFIELD-PLAN (resequence WP-O3), BUILD-RUNSHEET (security contracts referenced in earlier sessions), WP-D1 (authoring the new contracts), and REVIEW-DISPOSITION (these are D-030-class "change before build" structural calls, per the brief's request to separate those).

**confidence.** high.

---

## 3. WHAT TO PROTECT

Choices strong enough to guard against improvement-churn during the build:

- **R5.0 onboarding gate (D-022).** The `pending→qa→active→disabled` allowlist with "auto-discovery never equals auto-production" and per-agent-folder structural isolation is genuinely excellent security engineering — it is the one place the design already applies access-control discipline with an enforced check, and it directly kills the "rogue folder auto-enters production" class. Do not let a build agent "simplify" it back toward the earnest-v3 `SELECT DISTINCT agent_id` auto-discovery the doc explicitly rejects (DATA-WORKFLOW-RULES:149). My FINDING-SEC.4 asks for a *human-plane* analog to this — not a change to it.
- **All-private, proprietary repos (D-036/D-061)** and secrets-never-in-component-repos (D-050). The default-closed posture and keeping ciphertext-only secrets in a single `ear-fleet` repo (with `.gitignore` for plaintext) are correct and should not be relaxed for convenience.
- **sops+age as the *tool* (D-050).** The choice of mechanism is right for a solo operator; my FINDING-SEC.2 targets the missing key-management *contract around it*, not the tool. Don't churn to a heavier secrets manager — fix the escrow/rotation/blast-radius documentation instead.
- **Physical enforcement of the L1/L2 boundary and folder-authoritative identity's *isolation* intent** (D-024, EVENT-DATASET-HANDOFF: "QA isolation is literal"). The structural (not merely policced) isolation is a security asset — a `qa` agent's partition simply isn't globbed. Protect it. (FINDING-SEC.3 adds an *integrity* caveat to folder-authoritative identity under stolen creds; it does not weaken the isolation property.)
- **Retention discipline as configuration, versioned (RECOVERY-RETENTION).** Enforcing retention via lifecycle policy and job-start pruning (never ad-hoc deletion) is exactly right; it just needs a privacy-minimization *framing* added (FINDING-SEC.1), not a mechanism change.

---

## 4. COVERAGE STATEMENT

**Examined in full and found to warrant findings:** Q-23/legal framing (SEC.1), D-050 key management (SEC.2), D-002/R4/Q-12 credential scope (SEC.3), D-056/dashboard access control (SEC.4), warehouse-PC physical threat model (SEC.5), at-rest encryption (SEC.6), access/export audit (SEC.7), D-004 isolation (SEC.8), D-049 alert channel (SEC.9), D-051 update integrity (SEC.10), WP-O3 sequencing (SEC.11).

**Examined and found sound (why they held):**
- **R5.0 / D-022 onboarding gate** — held: it is a properly-specified, enforced, structurally-isolated data-plane access control; the only gap is that it has no human-plane sibling (raised as SEC.4, an addition not a defect in R5.0 itself).
- **D-036 / D-061 visibility & licensing** — held as *posture*; the finding (SEC.1) is that the *threat-model framing* around them is too narrow (repo-visibility ≠ lawful-basis), not that private/proprietary is wrong.
- **Schema field sensitivity (observation-signal.v5, heartbeat.v1)** — checked for PII/location leakage. `latitude/longitude/cell_id/pci` are optional and explicitly *unpopulated* in this build (LOCAL-HOST-DATA-FLOW:105) — so no GPS/PII is captured at the record level, which is a genuine privacy *reduction* and correctly noted. `heartbeat.v1` carries only operational fields (disk, queue, ntp_offset). No finding on field-level PII; the sensitivity is behavioral (the *pattern* of a device's transmissions is the identifying signal), which the taxonomy and corpus handle — and which SEC.7's audit-on-corpus addresses. Sound as far as record schema goes.
- **Data lineage audit (R8/R9)** — held as *lineage*; SEC.7 distinguishes it from the missing *access* audit rather than faulting it.
- **BACKUP-DR sacred/rebuildable model** — held as a *durability* design; SEC.5/SEC.6 add the *disclosure/at-rest* dimension it doesn't cover, but the availability reasoning is sound and the "un-uploaded pending/ is the only true loss window" honesty is a model of good documentation.
- **Store-and-forward / disk-ceiling (D-045/L5)** — held for reliability; SEC.5 reuses the disk ceiling as a confidentiality blast-radius bound rather than disputing it.

**Not read (isolation rule):** the entire `audit/` folder. Two incidental grep-preview lines from `audit/synthesis/load-bearing-decisions.md` were seen but not used; all conclusions above derive from primary documents only.

**Net posture judgment:** The security lens is under-developed exactly as the authors flagged, but the gaps are overwhelmingly *contract absences* (no access-control contract, no audit contract, no key-management contract, no at-rest statement, no physical-compromise/revocation playbook, no credential-scope decision) rather than *bad decisions*. Because the system is pre-code, every one of these is a document to write, not a rewrite — which is the best possible state to find them in. The two decisions that most deserve to move *before* repo creation (per the brief's request to separate structural calls): **credential scoping (SEC.3)** and the **access-control/audit contract seams (SEC.4/SEC.7)**, plus treating **Q-23 as a hard blocking gate (SEC.1)**.
