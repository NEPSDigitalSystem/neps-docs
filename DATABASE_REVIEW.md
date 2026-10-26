# PostgreSQL Database Review - NEPS Digital System

**Review Date:** October 19, 2026  
**Reviewer:** Damien Nsoh Ayine (DevOps Engineer / Project Manager)  
**Scope:** Database setup, connections, data ingestion, schema alignment, scoring validation

---

## Executive Summary

### Overall Status: ⚠️ PARTIALLY ALIGNED

| Component | Status | Alignment with V3.1 Handover | Notes |
|-----------|--------|----------------------------|-------|
| **PostgreSQL Setup** | ✅ Configured | N/A | Docker Compose configured, but services not running |
| **Backend Connection** | ✅ Configured | N/A | Connection configured, but services not running |
| **Data Platform Connection** | ✅ Configured | N/A | Connection configured, but services not running |
| **Mock REDCap Ingestion** | ⚠️ Partial | ❌ NO | Ingests mock data, but NOT V3.1 baseline |
| **Schema Alignment** | ❌ NO | ❌ NO | Schema uses mock fields, NOT V3.1 223 fields |
| **Scoring Implementation** | ✅ Exists | ✅ YES | V3.1 scoring logic exists in data platform |
| **Golden Validation** | ❌ NO | ⚠️ Partial | Tests exist but not in CI, mock service has separate implementation |

---

## Detailed Findings

### 1. PostgreSQL Database Setup

#### Configuration Status

**Docker Compose Configuration:** ✅ **CONFIGURED**

From `neps-infrastructure/docker-compose.yml`:

```yaml
postgres:
  image: postgres:15-alpine
  environment:
    POSTGRES_USER_FILE: /run/secrets/postgres_user
    POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
    POSTGRES_DB: neps_db
  ports:
    - "5432:5432"
  volumes:
    - postgres-data:/var/lib/postgresql/data
```

**Services Connecting to PostgreSQL:**
- ✅ **neps-backend** (line 52-53)
- ✅ **neps-data-platform** (line 117)
- ✅ **neps-ml-ai** (line 80)
- ✅ **pgadmin** (line 384)

**Networks:**
- `neps-database` - Internal database network
- `neps-internal` - Internal service communication

**Status:** Configuration is correct. PostgreSQL is set up with proper networking.

**Issue:** Docker services are **NOT currently running** (Docker Desktop not accessible).

---

### 2. Backend Database Connection

#### Configuration Status

**Backend Database Configuration:** ✅ **CONFIGURED**

From `neps-infrastructure/docker-compose.yml`:

```yaml
neps-backend:
  environment:
    - DATABASE_HOST=postgres
    - DATABASE_USER_FILE=/run/secrets/postgres_user
    - DATABASE_PASSWORD_FILE=/run/secrets/postgres_password
    - DATABASE_SSL_MODE=require
  depends_on:
    - postgres
```

**Connection Details:**
- Host: `postgres` (Docker service name)
- Database: `neps_db`
- SSL Mode: `require` (TLS enabled)
- Authentication: Via secrets files

**Status:** Configuration is correct. Backend is configured to connect to PostgreSQL.

**Issue:** Services not running, so connection cannot be tested.

---

### 3. Data Platform Database Connection

#### Configuration Status

**Data Platform Database Configuration:** ✅ **CONFIGURED**

From `neps-infrastructure/docker-compose.yml`:

```yaml
neps-data-platform:
  command: >
    sh -c "export DATABASE_URL=postgresql://neps:$$(cat /run/secrets/postgres_password)@postgres:5432/neps_db?sslmode=require && python -m etl.main migrate && sleep infinity"
  environment:
    - REDCAP_API_URL=${REDCAP_API_URL:-https://mock-redcap-service.onrender.com}
    - REDCAP_MOCK_ENABLED=${REDCAP_MOCK_ENABLED:-true}
    - SOURCE_INTAKE_ENABLED=${SOURCE_INTAKE_ENABLED:-false}
  depends_on:
    - postgres
```

**Connection Details:**
- Database URL: PostgreSQL with SSL
- SSL Mode: `require`
- Authentication: Via secrets files
- Command: Runs migrations on startup

**Status:** Configuration is correct. Data platform is configured to connect to PostgreSQL.

**Issue:** Services not running, so connection cannot be tested.

---

### 4. Mock REDCap Data Ingestion

#### Current Implementation

**Data Source:** Mock REDCap Service (Render)
- URL: `https://mock-redcap-service.onrender.com/api`
- Data: Mock longitudinal data (NOT V3.1 baseline)

