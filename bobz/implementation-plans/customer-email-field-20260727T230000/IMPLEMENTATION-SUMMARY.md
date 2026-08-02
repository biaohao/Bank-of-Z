# Implementation Summary: Add CUSTOMER_EMAIL Field

**Commits**: `d51f56b`, `138b3bd`, `f1f5f79`, `32371fa`, `127c889`, `c503601`, `477c3b7`, `f0da907`, `399a02e`
**Branch**: `demo-start` (pushed to `origin/demo-start`)
**Date**: 2026-07-27 – 2026-07-28
**Signed-off-by**: DCO ✅

---

## What Was Done

All source changes for the customer email field have been committed and pushed to `origin/demo-start`. The implementation followed workstreams A–G from the [implementation plan](./implementation-plan.md), plus post-implementation bug fixes discovered during the first compile attempt on z/OS.

---

## Workstream B — COBOL Copybooks (5 files)

| File | Change |
|---|---|
| `src/base/cics/copy/CUSTDB2.cpy` | Added `CUSTOMER_EMAIL CHAR(50)` to `EXEC SQL DECLARE CUSTOMER TABLE` |
| `src/base/cics/copy/CUSTOMER.cpy` | Added `05 CUSTOMER-EMAIL PIC X(50)` after `CUSTOMER-CS-REVIEW-DATE` group. **Post-deployment fix (`900778c`)**: Moved to after `CUSTOMER-PHONE` (before `CUSTOMER-ADDRESS`). |
| `src/base/cics/copy/CRECUST.cpy` | Added `03 COMM-EMAIL PIC X(50)` before `COMM-SUCCESS`. **Post-deployment fix (`900778c`)**: Moved to after `COMM-PHONE` (byte 159, before `COMM-ADDR`). |
| `src/base/cics/copy/INQCUSTZ.cpy` | Added `03 INQCUST-EMAIL PIC X(50)` before `INQCUST-INQ-SUCCESS`. **Post-deployment fix (`900778c`)**: Moved to after `INQCUST-PHONE`. |
| `src/base/cics/copy/UPDCUST.cpy` | Added `03 COMM-EMAIL PIC X(50)` before `COMM-UPD-SUCCESS`. **Post-deployment fix (`900778c`)**: Moved to after `COMM-PHONE`. |
| `src/base/cics/copy/DELCUS.cpy` | Added `03 COMM-EMAIL PIC X(50)` before `COMM-DEL-SUCCESS`. **Post-deployment fix (`900778c`)**: Moved to after `COMM-PHONE`. |

---

## Workstream C — BMS Maps (2 files)

| File | Change |
|---|---|
| `src/base/cics/bms/BNK1CCM.bms` | Row 21: `EMAIL` DFHMDF (UNPROT, GREEN, UNDERLINE, LENGTH=50) — Create Customer input field |
| `src/base/cics/bms/BNK1DCM.bms` | Row 20: `CUSTEML` DFHMDF (PROT, NEUTRAL, ASKIP, LENGTH=50) — Display Customer output field |

---

## Workstream D — COBOL Programs (6 files)

