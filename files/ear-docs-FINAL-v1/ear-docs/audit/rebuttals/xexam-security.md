# Phase 2 — Adversarial Cross-Examination: SECURITY cluster

**Cluster:** security scope vs single-operator simplicity.
**Findings under examination:** SEC.1 (Q-23 legal gate under-gated), SEC.2 (operator age-key SPOF), SEC.3 (R2 credential scope / forged-under-active-prefix), SEC.11 (WP-O3 sequencing bakes structural seams in early).
**Cross-examiner stance:** default skeptical. A finding survives only if it withstands the single-operator/simplicity steelman plus D-036/D-050/D-061 rationale plus cross-lens priors. Documents read: DECISION-LOG D-002/022/036/045/048/049/050/051/056/061, OPEN-QUESTIONS Q-02/04/11/12/16/23, DATA-WORKFLOW-RULES-v3 R5.0/R5.1, both lead passes, the security reviewer's full text.

The organizing question the brief poses: **which of these are genuine pre-repo *structural* calls (cheap now, a rewrite later) vs deploy-time config (correctly deferred to WP-O3)?** I answer it per finding and then in the roll-up.

---

## TARGET: FINDING-SEC.1 — Q-23 legal gate under-gated; elevate to a blocking provisioning checkbox

### Rebuttal (strongest case for killing / addressing it)

The DECISION-LOG and OPEN-QUESTIONS do more here than the finding credits. Q-23 (OPEN-QUESTIONS:41) is not a hand-wave — it names the exact statute families ("employee privacy, wiretap/ECPA-style statutes on signal interception, jurisdiction-specific rules"), states the priority ordering explicitly ("This gates deployment harder than any technical risk"), and correctly assigns ownership ("the clearance is a you-decision, not a design one"). That is a textbook example of a design document doing the *right* thing with a legal question: surfacing it, ranking it above every technical risk, and refusing to fake a legal opinion it has no standing to give. The steelman the reviewer itself wrote is decisive on the *substance*: engineering says "here is the gate," the principal (who owns the facilities, the liability, and the ability to retain counsel) clears it. You cannot software-engineer your way to a lawful-basis determination, and pretending a checkbox confers one is worse than honest deferral.

So on the **legal conclusion**, this is `rationale_addressed` — Q-23 already owns it, at the correct altitude, with the correct priority language.

**But** the finding is not actually asking engineering to render a legal opinion. Re-read the residual claim: it asks to convert an *advisory* question into an *enforced* gate — "a rule with no check is not a rule" (R0.3), the design's own stated discipline applied to itself. That is an engineering ask, and it is one the DECISION-LOG does *not* answer. Here is where the rebuttal fails to kill:

1. The system's own rigor cuts against the operator. Every *technical* precondition to going live is a hard, checkable gate: UAT journeys J1–J7 must be green (D-056, S19e), soak must pass, HOST-PROVISIONING-CHECKLIST has 30+ mechanical checkboxes down to antenna bearings and NTP. Q-23 — ranked by the doc itself *above all of those* — is the **only** deployment precondition with no checkbox, no milestone-exit criterion, and no runsheet ASK-gate. A momentum-driven part-time build (GREENFIELD-PLAN) that clears every green light will roll a live antenna at WP-O2 without ever being *structurally forced* to stop for the one gate the doc says matters most. "It's a you-decision" is true and also exactly how a you-decision silently never happens.

2. The fact pattern is genuinely the worst case for deferral, not a generic one. The system captures LTE **uplink** (`direction: 'ul'`) — phone-to-tower, i.e. the transmissions of employees' and visitors' *personal* devices — and classifies them as "cell phone" as a routine output. That is squarely the interception fact pattern the Wiretap Act / two-party-consent statutes are built for. And the exposure compounds with retention: handoff data is retained **indefinitely** (RECOVERY-RETENTION), so a legal problem discovered late sits on top of months of discoverable recorded interception.

3. The cost asymmetry is total. Adding one blocking checkbox is near-zero effort now; discovering the gate was skipped after six months of live interception is catastrophic and irreversible. This is the cleanest possible "cheap now, ruinous later."