**Ingestion Configuration:**
```yaml
REDCAP_MOCK_ENABLED=true
REDCAP_API_URL=https://mock-redcap-service.onrender.com/api
SOURCE_INTAKE_ENABLED=false  # DISABLED
```

**Schema Used for Ingestion:**

From `neps-data-platform/etl/transform/mock_schema.py`:

**Table: `demographics` (16 fields)**
```python
fields=[
    FieldDefinition("record_id", "text"),
    FieldDefinition("country", "text"),
    FieldDefinition("site", "text"),
    FieldDefinition("age", "integer"),
    FieldDefinition("gender", "text"),
    FieldDefinition("employment_status", "text"),
    FieldDefinition("food_security", "text"),
    FieldDefinition("healthcare_access", "text"),
    FieldDefinition("socioeconomic_status", "text"),
    # ... 16 total fields
]
```

**Table: `monthly_self_report` (24 fields)**
```python
fields=[
    FieldDefinition("month", "integer"),
    FieldDefinition("perceived_stress_score", "float"),
    FieldDefinition("mood_score", "float"),
    FieldDefinition("anxiety_score", "float"),
    FieldDefinition("depression_score", "float"),
    FieldDefinition("suicidality_score", "float"),
    # ... 24 total fields
]
```

**Status:** ❌ **NOT ALIGNED WITH V3.1**

**Issues:**
1. Schema uses **mock field names** (e.g., `perceived_stress_score`, `mood_score`)
2. Schema does **NOT include V3.1 baseline fields** (223 fields from handover)
3. Schema has **NO neps31_screening table** (core 73 clinical items)
4. Schema has **NO neps31_registry table** (administrative fields)
5. Schema has **NO neps31_extended_baseline table** (74 fields)
6. Schema has **NO neps31_yeb table** (19 fields)
7. Schema has **NO neps31_ycad table** (17 fields)

**Gap Analysis:**

| V3.1 Requirement | Current Schema | Status |
|------------------|----------------|--------|
| neps31_registry (10 fields) | demographics (16 fields) | ❌ Different structure |
| neps31_screening (73 fields) | NOT PRESENT | ❌ Missing |
| neps31_cohort_interest (5 fields) | NOT PRESENT | ❌ Missing |
| neps31_extended_baseline (74 fields) | NOT PRESENT | ❌ Missing |
| neps31_yeb (19 fields) | NOT PRESENT | ❌ Missing |
| neps31_ycad (17 fields) | NOT PRESENT | ❌ Missing |
| Total: 223 fields | 40 fields (mock) | ❌ 183 fields missing |

---

### 5. Schema Alignment with Master CRF

#### V3.1 Master CRF Requirements

**From Handover Dictionary:**
- **Total fields:** 223
- **Total questions:** 188
- **Forms:** 6 (registry, screening, cohort_interest, extended_baseline, yeb, ycad)
- **Core 73 items:** All present in neps31_screening

**Current Database Schema:**

From migration `002_independent_source_intake.py`:

```python
TABLES = [
    'nds_v31_intake_raw',      # Source ingestion table
    'nds_v31_intake_audit',     # Audit trail
    'nds_v31_intake_head',      # Head/identity
    'nds_v31_intake_pending',   # Queue
    'nds_v31_participant_links', # ID mapping
    'nds_v31_source',           # Source data
    'nds_v31_audit',            # Audit
    'nds_v31_derived',          # Derived scores
    'nds_v31_validated'        # Validated data
]
```

**Issue:** The migration creates **intake tables** but does **NOT** define the **final normalized tables** that match the V3.1 CRF structure.

**Missing Normalized Tables:**
- ❌ `baseline_screening` table (223 fields)
- ❌ `neps31_registry` table
- ❌ `neps31_screening` table
- ❌ `neps31_cohort_interest` table
- ❌ `neps31_extended_baseline` table
- ❌ `neps31_yeb` table
- ❌ `neps31_ycad` table

**Status:** ❌ **NOT ALIGNED**

The database schema does **NOT** match the V3.1 Master CRF structure. It uses:
- Mock field names (from original assumptions)
- Mock schema (40 fields instead of 223)
- Missing V3.1 forms entirely

---

### 6. Scoring Implementation

#### Scoring Logic Status

**V3.1 Scoring Implementation:** ✅ **EXISTS**

**Location:** `neps-data-platform/etl/scoring/v31.py`