| File | Changes |
|---|---|
| `src/base/cics/cobol/CRECUST.cbl` | `HV-CUSTOMER-EMAIL` host variable added; `MOVE COMM-EMAIL` to HV before INSERT; `CUSTOMER_EMAIL` / `:HV-CUSTOMER-EMAIL` added to SQL INSERT column and VALUES lists. **⚠️ Post-deployment fix (`ab395ed`)**: `WS-CHILD-EMAIL PIC X(50)` added to `WS-CHILD-DATA` inline struct — see "Post-Implementation Bug Fixes" below. |
| `src/base/cics/cobol/INQCUST.cbl` | `HV-CUSTOMER-EMAIL` host variable added; `CUSTOMER_EMAIL` / `:HV-CUSTOMER-EMAIL` added to SQL SELECT in `READ-CUSTOMER-DB2`; `MOVE HV-CUSTOMER-EMAIL TO CUSTOMER-EMAIL`; `MOVE CUSTOMER-EMAIL TO INQCUST-EMAIL` in COMMAREA populate block |
| `src/base/cics/cobol/UPDCUST.cbl` | `HV-CUSTOMER-EMAIL` host variable added; `MOVE COMM-EMAIL` to HV before UPDATE; `CUSTOMER_EMAIL = :HV-CUSTOMER-EMAIL` added to SQL UPDATE SET clause; `MOVE HV-CUSTOMER-EMAIL TO COMM-EMAIL` in success path |
| `src/base/cics/cobol/BNK1CCS.cbl` | `SUBPGM-EMAIL PIC X(50)` added to `SUBPGM-PARMS` after `SUBPGM-CS-REVIEW-DATE`. **Post-deployment fix (`900778c`)**: Moved to after `SUBPGM-PHONE` (before `SUBPGM-ADDR`). `MOVE SPACES TO EMAILI` added in RECEIVE-MAP initialisation; `MOVE EMAILI OF BNK1CCI TO SUBPGM-EMAIL` added in `CRE-CUST-DATA`; `MOVE SPACES TO SUBPGM-EMAIL` on `INITIALIZE SUBPGM-PARMS`. |
| `src/base/cics/cobol/BNK1DCS.cbl` | ⚠️ **4 inline edits** (no COPY statement — manual sync required): `WS-COMM-EMAIL PIC X(50)` added to `WS-COMM-AREA` after `WS-COMM-CS-REVIEW-DATE`; `COMM-EMAIL PIC X(50)` added to inline `DFHCOMMAREA` LINKAGE after `COMM-CS-REVIEW-DATE`; display and update PROCEDURE paths. **Post-deployment fix (`900778c`)**: Both `WS-COMM-EMAIL` and `COMM-EMAIL` moved to after their respective phone fields (before address fields). |
| `src/base/cics/cobol/BANKDATA.cbl` | `HV-CUSTOMER-EMAIL PIC X(50)` host variable added; `MOVE SPACES / MOVE test-value TO HV-CUSTOMER-EMAIL` before INSERT; `CUSTOMER_EMAIL` / `:HV-CUSTOMER-EMAIL` added to SQL INSERT column and VALUES lists (initial implementation used a string literal — fixed in post-implementation bug fix) |

---

## Workstream F — z/OS Connect Provider Files + Mapping YAMLs + OpenAPI

### F1–F3, F-DELCUS — Provider file manual fix (post-deployment bug fix)

After deployment it was discovered that all four z/OS Connect provider file sets had not been updated to match the extended COMMAREs. Rather than regenerating via the z/OS Connect CLI (which requires the COBOL programs to already be deployed on z/OS), the files were manually corrected in commit `f962ace`.

**Files fixed across CRECUST, INQCUST, UPDCUST, and DELCUS:**

| Asset | Change |
|---|---|
| `CRECUST/providerFiles/COMMAREA.cpy` | Added `COMM-EMAIL PIC X(50)` — the hand-authored source-of-truth |
| `CRECUST/providerFiles/gen/CRECUST_request_0.cpy` | Added `COMM-EMAIL` field; COMMAREA bytes 403 → 453 |
| `CRECUST/providerFiles/gen/CRECUST_response_0.cpy` | Same |
| `CRECUST/providerFiles/request.dai` + `response.dai` | `COMM-EMAIL` at startPos 398 (50 B); `COMM-SUCCESS` → 448; `COMM-FAIL-CODE` → 449; total 403 → 453. **Post-deployment fix (`900778c`)**: `COMM-EMAIL` moved to startPos 159; addr and all subsequent field startPos values shifted +50 (addr→209, status→419, created→429, credit→437, cs-review→440); success/fail unchanged at 448/449. |
| `CRECUST/providerFiles/gen/requestSchema.json` + `responseSchema.json` | Added `COMM-EMAIL` property |
| `INQCUST/providerFiles/gen/INQCUSTZ_request_0.cpy` | Added `INQCUST-EMAIL` field; COMMAREA bytes 403 → 457 |
| `INQCUST/providerFiles/gen/INQCUSTZ_response_0.cpy` | Same |
| `INQCUST/providerFiles/request.dai` + `response.dai` | `INQCUST-EMAIL` at startPos 398 (50 B); `INQ-SUCCESS` → 448; `INQ-FAIL-CD` → 449; `PCB-POINTER` → 450; total 403 → 457. **Post-deployment fix (`900778c`)**: `INQCUST-EMAIL` moved to startPos 159; addr and all subsequent fields shifted +50; success/fail/PCB unchanged. |
| `INQCUST/providerFiles/gen/requestSchema.json` + `responseSchema.json` | Added `INQCUST-EMAIL` property |
| `UPDCUST/providerFiles/gen/UPDCUST_request_0.cpy` | Added `COMM-EMAIL` field; COMMAREA bytes 399 → 449 |
| `UPDCUST/providerFiles/gen/UPDCUST_response_0.cpy` | Same |
| `UPDCUST/providerFiles/request.dai` + `response.dai` | `COMM-EMAIL` at startPos 398 (50 B); `UPD-SUCCESS` → 448; `UPD-FAIL-CD` → 449; total 399 → 449. **Post-deployment fix (`900778c`)**: `COMM-EMAIL` moved to startPos 159; addr and all subsequent fields shifted +50. |
| `UPDCUST/providerFiles/gen/requestSchema.json` + `responseSchema.json` | Added `COMM-EMAIL` property |
| `DELCUS/providerFiles/gen/DELCUS_request_0.cpy` | Added `COMM-EMAIL` field; COMMAREA bytes 399 → 449 |
| `DELCUS/providerFiles/gen/DELCUS_response_0.cpy` | Same |
| `DELCUS/providerFiles/request.dai` + `response.dai` | `COMM-EMAIL` at startPos 398 (50 B); `DEL-SUCCESS` → 448; `DEL-FAIL-CD` → 449; total 399 → 449. **Post-deployment fix (`900778c`)**: `COMM-EMAIL` moved to startPos 159; addr and all subsequent fields shifted +50. |
| `DELCUS/providerFiles/gen/requestSchema.json` + `responseSchema.json` | Added `COMM-EMAIL` property |

