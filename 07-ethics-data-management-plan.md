# 07 — Ethics Data Management Plan (DMP)

**Document Status:** Draft — Requires Dr. Linda Interview + Institutional/IRB Review
**Owner:** Damien + Dr. Linda (Documentation Lead facilitates)
**Interview Required:** Dr. Linda (PI) + Damien — Priority 1 (IRB blocker)
**Priority:** 1 (IRB/Ethics approval may block enrollment)

---

## 1. Purpose

This Data Management Plan (DMP) documents how the NEPS Digital System collects, stores, protects, accesses, retains, and disposes of participant data. It is a **required submission for IRB/Ethics Board review** before the study can publish results or expand enrollment.

**This document is legally and ethically binding.** All items marked "CONFIRM WITH DR. LINDA" must be reviewed, corrected if wrong, and signed off by the PI before the DMP is submitted.

---

## 2. Study Overview

| Field | Value | Source / Confirm? |
|-------|-------|-------------------|
| Study Name | NEPS (Nutrition, Education, Psychology Study — confirm official full name with Dr. Linda) | **CONFIRM WITH DR. LINDA** |
| Study Type | Longitudinal cohort study with WP6 intervention arm | **CONFIRM WITH DR. LINDA** |
| Study Duration | 24 months (per ML dataset description — confirm start/end dates) | **CONFIRM WITH DR. LINDA** |
| Study Countries | Ghana, Sierra Leone, Tanzania | Confirmed (data + infrastructure) |
| Target Enrollment | TBD (per ML mock: 150 participants; confirm actual target) | **CONFIRM WITH DR. LINDA** |
| Principal Investigator | Dr. Linda (confirm full name, affiliation, KNUST) | **CONFIRM WITH DR. LINDA** |
| Co-Investigators / Country Leads | Ghana: TBD, Sierra Leone: TBD, Tanzania: TBD | **CONFIRM WITH DR. LINDA** |
| Funding Body / Funder | TBD | **CONFIRM WITH DR. LINDA** |
| IRB / Ethics Board | KNUST IRB? (confirm name, reference number, approval date) | **CONFIRM WITH DR. LINDA** |

---

## 3. Data Types Collected

The NEPS Digital System collects the following categories of participant data:

| Category | Data Items | Identifiability | Stored In |
|----------|-----------|-----------------|-----------|
| **Core Participant Demographics** | Name, date of birth, gender, country, site/school, participant ID, contact (phone/next of kin) | **Identifiable (PHI)** | `neps_core.participant` table |
| **Informed Consent** | Consent date, consent version, witness name, signature (digitized? paper only?), withdrawal flag, withdrawal date | **Identifiable** | `neps_core.consent_record` table + paper forms |
| **Monthly Self-Report Surveys** | PHQ-9, GAD-7, suicidality screening items, employment, food security, socioeconomic status (Yasmine v3 expanded schema) | **Identifiable (linked to participant)** | `neps_core.survey_response` (JSONB) |
| **Distress / Risk Screening** | Distress scale scores, suicidality flag, ML risk prediction score, ML sentiment/emotion labels | **Identifiable (linked)** | `neps_core.distress_screening` |
| **WP6 Intervention Sessions** | Session attendance, session topics, intervention outcomes, fidelity checklists | **Identifiable (linked)** | `neps_core.wp6_session` |
| **Safeguarding Referrals** | Referral date, responder assigned, action taken, case resolution | **Identifiable (linked)** | `neps_core.referral` |
| **Free-Text / Qualitative Data** | Open-ended survey responses, NLP sentiment/emotion training data | **Potentially identifiable (content may reveal identity)** | `survey_response` JSONB + `neps-ml-ai/data/raw/` CSVs |
| **System-Generated Metadata** | Timestamps, user IDs of data viewers/editors, audit logs, export records | Operational (not study data) | Audit log (TBD implementation) |
| **REDCap Source Data** | All of the above, in REDCap instrument format | **Identifiable (source of truth)** | REDCap server (external to NEPS Digital) |

**CONFIRM WITH DR. LINDA:**
- [ ] Are paper consent forms stored separately? Where? Who has access?
- [ ] Is there any biological/genetic data? (Appears no — confirm)
- [ ] Are school/workplace identifiers considered indirect identifiers?
- [ ] Is there audio/video recording of WP6 sessions or interviews?

---

## 4. Identifiability & Anonymization

### 4.1 Identifiable vs. De-Identified Data

