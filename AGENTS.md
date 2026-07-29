# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Workspace Type

**Type:** IBM Enterprise COBOL for z/OS hybrid banking application
**Z Understand Project:** `BankofZ` (project ID: `506ac666-9c9b-4fcc-882a-9b93de6fe85f`, stored in `.bobz/local-settings.json`)
**Detected Languages:** IBM Enterprise COBOL for z/OS, PL/I, HLASM (Assembler), JCL, Java (IMS bridge), JavaScript/HTML (frontend)
**Build System:** IBM Dependency Based Build (DBB) / zBuilder, Wazi Deploy, Gradle (z/OS Connect API + IMS Java)

## Repository Structure (Non-Obvious)

- `src/base/cics/` — COBOL programs, BMS maps, and copybooks for the **CICS transaction path** (customers with ID pattern `Cnnnn`)
- `src/base/ims/` — COBOL, PL/I programs and copybooks for the **IMS transaction path** (customers with ID pattern `Innn`)
- `src/base/batch/` — Batch programs: one PL/I program (`BNKSTMT.pli`) + one JCL (`BNKSTMT.jcl`) for monthly statement generation
- `src/base/ims/PSB/` and `src/base/ims/DBD/` — Assembler sources for IMS PSB and DBD definitions; assembled by IMS ACBGEN, deployed to `PSBLOAD`/`DBDLOAD` libraries — **not** compiled as programs
- `src/base/ims/java/` — Gradle-based IMS Java JMP project bridging 31-bit COBOL to 64-bit JVM via `IBTRAN.cbl`
- `src/api/` — z/OS Connect API project (Gradle). Operations live in URL-encoded path directories under `src/api/src/main/operations/`
- `src/api/src/main/zosAssets/*/providerFiles/gen/` — Generated `.cpy` provider files for z/OS Connect; **NEVER hand-edit** — regenerate via z/OS Connect CLI after COBOL programs are deployed
- BMS map sources use `.bms` extension; located in `src/base/cics/bms/`

## Build Commands (All run on z/OS USS, not locally)

| Workflow | Local trigger |
|---|---|
| Full environment setup (first time) | `bash .setup/setup-local.sh` from repo root, OR VS Code task: **Setup Bank of Z Environment** |
| Incremental build + deploy | `bash .setup/pipeline-local.sh` from repo root, OR VS Code task: **Run Pipeline Simulation** |
| GRUB workflow (no push required) | Trigger GRUB sync from IDE; auto-runs `setup-remote.sh` on USS |

- **Zowe CLI workflow**: Requires pushing changes to remote before running — the script clones the branch from GitHub on USS
- **GRUB workflow**: Does NOT require a commit/push; detects Bank-of-Z repo and uses local files directly; incremental updates take ~5–10 seconds vs 5–8 minutes for Zowe CLI
- Single-program user build: `dbb build user` — runs compile + optional TAZ unit test
- DBB build lifecycles: `full`, `impact` (incremental), `pipeline`, `user` (single program), `merge`, `file`, `metadata`

## Git Commit Convention (MANDATORY)

All commits **MUST** include DCO sign-off — PRs with unsigned commits are automatically rejected.

```bash
git commit -s -m "Your message"   # -s flag is MANDATORY
```

To fix already-made commits without sign-off:
```bash
git commit --amend -s --no-edit        # last commit only
git rebase HEAD~N --signoff            # N commits
git push --force-with-lease origin <branch>
```

## COBOL-Specific Conventions

