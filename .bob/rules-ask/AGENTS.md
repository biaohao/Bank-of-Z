# AGENTS.md — Ask Mode

This file provides documentation-context guidance for AI agents answering questions in this repository.

## Non-Obvious Code Organization

- `src/base/cics/` and `src/base/ims/` are **two separate transaction paths** for the same business domain — CICS handles customers with ID prefix `C`, IMS handles prefix `I`. They do NOT share copybooks (each has its own `copy/` subdirectory).
- `src/api/` is a **Gradle z/OS Connect project** — it is NOT a Node.js/Java web API. Operations live in URL-encoded directory names under `src/api/src/main/operations/`.
- `src/frontend/` is served from a **WebSphere Liberty server on z/OS** as a WAR file — it is not a static site or Node app. Port 3001 is a Docker dev proxy only.
- `dbb-app.yaml` is the **entire build configuration** for all z/OS languages (COBOL, PL/I, Assembler, BMS). There are no separate Makefiles or build scripts for z/OS.
- `citi-demo/` contains **pre-built HTML demo pages** for sales presentations — not source code.
- `bobz/` is the **agent working directory** for planning artifacts (impact analyses, implementation plans). Not part of the application.

## Program Naming Confusion

- `BNK1DCS.cbl` = "Delete Customer Screen" — the presentation layer for customer delete/display, NOT a delete business program. The actual delete is `DELCUS.cbl`.
- `BNK1CCS.cbl` = "Create Customer Screen" — the CICS BMS frontend; `CRECUST.cbl` is the business logic.
- `IBTRAN.cbl` is **not** an IMS transaction — it is a 31-bit COBOL shim that bridges to a 64-bit JVM for IMS Java Message Processing.
- `BANKDATA.cbl` is **not** a runtime program — it is a test-data seeder that bulk-inserts/deletes all four DB2 tables.

## Copybook Naming Patterns

- `*DB2.cpy` — DB2 DECLARE TABLE statements (e.g., `CUSTDB2.cpy`, `ACCDB2.cpy`)
- `*Z.cpy` — COMMAREA definitions for inquiry/response programs (e.g., `INQCUSTZ.cpy`, `INQACCZ.cpy`, `DELACCZ.cpy`)
- Files without a suffix (e.g., `CUSTOMER.cpy`, `ACCOUNT.cpy`) — COBOL data structures (not COMMAREAs, not DB2)
- `SORTCODE.cpy` — widest-impact copybook: included by virtually every CICS program; a change triggers a rebuild of the entire CICS layer

## Documentation Gaps

- `docs/docs/tutorials/debug-cics-transaction.md` references `INQCUST` specifically with DPS/RDS port numbers and debugger setup — useful context when diagnosing CICS issues.
- `docs/docs/development-workflows/workflow-comparison.md` explains why GRUB does NOT require a commit (direct USS file sync) while Zowe CLI workflow does (clones from GitHub branch).
- `BNK1DDM.cpy` in `src/base/cics/copy/` is unused — it is a pre-generated copybook with `COMPANY`/`MESSAGE` fields that no current program includes. Do not reference it as a live BMS copybook.
- `BNK1B2M.bms` defines a secondary map (`BNK1B2`) inside the `BNK1TFM` mapset — it is not an independent BMS screen and no program COPYs it separately.

## z/OS Connect CICS Transaction

All CICS business programs exposed via z/OS Connect share **one** CICS transaction ID: `OMEN`. z/OS Connect routes to the correct COBOL program via the `program:` field in `zosAsset.yaml`, not via separate CICS transaction IDs.