| State | When Used | Who Can Access |
|-------|-----------|---------------|
| **Fully Identifiable** | Default in backend DB and REDCap. All data linked to participant by UUID + demographics. | Admin + Damien (system-level). No other role should see raw PHI without need. *(Current state: API is unauthenticated — everyone can see everything. CRITICAL GAP — see Auth in readiness review.)* |
| **Pseudonymized / Minimized** | Dashboards for PI and Country Lead (aggregated, masked participant names if required). | PI + Country Lead (future RBAC) |
| **De-Identified / Aggregated** | Funder reports, publications, public datasets. | Research team for analysis; public post-publication |

### 4.2 Anonymization Approach

**CONFIRM WITH DR. LINDA:**
- [ ] HIPAA Safe Harbor vs. Expert Determination — which standard applies?
- [ ] What fields must be stripped before any data leaves the system (export for analysis)?
- [ ] Is participant ID (UUID) considered an identifier under your IRB's rules?
- [ ] Are there re-identification risk thresholds (k-anonymity, l-diversity) required?

### 4.3 ML Data Considerations

`neps-ml-ai/data/raw/` contains CSVs with participant-linked NLP training data.
**IRB QUESTIONS FOR DR. LINDA:**
- [ ] Do these CSVs need a separate IRB data-use agreement?
- [ ] Are model outputs (sentiment scores, emotion labels) considered "derived PHI"?
- [ ] Can trained model weights (PKL files) be shared post-publication without IRB re-review?

---

## 5. Data Storage

### 5.1 Primary Storage Location

| Storage | What | Physical Location | Encryption |
|---------|------|-------------------|------------|
| **PostgreSQL (Docker volume)** | Study data (participants, surveys, screenings, sessions, referrals) | KNUST on-prem server (confirm: data center / building / rack?) | **At-rest:** TBD — confirm host-level volume encryption or DB-level TDE? (PostgreSQL TDE not in current compose; PASSWORD encryption not set)<br>**In-transit:** PostgreSQL inside Docker overlay network is currently **unencrypted** (medium gap per review — SSL for Postgres TBD) |
| **REDCap Server (external)** | Source-of-truth data entry | KNUST REDCap server? (confirm URL and physical hosting) | REDCap manages its own encryption |
| **Docker Secrets (`/run/secrets/`)** | DB passwords, JWT keys, API tokens, Discord webhooks | KNUST on-prem server | in-memory tmpfs mounts (Docker native) |
| **Paper Consent Forms** | Original signed consents | TBD (locked file cabinet at KNUST? each country site?) | Physical lock |
| **MinIO Object Storage** | Backup target (base backups, WAL archives) | KNUST on-prem server (same host or separate volume?) | TBD — MinIO SSE not configured in current compose |
| **Staging PostgreSQL** | Identical copy of production data for testing | Same server OR separate staging VM | Same caveats as prod — staging must be treated as PHI |

**CONFIRM WITH DR. LINDA:**
- [ ] Is "data physically stored in Ghana (KNUST)" sufficient for all three countries' IRBs?
- [ ] Does Sierra Leone or Tanzania require a local data copy or local IRB registration?
- [ ] Is host-level (VM-level) disk encryption in place at KNUST? Get IT confirmation.
- [ ] Is there an approved off-site / cloud backup? (Current: MinIO on same host — single point of failure.)

### 5.2 Data in Transit

| Link | Encryption |
|------|-----------|
| End user browser → Nginx | **TLS 1.2+ enforced** (Nginx port 443; port 80 301 redirects to HTTPS)<br>Prod: CA-signed cert TBD (currently self-signed dev cert — CRITICAL GAP per review)<br>Staging: staging self-signed cert + `X-Robots-Tag: noindex` |
| Nginx → Backend / Portal (internal network) | HTTP only (internal Docker network — acceptable but confirm with IRB) |
| Backend → PostgreSQL (internal) | **Unencrypted TCP** (medium gap per review) |
| Backend → REDCap API | HTTPS (REDCap server TLS) |
| CI/CD SSH to deploy server | SSH + `DEPLOY_KEY` secret |

---

## 6. Data Retention

