# NEPS Digital System - Pipeline Testing Guide

This guide provides step-by-step commands to verify the entire data pipeline:
Mock REDCap → Data Platform → PostgreSQL → Backend → Frontend

And verify V3.1 schema alignment with handover requirements.

## Prerequisites

- Docker Desktop running
- All services started: `docker-compose up -d`
- 150 mock records ingested (should already be done)

---

## Test 1: Verify Docker Services Running

Check all services are healthy:

```bash
cd neps-infrastructure
docker-compose ps
```

**Expected output:**
- `neps-infrastructure-postgres-1` - Running
- `neps-infrastructure-neps-data-platform-1` - Running
- `neps-infrastructure-neps-backend-1` - Running
- `neps-infrastructure-redis-1` - Running

---

## Test 2: Verify Mock REDCap Service

Test the hosted mock REDCap on Render:

```bash
curl https://mock-redcap-service.onrender.com/api/v31/baseline | python -c "import sys, json; d=json.load(sys.stdin); print(f\"Records: {len(d['data'])}\"); print(f\"Fields per record: {len(d['data'][0])}\")"
```

**Expected output:**
- Records: 150
- Fields per record: ~232 (223 contract + 9 metadata)

---

## Test 3: Verify Data Platform Ingestion

Check data platform logs for successful ingestion:

```bash
docker-compose logs neps-data-platform | grep -i "v31"
```

**Expected output:**
- `"event": "starting_v31_ingestion"`
- `"count": 150, "event": "fetched_v31_data"`
- `"count": 150, "event": "inserted_v31_source"`
- `"count": 150, "event": "applied_scoring"`
- `"status": "success"`

---

## Test 4: Verify PostgreSQL Data

### 4.1 Check table counts

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT 
  (SELECT COUNT(*) FROM nds_v31_source) as source_count,
  (SELECT COUNT(*) FROM nds_v31_derived) as derived_count,
  (SELECT COUNT(DISTINCT country) FROM nds_v31_source) as countries;
"
```

**Expected output:**
- source_count: 150
- derived_count: 150
- countries: 3 (Ghana, Tanzania, Sierra Leone)

### 4.2 Check source table structure

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "\d nds_v31_source"
```

**Expected columns:**
- id
- identity_key
- identity
- revision
- change_hash
- source_version
- contract_version
- payload (JSONB)
- country
- observed_at
- contract_hash
- storage_destination
- nds_participant_id
- intake_revision_id
- metadata

### 4.3 Check derived table structure

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "\d nds_v31_derived"
```

**Expected columns:**
- id
- source_id
- calculation_version
- result (JSONB)

### 4.4 Inspect a sample record

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT 
  id,
  country,
  source_version,
  contract_version,
  jsonb_object_keys(payload) as field_count
FROM nds_v31_source 
LIMIT 1;
"
```

**Expected output:**
- field_count should show 200+ keys (the 223 V3.1 fields)

### 4.5 Inspect sample scoring

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT result FROM nds_v31_derived LIMIT 1;
"
```

**Expected output:**
- JSON with EPI, ESI, SBI, MHDI, K6_total, BRS_100, MSPSS_100, GSE_100, PFI, NSES

---

## Test 5: Verify V3.1 Schema Alignment

### 5.1 Extract field names from database

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT jsonb_object_keys(payload) 
FROM nds_v31_source 
LIMIT 1;
" > /tmp/db_fields.txt
cat /tmp/db_fields.txt
```

### 5.2 Compare with handover contract

```bash
cd neps-data-platform
python -c "
import json
with open('etl/contracts/data_contract_v31.json') as f:
    contract = json.load(f)
fields = list(contract.get('fields', {}).keys())
print(f'Contract fields: {len(fields)}')
print('Sample fields:', fields[:10])
"
```

### 5.3 Verify specific V3.1 fields exist

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT 
  payload->>'study_id' as study_id,
  payload->>'age_years' as age_years,
  payload->>'sex_report' as sex_report,
  payload->>'country' as country,
  payload->>'site_id' as site_id,
  payload->>'epi_expectations_1' as epi_1,
  payload->>'k6_1' as k6_1,
  payload->>'brs_01' as brs_01
