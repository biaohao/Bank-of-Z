# Z Understand Analysis — BankofZ

This document captures the additional insights discovered via the Z Understand container
(`BankofZ` project, ID: `506ac666-9c9b-4fcc-882a-9b93de6fe85f`) that were not available
from local file inspection alone.

---

## DB2 Table Ownership

Four DB2 tables are used at runtime. Each is declared via a copybook included by programs
that access it.

| Table | Declared in copybook | Programs that READ | Programs that WRITE (INSERT/UPDATE/DELETE) |
|---|---|---|---|
| `CUSTOMER` | `src/base/cics/copy/CUSTDB2.cpy` | `INQCUST`, `UPDCUST`, `DELCUS`, `BNKSTMT` (PL/I) | `CRECUST` (INSERT), `UPDCUST` (UPDATE), `DELCUS` (DELETE), `BANKDATA` (INSERT/DELETE) |
| `ACCOUNT` | `src/base/cics/copy/ACCDB2.cpy` | `INQACC`, `INQACCS`, `INQACCCU`, `DELACC`, `UPDACC`, `DBCRFUN`, `XFRFUN`, `BNKSTMT` (PL/I) | `CREACC` (INSERT), `UPDACC` (UPDATE), `DELACC` (DELETE), `DBCRFUN` (UPDATE), `XFRFUN` (UPDATE), `BANKDATA` (INSERT/DELETE) |
| `PROCTRAN` | `src/base/cics/copy/PROCDB2.cpy` | `INQTRANL`, `INQTRAND`, `BNKSTMT` (PL/I) | `DBCRFUN` (INSERT), `XFRFUN` (INSERT), `BANKDATA` (INSERT/DELETE) |
| `STTESTER.CONTROL` | `src/base/cics/copy/CONTDB2.cpy` | `CRECUST`, `BANKDATA` | `CREACC` (UPDATE), `BANKDATA` (INSERT/UPDATE/DELETE) |

> ⚠️ **`STTESTER.CONTROL`** (schema prefix `STTESTER`) is a test-harness control table — not a
> business data table. Do not include it in production data model or impact analysis. It is
> accessed by `BANKDATA`, `CREACC`, and `CRECUST` only to track test execution state.

> ⚠️ **`BANKDATA`** is a data-loader/test-data seeding program — not a runtime transaction
> program. It inserts, updates, and deletes all four tables. It should never appear as a
> production impact target.

---

## BMS Screen to Program Mapping (CICS only)

The 10 BMS screens each have a single owning COBOL program. Each `BNK1xxx` program is a
thin presentation-layer handler that owns exactly one screen and delegates to a business
program.

| BMS source file | Map name prefix | Owner COBOL program | Purpose |
|---|---|---|---|
| `BNK1ACC.bms` | `BNK1ACC` | `BNK1CCA` | Create account |
| `BNK1CAM.bms` | `BNK1CA` | `BNK1CAC` | Create account (CICS menu flow) |
| `BNK1CCM.bms` | `BNK1CC` / `BNK1CCM` | `BNK1CCS` | Create customer |
| `BNK1CDM.bms` | `BNK1CD` | `BNK1CRA` | Credit/Debit account |
| `BNK1DAM.bms` | `BNK1DA` | `BNK1DAC` | Delete account |
| `BNK1DCM.bms` | `BNK1DC` | `BNK1DCS` | Delete customer |
| `BNK1MAI.bms` | `BNK1ME` | `BNKMENU` | Main menu |
| `BNK1TFM.bms` | `BNK1TF` | `BNK1TFN` | Transfer funds |
| `BNK1UAM.bms` | `BNK1UA` | `BNK1UAC` | Update account |
| `BNK1B2M.bms` | *(unmapped)* | No direct COBOL match found in project | Likely secondary screen for `BNK1CCA` |

---

## Key Shared Copybooks — High-Impact Change Targets

| Copybook | Programs using it | Notes |
|---|---|---|
| `SORTCODE.cpy` | 21 programs (all CICS programs + `BANKDATA`) | **Highest-impact copybook** — a change triggers rebuild of virtually the entire CICS codebase |
| `ABNDINFO.cpy` | `ABNDPROC` only | Abend/error information structure for the shared error handler |
| `CUSTDB2.cpy` | All programs accessing `CUSTOMER` | DB2 DECLARE TABLE — also includes the `CUSTOMER` copybook (struct) |
| `ACCDB2.cpy` | All programs accessing `ACCOUNT` | DB2 DECLARE TABLE for `ACCOUNT` |
| `PROCDB2.cpy` | All programs accessing `PROCTRAN` | DB2 DECLARE TABLE for `PROCTRAN` |
| `CONTDB2.cpy` | `BANKDATA`, `CREACC`, `CRECUST` | DB2 DECLARE TABLE for `STTESTER.CONTROL` (test harness only) |

---

## Program Groups by Functional Layer

### CICS — Presentation layer
Own a BMS screen, handle COMMAREA, call business programs below:

`BNKMENU`, `BNK1CAC`, `BNK1CCA`, `BNK1CCS`, `BNK1CRA`, `BNK1DAC`, `BNK1DCS`, `BNK1TFN`, `BNK1UAC`

### CICS — Business / service layer
Called by presentation programs or by z/OS Connect directly; contain all SQL and business logic:

`CREACC`, `CRECUST`, `DELACC`, `DELCUS`, `DBCRFUN`, `INQACC`, `INQACCS`, `INQACCCU`,
`INQCUST`, `INQTRAND`, `INQTRANL`, `UPDACC`, `UPDCUST`, `XFRFUN`, `GETSCODE`, `GETCOMPY`

### CICS — Infrastructure / shared
- `ABNDPROC` — shared CICS abend handler; called by all CICS programs on error
- `CRDTAGY1`–`CRDTAGY5` — credit agency stub programs (5 identical-structure variants, all include `SORTCODE.cpy`)

### IMS — COBOL programs
`IBACSUM`, `IBGCUDAT`, `IBLOGIN1`, `IBLOGOUT`, `IBSCUDAT`, `IBTRAN` (Java bridge),
`LOADACCT`, `LOADCUSA`, `LOADCUST`, `LOADHIST`, `LOADTSTA`

### IMS — PL/I programs
- `IBLOGIN` — IMS transaction entry program
- `BNKSTMT` — Batch monthly statement generator; reads `CUSTOMER`, `ACCOUNT`, and `PROCTRAN` via DB2 cursors

### Data loader (not runtime — test/setup only)
- `BANKDATA` — seeds all four DB2 tables; also manages `STTESTER.CONTROL` test state

---

## CICS Transaction Count

Z Understand identified **10 CICS transactions** registered in the project. Transaction IDs
route through z/OS Connect to the CICS programs. See:
- `src/api/src/main/liberty/config/cics.xml` — CICS connection definition
- `src/api/src/main/operations/` — z/OS Connect operation-to-transaction mappings