| Data Type | Retention Period After Study End | Disposal Method | Confirm? |
|-----------|----------------------------------|-----------------|----------|
| Identifiable participant data (DB) | **TBD years** (standard: 7–10 years after publication for research; confirm IRB requirement) | Secure database wipe + overwrite all backup volumes | **CONFIRM WITH DR. LINDA** |
| Paper consent forms | **TBD years** (typically same as electronic or longer for legal proof) | Shredding (cross-cut) at KNUST records management | **CONFIRM WITH DR. LINDA** |
| De-identified / aggregated analysis datasets | Indefinitely (for reproducibility, funder archives, re-analysis) | N/A — retained | Standard, confirm no objection |
| ML trained models (PKL files) | Indefinitely (research artifact; may be published) | N/A — retained | Confirm no PHI leakage in weights |
| Raw ML training CSVs | TBD (delete after model publication? Or keep with PHI data?) | Secure delete if identifiable | **CONFIRM WITH DR. LINDA** |
| Audit logs / access logs | TBD (1–3 years standard for incident investigation) | Log rotation + secure delete | **CONFIRM WITH DR. LINDA** |
| Docker secrets / credentials | Rotate every 90 days (or on suspected compromise) | Overwrite old secret + force redeploy | Standard, confirm no objection |
| Backups (WAL archives, base backups) | 30-day rolling retention (PITR window). Long-term monthly snapshot TBD. | Automated deletion via `backup-pitr.sh` retention policy | 30-day current config; confirm IRB requires longer archival backup |

---

## 7. Participant Withdrawal & Data Deletion

### 7.1 Withdrawal Rights

Participants have the right to withdraw from the study at any time.

### 7.2 Deletion SLA

- **Time from withdrawal request to data delete:** **Within 30 days** (or sooner if IRB requires). Confirm with Dr. Linda.

### 7.3 Deletion Scope

When a participant withdraws:
- [ ] Delete identifiable fields in `participant` row (name, contact, DOB → replaced with placeholder/NULL). Participant UUID retained for linkage only if IRB requires "soft delete."
- [ ] Delete or nullify `consent_record` (or mark withdrawn per IRB rules).
- [ ] Delete or de-identify `survey_response`, `distress_screening`, `wp6_session`, `referral` rows linked to that participant.
- [ ] **Hard problem:** Delete (or de-identify) the participant's records in REDCap source data (requires coordination with REDCap admin).
- [ ] **Hard problem:** Delete (or de-identify) any free-text data used in ML training. If the model is already trained, this may be impossible — document this limitation and get IRB sign-off that ML weights are not PHI.
- [ ] **Hard problem:** Remove from backups — PITR WAL archives contain historical state. Document that backup retention is 30 days; prior backups may retain deleted data until natural rotation expires. If IRB requires immediate backup purge, a full base-backup reset + WAL purge is needed (document the procedure in 08-operational-runbook.md).

**CONFIRM WITH DR. LINDA:**
- [ ] Is "soft delete" (retain UUID + remove all other fields) acceptable, or must the UUID also be purged?
- [ ] If data has already been used in a published analysis, what is the deletion policy?
- [ ] What is the procedure for a withdrawal request? (Phone to whom? Email to admin? Form?)

---

## 8. Data Access Control

### 8.1 Roles & Access Matrix (Target State — current state: NO AUTH)

| Role | Can See Identifiable Data? | Can Edit Data? | Can Export? | System Actions |
|------|----------------------------|----------------|-------------|----------------|
| **Enumerator** | Only participants assigned to their site/school, during their visit | Submit new forms only | No | Participant search (site-scoped), form submission |
| **Country Lead** | Only participants in their country (aggregated dashboard; drill-down to individual?) | Limited (reassign enumerators, site config?) | Country-scoped export (confirm if allowed) | Site-level dashboards, consent status tracking |
| **PI (Dr. Linda)** | Study-wide aggregated + identifiable individual records (confirm — does PI need PHI or only pseudonymized?) | Read-only? Or case management? | Full study export (CSV/PDF report) | Study-wide dashboards, funder reports, IRB reports |
| **Admin** | Full system access (ALL PHI) | Create/edit users, manage participants, resolve alerts | Full export + audit log export | User management, safeguarding case closure, system settings |
| **Damien / DevOps** | Full system-level access (DB, logs, backups) — incidental access required for operations | Infrastructure only; no study data edits (except incident response) | Can export at DB level (restrict via policy + audit) | Infrastructure maintenance, backup, disaster recovery |
| **ML service (neps-ml-ai)** | Reads distress_screening + survey_response (limited scope for prediction) | Writes ML score columns only | No export | Batch or real-time prediction calls only |
| **ETL service (data-platform)** | Reads REDCap, writes to DB | Write-only on sync | No export | Sync only |

