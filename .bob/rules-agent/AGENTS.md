# AGENTS.md — Agent (Code) Mode

This file provides coding-specific guidance for AI agents making code changes in this repository.

## MANDATORY Before Any Commit

```bash
git commit -s -m "message"   # -s is NON-NEGOTIABLE — DCO enforced by PR checks
```

## COBOL Editing Rules

- **`BNK1DCS.cbl` LINKAGE SECTION is fully inline** — no COPY statement. When any CICS COMMAREA copybook changes, you MUST manually add matching fields to BOTH the `WS-COMM-AREA` working-storage struct AND the inline `DFHCOMMAREA` LINKAGE SECTION in `BNK1DCS.cbl`, or data corruption occurs silently.
- **`BNK1CCS.cbl` `SUBPGM-PARMS` is inline** — manually sync it with CRECUST.cpy when the create-customer COMMAREA changes.
- CICS programs require `PROCESS CICS,...` + `CBL CICS(...)` **before** `IDENTIFICATION DIVISION` — not just `CBL`.
- IMS COBOL programs use `CBL LIST,MAP,XREF,FLAG(I)` — no `PROCESS` directive.
- 88-level condition names **must** be prefixed `TEST` — enforced by ZCodeScan `ConditionNamePrefixRule`.
- END-IF / END-EVALUATE / END-READ / END-SEARCH / END-CALL are **required** — `RequireEndClauseRule` is active.
- Never use `SELECT *` in embedded SQL — `SqlAvoidSelectStarRule` is HIGH severity.
- Never use CICS HANDLE CONDITION/HANDLE AID — use RESP/RESP2 pattern.
- Inline PERFORM body limit: 30 lines. Nested IF limit: 6. Paragraph body limit: 100 lines.

## z/OS Connect Editing Rules

- **Never hand-edit** `gen/` directories under `src/api/src/main/zosAssets/*/providerFiles/gen/` — regenerate via z/OS Connect CLI after COBOL deploy.
- `.dai` files (`request.dai`, `response.dai`) ARE hand-authored — located at the `providerFiles/` level (not in `gen/`).
- For optional JSON body fields in mapping YAMLs, use JSONata `$exists()` null-guard: `"{{$exists($body.field) ? $body.field : \"\"}}"`
- All `zosAsset.yaml` files use `transid: "OMEN"` and `connectionRef: "bankzCicsConnection"` — do not change these.

## Web UI Editing Rules

- No npm/build step — vanilla ES modules; import from relative paths only.
- `src/frontend/config.js` controls API base URL — do not hardcode API endpoints in HTML/JS files.
- Optional API fields: pass `fieldValue || undefined` (not `""`) so the JSON body omits the field when blank.
- Carbon Design System web components are pre-bundled in `src/frontend/js/carbon-web-components.min.js` — do not add CDN script tags.

## DB2 Schema Changes

When adding a column to the `CUSTOMER` table, the following must ALL be updated in lockstep:
1. `src/base/cics/copy/CUSTDB2.cpy` — DB2 DECLARE TABLE
2. `src/base/cics/copy/CUSTOMER.cpy` — COBOL struct
3. Relevant COMMAREA copybooks (`CRECUST.cpy`, `INQCUSTZ.cpy`, `UPDCUST.cpy`)
4. Business programs (`CRECUST.cbl`, `INQCUST.cbl`, `UPDCUST.cbl`) — host variables + SQL
5. Presentation programs (`BNK1CCS.cbl`, `BNK1DCS.cbl`) — **inline** structs, no COPY
6. BMS maps if a new screen field is required
7. z/OS Connect mapping YAMLs + `openapi.yaml`
8. Web UI HTML + `api.js`
9. DB2 DDL (`ALTER TABLE CUSTOMER ADD COLUMN ...`) — executed on target DB2 before DBRM bind

## Build Ordering

1. BMS assembly must complete before COBOL compile for `BNK1CCS.cbl` and `BNK1DCS.cbl` — DBB handles this automatically via `.bms` dependency in `dbb-app.yaml`.
2. For `INQCUST.cbl` schema changes: only add columns to the SQL SELECT in `READ-CUSTOMER-DB2`. Do NOT add columns to `GET-LAST-CUSTOMER-DB2` — that paragraph only retrieves the last customer number.

## secrets baseline

If new string literals are added to `BANKDATA.cbl` (test-data seeder), run `detect-secrets scan` and update `.secrets.baseline` before committing, or the pre-commit hook will block the commit.