FROM nds_v31_source 
LIMIT 1;
"
```

**Expected output:**
- All fields should have values (not NULL)

### 5.4 Verify form structure

Check that the 6 V3.1 forms are represented:

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT 
  COUNT(DISTINCT CASE WHEN payload->>'screen_date' IS NOT NULL THEN 1 END) as has_screening,
  COUNT(DISTINCT CASE WHEN payload->>'cohort_interest' IS NOT NULL THEN 1 END) as has_cohort,
  COUNT(DISTINCT CASE WHEN payload->>'epi_expectations_1' IS NOT NULL THEN 1 END) as has_extended,
  COUNT(DISTINCT CASE WHEN payload->>'yeb_01' IS NOT NULL THEN 1 END) as has_yeb,
  COUNT(DISTINCT CASE WHEN payload->>'ycad_01' IS NOT NULL THEN 1 END) as has_ycad
FROM nds_v31_source;
"
```

**Expected output:**
- All counts should be 150 (all records have all form fields)

---

## Test 6: Verify Backend API

### 6.1 Test baseline endpoint

```bash
curl -s http://localhost:8000/api/v31/baseline?limit=1 | python -c "
import sys, json
d = json.load(sys.stdin)
print(f\"Status: {d['metadata']['status']}\")
print(f\"Source: {d['metadata']['source']}\")
print(f\"Version: {d['metadata']['version']}\")
print(f\"Records returned: {len(d['data'])}\")
print(f\"Has scores: {'_scores' in d['data'][0]}\")
print(f\"Sample scores: {d['data'][0].get('_scores', {})}\")
"
```

**Expected output:**
- Status: VERIFIED
- Source: postgresql
- Version: 3.1
- Records returned: 1
- Has scores: True
- Sample scores: shows EPI, ESI, SBI, etc.

### 6.2 Test participants endpoint

```bash
curl -s http://localhost:8000/api/v31/participants?limit=1 | python -c "
import sys, json
d = json.load(sys.stdin)
print(f\"Source: {d['metadata']['source']}\")
print(f\"Records: {len(d['data'])}\")
print(f\"Sample: {d['data'][0]}\")
"
```

**Expected output:**
- Source: postgresql
- Records: 1
- Sample shows study_id, age_years, sex_report, country, etc.

### 6.3 Test country filtering

```bash
curl -s "http://localhost:8000/api/v31/baseline?country=Ghana&limit=5" | python -c "
import sys, json
d = json.load(sys.stdin)
print(f\"Records: {len(d['data'])}\")
print(f\"All Ghana: {all(r['country'] == 'Ghana' for r in d['data'])}\")
"
```

**Expected output:**
- Records: 5
- All Ghana: True

### 6.4 Test POST endpoint is disabled

```bash
curl -s -X POST http://localhost:8000/api/v31/baseline -H "Content-Type: application/json" -d '{"test": "data"}'
```

**Expected output:**
- HTTP 501 Not Implemented
- Detail: "Direct V3.1 baseline submission is not supported"

---

## Test 7: Verify Scoring Logic

### 7.1 Check scoring values are within valid ranges

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT 
  AVG((result->>'EPI')::float) as avg_epi,
  AVG((result->>'ESI')::float) as avg_esi,
  AVG((result->>'NSES')::float) as avg_nses,
  COUNT(*) FILTER (WHERE (result->>'EPI')::float BETWEEN 0 AND 100) as valid_epi,
  COUNT(*) FILTER (WHERE (result->>'ESI')::float BETWEEN 0 AND 100) as valid_esi
FROM nds_v31_derived
WHERE (result->>'EPI') IS NOT NULL;
"
```

**Expected output:**
- All averages should be between 0 and 100
- valid_epi and valid_esi should be equal to total count

### 7.2 Verify BRS reverse scoring

Check that BRS_100 is calculated correctly (items 02, 04, 06 reversed):

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT 
  payload->>'brs_01' as brs_01,
  payload->>'brs_02' as brs_02,
  payload->>'brs_03' as brs_03,
  payload->>'brs_04' as brs_04,
  payload->>'brs_05' as brs_05,
  payload->>'brs_06' as brs_06,
  (result->>'BRS_100')::float as brs_100
FROM nds_v31_source s
JOIN nds_v31_derived d ON s.id = d.source_id
WHERE (result->>'BRS_100') IS NOT NULL
LIMIT 5;
"
```

**Expected output:**
- BRS_100 should be a value between 0 and 100
- It should reflect the reverse scoring logic

### 7.3 Verify complete-items-only behavior

Check that missing data results in NULL scores:

