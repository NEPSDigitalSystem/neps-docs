# 01 — API Reference

**Document Status:** Draft — Pending Interviews
**Owner:** Documentation Lead
**Interview Required:** Backend Lead + ML/AI Lead (Yasmine)
**Priority:** 3

---

## Router Inventory (from `neps-backend/main.py`)

Every router except `health`, `auth`, and `/` is mounted with the
`require_research_authorisation` guard at the router level; guarded routers also
re-check at the endpoint. Research access is fail-closed and additionally gated
by `RESEARCH_ACCESS_ENABLED` (default `false`).

| Prefix | Module | Guard | Notes |
|--------|--------|-------|-------|
| `/` | `main.py` | Public | Service banner (`app_name`, `app_env`) |
| `/health` | `app.api.health` | Public | Liveness/readiness for Nginx, Prometheus, blackbox |
| `/api/auth` | `app.api.v1.auth` | Public | Login/session; must work without a token |
| `/api/portal` | `app.routers.portal` | Research | Portal (NDS registry, consent, participants) |
| `/api/redcap` | `app.routers.redcap` | Research | REDCap proxy — reference only |
| `/api/v31` | `app.routers.v31` | Research | V3.1 baseline reads/exports (Track A) |
| `/api/v1/sync` | `app.api.v1.redcap_sync` | Research | REDCap sync control |
| `/api/v1/analytics` | `app.api.v1.analyst_dash` | Research | Analytics — Track B (provisional) |
| `/api/ml` | `app.routers.ml` | Research | ML/AI proxy — Track B (provisional) |

Every response carries an `X-Request-ID`. Paths under `TRACK_B_PATH_PREFIXES`
(`/api/portal/surveys`, `/api/portal/distress`, `/api/portal/wp6`,
`/api/portal/referrals`, `/api/v1/analytics`, `/api/ml`) additionally carry
`X-NEPS-Track: Track B` and `X-NEPS-Provisional: true`.

---

## ML/AI Proxy Endpoints (`/api/ml`)

The frontend must call these endpoints — never `neps-ml-ai` directly. The
backend authenticates the human, audits intent/completion (fail-closed), applies
rate limiting, and forwards a *minimised* payload to ML/AI with a separate
service token. See
[06-data-flow-and-architecture.md §4.2](06-data-flow-and-architecture.md#42-mlai-request-lifecycle-frontend--backend--mlai--backend--frontend)
for the full round trip.

| Method | Path | Auth | Rate limit | Request | Response |
|--------|------|------|-----------|---------|----------|
| POST | `/api/ml/sentiment` | Research authorisation | 10/min | `{ "text": string (1–10000 chars) }` | `{ "prediction": string, "scores": { model: float } }` |
| POST | `/api/ml/emotion` | Research authorisation | 10/min | `{ "text": string (1–10000 chars) }` | `{ "prediction": string, "scores": { model: float } }` |

**Behaviour and status codes**

- Both endpoints are decorated `@no_participant_data` and require research
  authorisation; they are not general-purpose public NLP endpoints.
- `503` if `ML_AI_ENABLED` is `false` (default) or the ML service token is not
  configured — the backend never contacts ML in that case.
- `503` if the required audit write fails (fail-closed).
- `503` if ML/AI is unreachable or returns an invalid payload.
- Results are cached per user for 300s, keyed by `ml:{model}:{user_id}:{sha256(text)}`.
- Only text *length* — never raw text — is written to the audit log.

> **Wiring note.** The portal's BFF proxy allow-list currently covers only
> `api/v31`, `api/portal`, and `health`, and there is no ML UI surface yet. Until
> `api/ml` is added to that allow-list, browser-originated ML calls return `404`.

---

## Information Gathering Checklist

- [ ] Interview scheduled with Backend Lead
- [ ] Interview scheduled with ML/AI Lead (Yasmine)
- [ ] Exported OpenAPI JSON from backend `/openapi.json`
- [ ] Captured FastAPI `/docs` screenshots for all routers
- [ ] Documented auth scheme & role requirements per endpoint
- [ ] Backend lead review signed off
- [ ] ML lead review signed off