The lead-consistency pass independently reinforces the *structural* half of this: the index docs undercount questions such that Q-23 (the legal gate) is implicitly out of scope for a reviewer trusting the front-matter (lead-consistency.md:86–89). So the gate is not merely un-checkboxed — the navigation actively de-emphasizes it. That *hardens* the "under-gated as written" claim.

### Verdict: **survives** (with a `rationale_addressed` split on the sub-claim)

The *legal conclusion* is fully owned by Q-23 and must NOT be re-litigated by engineering — quote: *"the clearance is a you-decision, not a design one"* (OPEN-QUESTIONS:41). But the finding's actual ask — **enforce** the gate with the same "no rule without a check" discipline the data pipeline gets — is not addressed anywhere and survives intact. The distinction matters: the reviewer is not asking the design to decide the law; it is asking the design to refuse to *silently proceed* without the operator's own recorded answer.

### Residual claim (narrowed and hardened)

Add a **first-line, blocking** checkbox to HOST-PROVISIONING-CHECKLIST — `[ ] Written legal clearance on file for THIS facility/jurisdiction (counsel ref: __; date: __); host not pending-eligible until checked` — structurally analogous to the existing NTP hard-requirement, plus a milestone-exit gate on GREENFIELD-PLAN M5/WP-O2. Do **not** add legal-basis text pretending to be advice; the checkbox records the operator's own clearance. Split per-jurisdiction (Q-14 facility codes may span differing two-party-consent regimes). This is a documentation/gate change, not a legal opinion — it makes the you-decision impossible to skip rather than merely recommended.

### Downstream