```bash
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT 
  COUNT(*) as total,
  COUNT(*) FILTER (WHERE (result->>'EPI') IS NULL) as null_epi,
  COUNT(*) FILTER (WHERE (result->>'ESI') IS NULL) as null_esi,
  COUNT(*) FILTER (WHERE (result->>'NSES') IS NULL) as null_nses
FROM nds_v31_derived;
"
```

**Expected output:**
- Some records should have NULL scores (due to missing items)
- This confirms complete-items-only behavior

---

## Test 8: Verify End-to-End Pipeline

### 8.1 Trace a single record through the pipeline

```bash
# Get a study_id from mock REDCap
STUDY_ID=$(curl -s https://mock-redcap-service.onrender.com/api/v31/baseline | python -c "import sys, json; print(json.load(sys.stdin)['data'][0]['study_id'])")
echo "Study ID: $STUDY_ID"

# Check if it exists in PostgreSQL
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT id, country, source_version 
FROM nds_v31_source 
WHERE payload->>'study_id' = '$STUDY_ID';
"

# Check if it has scores
docker-compose exec postgres psql -U neps -d neps_db -c "
SELECT result 
FROM nds_v31_derived d
JOIN nds_v31_source s ON d.source_id = s.id
WHERE s.payload->>'study_id' = '$STUDY_ID';
"

# Check if backend can return it
curl -s "http://localhost:8000/api/v31/baseline?limit=150" | python -c "
import sys, json
d = json.load(sys.stdin)
record = next((r for r in d['data'] if r['study_id'] == '$STUDY_ID'), None)
if record:
    print(f\"Found in backend: {record['study_id']}\")
    print(f\"Has scores: {'_scores' in record}\")
else:
    print(\"Not found in backend\")
"
```

**Expected output:**
- Record exists in PostgreSQL
- Record has scores in derived table
- Backend returns the record with scores

---

## Test 9: Verify Golden Scoring (Optional)

If you want to verify against the golden fixture:

```bash
cd neps-data-platform
python -c "
import json
from etl.scoring.v31 import Scorer

# Load contract
with open('etl/contracts/data_contract_v31.json') as f:
    contract = json.load(f)

# Load golden fixture
with open('../NDS_Team/review/SCORING_GOLDEN.json') as f:
    golden = json.load(f)

# Test a case
test_case = golden['cases'][0]
if test_case['fixture_kind'] == 'composite_only':
    inputs = test_case['component_inputs']
    expected = test_case['expected']['NSES']
    
    scorer = Scorer(contract)
    # Run scoring with the inputs
    print(f\"Expected NSES: {expected}\")
    print(f\"Test case: {test_case['id']}\")
"
```

---

## Test 10: Verify Data Platform Configuration

Check that data platform is using mock REDCap:

```bash
docker-compose exec neps-data-platform python -c "
from etl.config import get_settings
settings = get_settings()
print(f\"Mock enabled: {settings.redcap_mock_enabled}\")
print(f\"API URL: {settings.redcap_api_url}\")
print(f\"Source intake enabled: {settings.source_intake_enabled}\")
"
```

**Expected output:**
- Mock enabled: True
- API URL: https://mock-redcap-service.onrender.com/api
- Source intake enabled: True

---

## Summary Checklist

Run this quick verification:

```bash
cd neps-infrastructure

# 1. Services running
docker-compose ps

# 2. Database counts
docker-compose exec postgres psql -U neps -d neps_db -c "SELECT COUNT(*) FROM nds_v31_source;"

# 3. Backend API
curl -s http://localhost:8000/api/v31/baseline | python -c "import sys, json; print(json.load(sys.stdin)['metadata'])"

# 4. Scoring exists
docker-compose exec postgres psql -U neps -d neps_db -c "SELECT COUNT(*) FROM nds_v31_derived WHERE result->>'NSES' IS NOT NULL;"
```

**All should pass:**
- ✅ Services: Running
- ✅ Source count: 150
- ✅ Backend metadata: source=postgresql, status=VERIFIED
- ✅ Scoring: Count > 0

---

## Troubleshooting

### Backend returns "Internal Server Error"

Check logs:
```bash
docker-compose logs neps-backend --tail 50
```

### Database is empty

Re-run ingestion:
```bash
docker-compose exec neps-data-platform python -m etl.main ingest-v31
```

### Mock REDCap unreachable

Check Render service:
```bash
curl https://mock-redcap-service.onrender.com/api/v31/baseline
```

### Scores not calculated

Check data platform logs for scoring errors:
```bash
docker-compose logs neps-data-platform | grep -i "score"
```