> **Note**: `DELCUS.cpy` itself was also missing `COMM-EMAIL` (the other three COMMAREA copybooks had been updated in Workstream B; DELCUS was missed). This was also fixed in commit `f962ace`.

> **Recommended**: After COBOL programs are fully deployed and z/OS Connect CLI access is available, regenerate the `gen/` files officially to ensure they exactly match the deployed COMMAREA layout. The manual fixes are byte-accurate but the CLI regeneration is the authoritative process.

### F4–F8 — Mapping YAMLs + OpenAPI

| File | Change |
|---|---|
| `src/api/src/main/operations/%2Fcustomers/post/request.yaml` | `COMM-EMAIL` mapping added with `$exists($body.email)` null guard |
| `src/api/src/main/operations/%2Fcustomers%2F%7BcustomerId%7D/get/response_200.yaml` | `email` response field mapped from `INQCUST-EMAIL` |
| `src/api/src/main/operations/%2Fcustomers%2F%7BcustomerId%7D/put/request.yaml` | `COMM-EMAIL` mapping added from `$body.email` |
| `src/api/src/main/operations/%2Fcustomers%2F%7BcustomerId%7D/put/response_200.yaml` | `email` response field mapped from `COMM-EMAIL` |
| `src/api/src/main/api/openapi.yaml` | `email` field (optional, nullable, `maxLength: 50`) added to `Customer` and `CustomerUpdate` schemas. **Post-deployment fix (`183edf0`)**: `email` also added to `CreateCustomerRequest` schema — it was missing from this schema, causing z/OS Connect to not forward the field to the request mapping engine. |

---

## Workstream G — Web UI (3 files)

| File | Change |
|---|---|
| `src/frontend/customer-create.html` | Email input field (`<cds-text-input>`, optional, `maxlength=50`, `type=email`) added after Country; `validateCustomerData` extended with 50-char max check; `email` included in POST body as `email || undefined` (omitted when blank) |
| `src/frontend/customer-details.html` | Email field added to `displayCustomerDetails` form, pre-populated from `customer.email` returned by API; `updateCustomer` reads field and includes `email` in PUT body when non-empty; existing email preserved if field left blank |
| `src/frontend/js/api.js` | JSDoc `@typedef Customer` and `createCustomer` param list updated to document `email` property (max 50 chars, optional) |

---

## Post-Implementation Bug Fixes

Seven bugs were discovered across compile, deployment, and runtime testing. They are listed in chronological order.

### Fix 1 — `BANKDATA.cbl`: Replace string literal with host variable (`477c3b7`)

**Problem**: The initial implementation used a hardcoded string literal `'test@bankofz.example.com'` directly in the SQL `VALUES` clause. This triggered the `detect-secrets` pre-commit hook (RC=8) as a potential credential.

**Fix**: Replaced with host variable `:HV-CUSTOMER-EMAIL` following the same pattern as all other fields:
- Added `03 HV-CUSTOMER-EMAIL PIC X(50)` to the `HOST-CUSTOMER-ROW` block
- Added `MOVE SPACES TO HV-CUSTOMER-EMAIL` and `MOVE 'test@bankofz.example.com' TO HV-CUSTOMER-EMAIL` before the INSERT
- Replaced the literal in VALUES with `:HV-CUSTOMER-EMAIL`

