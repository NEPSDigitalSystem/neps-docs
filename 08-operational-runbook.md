# 08 — Operational Runbook

**Document Status:** Draft — Damien-owned; add KNUST IT on-call contacts
**Owner:** Damien (DevOps / System Owner)
**Interview Required:** Damien (primary author) + Infrastructure review
**Priority:** 8 (Critical for production but only Damien needs it now)

---

## 1. Audience & Scope

**Audience:** Damien (primary), any future SRE/DevOps on-call, KNUST IT (for escalations), PI (Dr. Linda) for incident notifications.

**Scope:** All operational procedures for the NEPS Digital System running on the KNUST on-premise server via Docker Compose. Covers: incident triage, service restarts, database restore, deployment, rollback, secret rotation, safeguarding escalation.

This document is **not a training manual for new DevOps**. It is a step-by-step checklist for Damien and named successors during outages.

---

## 2. Key Contacts — FILL THESE IN BEFORE PRODUCTION

| Role | Name | Phone | Email | Discord Handle | Response SLA |
|------|------|-------|-------|----------------|--------------|
| Primary on-call (DevOps) | Damien Nsoh | | | | 24/7 for P0 |
| Backup on-call | | | | | P0: 30 min if primary unreachable |
| KNUST IT (host/network) | | | | | P0 host down: 1 hour |
| Safeguarding Lead (study) | Dr. Linda / named person | | | | P0 safeguarding: 30 min |
| Backend Lead | | | | | P1/P2: 4 hours workday |
| Portal / Frontend Lead | | | | | P1/P2: 4 hours workday |
| PI (Dr. Linda) | | | | | Notify within 4 hours for P0/P1 incidents |

---

## 3. URLs & Ports Reference

### 3.1 Production

| Service | Internal (Docker DNS) | Host URL | Credentials Location |
|---------|----------------------|----------|---------------------|
| Portal (Nginx HTTPS) | nginx:443 | `https://[PROD_DOMAIN]/` | Nginx TLS: `./nginx/ssl/neps.crt / neps.key` (dev certs now; prod CA cert TBD) |
| Backend API | neps-backend:8000 | `https://[PROD_DOMAIN]/api/*` (via Nginx) | JWT signing secret: Docker secret `jwt_private.pem` |
| Backend FastAPI Docs | — | `https://[PROD_DOMAIN]/docs` | Same auth as API (when implemented) |
| Backend Health | neps-backend:8000 | `https://[PROD_DOMAIN]/health` | Public |
| Grafana Dashboards | grafana:3000 | `http://[PROD_HOST]:3001` (restricted) | `app_secret_key` Docker secret (currently shared with MinIO/PgAdmin — fix per review) |
| Prometheus UI | prometheus:9090 | `http://[PROD_HOST]:9090` (restricted — consider NOT exposing on host) | No auth (IP-restrict at host firewall) |
| Alertmanager UI | alertmanager:9093 | `http://[PROD_HOST]:9093` (restricted) | No auth (IP-restrict) |
| PgAdmin | pgadmin:80 | `http://[PROD_HOST]:5050` (restricted) | `app_secret_key` |
| MinIO Console | minio:9001 | `http://[PROD_HOST]:9001` (restricted) | `app_secret_key` |

### 3.2 Staging (if Option A — side-by-side on same host)

All ports shifted +10,000. Container names prefixed `neps-staging-*`.

| Service | Host URL | Notes |
|---------|----------|-------|
| Nginx HTTP/HTTPS | `http://[HOST]:18080` / `https://[HOST]:18443` | `X-NEPS-Environment: staging` header + `X-Robots-Tag: noindex` |
| Grafana | `http://[HOST]:13001` | Staging volumes separate |
| Prometheus | `http://[HOST]:19090` | |
| Alertmanager | `http://[HOST]:19093` | Routes to `#staging-alerts` Discord, not safeguarding channel |
| PgAdmin | `http://[HOST]:15050` | |
| Postgres direct | `[HOST]:15432` | (if PgAdmin fails) |
| Backend direct | `http://[HOST]:18000/health` | (bypass Nginx for debugging) |