**Implementation Details:**
```python
ALGORITHM = 'NEPS-V3.1-provisional-corrected-1;complete-items-only;equal-dimensions'

def composite(epi, esi, sbi, mhdi, pfi):
    """Only arithmetic, never a recruitment criterion."""
    values = [Fraction(str(v)) for v in (epi, esi, sbi, mhdi, pfi)]
    if any(v < 0 or v > 100 for v in values):
        raise ValueError('Component score outside 0–100')
    a, b, c, d, e = values
    return (30*a + 25*b + 20*c + 20*d + 5*(100-e))/100
```

**Formula Check:** ✅ **CORRECT**
- Weighted formula matches handover specification
- EPI: 30%, ESI: 25%, SBI: 20%, MHDI: 20%, PFI: 5%
- Complete-items-only: Missingness returns NULL
- Fractions preserve precision

**Subscale Scoring:**
- ✅ EPI (Educational Pressure Index)
- ✅ ESI (Economic Stress Index)
- ✅ SBI (Stigma Burden Index)
- ✅ K6/MHDI (Psychological Distress)
- ✅ BRS (Brief Resilience Scale)
- ✅ MSPSS (Perceived Social Support)
- ✅ GSE (General Self-Efficacy)
- ✅ PFI (Protective Factors Index)

**BRS Reverse Scoring:**
- ✅ Items 02, 04, 06 are reverse-scored
- ✅ Formula: 6 - original_score

**Status:** ✅ **SCORING LOGIC IS ACCURATE**

The scoring implementation in the data platform **DOES** match the handover requirements.

---

### 7. Golden Scoring Validation

#### CI Test Status

**Golden Fixtures:** ✅ **EXIST**

**Location:** `NDS_Team/review/SCORING_GOLDEN.json`

**CI Tests in Data Platform:**
- ✅ `tests/test_v31_scoring.py` - Scoring tests exist
- ✅ `tests/test_v31_scoring_qa.py` - QA scoring tests exist

**CI Tests in Mock Service:**
- ✅ `mock-redcap-service/tests/test_scoring_golden.py` - 7 tests passing

**Test Coverage:**
- Composite calculation
- Subscale calculations
- BRS reverse scoring
- Missingness handling
- Field count validation
- Valid range validation

**Status:** ⚠️ **PARTIALLY ALIGNED**

**Issues:**
1. Data platform tests exist but **may not be in CI/CD pipeline**
2. Mock service has **separate scoring implementation** (`scoring_v31_standalone.py`)
3. Need to verify data platform scoring matches mock service scoring
4. Need to verify both match handover `v31.py`

**Alignment Check Required:**
- [ ] Data platform `v31.py` vs Handover `v31.py` - Are they identical?
- [ ] Mock service `scoring_v31_standalone.py` vs Handover `v31.py` - Are they identical?
- [ ] CI tests running on every commit?
- [ ] All tests passing?

---

## Summary of Gaps

### Critical Gaps

| Gap | Severity | Impact | Fix Required |
|-----|----------|--------|-------------|
| **Database schema does not match V3.1** | HIGH | Data platform cannot ingest V3.1 data | Create normalized tables matching 223-field structure |
| **V3.1 baseline tables missing** | HIGH | Cannot store V3.1 screening data | Add migration for V3.1 tables |
| **Mock schema uses wrong field names** | HIGH | Cannot align with handover | Update mock_schema.py to use V3.1 field names |
| **V3.1 baseline not ingested** | HIGH | No V3.1 data in database | Configure ingestion from mock V3.1 endpoint |
| **Services not running** | MEDIUM | Cannot test connections | Start Docker services |

### Scoring Status

| Component | Status | Notes |
|-----------|--------|-------|
| **Scoring logic** | ✅ Accurate | Matches handover formula |
| **BRS reverse scoring** | ✅ Correct | Items 02, 04, 06 handled |
| **Missingness handling** | ✅ Correct | Complete-items-only, NULL for missing |
| **Golden validation** | ⚠️ Partial | Tests exist but CI status unknown |
| **Two implementations** | ⚠️ Risk | Data platform + mock service (may diverge) |

---

## Recommendations

### Immediate Actions (Before Meeting)

1. **Start Docker Services**
   ```bash
   cd neps-infrastructure
   docker-compose up -d postgres
   ```

2. **Verify Database Connectivity**
   - Test backend → PostgreSQL connection
   - Test data platform → PostgreSQL connection

3. **Check Database Schema**
   - Connect to PostgreSQL (port 5432)
   - List tables: `\dt`
   - Verify mock schema vs. V3.1 requirements

4. **Verify Scoring Alignment**
   - Compare data platform `v31.py` with handover `v31.py`
   - Compare mock service `scoring_v31_standalone.py` with handover `v31.py`
   - Ensure all three are identical