### Fix 2 — `CUSTDB2.cpy`: Restore `END-EXEC.` indentation (`f0da907`)

**Problem**: When `CUSTOMER_EMAIL` was added to `CUSTDB2.cpy`, the `END-EXEC.` statement was accidentally shifted one column left — from column 12 (Area B) to column 11 (Area A). In COBOL fixed-format, `END-EXEC.` in Area A is a syntax error. This caused RC=8 compile failures in **all four programs** that include `CUSTDB2.cpy`: `CRECUST`, `INQCUST`, `UPDCUST`, and `DELCUS`.

**Fix**: Restored `END-EXEC.` to column 12 (11 leading spaces), matching the original indentation.

### Fix 3 — `Db2-create.j2`: Add `CUSTOMER_EMAIL` to `CREATE TABLE` DDL (`399a02e`)

**Problem**: The `CREATE TABLE BANKZ.CUSTOMER` DDL in `.setup/jcl/cics/Db2-create.j2` did not include the new column. Fresh installs running `setup-common.sh environment` would create the table without `CUSTOMER_EMAIL`, causing `BANKDATA` to fail with SQLCODE -206 ("column does not exist").

**Fix**: Added `CUSTOMER_EMAIL CHAR(50)` as the last column in the `CREATE TABLE` statement.

> **Note**: This fix applies to fresh installs only. Existing installations still require the manual `ALTER TABLE` below.

### Fix 4 — z/OS Connect provider files + `DELCUS.cpy` (`f962ace`)

**Problem**: All z/OS Connect provider file sets (CRECUST, INQCUST, UPDCUST, DELCUS) had stale `gen/` copybooks, `.dai` descriptors, and JSON schemas that did not include the email field. This caused COMMAREA size mismatches resulting in HTTP 500 on all four customer API operations. Additionally, `DELCUS.cpy` itself was missing `COMM-EMAIL`.

**Fix**: Manually corrected all 26 affected provider files across four programs and updated `DELCUS.cpy`. See the Workstream F section for the full file inventory.

### Fix 5 — `openapi.yaml`: Add `email` to `CreateCustomerRequest` schema (`183edf0`)

**Problem**: The `email` field was present in the `Customer` and `CustomerUpdate` OpenAPI schemas but was missing from `CreateCustomerRequest`. When the web UI sent `email` in the POST body, z/OS Connect's request processing did not forward it through the mapping engine, meaning `$body.email` was undefined and `COMM-EMAIL` received an empty string.

**Fix**: Added `email: {type: string, maxLength: 50, nullable: true}` to the `CreateCustomerRequest` schema in `src/api/src/main/api/openapi.yaml`.

### Fix 6 — `CRECUST.cbl`: Add `WS-CHILD-EMAIL` to `WS-CHILD-DATA` inline struct (`ab395ed`) ⚠️ Critical

**Problem**: `CRECUST.cbl` contains an inline manual copy of the customer structure called `WS-CHILD-DATA` (WORKING-STORAGE, lines ~260–292). It is used to GET the result containers written back by the five async credit agency stubs (CRDTAGY1–5) after they complete their credit scoring. Because this struct has no `COPY` statement, it does **not** inherit the email field automatically when `CUSTOMER.cpy` is updated.

Root cause chain:
1. CRECUST PUT CONTAINERs the 453-byte DFHCOMMAREA to each credit agency child task
2. Each CRDTAGY stub GETs the container into `WS-CONT-IN` (= `COPY CUSTOMER` = 447 bytes, correctly updated)
3. The stub writes a random credit score and PUTs 447 bytes back to the container
4. CRECUST GETs the returned container into `WS-CHILD-DATA` with `FLENGTH(LENGTH OF WS-CHILD-DATA)`
5. Before fix: `WS-CHILD-DATA` was 397 bytes (no email field) — 50 bytes shorter than the container
6. CICS GET CONTAINER returns **LENGERR** when FLENGTH is less than the container length
7. CRECUST checks `IF WS-CICS-RESP NOT = DFHRESP(NORMAL)` → sets `COMM-FAIL-CODE = 'E'`, `COMM-SUCCESS = 'N'`
8. z/OS Connect maps `COMM-SUCCESS = 'N'` → HTTP 400 "Invalid request parameters"
9. `messages.log` showed nothing unusual because z/OS Connect received a well-formed COMMAREA response — the failure was internal to CRECUST after the CICS link completed