HOST-PROVISIONING-CHECKLIST (new blocking item), GREENFIELD-PLAN M5/WP-O2 (exit gate), RECOVERY-RETENTION (retention becomes a compliance parameter), D-036 threat-model framing widens from repo-visibility to lawful-basis. Interlocks with SEC.8 (per-jurisdiction legal answers feed the D-004 topology call) and the lead-consistency index fix (so Q-23 isn't implicitly out of scope). **This is a pre-repo structural/gate call, not deploy-time config.**

---

## TARGET: FINDING-SEC.2 — operator age-key single point of failure; no escrow/revocation contract

### Rebuttal (strongest case for killing / softening)

D-050's sops+age is exactly right for a solo operator and the finding concedes this. The simplicity steelman is strong: no secrets-manager service to run and himself secure, ciphertext-only in one `ear-fleet` repo, git-versioned encrypted blobs, plaintext `.env` only rendered at provision/update time. For one maintainer this beats every heavier alternative. The reviewer explicitly protects the *tool* choice (WHAT-TO-PROTECT §, security.md:286) — so this is not a churn-to-Vault argument.

The genuinely killing sub-argument the rebuttal can mount: **"single point of failure" is partly inherent to single-operator and cannot be engineered away.** A solo operator IS the bus-factor-1. The dev box already concentrates the git hub (all repos), Postgres, MinIO, and the operator. Adding "and the age key" to that pile does not create a new SPOF class — the operator's workstation was already the crown-jewel box. If the threat model is "dev box stolen / operator laptop compromised," the age key is the least of it; the attacker already has every repo and the running database. So the *leak* half of SEC.2, viewed against the actual TCB, adds little marginal blast radius that D-048's own concentration didn't already create.

That argument softens the *leak* framing. It does **not** touch the two sub-claims that survive:

1. **Key LOSS is an unrecovered failure mode, and it is not the same as leak.** BACKUP-DR lists `ear-fleet` as SACRED and notes the sops ciphertext is committed — but nothing escrows the age *private* key. Lose the dev box with no escrow copy and you cannot re-render *any* host config or decrypt your *own* R2 credentials. This is not a confidentiality attack; it is a durability gap in a system that is otherwise obsessive about durability (recovery-by-rebuild, sacred/rebuildable classification). BACKUP-DR drills Postgres restore but not "operator key lost." The lead-seams pass reached this independently: *"lose the laptop, lose the ability to re-render any config"* (lead-seams.md:122, D.4) — cross-lens convergence (ops + security) hardens it. This survives cleanly; the simplicity steelman has no answer to it because escrowing one key is *itself* zero-ceremony.

2. **"Rotation = re-encrypt + re-render" does not rotate the underlying secrets.** D-050's rotation clause only re-keys the sops files. After a *leak*, the step that actually matters — rotating the R2 credentials, DB creds, and ALERT_URL that the old key could read — is unstated. So even granting the "dev box already concentrated" softening, the post-incident runbook is absent: there is no "operator key suspected leaked → generate new key, re-key files, AND rotate every underlying secret" procedure. This is a missing playbook, not a missing tool, and survives.

The proposed per-facility age keys (recipient = operator + that facility's host) is over-engineering for v1 and should be dropped from the residual — it fights the single-operator simplicity the design correctly chose, and the R2-blast-radius half it's meant to bound is better handled by SEC.3's scoped R2 tokens (which bound the *value* of a leaked key without multiplying keys). So that sub-recommendation is **softened away.**

### Verdict: **softened**

The *leak-blast-radius* alarm drops in severity: against D-048's already-concentrated dev-box TCB, the age key adds little marginal exposure, so "master key = whole-fleet compromise" overstates a *new* risk. Reframe from blocker to major. The **key-loss escrow gap** and the **rotate-the-underlying-secrets-on-leak playbook gap** survive fully and are the real residual. Drop the per-facility-keys sub-proposal (churn against justified simplicity).

### Residual claim

Write a short KEY-MANAGEMENT note (the missing half of WP-O3) covering exactly two things, both zero-ceremony: (1) **one offline escrow copy** of the operator private age key (paper/hardware token in a physical safe) + a BACKUP-DR row and a key-loss drill — closes the durability gap; (2) a **"key suspected leaked" runbook** that re-keys sops files *and rotates the underlying R2/DB/ALERT_URL secrets* — because today's "re-encrypt + re-render" does not. Also state the key MUST be passphrase-protected at rest and excluded from any synced/cloud-backed directory. Do NOT adopt per-facility keys.

### Downstream

BACKUP-DR (escrow row + key-loss drill), RUNBOOK (leak-revocation playbook next to the R2-outage playbook), HOST-PROVISIONING-CHECKLIST (age-key handling). Interlocks with SEC.3 (scoped R2 tokens bound the leaked-key R2 value, replacing per-facility age keys as the blast-radius control) and lead-seams D.4 (bus-factor). **The escrow + rotation playbook is a documentation call writable now; the mechanism (sops+age) is untouched — this is contract-writing, not a rewrite, and not deploy-time.**

---

## TARGET: FINDING-SEC.3 — R2 credential scope undefined; one stolen warehouse PC = whole-fleet confidentiality+integrity compromise via forged-under-active-prefix

### Rebuttal (strongest case for killing / deferring)

The deferral steelman is the strongest here and the brief asks me to test it directly. Cloudflare R2 scoped API tokens are a **console/deploy-time artifact** — you mint them against the real production bucket, which doesn't exist yet (Q-12 is literally "fresh R2 bucket name/account for production"). You cannot scope a token to a bucket you haven't created. So on its face, credential scoping looks like the archetypal deploy-time config that WP-O3 correctly holds: no code changes, just a better token issued at provision time. That is the SEC.11-adjacent "this is deploy-time, not structural" defense, and for the *confidentiality* half (a) — "stolen cred reads every facility's history" — it largely holds. Confidentiality scoping is genuinely a token-minting decision; issuing a prefix-scoped token instead of a fleet-wide one is a HOST-PROVISIONING-CHECKLIST line, not a design rewrite. That half **softens to deploy-time config** — real, but WP-O3-appropriate, provided a checklist line captures it.

Now the two harder sub-claims, where the rebuttal must be tested against the actual contract text rather than asserted:

**(b) overwrite/delete of other facilities' objects.** R4's "objects never mutated in place" is a *producer discipline*, not an enforced bucket ACL — I confirmed nothing in the contracts enforces immutability server-side. A holder of an un-scoped write cred can PUT-over or DELETE. The rebuttal: object-lock/WORM is *also* a deploy-time bucket-config artifact, so this too looks deferrable. **But** whether the write path ever needs DeleteObject at all is a *design* question (agents only ever create immutable write-once keys per R3), and that shapes what the agent uploader is even permitted to do — that's an M2 build-time property of the uploader, not a M5 deploy toggle. Partial.

**(c) forged-under-active-prefix — this is the load-bearing claim, and I verified it against R5.0/R5.1 directly.** The rebuttal the brief invites is: "R5.0 folder-authoritative identity + the onboarding gate defeat the forged-object attack." **This rebuttal fails, and verifying it actually hardens the finding.** Here is the mechanism, checked against DATA-WORKFLOW-RULES-v3:

- R5.0's gate stops **unregistered/`pending` prefixes** from auto-entering production: *"Auto-discovery never equals auto-production. An unregistered/`pending` folder is never ingested"* (R5.0 NEVER). It gates *new* prefixes. It does **nothing** against forgery under an **already-`active`** prefix — `sync iterates Sensor.objects.filter(status='active')` and ingests whatever objects sit under those active prefixes.
- R5.1 makes identity **folder-authoritative**: *"agent identity is FOLDER-authoritative — the R2 prefix wins; a row whose internal agent_id disagrees is flagged."* So an attacker who forges an object under an active agent's prefix and writes the *matching* internal agent_id is not even flagged — the forgery agrees with the folder, which is the whole point of forging it there.
- Every integrity check R5.1 enforces is **satisfiable by the forger**: the ledger uniqueness is on filename+size/etag (the forger picks a fresh filename); the post-load assertion is `observations delta == Σ ledger row_counts` (holds for a well-formed forged batch); R4's download check is size/etag match (trivially satisfied by the object the attacker wrote). There is **no producer authentication** anywhere in the chain — nothing binds an object to the agent that legitimately owns the prefix.

So the onboarding gate — which the reviewer itself (correctly) protects as excellent engineering — is **orthogonal** to this attack. It is an *authorization* gate on prefixes, not an *authentication* of object provenance. The rebuttal conflates the two. Confirming this makes the finding *worse* than filed: the design's own strongest security control gives a false sense of coverage against exactly this class, and a build agent (or a later reviewer) is likely to say "R5.0 handles rogue folders" and stop — precisely the trap.

Prefix-scoped write tokens (SEC.3's own primary fix) *do* bound (c): a stolen `ral` host token can only forge under `ral/`, so the blast radius drops from fleet to one facility. That is the real mitigation and it is the same token-scoping artifact as (a). The residual integrity risk *within* a facility (forging under your own facility's active prefix with your own stolen cred) is only closed by producer-side signing — which the finding lists as a "consider," appropriately, since per-object signing is real added ceremony that a solo operator may reasonably defer. So the signing sub-recommendation **softens to "consider / deferrable"**; the token-scoping sub-recommendation **survives as the core**.

### Verdict: **hardened**

The forged-under-active-prefix mechanism is **confirmed real** against the contract text, and the R5.0-defeats-it rebuttal the brief invited is **refuted**: R5.0 authorizes prefixes, it does not authenticate object provenance, so forgery under an active prefix is ingested as authentic with every integrity check satisfied. Cross-examination surfaced the additional reason it is worse than filed: the design's flagship security control (the onboarding gate) *appears* to cover this and does not, which is an active false-assurance trap for the build. The confidentiality half (a) softens to a deploy-time checklist line, but the integrity half (b)/(c) is a genuine structural gap.

### Residual claim

Two tiers. **Structural / decide now:** state the **credential model** (per-agent or at-minimum per-facility prefix-scoped R2 tokens) as an explicit Q-12 sub-question and a HOST-PROVISIONING-CHECKLIST item, and decide whether the agent uploader is built with DeleteObject at all (R3 says it never needs it) — that shapes the M2 uploader, not the M5 deploy. **Deploy-time / WP-O3-fine:** minting the actual scoped tokens and enabling object-lock/WORM on `obs_`/`ctx_` prefixes against the real bucket. **Defer as a "consider":** producer-side batch-manifest signing (real ceremony; the token scoping already drops fleet→facility, so signing is the second-order within-facility residual). The one thing that MUST be recorded pre-repo is the *decision that the credential model is prefix-scoped* so the uploader and provisioning are built expecting it — retrofitting scoped-token issuance onto a fleet built around one shared cred is the avoidable rework.

### Downstream

D-002, R4, R5.0/R5.1 (add an explicit "prefix authorizes; it does not authenticate provenance" caveat so the false-assurance trap is documented), Q-12 (credential model sub-question), HOST-PROVISIONING-CHECKLIST, BACKUP-DR (object-lock vs versioning). Interlocks with SEC.2 (scoped tokens bound the leaked-age-key R2 value, replacing per-facility age keys) and SEC.5 (host-compromise revocation is trivial iff tokens are scoped). Lead-seams flagged this exact tension and pre-judged the simplicity rebuttal weak (lead-seams.md:134): *"scoped tokens are near-zero added ceremony"* — cross-exam confirms.

---

## TARGET: FINDING-SEC.11 — security scheduled in WP-O3 (M5) but the structural seams get baked in at M2/M3

### Rebuttal (strongest case for killing / softening)

The deferral steelman is legitimately strong and the brief asks whether WP-O3 timing actually costs a rewrite. Much of what WP-O3 holds genuinely *cannot* be finalized earlier because it is deploy-target-dependent (Q-11 open: facility box vs Hetzner vs Render; Postgres managed vs in-compose): TLS certificates, host hardening, prod token *issuance*, prod firewalling. Sequencing that class of work near go-live is standard and correct — you cannot mint a prod-bucket-scoped token before the prod bucket exists (the SEC.3 deploy-time half), and dashboard auth comes largely free from Pegasus earlier regardless. So the blanket claim "security is scheduled too late" is, for the *config* half of WP-O3, **wrong** — that half is scheduled exactly right.

The finding's severity is also only `minor`, and the rebuttal can fairly say: the reviewer is really re-packaging SEC.3/SEC.4/SEC.7's "these specific items are structural" as a sequencing meta-finding. If those underlying items are handled at the right milestone, SEC.11 has no independent content. That argues for **softening to a pure sequencing note with no standalone severity.**

Where the rebuttal fails to kill: the *distinction* SEC.11 draws is correct and is not made anywhere in the plan. The verification I did on SEC.3 proves the point concretely — the *credential-model decision* (prefix-scoped vs shared) shapes the M2 agent uploader and provisioning, not the M5 token mint. Cross-checking the plan: R5.0's data-plane gate is built at M3 (WP-S2) with no human-plane analog; Postgres models (WP-S1) and the dashboard (WP-O1/L4) are built M3–M4.5 with no access/audit contract to build *against*; and Q-04 topology is due *before M3* (OPEN-QUESTIONS:8) — strictly earlier than the M5 milestone where "security basics" live. So the plan does contain a real ordering inversion: at least one security-relevant *structural* decision (topology, Q-04) is due before the milestone that nominally owns security, and several *contract seams* (credential model, access/audit shape) want to exist before the surfaces that consume them are built. That is not config-that-can-wait; that is contract-first discipline the data pipeline already enjoys (schemas authored in WP-D1) being denied to the security surface for no reason other than the WP-O3 label.

This is a real, if modest, structural finding. It survives *as a sequencing correction*, not as an alarm.

### Verdict: **softened**

The blanket "security scheduled too late" is wrong for the config half of WP-O3 (TLS, host hardening, prod token issuance are correctly deferred and genuinely deploy-target-dependent). What survives is the narrower, correct claim: a subset of security items are **contract/structural** (credential-model decision, human access/audit contract shape, topology) and are due at M0–M3, *earlier* than the M5 milestone that nominally owns them — so leaving them under the WP-O3 label risks building the uploader, models, and views without the seams. Reframe from "security is last" to "split WP-O3: pull the structural subset forward to the milestone that builds the affected surface; leave genuine deploy config at M5."

### Residual claim

Split WP-O3. **Pull forward:** (i) the credential-model *decision* (prefix-scoped) to M0/M2 with the agent uploader — verified structural via SEC.3; (ii) the access-control + audit *contract* shape to WP-D1/M3 alongside the schemas, so WP-S1 models and WP-O1/L4 views are built against it (contract-first, same discipline the data pipeline gets); (iii) topology/isolation (Q-04) is *already* due before M3 per OPEN-QUESTIONS — just make its security criterion explicit. **Leave at M5:** TLS certs, host hardening, prod token *issuance* against the real bucket, object-lock enablement — all genuinely deploy-target-dependent. The test for "pull forward" is precise: does it change what a build agent *writes* (uploader permissions, data model, view auth) → structural, pull forward; or only what an operator *configures at provision time* → deploy-time, leave at M5.

### Downstream

GREENFIELD-PLAN (resequence WP-O3 into structural-forward + config-at-M5), BUILD-RUNSHEET (security contracts referenced in earlier sessions), WP-D1 (author the credential-model note + access/audit contract shape alongside schemas). This is the meta-finding that operationalizes SEC.1/SEC.3's "pre-repo structural" verdicts; it has no independent alarm value but the sequencing correction is real. **Pure documentation/planning change, no code — writable now.**

---

## Cross-finding adjudication and roll-up

**No genuine cross-lens dissents in this cluster** — the four findings are mutually reinforcing, and where they touch other lenses (ops on SEC.2 bus-factor, RF/reliability on the D-004 topology that SEC.8 shares) the other lenses *agree*, they do not pull against. Specifically:

- **SEC.2's per-facility-age-keys vs SEC.3's scoped-R2-tokens** was a latent internal tension (two different blast-radius controls). Adjudicated: **SEC.3's token scoping wins.** It bounds the leaked-key R2 value without multiplying age keys, preserving the single-operator simplicity D-050 correctly chose. SEC.2 drops its per-facility-key proposal; the two findings converge on "scope the R2 tokens, keep one age key, escrow it."

**Answering the brief's structural-vs-deploy-time question directly:**

| Item | Structural (pre-repo, cheap now / rewrite later) | Deploy-time (WP-O3-appropriate) |
|---|---|---|
| SEC.1 legal gate | **YES** — blocking checkbox + milestone-exit gate is a doc/gate call, and the *enforcement* is not addressed by Q-23 (only the legal conclusion is) | — |
| SEC.2 key mgmt | **YES** — escrow + leak-rotation playbook are contract-writing now; mechanism untouched | (prod token issuance touches it, minor) |
| SEC.3 cred model | **YES (integrity half)** — the *decision* to prefix-scope shapes the M2 uploader + provisioning; forged-under-active-prefix is real and R5.0 does NOT cover it | **YES (issuance half)** — minting scoped tokens + object-lock against the real bucket is correctly deferred |
| SEC.11 sequencing | **YES** — the plan really does have an ordering inversion (Q-04/credential-model due before the M5 security milestone) | The TLS/hardening/issuance subset it wants to *leave* at M5 is correctly deploy-time |

**Is private-repos + sops/age + you-decide-the-law an adequate posture for a solo operator?** As a *tool and posture* baseline: yes, and the reviewer rightly protects all three (D-036/D-050/D-061). As a *complete* posture: no, but the gaps are contract absences writable pre-repo, not bad decisions — (1) the legal gate is un-enforced, (2) the age key has no escrow/leak-rotation runbook, (3) the credential model is undecided and R5.0 does not authenticate object provenance. All three are cheap now and expensive-or-catastrophic later, which is the exact profile a pre-code audit exists to catch.

**Net:** SEC.1 survives, SEC.3 hardens, SEC.2 and SEC.11 soften. None killed. None fully rationale-addressed (SEC.1's legal *sub-claim* is, but its enforcement ask is not). The strongest single result is SEC.3: the R5.0-defeats-forgery rebuttal the brief asked me to test is refuted against the contract text, and refuting it revealed a false-assurance trap that makes the finding worse than filed.
