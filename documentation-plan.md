# NEPS Documentation Plan

## Document 1: Data Security
What this doc will explain:
- How user logins work (authentication)
- Who can see what (authorization/roles)
- How data is encrypted in transit and at rest
- How access is tracked (audit logs)

Information I need:
- [ ] What authentication method is used? (2FA? SSO? Password only?)
- [ ] What are the user roles? (admin, clinician, enumerator, researcher, etc.)
- [ ] Is all data encrypted on disk?
- [ ] Where are audit logs stored and who reviews them?

Source team(s): neps-backend, neps-infrastructure

---

## Document 2: Three-Country Hosting
What this doc will explain:
- Where servers are physically located
- How Ghana, Sierra Leone, and Tanzania teams connect
- What happens if the internet goes down in a remote site
- Backup and disaster recovery

Information I need:
- [ ] Exact server location (KNUST only? Any cloud?)
- [ ] Do Sierra Leone/Tanzania teams use VPN or just HTTPS?
- [ ] Can field teams work offline? If yes, how does sync work?
- [ ] How often are backups tested?

Source team(s): neps-infrastructure

---

## Document 3: End-to-End Data Flow
What this doc will explain:
- Path from user input → database → back to user
- Which systems touch the data at each step

Information I need:
- [ ] Is the information flow diagram I received still accurate?
- [ ] What happens at each numbered step (1-7) in the diagram?
- [ ] Which team owns each step?

Source team(s): All teams

---

## Document 4: Dashboard & Frontend
What this doc will explain:
- What users see in the web portal
- How data is filtered by country
- How the "Quick Actions" buttons work

Information I need:
- [ ] What are the main user roles and what does each role see on their dashboard?
- [ ] How does country switching work?
- [ ] Are there mobile apps or only web?

Source team(s): neps-portal

---

## Document 5: Risk Detection (AI)
What this doc will explain:
- How the system flags "High Distress Alerts"
- What triggers a safeguarding alert
- How a clinician responds to an alert

Information I need:
- [ ] What data feeds into the AI models?
- [ ] How is a risk score calculated?
- [ ] Who gets notified when an alert triggers?

Source team(s): neps-ai-analytics, neps-backend

---

## Document 6: Forms & REDCap
What this doc will explain:
- How surveys and forms are created
- How data from forms gets stored
- How form completion is tracked

Information I need:
- [ ] Who creates/modifies REDCap forms?
- [ ] How does data validation work?
- [ ] What happens to incomplete forms?

Source team(s): neps-backend, neps-data-platform

---

## Document 7: Backups & Disaster Recovery
What this doc will explain:
- What gets backed up and how often
- How long it takes to restore after a failure
- What data might be lost (RPO)

Information I need:
- [ ] Is RTO 24-48 hours and RPO 24 hours (from diagram) correct?
- [ ] Are backups tested? How often?
- [ ] Is there an off-site/cloud backup?

Source team(s): neps-infrastructure