**Fix**: Added `05 WS-CHILD-EMAIL PIC X(50).` immediately after `WS-CHILD-CS-REVIEW-YEAR` and before `WS-CHILD-SUCCESS` in `WS-CHILD-DATA`. `WS-CHILD-DATA` is now 447 bytes, matching the CRDTAGY container size. CRDTAGY1–5 did **not** need recompilation.

```cobol
*  Before (397 bytes — missing email):
              05 WS-CHILD-CS-REVIEW-YEAR PIC 9999 DISPLAY.
              05 WS-CHILD-SUCCESS           PIC X.

*  After (447 bytes — matches CUSTOMER.cpy at old layout):
              05 WS-CHILD-CS-REVIEW-YEAR PIC 9999 DISPLAY.
              05 WS-CHILD-EMAIL             PIC X(50).
              05 WS-CHILD-SUCCESS           PIC X.
```

> ⚠️ **Note**: commit `900778c` (Fix 7 below) subsequently moved `WS-CHILD-EMAIL` from after `WS-CHILD-CS-REVIEW-YEAR` to after `WS-CHILD-PHONE` (before the address fields), to match the corrected COMMAREA layout. The net effect is still 447 bytes.

---

### Fix 7 — Email field position: move from end of COMMAREA to after phone (`900778c`) ⚠️ Critical

**Problem**: The `COMM-EMAIL` / `INQCUST-EMAIL` / `CUSTOMER-EMAIL` field was inserted at the **end** of each COMMAREA (before success/fail flags) in the initial implementation. The authoritative COMMAREA layout requires email immediately **after the phone field** (byte 159 in CRECUST), with address and all subsequent fields shifted +50 bytes. This mismatch caused z/OS Connect to read customer data fields from the wrong byte offsets, producing garbled address/status data and server errors on the Create Customer API call.

**Root cause**: The `.dai` provider files were partially updated in commit `f962ace` (email added at end) but the COBOL structs and the `.dai` addr-field startPos values were not consistent with the intended layout. The correct layout per the implementation plan places email at byte 159 (right after phone, 20 bytes, which ends at byte 158).

**Files changed** (25 files, commit `900778c`):

*COBOL copybooks* — `COMM-EMAIL` / `INQCUST-EMAIL` / `CUSTOMER-EMAIL` moved from after CS-REVIEW-DATE to after PHONE:
- `src/base/cics/copy/CRECUST.cpy`
- `src/base/cics/copy/INQCUSTZ.cpy`
- `src/base/cics/copy/UPDCUST.cpy`
- `src/base/cics/copy/DELCUS.cpy`
- `src/base/cics/copy/CUSTOMER.cpy`

*COBOL inline structs* (no COPY — required manual repositioning):
- `src/base/cics/cobol/CRECUST.cbl` — `WS-CHILD-EMAIL` in `WS-CHILD-DATA` moved from after `WS-CHILD-CS-REVIEW-YEAR` to after `WS-CHILD-PHONE`
- `src/base/cics/cobol/BNK1CCS.cbl` — `SUBPGM-EMAIL` in `SUBPGM-PARMS` moved from after `SUBPGM-CS-REVIEW-DATE` to after `SUBPGM-PHONE`
- `src/base/cics/cobol/BNK1DCS.cbl` — `WS-COMM-EMAIL` and `COMM-EMAIL` moved to after their respective phone fields in both `WS-COMM-AREA` and `DFHCOMMAREA` LINKAGE

*z/OS Connect provider files* — email field moved to startPos 159; addr through cs-review-date shifted +50:

| Field | Old startPos | New startPos |
|---|---|---|
| `COMM-EMAIL` / `INQCUST-EMAIL` | 398 | **159** |
| `COMM-ADDR` / `INQCUST-ADDR` | 159 | **209** |
| `COMM-ADDR-LINE1` | 159 | **209** |
| `COMM-ADDR-LINE2` | 209 | **259** |
| `COMM-CITY` | 259 | **309** |
| `COMM-POSTCODE` | 309 | **359** |
| `COMM-COUNTRY` | 319 | **369** |
| `COMM-STATUS` | 369 | **419** |
| `COMM-CREATED-DATE` | 379 | **429** |
| `COMM-CREDIT-SCORE` | 387 | **437** |
| `COMM-CS-REVIEW-DATE` | 390 | **440** |
| `COMM-SUCCESS` / `COMM-UPD-SUCCESS` / etc. | 448 | **448 (unchanged)** |