---

## 4. Severity Definitions

| Level | Label | Example | First Response | Resolution SLA |
|-------|-------|---------|----------------|----------------|
| **P0** | CRITICAL — User-impacting outage or safeguarding | Portal returns 5xx for 100% of users / PostgreSQL down / SafeguardingCrisisAlert firing and Discord route down | 5 min | 1 hour |
| **P1** | HIGH — Partial outage or data pipeline broken | One role dashboard returns 5xx / REDCap sync stale >24h / ETL failed 2 nights in a row | 15 min workday | 4 hours |
| **P2** | MEDIUM — Degraded non-critical | Prometheus scrape partial / high latency >500ms p95 / Grafana single panel broken | 2 hours workday | Next workday |
| **P3** | LOW — Cosmetic or advisory | Typo in UI / outdated doc / Trivy found LOW CVEs | Next workday | Best effort |

---

## 5. P0 Playbook — "Website Is Down"

**If the website is down, follow these steps IN THIS EXACT ORDER.** Do not skip.

### Step 1 — Confirm (2 min)
```bash
# From your laptop
curl -Ik https://[PROD_DOMAIN]/ | head -5
# Also check staging to compare
curl -Ik https://[PROD_HOST]:18443/ | head -5
```
- If prod returns 200 but you got a complaint — it's user's internet / DNS. Ask for screenshot.
- If prod returns 5xx or timeout or connection refused — proceed.

### Step 2 — SSH to server & check container status (2 min)
```bash
ssh [DEPLOY_USER]@[DEPLOY_HOST]
cd /opt/neps-infrastructure   # or wherever the repo is on server

# Check compose state
docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.discord-alerts.yml ps
```
Scan output for any service with State != `Up (healthy)`.

### Step 3 — Check Nginx first (3 min)
80% of outages are either Nginx or a container crash-loop.
```bash
# Nginx logs
docker compose logs nginx --tail=100 | less
# If nginx is restarting: check cert
ls -la nginx/ssl/
# Verify nginx config syntax
docker compose exec nginx nginx -t
# If syntax fails, revert last nginx.conf change
```

### Step 4 — Check PostgreSQL (3 min)
```bash
docker compose logs postgres --tail=50
# Connect and check
docker compose exec postgres psql -U neps neps_core -c "SELECT 1;"
# Check connections
docker compose exec postgres psql -U neps neps_core -c "SELECT count(*) FROM pg_stat_activity;"
# Max_connections check (default 100; threshold alert at >80%)
docker compose exec postgres pg_isready
```
- If Postgres is `FATAL:  sorry, too many clients already` → kill idle transactions or restart backend/data-platform containers (they may be leaking connections).
- If Postgres data volume corrupt → go to Section 9 (PITR Restore).

### Step 5 — Check Backend & Portal (3 min)
```bash
docker compose logs neps-backend --tail=100
docker compose logs neps-portal --tail=100
# Direct health check (bypass Nginx)
curl http://neps-backend:8000/health   # from inside server; or use curl to localhost:8000 if mapped
curl http://neps-portal:3000/
```
- If backend shows `ImportError`, `ModuleNotFoundError` → bad image. **Roll back immediately** (Section 8).
- If portal shows Next.js `Application error` → bad deploy. **Roll back immediately** (Section 8).

### Step 6 — Check Disk / Memory / CPU (2 min)
```bash
df -h
free -m
uptime
docker stats --no-stream
```
- If `/var/lib/docker` >90% → **critical**. Check backup volumes. Clean old images: `docker image prune -af` (only after confirming rollback history retention).
- If OOM kills in dmesg → `dmesg -T | grep -i oom`. Increase memory limits or kill staging containers if side-by-side.

