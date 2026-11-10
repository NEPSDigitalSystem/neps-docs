# REDCap Data Dictionary V3.2 Integration Record

**Status:** Completed & Integrated  
**Version:** Master CRF V3.2 (Dated 26-OCT-2026)  
**Baseline Reference:** Master CRF V3.1 (223 fields)  
**Updated Field Count:** 227 fields (+4 fields)  

---

## 1. Executive Summary

On October 26, 2026, the NEPS clinical and survey team released the REDCap Data Dictionary Version 3.2 (`NEPSPROJECT_DataDictionary_Version 3.2_Dated26OCT2026.csv`). This update introduces explicit participant consent, automated timestamping, DOB-driven calculated age, form consolidation, and survey-wide gatekeeping branching logic.

This document records the architectural changes and end-to-end integration status across all NEPS subsystems.

---

## 2. Specification of Changes (V3.1 → V3.2)

### 2.1 Added Variables (4 New Fields)
1. **`survey_consent`** (`neps31_screening`, `radio`, required: `y`):
   - Label: *"I consent to take part in this NEPS survey."*
   - Choices: `1, Yes, I consent | 0, No, I do not consent`
   - Gatekeeping: When `survey_consent != 1`, branching logic across the instrument hides respondent screening items.
2. **`tdate`** (`neps31_screening`, `text`, `@HIDDEN @TODAY-UTC`):
   - Label: *"Today's Date: "*
   - Captures UTC survey assessment timestamp for chronological auditing and age computation.
3. **`dob`** (`neps31_screening`, `text`, validation: `date_dmy`, required: `y`):
   - Label: *"Date of Birth: "*
   - Formatted DD-MM-YYYY, used to determine participant eligibility without relying on self-reported whole numbers.
4. **`lblage`** (`neps31_screening`, `descriptive`):
   - Section header / advisory banner regarding age criteria.

### 2.2 Calculated Age
* In V3.1, `age_years` was a manually entered integer text field.
* In V3.2, `age_years` is converted to a REDCap calculated field (`calc`):
  ```redcap
  rounddown(datediff([dob],[tdate], 'y', true), 0)
  ```
  Branching logic: `[survey_consent] = '1' and [dob] <> ''`.

### 2.3 Form Reorganization
* In V3.1, administrative fields (`study_id`, `country`, `site_id`, `site_setting`, `recruit_src`, `screen_date`, `crf_version`, `form_lang`, `age_years`, `sex_report`) belonged to `neps31_registry`.
* In V3.2, all these fields were consolidated under `neps31_screening`.

### 2.4 Universal Gatekeeping Branching Logic
* In V3.2, survey and clinical items throughout the instrument enforce:
  ```redcap
  [age_years] >= 12 and [age_years] <= 24 and [survey_consent] = '1'
  ```
  Nested conditional branching (e.g. trauma timing in YCAD) prepends `[survey_consent] = '1'`.

---

## 3. Subsystem Integration Matrix

| Subsystem / Service | Status | Integration Scope & Verification |
| :--- | :--- | :--- |
| **`mock-redcap-service`** | Verified | Served at `/api/v32/dictionary` and default `/api/v31/dictionary`; backwards-compatibility via `?version=3.1`; synthetic generator synthesizes `survey_consent`, `dob`, `tdate`; test suite passing (10/10). |
| **`neps-backend`** | Verified | Dictionary served with fallback; dynamic `V32_FIELD_COUNT = 227` / `V31_FIELD_COUNT = 223`; participant queries surface `dob`, `tdate`, `survey_consent`; test suite passing (118/118). |
| **`neps-portal`** | Verified | TypeScript types `V31BaselineRecord`, `V31Participant`, and `Participant` updated with `survey_consent`, `dob`, `tdate`, `lblage`; `apiClient.getV31Dictionary("3.2")` supported; type-check and tests passing. |
| **`neps-data-platform`** | Verified | V3.2 reference fixture placed in `tests/fixtures/v31_reference/`; validated with automated fixture tests (`tests/test_v32_reference_fixture.py`); dictionary documentation generated in `docs/DATA_DICTIONARY_V32.md` via `scripts/build_v32_dictionary.py`. |
| **`neps-ml-ai`** | Verified | Baseline dataset `data/raw/v3.1/v31_baseline_dataset.csv` updated to 240 columns with V3.2 fields; test suite passing (23/23). |

---

## 4. Backwards Compatibility Protocol

All API consumers querying `/api/v31/dictionary` receive the latest V3.2 schema by default (227 fields), ensuring new intake forms present consent and DOB fields. Legacy pipelines that require the historical 223-field specification can supply `?version=3.1`, returning the original 18-column V3.1 dictionary without breaking contract guarantees.
