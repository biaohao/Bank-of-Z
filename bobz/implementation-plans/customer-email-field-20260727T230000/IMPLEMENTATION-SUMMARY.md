# Implementation Summary: Add CUSTOMER_EMAIL Field

**Commit**: `27ac33b`  
**Branch**: `main`  
**Date**: 2026-07-27  
**Signed-off-by**: DCO ✅  

---

## What Was Done

All source changes for the customer email field have been committed. The implementation followed workstreams A–F from the [implementation plan](./implementation-plan.md).

---

## Workstream B — COBOL Copybooks (5 files)

| File | Change |
|---|---|
| `src/base/cics/copy/CUSTDB2.cpy` | Added `CUSTOMER_EMAIL CHAR(50)` to `EXEC SQL DECLARE CUSTOMER TABLE` |
| `src/base/cics/copy/CUSTOMER.cpy` | Added `05 CUSTOMER-EMAIL PIC X(50)` after `CUSTOMER-CS-REVIEW-DATE` group |
| `src/base/cics/copy/CRECUST.cpy` | Added `03 COMM-EMAIL PIC X(50)` before `COMM-SUCCESS` |
| `src/base/cics/copy/INQCUSTZ.cpy` | Added `03 INQCUST-EMAIL PIC X(50)` before `INQCUST-INQ-SUCCESS` |
| `src/base/cics/copy/UPDCUST.cpy` | Added `03 COMM-EMAIL PIC X(50)` before `COMM-UPD-SUCCESS` |

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
| `src/base/cics/cobol/CRECUST.cbl` | `HV-CUSTOMER-EMAIL` host variable added; `MOVE COMM-EMAIL` to HV before INSERT; `CUSTOMER_EMAIL` / `:HV-CUSTOMER-EMAIL` added to SQL INSERT column and VALUES lists |
| `src/base/cics/cobol/INQCUST.cbl` | `HV-CUSTOMER-EMAIL` host variable added; `CUSTOMER_EMAIL` / `:HV-CUSTOMER-EMAIL` added to SQL SELECT in `READ-CUSTOMER-DB2`; `MOVE HV-CUSTOMER-EMAIL TO CUSTOMER-EMAIL`; `MOVE CUSTOMER-EMAIL TO INQCUST-EMAIL` in COMMAREA populate block |
| `src/base/cics/cobol/UPDCUST.cbl` | `HV-CUSTOMER-EMAIL` host variable added; `MOVE COMM-EMAIL` to HV before UPDATE; `CUSTOMER_EMAIL = :HV-CUSTOMER-EMAIL` added to SQL UPDATE SET clause; `MOVE HV-CUSTOMER-EMAIL TO COMM-EMAIL` in success path |
| `src/base/cics/cobol/BNK1CCS.cbl` | `SUBPGM-EMAIL PIC X(50)` added to `SUBPGM-PARMS`; `MOVE SPACES TO EMAILI` added in RECEIVE-MAP initialisation; `MOVE EMAILI OF BNK1CCI TO SUBPGM-EMAIL` added in `CRE-CUST-DATA`; `MOVE SPACES TO SUBPGM-EMAIL` on `INITIALIZE SUBPGM-PARMS` |
| `src/base/cics/cobol/BNK1DCS.cbl` | ⚠️ **4 inline edits** (no COPY statement — manual sync required): `WS-COMM-EMAIL PIC X(50)` added to `WS-COMM-AREA`; `COMM-EMAIL PIC X(50)` added to inline `DFHCOMMAREA` LINKAGE SECTION; `MOVE COMM-EMAIL OF DFHCOMMAREA TO WS-COMM-EMAIL` added in COMMAREA copy-out block; display path: `MOVE INQCUST-EMAIL OF INQCUST-COMMAREA TO CUSTEMLO OF BNK1DCO`; update path: `MOVE WS-COMM-EMAIL TO COMM-EMAIL OF UPDCUST-COMMAREA` |
| `src/base/cics/cobol/BANKDATA.cbl` | `CUSTOMER_EMAIL` added to INSERT column list; literal `'test@bankofz.example.com'` added to VALUES |