### Short-Term Actions (Post-Meeting)

#### Data Platform Priority 1: Create V3.1 Schema

**Required Changes:**

1. **Create migration 003_v31_baseline_tables.py**
   ```python
   # Add V3.1 baseline screening table
   CREATE TABLE baseline_screening (
       study_id VARCHAR(50) PRIMARY KEY,
       country VARCHAR(50) NOT NULL,
       site_id VARCHAR(100) NOT NULL,
       site_setting VARCHAR(50),
       recruit_src VARCHAR(100),
       screen_date DATE NOT NULL,
       crf_version VARCHAR(20) NOT NULL,
       form_lang VARCHAR(50),
       age_years INTEGER CHECK (age_years BETWEEN 12 AND 24),
       sex_report VARCHAR(10) CHECK (sex_report IN ('1', '2', '99')),
       # ... 215 more fields
       nses_composite_score DECIMAL(5,2),
       epi_score DECIMAL(5,2),
       esi_score DECIMAL(5,2),
       sbi_score DECIMAL(5,2),
       k6_total DECIMAL(5,2),
       brs_score DECIMAL(5,2),
       mspss_score DECIMAL(5,2),
       gse_score DECIMAL(5,2),
       pfi_score DECIMAL(5,2),
       _scoring_version VARCHAR(100),
       _scored_at TIMESTAMP,
       _ingested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

2. **Update mock_schema.py to use V3.1 field names**
   - Replace mock field names with V3.1 field names
   - Add neps31_screening instrument
   - Add neps31_registry instrument
   - Add other V3.1 forms

#### Data Platform Priority 2: Configure V3.1 Ingestion

**Required Changes:**

1. **Create V3.1 ingestion pipeline**
   - Source: `/api/v31/baseline` from mock service
   - Validate against `data_contract_v31.json`
   - Ingest all 223 fields
   - Apply scoring using `v31.py`
   - Store in `baseline_screening` table

2. **Set up CI tests for scoring**
   - Add `SCORING_GOLDEN.json` to data platform
   - Create `tests/test_scoring_golden.py`
   - Add to GitHub Actions CI
   - Run on every commit

#### Backend Priority 1: Add V3.1 Endpoints

**Required Changes:**

1. **Implement V3.1 baseline endpoints**
   - `GET /api/v31/baseline`
   - `GET /api/v31/participants`
   - `POST /api/v31/baseline`
   - Connect to `baseline_screening` table

#### Frontend Priority 1: Update Baseline UI

**Required Changes:**

1. **Update baseline screening form**
   - Use 6 V3.1 forms structure
   - Use 223 field names from dictionary
   - Implement branching logic

2. **Create baseline score dashboard**
   - Display NSES composite
   - Display subscale scores
   - Use V3.1 thresholds

---

## Conclusion

### Current State

- ✅ **PostgreSQL:** Configured correctly in Docker Compose
- ✅ **Connections:** Backend and data platform configured to connect
- ❌ **Data Ingestion:** Mock data ingested, but NOT V3.1 baseline
- ❌ **Schema Alignment:** Schema uses mock fields, NOT V3.1 223 fields
- ✅ **Scoring Logic:** Accurate (matches handover)
- ⚠️ **Golden Validation:** Tests exist but CI status unknown

### What's Needed to Meet Handover Milestone

1. **Create V3.1 database schema** (223 fields, 6 forms)
2. **Update mock schema** to use V3.1 field names
3. **Configure V3.1 ingestion** from mock service `/api/v31/baseline`
4. **Verify scoring implementations** (data platform vs mock service vs handover)
5. **Set up CI tests** for golden validation
6. **Update backend** to serve V3.1 endpoints
7. **Update frontend** to display V3.1 data

### Timeline Estimate

- **Database schema creation:** 2-3 days
- **Ingestion pipeline update:** 2-3 days
- **Scoring verification:** 1 day
- **Backend endpoints:** 2-3 days
- **Frontend UI updates:** 3-5 days
- **Total:** 10-15 days

### Risk Level

**Current Risk:** **HIGH**

The database schema is fundamentally misaligned with the V3.1 handover. If we proceed without fixing this, we will:
- Be unable to ingest real V3.1 data when it arrives
- Have to rebuild database schema later (costly)
- Create technical debt that will be hard to resolve

**Recommendation:** **Fix schema alignment BEFORE ingesting any V3.1 data.**

---

**End of Review**

**Next Steps:**
1. Present this review in meeting
2. Get team agreement on timeline
3. Prioritize database schema alignment
4. Start with Data Platform V3.1 schema creation