- CICS programs start with `PROCESS CICS,NODYNAM,NSYMBOL(NATIONAL),TRUNC(STD)` followed by `CBL CICS(...)` **before** `IDENTIFICATION DIVISION`
- IMS COBOL programs start with `CBL LIST,MAP,XREF,FLAG(I)` — no `PROCESS` directive
- `IBTRAN.cbl` requires special compile parms: `LP(32),JAVAIOP(JAVA64),DLL,RENT,PGMNAME(LONGMIXED)` — handled automatically by `dbb-app.yaml`
- IMS batch COBOL programs (`IBACSUM`, `IBGCUDAT`, `IBLOGIN1`, `IBLOGOUT`, `IBSCUDAT`, `LOAD*`) require `ENTRY DLITCBL` in their link-edit streams — do not change link options for these without updating `dbb-app.yaml`
- ⚠️ **`BNK1DCS.cbl` has a fully inline `DFHCOMMAREA` LINKAGE SECTION and a parallel `WS-COMM-AREA` working-storage struct** — neither uses a COPY statement; both must be manually extended when any CICS COMMAREA copybook changes, or silent data corruption occurs
- ⚠️ **`BNK1CCS.cbl` has a local `SUBPGM-PARMS` working-storage struct** that mirrors the customer COMMAREA — it must be manually kept in sync with COMMAREA copybook changes
- COBOL deploy types: `LOAD` (batch default), `CICSLOAD` (auto-detected via `IS_CICS`), `IMSLOAD` (all `ims/cobol/*.cbl`)

## PL/I-Specific Conventions