**CONFIRM WITH DR. LINDA:**
- [ ] Does PI need to see raw PHI (names, DOB) or is pseudonymized dashboard sufficient?
- [ ] Is Country Lead allowed to export identifiable country data, or only aggregated?
- [ ] Is Enumerator allowed to see historical participant data, or only submit new forms?
- [ ] Is DevOps (Damien) access acceptable, or does IRB require a named "Data Custodian" role with separate access agreement?

### 8.2 Authentication

**Current state: CRITICAL GAP — no end-to-end auth (see readiness review).**

Target state for IRB documentation:
- All NEPS Digital Portal access requires unique username + strong password (minimum 12 chars, MFA TBD)
- All Backend API access requires JWT bearer token with role claims
- All DB access from app services uses least-privilege service accounts (not superuser) — confirm current postgres user is least-privilege
- Password resets: Admin-initiated or self-service via email (email server TBD)
- Account lockout: after N failed attempts (TBD), admin unlock required

**CONFIRM WITH DR. LINDA:**
- [ ] Is MFA (authenticator app / SMS) required for Admin and PI? (Highly recommended)
- [ ] Is there an institutional password policy to reference? (KNUST IT policy)
- [ ] Password expiry? (90 days standard for PHI systems)

### 8.3 Access Review & Audit

Target state:
- Quarterly access review by Admin + Damien: list all active users, confirm roles still required, disable leavers
- All data access (reads of identifiable data) and all exports logged in audit log with: timestamp, user_id, participant_id (if applicable), action, IP address
- Audit log review monthly by Admin or Damien
- Audit log retention: 3 years (or per IRB requirement)

**Current state: Audit logging NOT implemented (medium gap per review). Must be implemented before IRB submission.**

---

## 9. Safeguarding Protocol (Suicidality & Distress)

This is a **core requirement** for mental health research.

### 9.1 Detection

- Enumerator-administered PHQ-9 / GAD-7 includes suicidality screening item(s)
- ML risk prediction model flags elevated distress (TBD — risk_models.py currently 0 bytes)
- Automated Prometheus alert rule: `SafeguardingCrisisAlert` fires P0 when suicidality flag or extreme distress score detected

### 9.2 Alert Escalation Path

*(Confirm exact names and phone numbers with Dr. Linda; fill in placeholders below.)*

```
SafeguardingCrisisAlert fires (Prometheus → Alertmanager → Discord #safeguarding-alerts)
    │
    ▼
Step 1: Immediate (0–30 min)
  ├─ Admin acknowledges alert in portal (TBD UI — safeguarding workflow UI not yet built)
  ├─ Admin contacts on-country welfare responder:
  │    Ghana: [Name] / [Phone]
  │    Sierra Leone: [Name] / [Phone]
  │    Tanzania: [Name] / [Phone]
  └─ Welfare responder confirms receipt on Discord thread

Step 2: Follow-up (24 hrs)
  ├─ Welfare responder attempts contact with participant
  ├─ Documents action taken in Referral record (portal UI TBD)
  └─ If contact failed: Admin escalates to Country Lead → PI

Step 3: Case Resolution (7 days)
  ├─ Case outcome logged: Resolved / Referred to clinical services / Lost to follow-up
  ├─ PI notified of all safeguarding cases weekly
  └─ IRB adverse-event reporting (if required — confirm threshold with Dr. Linda)
```

### 9.3 Crisis Hotlines

Display in the Enumerator user guide and prominently in the portal (Admin dashboard widget):
| Country | Crisis / Mental Health Hotline | Confirm Number |
|---------|-------------------------------|----------------|
| Ghana | Mental Health Authority helpline | **CONFIRM** |
| Sierra Leone | Ministry of Health mental health line | **CONFIRM** |
| Tanzania | Ministry of Health mental health line | **CONFIRM** |
| International | Befrienders Worldwide / 988-equivalent | **CONFIRM** |

**CONFIRM WITH DR. LINDA:**
- [ ] Who is the named safeguarding lead for the study?
- [ ] What is the exact IRB adverse-event reporting threshold (e.g., "any suicidality attempt must be reported within 72h")?
- [ ] Are welfare responders trained and certified? Document training date/certification.
- [ ] What happens if Discord is down? (No secondary alert channel is configured — HIGH gap per review. Add Slack/email/SMS as fallback before IRB submission.)