---

## Workstream F — z/OS Connect Mapping YAMLs + OpenAPI (5 files, steps F4–F8)

| File | Change |
|---|---|
| `src/api/src/main/operations/%2Fcustomers/post/request.yaml` | `COMM-EMAIL` mapping added with `$exists($body.email)` null guard |
| `src/api/src/main/operations/%2Fcustomers%2F%7BcustomerId%7D/get/response_200.yaml` | `email` response field mapped from `INQCUST-EMAIL` |
| `src/api/src/main/operations/%2Fcustomers%2F%7BcustomerId%7D/put/request.yaml` | `COMM-EMAIL` mapping added from `$body.email` |
| `src/api/src/main/operations/%2Fcustomers%2F%7BcustomerId%7D/put/response_200.yaml` | `email` response field mapped from `COMM-EMAIL` |
| `src/api/src/main/api/openapi.yaml` | `email` field (optional, nullable, `maxLength: 50`) added to `Customer` and `CustomerUpdate` schemas |

---

## ⏳ Remaining Steps (require z/OS USS)

### Workstream A — DB2 DDL
Execute on the target DB2 subsystem **before** the DBRM bind:
```sql
ALTER TABLE CUSTOMER ADD COLUMN CUSTOMER_EMAIL CHAR(50);
```
- Column is nullable — no NOT NULL constraint — preserving all existing rows
- Must precede all DBRM binds (step E2)

### Workstream E — Build, Bind, Deploy
1. **E1** — `dbb build impact` — DBB impact build picks up all copybook changes; BMS reassembly runs before COBOL compile for `BNK1CCS` and `BNK1DCS`
2. **E2** — DB2 DBRM bind for `CRECUST`, `INQCUST`, `UPDCUST`, `BANKDATA` into plan `BANKZPLN`
3. **E3** — Wazi Deploy to CICS load library
4. **E4** — Smoke test via CICS terminal:
   - `BNK1CCM`: email input field visible at row 21
   - `BNK1DCM`: email display field visible at row 20

### Workstream F (steps F1–F3) — Provider CPY Regeneration
After E3 (COBOL deployed), regenerate the z/OS Connect provider `.cpy` files via z/OS Connect CLI:
- `src/api/src/main/zosAssets/CRECUST/providerFiles/gen/` — regenerate
- `src/api/src/main/zosAssets/INQCUST/providerFiles/gen/` — regenerate
- `src/api/src/main/zosAssets/UPDCUST/providerFiles/gen/` — regenerate

> ⚠️ Do **not** hand-edit these files. Always regenerate via CLI.

Then:
- **F9** — `dbb build impact` for z/OS Connect API project
- **F10** — Wazi Deploy — deploy updated z/OS Connect API artifact

---

## Rollback

| Trigger | Action |
|---|---|
| DDL deployed, programs not yet bound | `ALTER TABLE CUSTOMER DROP COLUMN CUSTOMER_EMAIL` (safe while column is all NULL) |
| Programs bound and deployed, defect found | Redeploy previous load modules; re-bind previous DBRMs; drop column |
| z/OS Connect API deployed, API defect | Redeploy previous z/OS Connect artifact; COBOL remains compatible (column is nullable) |

> ⚠️ DDL rollback is safe only while all `CUSTOMER_EMAIL` values are NULL. Once real email data is written, dropping the column loses that data.

---

**Reference**: [`implementation-plan.md`](./implementation-plan.md)  
**Impact analysis**: [`bobz/impact-analysis/customer-email-field-20260727T225450/IMPACT-ANALYSIS.md`](../../impact-analysis/customer-email-field-20260727T225450/IMPACT-ANALYSIS.md)