- IMS PL/I programs (`ims/pli/`) require `ENTRY CEESTART` and `INCLUDE RESLIB(DFSLI000)` in link-edit stream; `isIMS: true` is auto-applied only to `src/base/ims/pli/*.pli`
- Batch PL/I programs (`batch/pli/`) require `ENTRY CEESTART` only (no DFSLI000); `isIMS` is NOT set for batch PL/I — despite being in a similar IMS-adjacent context this is intentional
- `BNKSTMT.pli` uses named SQL columns — safe against schema additions (nullable columns don't break it)

## Assembler (IMS Control Blocks)

- PSB sources → deployed to `PSBLOAD`; DBD sources → deployed to `DBDLOAD` (not standard `LOAD` libraries)
- Changes to PSB/DBD sources require IMS ACBGEN (not a standard compile) — coordinate with IMS admin

## Static Analysis (ZCodeScan)

- Rules file: `zcodescan/zcodescan-rules.yaml`; activated via `zapp.yaml` (`zcodescan` profile)
- **Condition names (88-level) must have prefix `TEST`** (`ConditionNamePrefixRule`) — this is non-standard for COBOL but enforced here
- Inline PERFORM limited to 30 lines; nested IF limit is 6; procedure body limit is 100 lines
- `DISPLAY` to CONSOLE is flagged MEDIUM (`DisplayUponConsoleRule`) — use CICS messages instead
- `SELECT *` in SQL is flagged HIGH (`SqlAvoidSelectStarRule`) — always name columns explicitly
- CICS HANDLE CONDITION / HANDLE AID is flagged MEDIUM (`CicsNoHandleRule`) — use RESP/RESP2 pattern

## Unit Testing (TAZ)

- TAZ profile in `zapp.yaml` points to `BANKZ.V0R1M0.LOAD` as application load library
- Activated during `dbb build user` lifecycle
- `SYS1.PROCLIB` is the required procedure library

## Configuration

- All environment settings: `.setup/config/config.yaml` — uses `{{section.key}}` variable expansion and `${ENV_VAR}` environment variable references
- Application HLQ: `BANKZ`, version qualifier: `V0R1M0`
- `.zdx.json.template` must be copied to `.zdx.json` and filled in with your USS sandbox path and connection details for GRUB/debugger integration
- Pre-commit hook enforces `detect-secrets` via `ibm/detect-secrets` — run `pre-commit autoupdate` to update the hook revision

## Technical Documentation

Full documentation: [https://ibm.github.io/Bank-of-Z/](https://ibm.github.io/Bank-of-Z/)

| Topic | Location |
|---|---|
| Architecture overview | `docs/docs/about-bank-of-z/architecture-overview.md` |
| Application components | `docs/docs/architecture/application-components.md` |
| Request flow + routing logic (CICS vs IMS) | `docs/docs/architecture/application-flow.md` |
| Build and deployment | `docs/docs/architecture/build-and-deployment.md` |
| Repository structure | `docs/docs/reference/repository-structure.md` |
| Configuration reference | `docs/docs/reference/configuration-reference.md` |
| Commands reference | `docs/docs/reference/commands-reference.md` |
| Zowe CLI workflow | `docs/docs/development-workflows/zowe-cli-workflow.md` |
| GRUB workflow | `docs/docs/development-workflows/grub-workflow.md` |
| Workflow comparison | `docs/docs/development-workflows/workflow-comparison.md` |
| CICS enhancement tutorial | `docs/docs/tutorials/cics-enhancement-scenario.md` |
| Debug a CICS transaction | `docs/docs/tutorials/debug-cics-transaction.md` |
| Deploy Bank of Z (video) | `docs/docs/tutorials/deploy-bank-of-z.md` |

**Agent Planning Artifacts:** Store impact analyses and implementation plans under `bobz/` (e.g., `bobz/impact-analysis/<name>/`, `bobz/implementation-plans/<name>/`). This directory is agent-writable and gitignored for generated output.

**`citi-demo/`** — Pre-built HTML demo scenario pages for sales demos only. Not source code or documentation. Do not modify.

**COBOL Program to Documentation Mapping:**

| Program(s) | Documentation |
|---|---|
| All CICS COBOL (`src/base/cics/cobol/`) | `docs/docs/architecture/application-components.md`, `docs/docs/architecture/application-flow.md` |
| All IMS COBOL/PL/I (`src/base/ims/cobol/`, `src/base/ims/pli/`) | `docs/docs/architecture/application-components.md`, `docs/docs/architecture/application-flow.md` |
| `IBTRAN.cbl` | `dbb-app.yaml` (special build parms), `docs/docs/architecture/application-components.md` |
| `BNKSTMT.pli` + `BNKSTMT.jcl` | `docs/docs/architecture/application-components.md` (batch/statement generation) |
| PSB/DBD Assembler sources | `docs/docs/architecture/build-and-deployment.md` |
| `src/api/src/main/api/openapi.yaml` | `docs/docs/reference/configuration-reference.md`, OpenBanking UK spec |
| `INQCUST.cbl` | `docs/docs/tutorials/debug-cics-transaction.md` |

**Auto-Update Rules:**
1. When modifying COBOL programs, check if related docs/tutorials reference that program and update accordingly
2. When adding new programs, add them to the mapping table above and update `dbb-app.yaml` if they require non-default build options
3. When modifying IMS PSB/DBD sources, note that changes require IMS ACBGEN — coordinate with IMS admin
4. When updating z/OS Connect operations, regenerate provider `.cpy` files in `src/api/src/main/zosAssets/` — do not hand-edit

## z/OS Connect Conventions

- **Provider file layout per program**: `src/api/src/main/zosAssets/<PROGRAM>/providerFiles/`
  - `request.dai` / `response.dai` — hand-authored data interface descriptors (never generated)
  - `gen/<PROGRAM>_request_0.cpy` / `gen/<PROGRAM>_response_0.cpy` — **generated** COBOL copybooks; regenerate via z/OS Connect CLI after each COBOL deploy; **never hand-edit**
  - `gen/requestSchema.json` / `gen/responseSchema.json` — also generated, never hand-edit
  - `COMMAREA.cpy` at providerFiles level (CRECUST only as example) — source of truth for z/OS Connect; must match deployed COBOL COMMAREA exactly
- **Mapping YAML JSONata pattern**: use `$exists($body.field)` null-guards for optional fields to avoid forwarding empty strings to COBOL (e.g., `"{{$exists($body.email) ? $body.email : \"\"}}"`)
- **All z/OS Connect operations** use `transid: "OMEN"` and `connectionRef: "bankzCicsConnection"` — the CICS transaction ID is `OMEN` regardless of which business program is invoked

## Web UI Conventions (src/frontend/)

- **API base URL** is auto-detected in `src/frontend/config.js`: port 3001 → `/api` (Docker dev, proxied by nginx); any other port → `http://<hostname>:9080/api` (z/OS Liberty)
- **System routing** (CICS vs IMS) is determined purely by the leading letter of the customer ID: `C...` → CICS endpoints (`/api/customers/`), `I...` → IMS endpoints (`/api/ims/customers/`)
- Web UI uses Carbon Design System web components (`<cds-text-input>`, etc.) bundled as `src/frontend/js/carbon-web-components.min.js` — no npm build step; vanilla JS ES modules with native `import`
- For optional API fields (e.g., `email`), pass `value || undefined` so the field is omitted from the JSON body entirely when blank, rather than sending an empty string

## DB2 Table Ownership

| Table | Defined in Copybook | Programs that READ | Programs that WRITE |
|---|---|---|---|
| `CUSTOMER` | `src/base/cics/copy/CUSTDB2.cpy` | `INQCUST`, `UPDCUST`, `DELCUS`, `BNKSTMT` (PL/I) | `CRECUST` (INSERT), `UPDCUST` (UPDATE), `DELCUS` (DELETE), `BANKDATA` (INSERT/DELETE) |
| `ACCOUNT` | `src/base/cics/copy/ACCDB2.cpy` | `INQACC`, `INQACCS`, `INQACCCU`, `DELACC`, `UPDACC`, `DBCRFUN`, `XFRFUN`, `BNKSTMT` (PL/I) | `CREACC` (INSERT), `UPDACC` (UPDATE), `DELACC` (DELETE), `DBCRFUN` (UPDATE), `XFRFUN` (UPDATE), `BANKDATA` (INSERT/DELETE) |
| `PROCTRAN` | `src/base/cics/copy/PROCDB2.cpy` | `INQTRANL`, `INQTRAND`, `BNKSTMT` (PL/I) | `DBCRFUN` (INSERT), `XFRFUN` (INSERT), `BANKDATA` (INSERT/DELETE) |
| `STTESTER.CONTROL` | `src/base/cics/copy/CONTDB2.cpy` | `CRECUST`, `BANKDATA` | `CREACC` (UPDATE), `BANKDATA` (INSERT/UPDATE/DELETE) |

> ⚠️ **`STTESTER.CONTROL`** is a test-harness control table (schema prefix `STTESTER`) — accessed by `BANKDATA`, `CREACC`, and `CRECUST` to track test state. It is NOT a business table.

> ⚠️ **`BANKDATA`** is a data-loader/test-data program (inserts/deletes all four tables). It is **not** a runtime transaction — it exists only to seed the Db2 database.

## BMS Screen to Program Mapping

| BMS Map source | Map name prefix | Owner program |
|---|---|---|
| `BNK1ACC.bms` | `BNK1ACC` | `BNK1CCA` — Create account |
| `BNK1CAM.bms` | `BNK1CA` | `BNK1CAC` — Create account (CICS menu) |
| `BNK1CCM.bms` | `BNK1CC` / `BNK1CCM` | `BNK1CCS` — Create customer |
| `BNK1CDM.bms` | `BNK1CD` | `BNK1CRA` — Credit/Debit account |
| `BNK1DAM.bms` | `BNK1DA` | `BNK1DAC` — Delete account |
| `BNK1DCM.bms` | `BNK1DC` | `BNK1DCS` — Delete customer |
| `BNK1MAI.bms` | `BNK1ME` | `BNKMENU` — Main menu |
| `BNK1TFM.bms` | `BNK1TF` | `BNK1TFN` — Transfer funds |
| `BNK1B2M.bms` | `BNK1B2` (within mapset `BNK1TFM`) | Secondary transfer funds map; no program COPYs it directly — part of the same `BNK1TFN` transaction flow |
| `BNK1UAM.bms` | `BNK1UA` | `BNK1UAC` — Update account |

> The BNK1xxx programs are thin CICS presentation-layer programs that call the business logic programs (e.g., `BNK1CAC` calls `CREACC`, `BNK1CRA` calls `DBCRFUN`).
> **BMS reassembly must precede COBOL compile** for programs that `COPY` a BMS-generated symbolic map (e.g., `BNK1CCS` copies `BNK1CCM`, `BNK1DCS` copies `BNK1DCM`). DBB handles this via `.bms` dependency patterns in `dbb-app.yaml`.
> **`BNK1DDM.cpy`** in `src/base/cics/copy/` is a pre-generated symbolic copybook with only `COMPANY` and `MESSAGE` fields — it is NOT used by any current COBOL program (confirmed by grep). Do not mistake it for a BMS-derived copybook for a screen.

## Key Shared Copybooks

| Copybook | Used by | Purpose |
|---|---|---|
| `SORTCODE.cpy` | All CICS programs + `BANKDATA` (21 programs) | Bank sort code constant — **widest impact** copybook; a change triggers rebuild of virtually every CICS program |
| `CUSTDB2.cpy` | All programs accessing CUSTOMER table | DB2 DECLARE TABLE for CUSTOMER |
| `ACCDB2.cpy` | All programs accessing ACCOUNT table | DB2 DECLARE TABLE for ACCOUNT |
| `PROCDB2.cpy` | All programs accessing PROCTRAN table | DB2 DECLARE TABLE for PROCTRAN |
| `CONTDB2.cpy` | `BANKDATA`, `CREACC`, `CRECUST` | DB2 DECLARE TABLE for `STTESTER.CONTROL` |
| `ABNDINFO.cpy` | `ABNDPROC` | Abend/error information structure for the shared CICS error handler |

## CICS Program Groups

**Presentation layer** (have BMS screens, handle COMMAREA, call business programs):
`BNKMENU`, `BNK1CAC`, `BNK1CCA`, `BNK1CCS`, `BNK1CRA`, `BNK1DAC`, `BNK1DCS`, `BNK1TFN`, `BNK1UAC`

**Business/service layer** (called by presentation programs or z/OS Connect directly):
`CREACC`, `CRECUST`, `DELACC`, `DELCUS`, `DBCRFUN`, `INQACC`, `INQACCS`, `INQACCCU`, `INQCUST`, `INQTRAND`, `INQTRANL`, `UPDACC`, `UPDCUST`, `XFRFUN`, `GETSCODE`, `GETCOMPY`

**Infrastructure/shared**:
`ABNDPROC` — shared CICS abend handler, called by all CICS programs on error
`CRDTAGY1`–`CRDTAGY5` — credit agency stub programs

**IMS COBOL**: `IBACSUM`, `IBGCUDAT`, `IBLOGIN1`, `IBLOGOUT`, `IBSCUDAT`, `IBTRAN` (Java bridge), `LOADACCT`, `LOADCUSA`, `LOADCUST`, `LOADHIST`, `LOADTSTA`
**IMS PL/I**: `IBLOGIN` (IMS entry), `BNKSTMT` (batch monthly statement)
**Data loader** (not runtime): `BANKDATA` — seeds all Db2 tables

## COBOL Program Quirks

- **`INQCUST.cbl`** has three separate SQL SELECT statements: only the one in paragraph `READ-CUSTOMER-DB2` populates the COMMAREA. The `GET-LAST-CUSTOMER-DB2` paragraph looks up the most recently assigned customer number — it does NOT return customer data to the caller and does NOT need new columns added when extending the customer schema.
- **`BANKDATA.cbl`** uses literal string values directly in SQL `VALUES` clauses (not host variables) — this is intentional for a test-data seeder; `detect-secrets` baseline must be updated if new literals are added.