---

## 10. Data Sharing & Publication

**CONFIRM WITH DR. LINDA:**

- [ ] Will de-identified datasets be deposited in a public repository after publication? (e.g., Dryad, Figshare, Zenodo) — Which one?
- [ ] What is the embargo period? (Typically 6–12 months post-publication)
- [ ] Funder data-sharing requirements? (Include funder DMP template alignment if applicable)
- [ ] Authorship policy for researchers using the data?
- [ ] Can ML models/weights be shared?

---

## 11. Compliance & Certifications

| Item | Status | Notes |
|------|--------|-------|
| IRB / Ethics Approval | PENDING | Blocking everything — submit this DMP ASAP |
| Data Processing Agreement (DPA) between institutions (Ghana/SL/Tanzania) | TBD | Needed if cross-border transfers of personal data |
| GDPR (if EU funder) | TBD | May apply if funder is EU-based |
| National data protection laws (Ghana Data Protection Act, Sierra Leone/Tanzania equivalents) | TBD | Confirm applicability and registration |
| ISO 27001 / SOC 2 | No | Not required for academic research typically, but confirm with funder |
| HIPAA | TBD | Only if US funder/partner with HIPAA-covered entity |

---

## 12. Data Breach / Incident Response

Target procedure (document fully in 08-operational-runbook.md):

1. **Detection:** Prometheus alert, anomaly in audit log, user report, or Damien notices something during ops.
2. **Containment (0–1 hour):** Damien isolates affected container/DB; Revokes compromised credentials; Admin notifies PI (Dr. Linda).
3. **Assessment (1–4 hours):** Determine scope (what data exposed, which participants). Log everything.
4. **Notification (per law/IRB rules):**
   - Participants affected: within [TBD — 72h standard]
   - IRB: within [TBD]
   - KNUST IT Security: within [TBD]
   - Funder: per funder contract
   - National data protection authority: if legal requirement applies
5. **Remediation:** Restore from clean backup (PITR), patch vulnerability, force password resets.
6. **Lessons-learned report within 30 days to PI and IRB.**

**CONFIRM WITH DR. LINDA:**
- [ ] What is the IRB's required breach notification timeline?
- [ ] Who is the study's data protection officer / point person for incidents?

---

## 13. Roles & Responsibilities (Named Individuals)

**FILL IN NAMES WITH DR. LINDA + DAMIEN:**

| Role | Name | Affiliation | Email |
|------|------|-------------|-------|
| Principal Investigator | Dr. Linda | KNUST | |
| Ghana Country Lead | | | |
| Sierra Leone Country Lead | | | |
| Tanzania Country Lead | | | |
| Safeguarding Lead (named person) | | | |
| NEPS Admin (portal user management) | | | |
| System Owner / DevOps | Damien Nsoh | | |
| Data Custodian (legal owner of data) | Dr. Linda | KNUST | (confirm) |
| ML Lead | Yasmine | | |
| Backend Lead | | | |
| Portal / Frontend Lead | | | |
| Data Platform Lead | Bernard / Mr. Bernard? | | |

---

## Information Gathering Checklist

- [x] Damien provided infrastructure/storage/auth current-state details
- [ ] **Dr. Linda interview scheduled (30 min)**
- [ ] Dr. Linda confirms study overview (official name, dates, IRB info, funding)
- [ ] Dr. Linda confirms data types and whether paper/biological/audio data exists
- [ ] Dr. Linda confirms identifiability / anonymization rules and export restrictions
- [ ] Dr. Linda confirms physical storage location, on-prem IT encryption confirmation
- [ ] Dr. Linda confirms retention periods per data type
- [ ] Dr. Linda confirms withdrawal SLA, deletion scope, backup limitations
- [ ] Dr. Linda signs off role-access matrix
- [ ] Dr. Linda confirms MFA/password policy requirements
- [ ] Dr. Linda confirms safeguarding escalation path with NAMES and PHONE NUMBERS
- [ ] Dr. Linda confirms crisis hotline numbers for each country
- [ ] Dr. Linda confirms data sharing, publication, and funder DMP rules
- [ ] Dr. Linda confirms legal compliance requirements (GDPR / national DPAs)
- [ ] Dr. Linda confirms breach notification timelines and DPO
- [ ] All named individual roles filled in with real names and contact info
- [ ] IRB submission-ready version created from this draft with dates/signature pages
- [ ] **IRB submission**