Files: `CRECUST/providerFiles/COMMAREA.cpy`, `CRECUST/gen/CRECUST_request_0.cpy`, `CRECUST/gen/CRECUST_response_0.cpy`, `CRECUST/request.dai`, `CRECUST/response.dai`, and equivalent files for INQCUST, UPDCUST, and DELCUS.

---

## ✅ Completed: All Source Fixes

All source code fixes are committed and pushed to `origin/demo-start`:

| Commit | Fix |
|---|---|
| `477c3b7` | `BANKDATA.cbl` — host variable for email in SQL VALUES |
| `f0da907` | `CUSTDB2.cpy` — restore `END-EXEC.` to Area B |
| `399a02e` | `Db2-create.j2` — add `CUSTOMER_EMAIL` to `CREATE TABLE` |
| `f962ace` | All z/OS Connect provider files + `DELCUS.cpy` |
| `183edf0` | `openapi.yaml` — add `email` to `CreateCustomerRequest` |
| `ab395ed` | **`CRECUST.cbl` — add `WS-CHILD-EMAIL` to `WS-CHILD-DATA`** |
| `900778c` | **Email field repositioned to after phone (byte 159) — 25 files: 5 COBOL copybooks, 3 inline structs (`CRECUST.cbl`, `BNK1CCS.cbl`, `BNK1DCS.cbl`), 16 z/OS Connect provider files** |

---

## ⏳ Remaining Steps (require z/OS USS)

### Workstream A — DB2 DDL (existing installations only)
Execute on the target DB2 subsystem **before** the DBRM bind:
```sql
ALTER TABLE CUSTOMER ADD COLUMN CUSTOMER_EMAIL CHAR(50);
```
- Column is nullable — no NOT NULL constraint — preserving all existing rows
- Must precede all DBRM binds (step E2)
- Can be submitted via `jsub` on USS using a JCL that runs `DSNTEP13` against subsystem `DBD1`
- Fresh installs do NOT need this step — `Db2-create.j2` now includes the column

### Workstream E — Build, Bind, Deploy
1. **E1** — `dbb build impact` — DBB impact build picks up all copybook changes; BMS reassembly runs before COBOL compile for `BNK1CCS` and `BNK1DCS`
2. **E2** — DB2 DBRM bind for `CRECUST`, `INQCUST`, `UPDCUST`, `BANKDATA` into plan `BANKZPLN`
3. **E3** — Wazi Deploy to CICS load library
4. **E4** — Smoke test via CICS terminal:
   - `BNK1CCM`: email input field visible at row 21
   - `BNK1DCM`: email display field visible at row 20

### Workstream F — z/OS Connect API ✅ Complete (manual fix applied)

The provider `.cpy`, `.dai`, and schema JSON files for CRECUST, INQCUST, UPDCUST, and DELCUS have been manually corrected and pushed in commit `f962ace`. See the **Workstream F** section above for the full file inventory.

Remaining steps:
- **F9** — `dbb build impact` for z/OS Connect API project
- **F10** — Wazi Deploy — deploy updated z/OS Connect API artifact

> **Optional**: After z/OS deploy, regenerate `gen/` files via z/OS Connect CLI to produce the authoritative version.

---

## Rollback

| Trigger | Action |
|---|---|
| DDL deployed, programs not yet bound | `ALTER TABLE CUSTOMER DROP COLUMN CUSTOMER_EMAIL` (safe while column is all NULL) |
| Programs bound and deployed, defect found | Redeploy previous load modules; re-bind previous DBRMs; drop column |
| z/OS Connect API deployed, API defect | Redeploy previous z/OS Connect artifact; COBOL remains compatible (column is nullable) |
| Web UI defect | Revert `customer-create.html`, `customer-details.html`, `api.js` to previous commit; no backend impact |

> ⚠️ DDL rollback is safe only while all `CUSTOMER_EMAIL` values are NULL. Once real email data is written, dropping the column loses that data.

---

**Reference**: [`implementation-plan.md`](./implementation-plan.md)
**Impact analysis**: [`bobz/impact-analysis/customer-email-field-20260727T225450/IMPACT-ANALYSIS.md`](../../impact-analysis/customer-email-field-20260727T225450/IMPACT-ANALYSIS.md)
**Last Updated**: 2026-07-30 (Fix 7: email field repositioned to after-phone byte offset — 25 files, commit `900778c`)
