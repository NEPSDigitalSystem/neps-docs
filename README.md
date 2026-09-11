# neps-docs — NEPS Digital System Documentation

NEPS Digital: Documentation repository. Contains technical references, user guides, data governance, and operational runbooks.

---

## File Structure & Priority

| # | File | Owner | Status | Priority | Interview Needed |
|---|------|-------|--------|----------|-----------------|
| — | [confidentiality_and_trade_secret_register.md](confidentiality_and_trade_secret_register.md) | Legal / PI | ✅ Done | — | Done |
| — | [documentation-plan.md](documentation-plan.md) | Documentation Lead | Template (original 7-doc plan) | — | Reference only |
| 01 | [01-api-reference.md](01-api-reference.md) | Doc Lead | Draft skeleton | **3** | Backend Lead + ML Lead (Yasmine) |
| 02 | [02-user-guide-admin.md](02-user-guide-admin.md) | Doc Lead | Draft skeleton | **5** | Portal Lead + Damien |
| 03 | [03-user-guide-pi.md](03-user-guide-pi.md) | Doc Lead | Draft skeleton | **6** | Dr. Linda / AyineBia + Portal Lead |
| 04 | [04-user-guide-enumerator.md](04-user-guide-enumerator.md) | Doc Lead | Draft skeleton | **4** | Field Coordinator + Portal Lead |
| 05 | [05-user-guide-country-lead.md](05-user-guide-country-lead.md) | Doc Lead | Draft skeleton | **7** | Country Coordinators + Portal Lead |
| 06 | [06-data-flow-and-architecture.md](06-data-flow-and-architecture.md) | **Damien** + Doc Lead | **Damien content in — ready for Backend Lead review** | **2** | Backend Lead (verify) |
| 07 | [07-ethics-data-management-plan.md](07-ethics-data-management-plan.md) | **Damien + Dr. Linda** | **Damien content in — 40+ CONFIRM items for Dr. Linda** | **1** (IRB blocker) | Dr. Linda (30-min interview URGENT) |
| 08 | [08-operational-runbook.md](08-operational-runbook.md) | **Damien** | **Damien content in — contacts PIR checklist items pending** | **8** | Damien (fill contacts) + KNUST IT |
| 09 | [09-redcap-integration-guide.md](09-redcap-integration-guide.md) | Doc Lead | Draft skeleton | **9** | Backend Lead + Data Platform Lead |

---

## Recommended Next Steps (for Damien + Doc Lead)

1. **THIS WEEK** — Damien schedules Dr. Linda interview (30 min) to sign off doc 07 (IRB blocker).
2. **THIS WEEK** — Damien schedules Backend Lead review of doc 06 (architecture + data flow).
3. **THIS WEEK** — Doc Lead schedules 30-min interviews with: Portal Lead (docs 02, 03, 04, 05), Field Coordinator (doc 04), Yasmine/ML (doc 01), Data Platform Lead (doc 09).
4. **Damien fills names/phones** in doc 08 Section 2 (Key Contacts) before production.
5. **Doc Lead writes first drafts** of each interviewed document within 24 hours of each call.

---

## Repositories Referenced

- `neps-infrastructure` — Docker Compose, Nginx, Prometheus/Grafana/Alertmanager, Ofelia, PITR scripts
- `neps-backend` — FastAPI, PostgreSQL, Redis, REDCap sync scheduler
- `neps-portal` — Next.js 15, NextAuth, role-based dashboards
- `neps-data-platform` — REDCap ETL extract/transform/load pipeline
- `neps-ml-ai` — NLP models (sentiment, emotion, risk TBD)

Source architecture diagrams: `NEPSDigSystem-main/Info Files/` (information flow, deployment, component diagrams).