### Step 7 — Escalate
If after Step 6 the system is still down:
1. **Roll back to previous deployment** (Section 8) — this will solve 90% of post-deploy outages.
2. If still down after rollback → KNUST IT P0: host is bad.
3. Call backup on-call if you've spent >30 min.
4. Notify Dr. Linda via phone/email with: what's down, when it started, estimated recovery time.

---

## 6. P0 Playbook — Safeguarding Crisis Alert

A `SafeguardingCrisisAlert` fires when: suicidality screening item positive OR distress score above ML threshold. **This is a clinical safety event, not just a technical alert.**

### Step 1 — Confirm the alert is real (2 min)
- Open Discord `#safeguarding-alerts` channel (prod) / `#staging-alerts` (staging).
- Open the alert link → Alertmanager → Prometheus graph. Confirm the alert is not a test.
  - *Staging note:* Synthetic smoke test fires daily at 10:00 UTC on staging to verify routing. This is EXPECTED. Ignore unless it's outside 10:00–10:15 UTC.
- Log into Admin dashboard → AlertCard → find the corresponding participant ID and distress screening record.

### Step 2 — Acknowledge & route (5 min)
- **Acknowledge in Alertmanager UI** (silence 1 hour so it doesn't re-page).
- **Call the on-country welfare responder:**
  - Ghana: [phone]
  - Sierra Leone: [phone]
  - Tanzania: [phone]
- Confirm responder is acting. Get a name and ETA for participant contact.

### Step 3 — Document (10 min)
- Log into portal safeguarding case management UI (TBD).
- Assign responder, log: "Alert acknowledged [time], [Responder Name] contacted, ETA participant contact [time]."
- Set alert status to "In Progress."

### Step 4 — Escalate if no responder contact after 30 min
- Call Country Lead.
- If Country Lead unreachable, call Safeguarding Lead (Dr. Linda).
- If Dr. Linda unreachable, call PI backup and flag for KNUST IRB notification.

### Step 5 — Resolve (within 7 days)
- Welfare responder logs final outcome: Resolved / Clinical referral / Hospitalized / Lost to follow-up.
- Admin closes case in portal, adds case summary.
- **IRB adverse event report:** Filed within TBD days if the event meets IRB's threshold (confirm with Dr. Linda in DMP doc 07).

### If Discord alert route fails
- Fallback 1: Check Alertmanager UI directly (host:9093) for active alerts.
- Fallback 2: Check Prometheus → Alerts page for `SafeguardingCrisisAlert` firing.
- Fallback 3: Manually query distress_screening table for suicidality=TRUE in last 24h:
  ```sql
  SELECT participant_id, screening_date, suicidality_flag, distress_score
  FROM neps_core.distress_screening
  WHERE screening_date >= NOW() - INTERVAL '24 hours'
    AND (suicidality_flag = TRUE OR distress_score >= [THRESHOLD]);
  ```
- **CRITICAL GAP per readiness review:** No secondary alert channel (Slack/email/SMS). **Add before production deploy.**

---

## 7. Deployment Procedure

### Prerequisites
- `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_KEY` set in each repo's GitHub secrets (currently NOT provisioned per review — pending KNUST server)
- CI validated code via feature branch → PR to `staging` → smoke-tested on staging → PR to `main`
- Staging URL verified: all dashboards load, `/health` returns 200, ETL sync ran, no CVE >HIGH

### Automated Deploy (CI/CD — when vars set)
```
1. PR from staging → main → merge
2. GitHub Actions ci-cd.yml triggers on push to main
   ├─ validate: lint + test
   ├─ build: docker build → GHCR
   ├─ security: trivy image scan (exit-code=1 on HIGH/CRITICAL; FAIL if this fails)
   └─ deploy (only if DEPLOY_HOST != ''): uses DEPLOY_KEY to ssh to DEPLOY_HOST
       → cd /opt/neps-infrastructure
       → docker compose pull
       → docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.discord-alerts.yml up -d
       → loops /health checks for 3 minutes
       → if all healthy: SUCCESS, records SHA in .deployment-history
       → if any unhealthy: NOTIFIES, but does NOT auto-rollback (set this up per review gap)

3. Damien receives "deploy success/fail" notification (email / Discord webhook from GH Actions)
```

### Manual Deploy (break-glass, if CI down)
```bash
# On server, as DEPLOY_USER
cd /opt/neps-infrastructure

# 1. Get target image tags — either pin a SHA or use latest
export IMAGE_TAG=[sha or latest]

# 2. Backup DB first (ALWAYS before deploy)
./scripts/backup-pitr.sh
# Confirm backup exists in /backup/pitr/base/[timestamp].tar.gz

# 3. Pull new images
docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.discord-alerts.yml pull

# 4. Apply migrations (data-platform + backend)
docker compose run --rm neps-data-platform python -m etl.main migrate
# alembic in backend auto-runs on start; if not:
docker compose run --rm neps-backend alembic upgrade head

# 5. Restart services with new images
docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.discord-alerts.yml up -d

# 6. Health check loop
for i in {1..30}; do
  curl -skf https://localhost/health && echo "OK" && break
  echo "waiting attempt $i..."
  sleep 10
done

# 7. Record SHA
echo "$(date +%s) $(git rev-parse HEAD)" >> .deployment-history
```

### Staging Deploy
Same, but:
- Triggered by push to `staging` branch, not `main`
- `STAGING_DEPLOY_HOST` / `STAGING_DEPLOY_USER` / `STAGING_DEPLOY_KEY` vars
- Overlay: `docker-compose.staging.yml` AND `COMPOSE_PROJECT_NAME=neps-staging`
- Deploy URL check: `https://[STAGING_HOST]:18443/health`

---

## 8. Rollback Procedure

### Automated / Easy — Roll back to previous deploy
```bash
cd /opt/neps-infrastructure
# Roll to previous deployment (last SHA in .deployment-history)
./scripts/rollback.sh previous
```
Script does: reads `.deployment-history`, pulls previous GHCR SHAs, applies `docker-compose.rollback.yml` overlay, restarts, polls `/health`, declares success only if all healthy.

### Targeted — Roll back to specific SHA
```bash
./scripts/rollback.sh specific-sha [full-40-char-git-sha]
```

### Break-glass — Roll back manually if script fails
```bash
# 1. Find old image SHAs in GHCR (or .deployment-history)
# 2. Pin image tags in docker-compose.prod.yml or as env override
# 3. Redeploy same as manual deploy but with old tags
export IMAGE_TAG=[old-sha]
docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.discord-alerts.yml up -d
# 4. If DB migrated backwards-incompatibly, MUST restore DB to pre-deploy backup (Section 9)
```

**Auto-rollback note (HIGH gap per review):** CI currently does NOT auto-rollback on deploy failure. **Add this before production** by adding a GH Actions step that calls `./scripts/rollback.sh previous` if the health-check loop fails.

---

## 9. PostgreSQL Restore (PITR)

Use this when: database corruption, accidental `DELETE`/`DROP` without `WHERE`, bad migration, or a deploy wrote bad data.

### 9.1 Full Restore to a Specific Moment in Time
```bash
cd /opt/neps-infrastructure

# Step 1: CHOOSE A TARGET TIME
# Pick a time BEFORE the incident. Use ISO format: "2026-05-27 14:30:15"
# Confirm this time is after the last good base backup (base backups are daily at 02:00)
TARGET_TIME="YYYY-MM-DD HH:MM:SS"

# Step 2: MAKE A SAFETY BACKUP OF CURRENT BROKEN STATE
# YES — backup the broken DB first! You may need it for forensics.
./scripts/backup-pitr.sh
mv backups/pitr/base/$(ls -t backups/pitr/base/ | head -1) \
   backups/pitr/base/PRE-RESTORE-BROKEN-$(date +%s).tar.gz

# Step 3: EXECUTE PITR RESTORE
./scripts/pitr-restore.sh "$TARGET_TIME"
# Script will: stop postgres, extract base backup, replay WAL logs to target_time, start postgres

# Step 4: VERIFY
docker compose exec postgres psql -U neps neps_core -c "SELECT count(*) FROM participant;"
# Spot-check: find a participant deleted in incident; confirm they now exist
docker compose exec postgres psql -U neps neps_core -c "SELECT * FROM participant WHERE id = 'KNOWN-DELETED-UUID';"

# Step 5: NOTIFY
Send Slack/Discord: "DB restored to $TARGET_TIME. Any writes between then and now are LOST."
Notify Dr. Linda if clinical/survey data was lost.
```

### 9.2 PITR Drill (Test the restore WITHOUT touching production)
```bash
./scripts/pitr-drill.sh  # Uses isolated PG on alternate port
```
Runs restore in a throwaway container on port 15433, queries it. If this passes, your backup chain is valid.
**Schedule:** Monthly. Put a reminder in your calendar.

### 9.3 Participant Withdrawal Data Deletion (IRB compliance)
Per Section 7 of doc 07-ethics-data-management-plan.md:
```sql
-- SOFT DELETE (retain UUID, strip all PHI — confirm with Dr. Linda if this is acceptable)
BEGIN;
UPDATE neps_core.participant
SET first_name = '[WITHDRAWN]',
    last_name = '[WITHDRAWN]',
    date_of_birth = NULL,
    phone = NULL,
    next_of_kin_name = NULL,
    next_of_kin_phone = NULL,
    address = NULL,
    withdrawn_at = NOW()
WHERE id = 'PARTICIPANT_UUID_HERE';

-- Delete their survey responses (confirm with IRB — sometimes required to retain aggregates)
DELETE FROM neps_core.survey_response WHERE participant_id = 'PARTICIPANT_UUID_HERE';
-- OR de-identify only

-- Delete their distress screenings / referrals / wp6_sessions similarly
DELETE FROM neps_core.distress_screening WHERE participant_id = 'PARTICIPANT_UUID_HERE';
DELETE FROM neps_core.referral WHERE participant_id = 'PARTICIPANT_UUID_HERE';
DELETE FROM neps_core.wp6_session WHERE participant_id = 'PARTICIPANT_UUID_HERE';

COMMIT;
```
**Backup purge caveat:** Deleted participant records will still exist in base backups and WAL archives for up to 30 days until natural retention removes them. If IRB requires immediate purge, after the SQL above run:
```bash
# FORCE a new base backup, then purge all old WAL + old base backups (destroys ability to restore pre-delete state!)
# Confirm with Dr. Linda and IRB before running this — it is DESTRUCTIVE of backup history.
./scripts/backup-pitr.sh
# Then MANUALLY delete pre-date base backup files and WAL archives older than new base backup.
# Run a PITR drill to confirm backup chain is still valid after purge.
```

---

## 10. Secret Rotation

Rotate every 90 days OR on suspected compromise.

### 10.1 Full secret regeneration (all secrets)
```bash
cd /opt/neps-infrastructure
./scripts/setup-secrets.sh
# Outputs all new secrets to ./secrets/ directory with chmod 600
# Lists which services need redeploy: neps-backend, neps-portal, neps-ml-ai, neps-data-platform, postgres, grafana, minio, pgadmin, alertmanager
docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.discord-alerts.yml up -d --force-recreate
```

### 10.2 Individual secret: JWT key (invalidates all sessions)
```bash
# Generate new RSA key pair
openssl genpkey -algorithm RSA -out secrets/jwt_private.pem -pkeyopt rsa_keygen_bits:4096
openssl rsa -pubout -in secrets/jwt_private.pem -out secrets/jwt_public.pem
chmod 600 secrets/jwt_*.pem
# Restart backend and portal
docker compose up -d --force-recreate neps-backend neps-portal
# All users will be logged out; Admin communicates password reset flow if needed
```

### 10.3 Individual secret: PostgreSQL password
```bash
# 1. Generate new password
openssl rand -base64 32 > secrets/postgres_password
# 2. Apply in running DB AND in config (requires DB restart)
# 3. Restart everything that connects: backend, data-platform, ml-ai, postgres-exporter, pgadmin
```

### 10.4 Discord webhook URL rotation
```bash
echo "https://discord.com/api/webhooks/NEW/URL" > secrets/discord_webhook_url.txt
# Staging equivalent:
echo "https://discord.com/api/webhooks/NEW-STAGING/URL" > secrets/discord_webhook_url_staging.txt
docker compose up -d --force-recreate alertmanager
# Send test alert via Alertmanager API or wait for synthetic 10:00 UTC staging test
```

### 10.5 REDCap API token rotation
```bash
echo "NEW_REDCAP_API_TOKEN" > secrets/redcap_api_token
docker compose up -d --force-recreate neps-backend neps-data-platform
# Verify sync works
curl -X POST http://localhost:8000/api/v1/sync/trigger  # or use the manual trigger endpoint
curl http://localhost:8000/api/v1/sync/status
```

---

## 11. Data Pipeline Failure Playbook

### Symptom: `ETLJobFailure` or `REDCapSyncStaleness` alerts fire

### ETL (neps-data-platform) — nightly Ofelia sync 01:00 UTC
```bash
# Check Ofelia logs
docker compose logs ofelia --tail=50
# Look for the docker exec line — did it run?
# If Ofelia says "container not found": WRONG container name.
#    Staging uses namespaced names — check config.staging.ini references neps-staging-neps-data-platform-1
#    Prod vs staging naming conflict is a known sensitivity (per project_memory)

# Run manually
docker exec neps-data-platform python -m etl.main sync --full 2>&1 | tee /tmp/etl-run.log
# Check exit code
echo $?
# If non-zero: read /tmp/etl-run.log for which stage failed (extract / transform / load)
# Common failures:
#   - REDCap API unreachable → run mock client as fallback: REDCAP_MOCK_ENABLED=true python -m etl.main sync --full
#   - DB connection → check postgres alive + DATABASE_URL_FILE secret contents
#   - Normalizer field mismatch → REDCap schema updated without updating mock_schema.py; compare REDCap export to etl/transform/mock_schema.py; add missing fields

# Run dry-run to see planned writes without applying
docker exec neps-data-platform python -m etl.main sync --dry-run
# Check RunLog table for history:
docker compose exec postgres psql -U neps neps_core -c "SELECT * FROM neps_core.etl_run_log ORDER BY started_at DESC LIMIT 5;"
```

### Backend hourly REDCap sync (participants only — surveys NOT YET IMPLEMENTED)
```bash
# Check scheduler
docker compose logs neps-backend --tail=100 | grep -i sync
# Manual trigger
curl -X POST http://localhost:8000/api/v1/sync/trigger
# Check status
curl http://localhost:8000/api/v1/sync/status
# NOTE: LAST_SYNC_STATUS is IN-MEMORY GLOBAL, LOST ON RESTART.
# HIGH gap per review. Temporary workaround: trust Prometheus REDCapSyncStaleness alert for ground truth.
```

---

## 12. Adding a New User to the System

**Temporary manual workflow until Admin dashboard user management UI is built:**
```bash
# Note: No User DB model yet! See readiness review — AUTH is CRITICAL gap.
# Current workaround: if portal NextAuth mock mode is active, add to hardcoded list in neps-portal/auth.ts
# For production REAL workflow, after backend auth implemented:
docker compose exec neps-backend python -m app.scripts.create_user \
    --email user@country.neps \
    --role enumerator \   # or admin / country_lead / pi
    --country Ghana \
    --temporary-password "ChangeMe123!"
# Then Admin dashboard → Users → Reset password forces on first login.
```
**FIX BEFORE PROD:** Remove hardcoded portal users. Wire real backend User model + `/api/auth/login`.

---

## 13. Monitoring Dashboards Cheat Sheet

### Grafana — `http://host:3001` (prod) / 13001 (staging)
Currently only one dashboard: `NEPS Overview` (JSON in `monitoring/grafana-dashboards/neps-overview.json`).
HIGH gap per review — add dashboards for:
- Backend per-route latency, error rate (Prometheus: `http_request_duration_seconds_*`)
- PostgreSQL: slow queries, dead tuples, connection count
- Safeguarding-specific: alerts-per-day, time-to-acknowledge, time-to-resolve
- ETL: run duration, rows synced, data quality score

### Prometheus — `http://host:9090`
Useful PromQL queries:
```promql
# Is my service up?
up{job=~".*"}

# HTTP 5xx rate by route
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# Postgres connections % of max
pg_settings_max_connections / pg_stat_activity_count

# Safeguarding crisis alert count last 24h
sum_over_time(ALERTS{alertname="SafeguardingCrisisAlert",alertstate="firing"}[24h])

# REDCap sync age (seconds since last successful)
time() - redcap_last_sync_timestamp

# Container memory usage
container_memory_working_set_bytes / container_spec_memory_limit_bytes
```

### Alertmanager — `http://host:9093`
- Top navbar → Status: shows active alert routes, Discord webhook integration status
- Top navbar → Silences: temporary alert suppression (NEVER silence SafeguardingCrisisAlert)

### Loki logs — from Grafana Explore, datasource=Loki
```logql
# All error logs across stack in last hour
{compose_project=~"neps.*"} |= "error" | json | level="error"

# Safeguarding-related logs
{compose_project=~"neps.*"} |~ "(?i)safeguard|suicid|distress|referral"
```

---

## 14. Post-Incident Review (PIR)

For every P0 and any P1 that required rollback or restore:

1. Within **24 hours of resolution**, create a doc: Incident Report [date] — [slug]
2. Sections:
   - **Timeline** (UTC): when problem started, when detected, when acknowledged, each action taken, when resolved
   - **Root Cause** (5 Whys — get to system cause, not "human error")
   - **Impact**: how many users affected, how many safeguarding alerts delayed, data loss scope (if any)
   - **Detection gap**: why was it not caught before user reported?
   - **Action items**: owner + due date for each fix
3. Share with Dr. Linda within 48 hours.
4. Add action items to the relevant repo backlog.

---

## 15. Monthly Operations Checklist

Schedule these recurring tasks.

| When | Task | Command / Action |
|------|------|-----------------|
| 1st of month | PITR drill | `./scripts/pitr-drill.sh` |
| 1st of month | Backup verification: check MinIO/backup volumes have free space >50% | `df -h /backup /var/lib/docker` |
| 1st of month | Rotate JWT key (invalidates sessions) | Section 10.2 |
| Quarterly | Full secret rotation (all 90-day secrets) | Section 10.1 |
| Quarterly | User access review: disable leavers | Admin dashboard → Users (or DB if UI TBD) |
| Quarterly | Audit log review (when implemented) | Review all admin/data exports for anomaly |
| After any OS kernel patch | Restart server (planned maintenance window) and run health checks | |
| After Docker update | Full restart compose stack | `docker compose down && docker compose up -d` |

---

## Information Gathering Checklist

- [x] Damien provided all technical procedures (deploy, rollback, PITR, ETL debug, secret rotation, monitoring queries)
- [ ] **Damien fills in ALL contact names and phone numbers in Section 2 before production**
- [ ] Damien confirms actual deploy path on KNUST host (`/opt/neps-infrastructure` or other)
- [ ] KNUST IT contact added for host-level escalations
- [ ] Safeguarding welfare responder phone numbers per country confirmed with Dr. Linda
- [ ] Secondary alert channel (Slack/email/SMS) added (HIGH gap — must do before production)
- [ ] CI auto-rollback on deploy-health-failure wired up (HIGH gap)
- [ ] Backend `/api/auth/login` + real User model built (CRITICAL gap — else Section 12 and ALL access controls are fake)
- [ ] Additional Grafana dashboards created (backend latency, PG, safeguarding, ETL)
- [ ] Runbook reviewed by backup on-call (do a P0 tabletop exercise: "Pretend portal is down. Show me what you do.")
