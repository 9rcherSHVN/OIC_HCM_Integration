# Oracle Integration Cloud — Custom HCM Integration: Technical Design & Implementation Blueprint

| | |
| --- | --- |
| **Version** | 3.0 |
| **Date** | 2026-05-15 |
| **Audience** | Integration Developers, Solution Specialists |
| **Scope** | Custom HCM integrations using OIC App-Driven Orchestration with Oracle HCM Data Loader (HDL / FBDI) |
| **Status** | Finalized — V3 Technical Re-Assessment Applied |
| **Revision** | V3.0 incorporates all V2 corrections (A1–A12) plus V3 re-assessment findings (B1–B13). All V3 additions and corrections are marked **[V3]**. V2 marks are retained for traceability. |

---

## Table of Contents

1. Architecture Standard
2. Connection Design Standard
3. Configuration Management Standard (DVM)
4. FBDI / HDL Pipeline Standard
5. ESS Polling Standard
6. Error Handling Hierarchy Standard
7. Business Identifiers and Activity Logging Standard
8. Notification Design Standard
9. Audit and Traceability Standard
10. Deployment and Environment Management Checklist
11. Known Defect Register (Reference Implementation)
12. Architect Audit Register — V2 (A1–A12)
13. Architect Audit Register — V3 (B1–B13) **[V3]**

---

## 1. Architecture Standard

### 1.1 Integration Style

All Oracle HCM custom integrations use **Scheduled App-Driven Orchestration** in OIC. This is the only pattern that supports:

- Scheduled triggers (cron-style, configurable timezone)
- Multi-step orchestration with fault handling, loops, and conditional routing
- OIC Stage Files for intermediate file staging
- Full access to all OIC adapter types (HCM, FTP, SOAP, REST)

> **Do not use** Basic Map Data (cannot loop or conditionally route) or Event-Driven (not applicable for scheduled batch).

---

### 1.2 Integration Naming Convention

```text
{Type}_{Domain}_{Direction}_{Source}_{Version}
```

| Token | Values | Meaning |
| --- | --- | --- |
| `Type` | `IFP` | Integration Flow Process — standalone, scheduled (has its own schedule trigger) |
| `Type` | `IFS` | Integration Flow Sub — child, invoked synchronously by a parent via REST trigger |
| `Domain` | `EMPLO`, `JOB`, `DEPT`, `NAMEMAIL` | HCM data domain |
| `Direction` | `FROM_DW` | From data warehouse source |
| `Version` | `01.00.0000` | major.minor.patch |

**Examples:**

```text
IFP_COORD_HCM_DAILY_01.00.0000      ← Coordinator (single scheduled entry point)
IFS_EMPLO_FROM_DW_01.00.0000         ← Employee domain (REST-triggered, invoked by coordinator)
IFS_JOB_FROM_DW_01.00.0000           ← Job domain (REST-triggered, invoked by coordinator)
IFS_FBDI_SUBMIT_POLL_01.00.0000      ← Shared FBDI sub-integration
IFS_BIP_SRCID_PREFETCH_01.00.0000    ← Shared source ID pre-fetch sub-integration
IFS_ERROR_REPORT_01.00.0000          ← Shared error report sub-integration
```

---

### 1.3 Orchestration Pattern: Coordinator + Domain + Shared Sub-Integrations

```text
[IFP_COORD_HCM_DAILY]                         ← Single scheduled coordinator
        │                                         (e.g., 22:00 America/Vancouver)
        │
        ├─[1]─ invoke IFS_JOB                    (synchronous — wait for actual completion)
        │
        ├─[2]─ invoke IFS_DEPT                   (synchronous — starts only after [1] finishes)
        │
        ├─[3]─ invoke IFS_EMPLO                  (synchronous — starts only after [2] finishes)
        │
        └─[4]─ invoke IFS_NAMEMAIL               (synchronous — starts only after [3] finishes)


Each domain integration calls shared sub-integrations:

  IFS_FBDI_SUBMIT_POLL    ← UCM upload + importAndLoadData + ESS poll + getDataSetStatus
  IFS_BIP_SRCID_PREFETCH  ← Bulk BI Publisher lookup (returns XML result set pre-loop)
  IFS_ERROR_REPORT        ← BIP error report + UCM upload + SFTP audit write
```

**Why coordinator over time-stagger:**

Time-stagger assumes each integration finishes within a fixed window. The coordinator pattern uses the actual completion signal — if an upstream domain integration runs long (large file, HDL queue backlog), downstream integrations wait. If a critical upstream integration fails, the coordinator can halt all subsequent integrations to prevent cascading partial loads.

**[V3] Cross-domain HDL dependency (Audit B1):** The coordinator sequence order is architecturally enforced by HDL dependencies:

```text
[1] IFS_JOB    → Job codes must exist in HCM before Assignment.JobCode can reference them
[2] IFS_DEPT   → Department names must exist before Assignment.DepartmentName can reference them
[3] IFS_EMPLO  → Worker + Assignment must exist before AssignmentSupervisor can reference them
                  Phase 2 (AssignmentSupervisor) also references Manager Assignment from [3] itself
[4] IFS_NAMEMAIL → PersonName + PersonEmail update existing Worker records created in [3]
```

> **[V2] OIC Platform Constraint — Synchronous Invocation Requirement (Audit A2):** A Scheduled integration (schedule-triggered) **cannot** be invoked synchronously by another integration in OIC. Only integrations with a **REST trigger** or **SOAP trigger** can be invoked synchronously as child integrations. Therefore, domain integrations invoked by the coordinator **must use REST triggers** (type `IFS_`, not `IFP_`). Only the coordinator itself (`IFP_COORD_HCM_DAILY`) has a schedule trigger. The domain integrations are REST-triggered callable integrations that the coordinator invokes via the OIC REST adapter in synchronous request/reply mode.

---

### 1.4 OIC Project Health Gate

Before any IAR export or OIC deployment, the project **must** satisfy all of the following. **Any failing gate blocks deployment.**

| Gate | Requirement |
| --- | --- |
| Project warnings | `projectHasWarnings = false` |
| Design completion | `percentComplete = 100` |
| Router branch warnings | Zero `COMPOSER_ACTION_EMPTY_WARNING_ROUTE` in `project_messages.json` |
| Connection status | All referenced connections show `status = CONFIGURED` |

---

## 2. Connection Design Standard

### 2.1 Required Connections for HCM FBDI Integrations

| Connection Code | Adapter Type | Authentication | Role |
| --- | --- | --- | --- |
| `{PREFIX}_SFTP` | FTP Adapter | SFTP key-based or username/password | SOURCE_AND_TARGET |
| `{PREFIX}_HCM` | HCM Cloud Adapter | USERNAME_PASSWORD_TOKEN | SOURCE_AND_TARGET |
| `{PREFIX}_BIP` | SOAP Adapter | BASIC_AUTH (`oracle/wss_http_token_over_ssl_client_policy`) | SOURCE_AND_TARGET |

---

### 2.2 Environment Parameterization — Mandatory Rules

**Rule 1:** Every connection property that varies by environment **must** use the OIC `%%PROPERTY_NAME` token. No hostnames, ports, or endpoint URLs may appear as literal values in any connection definition.

**Rule 2:** No JCA file (`*_REQUEST.jca`) may contain a `<property name="endpointURL">` element. A JCA-level `endpointURL` overrides the connection-level property at runtime, bypassing all environment parameterization.

**Rule 3:** Enforce Rule 2 by running this scan on every IAR before deployment:

```powershell
# Run from the extracted IAR directory
Get-ChildItem -Recurse -Include "*.jca" | ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    if ($content -match 'endpointURL') {
        Write-Warning "HARDCODED ENDPOINT FOUND: $($_.FullName)"
    }
}
# Pass criterion: zero warnings
```

---

### 2.3 Standard Connection Property Reference

```text
# HCM Cloud Adapter
%%{PREFIX}_HCM_targetWSDLURL   → https://{hcm-hostname}/hcmCoreSetupService/...
%%{PREFIX}_HCM_Host            → {hcm-hostname}

# BI Publisher SOAP Adapter
%%{PREFIX}_BIP_targetWSDLURL   → https://{fusion-hostname}/xmlpserver/services/ExternalReportWSSService?wsdl

# SFTP FTP Adapter
%%{PREFIX}_SFTP_Host           → {sftp-hostname}
%%{PREFIX}_SFTP_Port           → 22
```

> The `targetWSDLURL` on the BIP SOAP adapter connection is the **only** place the BI Publisher hostname appears. No JCA file references any BIP URL.

---

## 3. Configuration Management Standard (DVM)

### 3.1 Standard DVM: `Common_Utility_Lookup`

Primary key: `RICE_ID` — a unique identifier assigned to each integration at project setup.

| Column | Type | Purpose |
| --- | --- | --- |
| `RICE_ID` | Key | Integration identifier (e.g., `I-03`) |
| `Name` | String | Integration display name |
| `FileName` | String | Source CSV filename (without extension) expected on SFTP |
| `DirectoryName` | String | SFTP source directory path |
| `ArchiveDirectory` | String | SFTP success archive directory |
| `Error_Directory` | String | SFTP error archive directory |
| `ReportAbsolutePath` | String | BI Publisher FBDI error report path — Phase 1 (Worker through Assignment) |
| `FusionURL` | String | Oracle Fusion base URL — must include trailing `/` |
| `IT_Distribution_List1` | String | IT operations notification email distribution list |
| `EnvironmentType` | String | Runtime label: `DEV`, `UAT`, `PROD` |
| `HDLStageBasePath` | String | OIC stage file base path for Phase 1 (e.g., `/Stage/Employee/`) |
| `MaxPollCount` | String | ESS polling iteration limit — default: `36` (= 6 minutes at 10s intervals) |
| `AssignSupStageBasePath` | **String** | **[V3]** OIC stage file base path for Phase 2 AssignmentSupervisor HDL (e.g., `/Stage/AssignSupHDL/`) |
| `Phase2ReportAbsolutePath` | **String** | **[V3]** BI Publisher FBDI error report path — Phase 2 (AssignmentSupervisor errors) |

> **[V2] DVM Type Clarification (Audit A3):** OIC Domain Value Maps store **all values as strings**. There is no Integer, Boolean, or typed column in a DVM. The `MaxPollCount` column stores a numeric value as a string. The integration must cast it to a number using `number($var_MaxPollCount)` in the WHILE condition XPath for arithmetic comparison. **[V3] Hardcoded Stage Path (Audit B3):** The reference implementation hardcodes `HDLASDirectory = "/Stage/AssignSupHDL"` directly in the flow variable expression, bypassing DVM control and breaking the configuration management standard. The `AssignSupStageBasePath` DVM column defined above must be used instead. Verify via `all_vars.txt` grep for `/Stage/AssignSupHDL` literal before deployment.

---

### 3.2 DVM Lookup Standard Pattern

All DVM lookups must use the integration flow variable `$Var_Rice_Id`. **Never use a literal RICE_ID string in any XPath expression.**

```xpath
<!-- CORRECT — uses the runtime variable -->
dvm:lookupValue('Common_Utility_Lookup', 'RICE_ID', $Var_Rice_Id, 'FusionURL', '')

<!-- INCORRECT — literal bypasses runtime RICE_ID variable; do not use -->
dvm:lookupValue('Common_Utility_Lookup', 'RICE_ID', 'I-03', 'FusionURL', '')
```

`$Var_Rice_Id` is set in the first ASSIGNMENT node immediately after the SCHEDULE_RECEIVE (or REST_RECEIVE for `IFS_` sub-integrations), before any DVM lookup executes.

---

### 3.3 Initialization Sequence (Start of Every Integration)

```text
SCHEDULE_RECEIVE (or REST_RECEIVE for IFS_ sub-integrations)
  └─ ASSIGNMENT block (single node, multiple assignments):
       Var_Rice_Id                = "{literal RICE_ID for this integration}"
       Var_Integration_Name       = dvm:lookupValue(..., 'Name', '')
       var_InputFileName          = dvm:lookupValue(..., 'FileName', '')
       var_SourceDirectory        = dvm:lookupValue(..., 'DirectoryName', '')
       var_ArchiveDirectory       = dvm:lookupValue(..., 'ArchiveDirectory', '')
       var_Error_Directory        = dvm:lookupValue(..., 'Error_Directory', '')
       var_ReportPath             = dvm:lookupValue(..., 'ReportAbsolutePath', '')
       var_Environment_Link       = dvm:lookupValue(..., 'FusionURL', '')
       var_EnvironmentType        = dvm:lookupValue(..., 'EnvironmentType', '')
       var_DistributionList       = dvm:lookupValue(..., 'IT_Distribution_List1', '')
       var_HDLStageBase           = dvm:lookupValue(..., 'HDLStageBasePath', '')
       var_MaxPollCount           = dvm:lookupValue(..., 'MaxPollCount', '36')
       var_AssignSupStageBase     = dvm:lookupValue(..., 'AssignSupStageBasePath', '')    ← [V3]
       var_Phase2ReportPath       = dvm:lookupValue(..., 'Phase2ReportAbsolutePath', '') ← [V3]
       var_import_status          = ""
       var_HireStatus             = ""    ← [V3] Phase 2 outcome; initialized here
       var_StartTimestamp         = fn:current-dateTime()
```

> If any lookup returns empty string for a critical field (`FileName`, `DirectoryName`, `FusionURL`), the integration must THROW a `CONFIG_FAIL` fault immediately before any SFTP or HCM operation is attempted. **[V3]** If `AssignSupStageBasePath` returns empty for integrations that process AssignmentSupervisor (Employee domain), also THROW `CONFIG_FAIL` — "AssignSupStageBasePath not configured in DVM for RICE_ID {id}".

---

## 4. FBDI / HDL Pipeline Standard

### 4.1 End-to-End Pipeline Sequence

**[V3] This section has been substantially updated to reflect the two-phase HDL import design. Single-domain integrations (JOB, DEPARTMENT, NAMEMAIL) follow Steps 1–14 only. The Employee domain integration (EMPLO) additionally follows Steps 10a–10g for the AssignmentSupervisor second phase.**

```text
━━━━━━━━━━━━━━━━━━━━━━ PHASE 1: MAIN OBJECTS ━━━━━━━━━━━━━━━━━━━━━━

 Step  1  SCHEDULE_RECEIVE (or REST_RECEIVE for IFS_ domain integrations)
 Step  2  ASSIGNMENT: initialize all config variables from DVM (Section 3.3)
          ASSIGNMENT: var_Phase2AssignSupCount = 0          ← [V3] Phase 2 write counter
          ASSIGNMENT: var_Phase2SkipCount = 0               ← [V3] no-manager skip counter
 Step  3  SFTP ListFile          → capture ItemCount
 Step  4  ROUTER: ItemCount > 0?
              No  → NOTIFY (NO_FILE template) → END (or RETURN for IFS_)
              Yes → continue
 Step  5  SFTP DownloadFile      → stage source CSV to {HDLStageBasePath}/source/
 Step 5a  SOURCE FILE VALIDATION [V2]:
              Verify downloaded file size > 0 bytes
              Optionally: validate header row matches expected CSV schema
              If invalid → THROW CONFIG_FAIL with detail "Source file empty or malformed"
 Step  6  [If source IDs needed] IFS_BIP_SRCID_PREFETCH
                                  → bulk BIP lookup, returns XML result set pre-loop
 Step  7  FOR each record in staged source CSV:
              STAGE_READ         → read record fields into loop variable
              ROUTER (Action?)   → branch per Action code:

                HIRE             → write: Worker.dat, PersonName.dat,
                                          PersonLegislativeData.dat, WorkRelationship.dat,
                                          WorkTerms.dat, Assignment.dat,
                                          ExternalIdentifier.dat,
                                          PersonEmail.dat (only if email field non-empty)

                REHIRE           → write: Worker.dat, WorkRelationship.dat,
                                          WorkTerms.dat, Assignment.dat

                ASG_CHANGE       → write: WorkRelationship.dat, WorkTerms.dat
                                   [V3] See Design Decision Note B2 re: Assignment.dat

                GLB_TRANSFER     → write: WorkRelationship.dat, WorkTerms.dat
                                   [V3] See Design Decision Note B2 re: Assignment.dat

                TERMINATION      → write: WorkRelationship.dat
                                          (TerminationType field required)

                [else/unknown]   → ACTIVITY_STREAM_LOGGER:
                                   {"step":"RECORD_SKIP","action":"{action}",
                                    "personNumber":"{personNumber}",
                                    "reason":"Action not recognised — no HDL written"}
                                   THROW fault code UNEXPECTED_ACTION

 Step  8  STAGE_ZIP              → zip all Phase 1 HDL staging files
 Step  9  IFS_FBDI_SUBMIT_POLL (Phase 1)
              → UCM upload + importAndLoadData + ESS poll + getDataSetStatus
              → returns: ESSStatus, ImportSuccess, ImportFailed,
                         LoadSuccess, LoadFailed, UCMContentId, ESSJobId
 Step 10  ROUTER: Phase 1 outcome?
              total = success AND error = 0  → var_import_status = "SUCCESS"
              total = error AND success = 0  → var_import_status = "ERROR"
              error > 0 AND error < total    → var_import_status = "PARTIAL"
              total = 0                      → var_import_status = "NO_RECORDS"
              [V3] All four branches must be explicit; no empty otherwise branch.

━━━━━━━━━━━━━━━━━━━━━━ PHASE 2: ASSIGNMENT SUPERVISOR ━━━━━━━━━━━━━━
         (Employee domain only — skip entirely for JOB, DEPT, NAMEMAIL)

Step 10a  GATE CHECK [V3]:
              ROUTER:
                $var_import_status = "SUCCESS" → proceed to Step 10b
                $var_import_status = "PARTIAL"  → proceed to Step 10b
                  (partial loads: write AssignmentSupervisor for successfully
                   loaded employees only; HDL MERGE is idempotent and safe)
                $var_import_status = "ERROR"    → SKIP Phase 2 → go to Step 11
                $var_import_status = "NO_RECORDS" → SKIP Phase 2 → go to Step 11
              ACTIVITY_STREAM_LOGGER:
                {"step":"PHASE2_GATE","phase1Status":"{var_import_status}",
                 "phase2Will":"RUN" or "SKIPPED"}

Step 10b  FOR each record in source CSV (second pass from AssignSupStageBasePath):
              STAGE_READ → read record fields into EachAssignSup variable
              ROUTER:
                (Action = "HIRE" or "ASG_CHANGE" or "REHIRE" or "GLB_TRANSFER")
                AND Manager_Person_Number != ''
                  → write AssignmentSupervisor.dat to {AssignSupStageBasePath}/
                     ROUTER: AssignSup_Header = "False"?
                       Yes → write METADATA header line first; set AssignSup_Header = "True"
                       No  → append MERGE line only
                     ASSIGNMENT: var_Phase2AssignSupCount = var_Phase2AssignSupCount + 1

                (Action = "HIRE" or "ASG_CHANGE" or "REHIRE" or "GLB_TRANSFER")
                AND Manager_Person_Number = ''
                  → ACTIVITY_STREAM_LOGGER [V3]:
                     {"step":"ASSUP_SKIP_NO_MANAGER","action":"{action}",
                      "personNumber":"{personNumber}",
                      "reason":"Manager_Person_Number is blank — AssignmentSupervisor not written"}
                     ASSIGNMENT: var_Phase2SkipCount = var_Phase2SkipCount + 1

                TERMINATION or unknown
                  → no AssignmentSupervisor written (correct by design)

Step 10c  ACTIVITY_STREAM_LOGGER [V3]:
              {"step":"PHASE2_ASSUP_WRITTEN",
               "assignSupWritten":"{var_Phase2AssignSupCount}",
               "skippedNoManager":"{var_Phase2SkipCount}"}
              ROUTER: var_Phase2AssignSupCount = 0?
                Yes → SKIP Phase 2 load (no records to submit) → go to Step 11
                      ASSIGNMENT: var_HireStatus = "NO_RECORDS"
                No  → continue to Step 10d

Step 10d  STAGE_ZIP → zip AssignmentSupervisor.dat from {AssignSupStageBasePath}/

Step 10e  IFS_FBDI_SUBMIT_POLL (Phase 2) [V3]:
              → UCM upload + importAndLoadData + ESS poll + getDataSetStatus
              → returns: Phase2_ESSStatus, Phase2_ImportSuccess, Phase2_ImportFailed,
                         Phase2_LoadSuccess, Phase2_LoadFailed,
                         Phase2_UCMContentId, Phase2_ESSJobId

Step 10f  ROUTER: Phase 2 outcome? [V3]
              total = success AND error = 0  → var_HireStatus = "SUCCESS"
              total = error AND success = 0  → var_HireStatus = "ERROR"
              error > 0 AND error < total    → var_HireStatus = "PARTIAL"
              total = 0                      → var_HireStatus = "NO_RECORDS"

Step 10g  IFS_ERROR_REPORT (Phase 2) [V3]:
              → BIP Phase 2 error report (using var_Phase2ReportPath)
              → UCM upload → capture Phase2_UCMErrorDocId
              → ACTIVITY_STREAM_LOGGER:
                {"step":"PHASE2_ERROR_REPORT","phase2UCMDocId":"{phase2UCMErrorDocId}",
                 "phase2ErrorUrl":"{phase2ErrorReportUrl}"}

━━━━━━━━━━━━━━━━━━━━━━ POST-PROCESSING ━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Step 11  IFS_ERROR_REPORT (Phase 1)
              → BIP Phase 1 error report + UCM upload + audit log write
 Step 12  SFTP MoveFile → archive source CSV:
              Phase1 SUCCESS AND Phase2 SUCCESS/NO_RECORDS → ArchiveDirectory
              Any PARTIAL or ERROR (either phase)         → Error_Directory
 Step 13  NOTIFY: outcome email (see Section 8.2 for combined Phase 1 + Phase 2 templates)
 Step 14  [Global CATCH_ALL] → see Section 6
```

> **[V3] Design Decision Note B2 — Assignment.dat for ASG_CHANGE and GLB_TRANSFER:**
> The reference implementation (`IFP_EMPLO_FROM_DATAW_TO_ORACL` v01.00.0011) does **not** write `Assignment.dat` for `ASG_CHANGE` or `GLB_TRANSFER` actions — only `WorkRelationship.dat` and `WorkTerms.dat` are written. This deviates from the V1/V2 blueprint specification (which stated Assignment is written for these actions).
>
> **Architectural context:** Oracle HCM's HDL engine may cascade certain WorkTerms-level changes (e.g., `AssignmentStatusTypeCode`, `BusinessUnitShortCode`) to the linked Assignment record automatically. However, Assignment-specific attributes (e.g., `GradeCode`, `JobCode`, `LocationCode` at assignment level, `PrimaryAssignmentFlag`) require an explicit `Assignment.dat` row to be updated.
>
> **Required team decision:** Validate against the specific HCM attribute mapping in scope. If all changed attributes are WorkTerms-level only, the current implementation is correct. If any Assignment-level attributes change on ASG_CHANGE/GLB_TRANSFER, add `Assignment.dat` write for these actions (register as Defect D16 in this case).
>
> Until this validation is completed, the pipeline spec shows WorkRelationship + WorkTerms only for these actions, matching the confirmed reference implementation.

---

### 4.2 Shared Sub-Integration: `IFS_FBDI_SUBMIT_POLL`

This sub-integration encapsulates the entire FBDI submission chain. It is invoked **twice** by the Employee domain integration (Phase 1 and Phase 2) and once by all other domain integrations.

**[V3] The `processLabel` input parameter must differentiate the two invocations for activity stream traceability.** Use `"EMPLO_PHASE1"` for the first call and `"EMPLO_ASSUP_PHASE2"` for the second call.

**Input parameters:**

| Parameter | Type | Description |
| --- | --- | --- |
| `zipStagePath` | String | OIC stage path of the prepared HDL zip |
| `hdlDocumentName` | String | UCM document name for the upload |
| `processLabel` | String | Domain/phase label for activity stream logging — e.g., `EMPLO_PHASE1`, `EMPLO_ASSUP_PHASE2` |

**Internal sequence:**

```text
1. HCM FileUpload (UCM)      → upload zip; capture UCMContentId
2. HCM importAndLoadData     → trigger HDL ESS job; capture ESSRequestId
3. ASSIGNMENT: pollCount = 0; maxPolls = number({input maxPollCount})
4. WHILE loop (see Section 5 for full standard)
5. ROUTER: maxPolls exceeded? → THROW ESS_POLL_TIMEOUT
6. HCM getDataSetStatus      → extract ImportSuccess, ImportFailed, LoadSuccess, LoadFailed
7. RETURN structured output:
     ESSStatus, ImportSuccess, ImportFailed, LoadSuccess, LoadFailed, UCMContentId, ESSJobId
```

---

### 4.3 Shared Sub-Integration: `IFS_BIP_SRCID_PREFETCH`

Eliminates the O(n × k) per-record BI Publisher call pattern by performing **one bulk BIP call before the FOR loop**.

**Problem it solves:** Without this sub-integration, each non-HIRE record requires individual BIP calls inside the loop to retrieve existing HCM source system IDs. For a 500-record file, this generates 2,500 sequential SOAP calls.

**Input parameters:**

| Parameter | Type | Description |
| --- | --- | --- |
| `personNumberList` | String | Comma-delimited list of all Person Numbers in the source file |
| `reportAbsolutePath` | String | BIP report path for bulk source ID lookup |

**Internal sequence:**

```text
1. BIP runReport             → pass personNumberList as parameter
                               returns XML: list of {PersonNumber, WR_SourceSystemId,
                               WT_SourceSystemId, WA_SourceSystemId, Manager_PersonNumber}
2. RETURN: XML result as response payload (for small datasets ≤ 5,000 records)
   — or —
   STAGE_WRITE               → write XML result to {HDLStageBasePath}/srcids/srcids.xml
   RETURN: stagePath of the written file (for large datasets)
```

> **[V2] BIP Parameter Size Constraint (Audit A5):** Maximum `personNumberList`: **1,000 person numbers per call**. For files exceeding 1,000 records: invoke `IFS_BIP_SRCID_PREFETCH` in a chunked loop (batches of 1,000). The BIP report SQL should use a table function or `XMLTABLE` to parse the comma-delimited input rather than a raw `IN` clause (Oracle DB `IN` clause limit: 1,000 items). **[V2] Source ID XML Access Pattern (Audit A7):** XPath key lookup inside the FOR loop: `$srcIdDoc/SourceSystemIDs/Person[PersonNumber = $currentPersonNumber]/WR_SourceSystemId`. OIC flow variables have a practical size limit (~5 MB); use stage file for larger datasets.

---

### 4.4 HDL File Format Standard

**File naming convention:**

```text
Source CSV (downloaded)        : {FileName}_{YYYYMMDD}.csv
Phase 1 HDL data file          : {Domain}_{YYYYMMDD}_{HHmmss}.dat
Phase 1 HDL zip (UCM upload)   : {Domain}_HDL_{YYYYMMDD}.zip
Phase 2 AssignSup HDL dat      : AssignmentSupervisor_{YYYYMMDD}_{HHmmss}.dat
Phase 2 AssignSup zip (UCM)    : AssignSup_HDL_{YYYYMMDD}.zip
Phase 1 error log zip          : {Domain}_Error_{YYYYMMDD}.zip
Phase 2 error log zip          : AssignSup_ImportError_{OICInstanceId}.zip    ← [V3]
Monthly audit log (SFTP)       : /Audit/{Domain}/audit_{YYYY-MM}.csv
```

**HDL section structure (pipe-delimited):**

```text
METADATA|Worker|SourceSystemOwner|SourceSystemId|EffectiveStartDate|PersonNumber|...
MERGE|HRC_SQLLOADER|PER_1001|2026-01-01|1001|...
MERGE|HRC_SQLLOADER|PER_1002|2026-01-01|1002|...
```

**Business key rules:**

| Field | Standard Value | Notes |
| --- | --- | --- |
| `SourceSystemOwner` | `HRC_SQLLOADER` | Constant for all integrations in this layer |
| `SourceSystemId` | `concat('PER_', PersonNumber)` | For Worker-level objects |
| HDL Action | `MERGE` | Idempotent — safe to re-run; do not use CREATE or UPDATE |

**[V3] Date format:** All date fields in HDL `.dat` files use `YYYY/MM/DD` format (e.g., `2026/01/15`). Open-ended dates use `4712/12/31`. This is confirmed from NXSD schema samples in the reference implementation. Timestamps (where required) use `YYYY/MM/DD HH:MM:SS`.

---

### 4.5 HDL Object Dependency Load Order **[V3 CORRECTED]**

HDL requires objects in dependency order. **Objects submitted out of order will cause HDL to fail with a dependency error.** The Employee domain uses two separate `importAndLoadData` submissions.

**Phase 1 — ZIP #1 (submitted first; must reach SUCCEEDED or WARNING before Phase 2):**

```text
Load Order    HDL Object                 Action Codes That Write This Object
──────────    ──────────────────────     ─────────────────────────────────────────────────
     1        Worker                     HIRE, REHIRE
     2        PersonName                 HIRE
     3        PersonLegislativeData      HIRE
     4        PersonEmail                HIRE (when email field is non-empty)
     5        WorkRelationship           HIRE, REHIRE, ASG_CHANGE, GLB_TRANSFER, TERMINATION
     6        WorkTerms                  HIRE, REHIRE, ASG_CHANGE, GLB_TRANSFER
     7        Assignment                 HIRE, REHIRE
              [Note: see Design Decision Note B2 re ASG_CHANGE, GLB_TRANSFER]
     8        ExternalIdentifier         HIRE
```

**Phase 2 — ZIP #2 (submitted separately; only after Phase 1 SUCCEEDED or WARNING):**

```text
Load Order    HDL Object                 Condition
──────────    ──────────────────────     ─────────────────────────────────────────────────
     1        AssignmentSupervisor       HIRE, REHIRE, ASG_CHANGE, GLB_TRANSFER
                                         AND Manager_Person_Number is non-empty
```

> **[V3] Why AssignmentSupervisor is Phase 2 (Audit B1):** `AssignmentSupervisor` references two Assignment records — the employee's (from Phase 1) and the manager's (from Phase 1 or a prior run). Oracle HCM requires the referenced Assignment records to be fully committed before the supervisor link can be created. Submitting AssignmentSupervisor in the same ZIP as the Assignment it references causes a dependency resolution failure in HDL because the HDL engine processes files within a single zip sequentially but the Assignment commitment is not guaranteed before the supervisor link validation runs. **[V3] ExternalIdentifier position (Audit B5):** ExternalIdentifier is included in Phase 1 (position 8 in ZIP #1), not after AssignmentSupervisor. The V1/V2 load order table listed it at position 9 after AssignmentSupervisor, which was misleading. ExternalIdentifier depends on Worker (Person), not on AssignmentSupervisor.

---

### 4.6 HDL File Generation Strategy

For complex integrations with multiple HDL object types, two implementation patterns are available.

#### Option A: Single-Pass XSL Transformation (Preferred When No Per-Record Conditional Logic)

Apply a single XSLT map to the **entire staged source CSV** to produce each HDL section in one transform.

```xslt
<!-- Worker section: emits METADATA once, then one MERGE line per Worker element -->
<xsl:if test="count(/SourceData/Record[Action='HIRE']) &gt; 0">
METADATA|Worker|SourceSystemOwner|SourceSystemId|EffectiveStartDate|PersonNumber|...
  <xsl:for-each select="/SourceData/Record[Action='HIRE']">
MERGE|HRC_SQLLOADER|<xsl:value-of select="concat('PER_', PersonNumber)"/>|...
  </xsl:for-each>
</xsl:if>
```

**Benefits:** No FOR loop; METADATA header only emitted when records exist; all mappings in one XSL file.

> **Recommendation:** Prefer Option A for integrations where all records follow the same mapping (JOB, DEPARTMENT). Use Option B for Action-code branching (EMPLOYEE with HIRE/REHIRE/TERMINATION/ASG_CHANGE/GLB_TRANSFER).

#### Option B: FOR Loop with STAGE_WRITE Append (Required When Per-Record Logic Is Needed)

> **[V2] OIC Platform Constraint (Audit A4):** OIC App-Driven Orchestration does **not** support incrementally appending XML elements to a flow variable inside a FOR loop. Use STAGE_WRITE in append mode instead.

```text
FOR each record:
  STAGE_READ → read record fields
  ROUTER (Action?) → determine which HDL objects apply
  STAGE_WRITE (append=true) → append pipe-delimited HDL line(s) to the .dat file
```

The METADATA header is written **once before the FOR loop** (or using a header-flag variable on first qualifying record).

---

### 4.7 Re-Run Safety and Idempotency **[V2]**

The `MERGE` HDL action provides **record-level idempotency** — re-processing the same employee record produces the same result in Oracle HCM. However, this does not prevent duplicate ESS job submissions.

**Mitigation strategies (implement at least one):**

| Strategy | Implementation | Trade-off |
| --- | --- | --- |
| **SFTP marker file** | After `importAndLoadData`, write a zero-byte `{FileName}_{YYYYMMDD}.submitted` marker. Check before Step 3. | Requires marker cleanup on completion |
| **Audit log check** | Before SFTP download, check audit log for matching `SourceFile` + date with `FinalStatus != FAULT`. | Depends on audit log written on partial success |
| **Accept duplicate MERGE** | Document that duplicate `MERGE` is safe; accept duplicate ESS job. | Simplest; acceptable if HDL queue capacity allows |

**[V3] Phase 2 re-run safety:** If Phase 1 succeeds but Phase 2 fails, a re-run will re-submit Phase 1 (duplicate ESS job — safe via MERGE). Phase 2 will also re-run. The SFTP marker file strategy must cover both phases — write the marker only after **both phases** complete successfully.

---

### 4.8 Source File Validation **[V2]**

After SFTP DownloadFile (Step 5) and before the FOR loop or XSL transform:

| Check | Method | Failure Action |
| --- | --- | --- |
| File size > 0 bytes | Stage file `size` property from download response | THROW `CONFIG_FAIL` — "Source file is empty (0 bytes)" |
| Header row matches expected schema | STAGE_READ first row; compare column count | THROW `CONFIG_FAIL` — "Source file header mismatch" |
| Record count within ceiling | Count records from STAGE_READ total | If > 10,000: log warning. Oracle recommends ≤ 10,000 records per HDL batch. |

---

### 4.9 AssignmentSupervisor Phase 2 — Design Standard **[V3]**

This section specifies the complete design requirements for the Phase 2 AssignmentSupervisor HDL submission that must be present in every Employee domain integration.

#### 4.9.1 Conditional Write Logic

AssignmentSupervisor rows are written **only when all three conditions are true simultaneously:**

1. Action code is one of: `HIRE`, `REHIRE`, `ASG_CHANGE`, `GLB_TRANSFER`
2. `Manager_Person_Number` field in the source CSV is non-empty (`!= ''`)
3. Phase 1 gate passed (Step 10a) — `var_import_status` is `SUCCESS` or `PARTIAL`

Records where `Manager_Person_Number` is blank must be explicitly logged (Step 10b) and counted in `var_Phase2SkipCount`. This count must appear in the Phase 2 activity stream entry (Step 10c) and the audit log.

#### 4.9.2 Phase 2 Outcome Classification

| Condition | `var_HireStatus` Value |
| --- | --- |
| `Phase2_total = Phase2_success AND Phase2_error = 0` | `SUCCESS` |
| `Phase2_total = Phase2_error AND Phase2_success = 0` | `ERROR` |
| `Phase2_error > 0 AND Phase2_error < Phase2_total` | `PARTIAL` |
| `Phase2_total = 0` (no AssignSup records written) | `NO_RECORDS` |
| Phase 2 not attempted (Phase 1 ERROR/NO_RECORDS) | `SKIPPED` |

#### 4.9.3 Combined Outcome Determination

The final integration outcome used for notifications and the audit log is the intersection of Phase 1 and Phase 2:

| `var_import_status` (Phase 1) | `var_HireStatus` (Phase 2) | Final Email Template |
| --- | --- | --- |
| SUCCESS | SUCCESS | SUCCESS |
| SUCCESS | NO_RECORDS | SUCCESS (note: no managers in source) |
| SUCCESS | PARTIAL | PHASE2_PARTIAL |
| SUCCESS | ERROR | PHASE2_PARTIAL |
| SUCCESS | SKIPPED | SUCCESS |
| PARTIAL | SUCCESS | PARTIAL (Phase 1 partial) |
| PARTIAL | PARTIAL or ERROR | ERROR (both phases degraded) |
| ERROR | SKIPPED | ERROR |
| NO_RECORDS | SKIPPED | NO_FILE (or NO_RECORDS) |

#### 4.9.4 Phase 2 Data Variables

| Variable | Populated From | Usage |
| --- | --- | --- |
| `var_HireStatus` | Phase 2 outcome router | Gate check; notification; audit log |
| `var_Phase2AssignSupCount` | Counter incremented in Step 10b FOR loop | Activity stream; audit log |
| `var_Phase2SkipCount` | Counter incremented on no-manager skip | Activity stream; audit log |
| `var_Phase2ESSJobId` | `IFS_FBDI_SUBMIT_POLL` Phase 2 response | Activity stream; audit log |
| `var_Phase2ESSStatus` | Phase 2 ESS poll terminal status | Activity stream; audit log |
| `var_Phase2UCMContentId` | Phase 2 UCM upload response | Audit log |
| `var_Phase2UCMErrorDocId` | Phase 2 error report UCM upload | Notification email |
| `var_Phase2ErrorReportUrl` | `concat($var_Environment_Link, 'cs/idcplg?IdcService=GET_FILE&amp;dID=', $var_Phase2UCMErrorDocId)` | Notification email |

---

## 5. ESS Polling Standard

### 5.1 Correct ESS Status Model for HCM Data Loader Jobs

| Status | Category | WHILE Loop Action |
| --- | --- | --- |
| `Not Started` | In-progress | Continue polling |
| `READY` | In-progress | Continue polling |
| `WAIT` | In-progress | Continue polling |
| `RUNNING` | In-progress | Continue polling |
| `PAUSED` | In-progress | Continue polling |
| `SUCCEEDED` | **Terminal — normal** | **Exit loop** |
| `ERROR` | **Terminal — error** | **Exit loop** |
| `WARNING` | **Terminal — partial success** | **Exit loop** |
| `CANCELLED` | **Terminal — cancelled** | **Exit loop + alert** |
| `BLOCKED` | **Terminal — blocked** | **Exit loop + alert** |
| `COMPLETED` | **[V3] Defensive terminal** | **Exit loop — treat as WARNING** |

> **[V2] `COMPLETED` guidance (Audit A10):** `COMPLETED` is not a documented terminal status for HCM Data Loader jobs in Oracle's HDL documentation. Do **not** include it in the WHILE continue condition.
>
> **[V3] Defensive handling (Audit B6):** The reference implementation **incorrectly** includes `COMPLETED` in the WHILE **continue** condition, causing the loop to keep polling if HCM returns this status — ultimately throwing `ESS_POLL_TIMEOUT` after 36 polls instead of exiting gracefully. The correct design: do **not** list `COMPLETED` in the WHILE loop condition (so the loop exits naturally if HCM returns it). In the POST-WHILE router, add an explicit branch for `$HDLJobStatus = "COMPLETED"` — treat as equivalent to `WARNING` and proceed to `getDataSetStatus`. Log a warning entry `{"step":"ESS_UNEXPECTED_TERMINAL","status":"COMPLETED","action":"treated_as_WARNING"}`. If your HCM version consistently returns `COMPLETED` for HDL jobs, raise an Oracle SR to clarify the correct terminal status enumeration for your release.

---

### 5.2 Standard WHILE Loop Implementation

```text
ASSIGNMENT:  pollCount   = 0
ASSIGNMENT:  maxPolls    = number($var_MaxPollCount)   ← cast string from DVM to number
ASSIGNMENT:  HDLJobStatus = "Not Started"

WHILE condition (XPath):
  (
    $HDLJobStatus = "Not Started"  or
    $HDLJobStatus = "READY"        or
    $HDLJobStatus = "WAIT"         or
    $HDLJobStatus = "RUNNING"      or
    $HDLJobStatus = "PAUSED"
  )
  and ($pollCount < $maxPolls)
  ← Note: COMPLETED is NOT listed here — loop exits naturally if received

  WHILE body:
    WAIT: 10 seconds
    HCM getESSJobStatus → update $HDLJobStatus from response
    ASSIGNMENT: pollCount = $pollCount + 1
    [Log only on first poll and every 10th poll — not every iteration]

POST-WHILE:
  ROUTER (three explicit branches):

  Branch 1 — Timeout:
    $pollCount >= $maxPolls
    and ($HDLJobStatus = "Not Started" or $HDLJobStatus = "READY"
         or $HDLJobStatus = "WAIT" or $HDLJobStatus = "RUNNING"
         or $HDLJobStatus = "PAUSED")
      → THROW fault: code=ESS_POLL_TIMEOUT
                     detail="ESS Job {essJobId}: status={HDLJobStatus} after {pollCount} polls"

  Branch 2 — Unexpected COMPLETED [V3]:
    $HDLJobStatus = "COMPLETED"
      → ACTIVITY_STREAM_LOGGER:
         {"step":"ESS_UNEXPECTED_TERMINAL","status":"COMPLETED",
          "action":"treated_as_WARNING","pollCount":"{pollCount}"}
      → continue to getDataSetStatus (same as WARNING path)

  Branch 3 — Normal terminal (SUCCEEDED, ERROR, WARNING, CANCELLED, BLOCKED):
    → continue to getDataSetStatus
```

> **[V2] XPath syntax:** Use lowercase `and` and `or`. Uppercase `AND`/`OR` is invalid XPath.
>
> **[V2] Activity stream budget (Audit A6):** Log on first poll and loop exit only. Per-iteration logging saturates the OIC activity stream (100–200 entry limit per instance).

---

### 5.3 ESS Service Binding — HCM Cloud Adapter Standard

All HCM FBDI integrations use the **HCM Cloud Adapter** to invoke the `getESSJobStatus` operation through the Oracle `ErpIntegrationService` service catalog.

> **[V2] Correction (Audit A1):** ALL four reference integrations use identical HCM adapter configuration: `adapter="hcm"`, service `{...erpIntegrationService/}ErpIntegrationService`, operation `getESSJobStatus`. The HCM Cloud Adapter surfaces this through its built-in service catalog — it is not the same as configuring a standalone SOAP adapter with a separate WSDL.

**Naming consistency rule:** Use `GetEssJobStatus` as the JCA reference name consistently across all integrations. Do not use `getCurrentESSStatus` or other variants.

---

## 6. Error Handling Hierarchy Standard

### 6.1 Three-Level Fault Architecture

```text
Level 1: Adapter-level CATCH (scoped per major adapter operation)
│
├─ CATCH FTP_FAULT (around ListFile, DownloadFile, MoveFile):
│    ACTIVITY_STREAM_LOGGER: {step, faultCode, faultDetail}
│    SFTP MoveFile → Error_Directory   (best-effort archive)
│    NOTIFY: SFTP_ERROR template
│    THROW  → re-raise to Level 2
│
├─ CATCH HCM_FAULT (around UCM upload, importAndLoadData, getESSJobStatus):
│    ACTIVITY_STREAM_LOGGER: {step, essJobId, faultCode, faultDetail, phase}  ← [V3] add phase
│    THROW  → re-raise to Level 2
│
└─ CATCH BIP_FAULT (around all BI Publisher runReport calls):
     ACTIVITY_STREAM_LOGGER: {step, reportPath, faultCode, faultDetail}
     THROW  → re-raise to Level 2

─────────────────────────────────────────────────────────────────

Level 2: Scope-level CATCH (wraps the FOR loop and FBDI chain as one scope)
│
└─ CATCH any fault from Level 1:
     ASSIGNMENT: var_FaultCode   = fault code from Level 1
     ASSIGNMENT: var_FaultDetail = fault message from Level 1
     ASSIGNMENT: var_FailedStep  = label of the step that threw
     ASSIGNMENT: var_FailedPhase = "PHASE1" or "PHASE2"    ← [V3]
     THROW  → re-raise to Level 3

─────────────────────────────────────────────────────────────────

Level 3: Global CATCH_ALL
│
└─ Actions (executed in order):
     1. ACTIVITY_STREAM_LOGGER:
           {"step":"GLOBAL_FAULT","instanceId":"{instanceId}",
            "faultCode":"{var_FaultCode}","faultDetail":"{var_FaultDetail}",
            "failedStep":"{var_FailedStep}","failedPhase":"{var_FailedPhase}",  ← [V3]
            "timestamp":"{timestamp}"}
     2. SFTP MoveFile → Error_Directory
           (ensures source file is always archived regardless of failure point)
     3. SFTP AUDIT WRITE
           (append error/fault record to monthly audit log — see Section 9)
     4. NOTIFY: FAULT template (includes failed phase in subject)
     5. END (or RETURN error response for IFS_ sub-integrations)
```

---

### 6.2 Fault Code Reference

| Fault Code | Trigger Condition |
| --- | --- |
| `CONFIG_FAIL` | DVM lookup returned empty for a required field |
| `SFTP_LIST_FAIL` | ListFile SFTP operation failed |
| `SFTP_DOWNLOAD_FAIL` | DownloadFile SFTP operation failed |
| `SFTP_ARCHIVE_FAIL` | MoveFile (archival) SFTP operation failed |
| `UCM_UPLOAD_FAIL` | FileUpload to Oracle WebCenter Content failed |
| `HDL_SUBMIT_FAIL` | `importAndLoadData` SOAP operation returned a fault |
| `ESS_POLL_TIMEOUT` | ESS job did not reach terminal status within `maxPolls` iterations |
| `ESS_POLL_FAIL` | `getESSJobStatus` SOAP operation returned a fault |
| `BIP_REPORT_FAIL` | BI Publisher `runReport` SOAP operation returned a fault |
| `XSLT_MAP_FAIL` | XSL transformation threw an exception |
| `UNEXPECTED_ACTION` | Source record contains an unrecognized `Action` code value |
| `SOURCE_INVALID` | Source file is empty (0 bytes) or header schema mismatch **[V2]** |
| `PHASE2_CONFIG_FAIL` | **[V3]** `AssignSupStageBasePath` or `Phase2ReportAbsolutePath` empty in DVM |
| `PHASE2_UCM_FAIL` | **[V3]** Phase 2 UCM upload for AssignmentSupervisor zip failed |
| `PHASE2_HDL_SUBMIT_FAIL` | **[V3]** Phase 2 `importAndLoadData` returned a fault |
| `PHASE2_ESS_POLL_TIMEOUT` | **[V3]** Phase 2 ESS job exceeded `maxPolls` |

---

### 6.3 Router Branch Completeness Rule

**Every content-based router must have an explicit action on every branch, including the else/otherwise branch.**

| Branch scenario | Required action |
| --- | --- |
| Known, processed `Action` code | Normal processing flow |
| Known, intentionally skipped `Action` code | `ACTIVITY_STREAM_LOGGER` — log the skip with reason |
| Unknown or unexpected `Action` code | `THROW` fault — code `UNEXPECTED_ACTION` |

> **Zero empty branches permitted.** Enforce via the OIC project health gate: `projectHasWarnings = false` at deploy time.

---

## 7. Business Identifiers and Activity Logging Standard

### 7.1 Mandatory Business Identifiers (OIC Monitoring)

Configure exactly **3 business identifiers** on every HCM integration via the OIC Designer tracking panel.

| Position | Label | XPath Expression | Example Value |
| --- | --- | --- | --- |
| 1 | `Source_File` | `$var_InputFileName` | `employees_20260514.csv` |
| 2 | `Run_Date` | `$var_RunDate` (pre-computed — see note) | `2026-05-14` |
| 3 | `Domain` | Literal constant per integration | `EMPLOYEE`, `JOB`, `DEPARTMENT`, `NAME_EMAIL` |

> **Implementation note:** Pre-compute `$var_RunDate = fn:format-dateTime(fn:current-dateTime(),'[Y0001]-[M01]-[D01]')` in the initialization ASSIGNMENT block and reference `$var_RunDate` as the tracking expression. Some OIC versions reject function calls directly in the tracking panel. **[V3] ESS Job ID cross-reference (Audit B9):** OIC supports only 3 business identifier fields. When an operator locates an HDL ESS Job ID in the HCM console and needs to find the originating OIC instance, they must query the SFTP audit log by `ESSJobId` column (see Section 9.1). The ESS Job ID is not directly searchable in OIC Monitoring. This is a known platform constraint; document it in the operations runbook.

---

### 7.2 Mandatory Activity Stream Entries **[V3 EXTENDED]**

All messages must be **valid JSON strings**. Phase 1 milestones are numbered 1–6. Phase 2 milestones use suffix `b` (Employee domain only).

**Phase 1 Milestones:**

| # | Trigger Point | Required JSON Fields |
| --- | --- | --- |
| 1 | After SFTP ListFile | `step`, `status` (`FOUND`/`NOT_FOUND`), `file`, `itemCount` |
| 2 | After Phase 1 `importAndLoadData` | `step:"PHASE1_HDL_SUBMIT"`, `essJobId`, `ucmDocId`, `hdlZipFile` |
| 3 | After Phase 1 ESS WHILE exits | `step:"PHASE1_ESS_COMPLETE"`, `essJobId`, `essStatus`, `pollCount`, `elapsedSeconds` |
| 4 | After Phase 1 `getDataSetStatus` | `step:"PHASE1_DATASET_STATUS"`, `importSuccess`, `importFailed`, `loadSuccess`, `loadFailed` |
| 5 | After Phase 1 error report UCM upload | `step:"PHASE1_ERROR_REPORT"`, `errorLogDocId`, `errorReportUrl` |
| 6 | At integration end (before NOTIFY) | `step:"INTEGRATION_COMPLETE"`, `phase1Status`, `phase2Status`, `archivePath`, `instanceId`, `timestamp` |

**Phase 2 Milestones (Employee domain only) — [V3]:**

| # | Trigger Point | Required JSON Fields |
| --- | --- | --- |
| 2b | Phase 2 gate decision (Step 10a) | `step:"PHASE2_GATE"`, `phase1Status`, `phase2Will:RUN/SKIPPED` |
| 2c | After Phase 2 AssignSup FOR loop (Step 10c) | `step:"PHASE2_ASSUP_WRITTEN"`, `assignSupWritten`, `skippedNoManager` |
| 2d | After Phase 2 `importAndLoadData` (Step 10e) | `step:"PHASE2_HDL_SUBMIT"`, `essJobId`, `ucmDocId`, `hdlZipFile:"AssignSup"` |
| 3b | After Phase 2 ESS WHILE exits (Step 10e) | `step:"PHASE2_ESS_COMPLETE"`, `essJobId`, `essStatus`, `pollCount` |
| 4b | After Phase 2 `getDataSetStatus` (Step 10e) | `step:"PHASE2_DATASET_STATUS"`, `importSuccess`, `importFailed`, `loadSuccess`, `loadFailed` |
| 5b | After Phase 2 error report UCM upload (Step 10g) | `step:"PHASE2_ERROR_REPORT"`, `phase2UCMDocId`, `phase2ErrorUrl` |

**Example — Milestone 3b:**

```json
{
  "step": "PHASE2_ESS_COMPLETE",
  "essJobId": "98765432",
  "essStatus": "SUCCEEDED",
  "pollCount": "3",
  "elapsedSeconds": "31"
}
```

**Example — Milestone 2c:**

```json
{
  "step": "PHASE2_ASSUP_WRITTEN",
  "assignSupWritten": "47",
  "skippedNoManager": "3"
}
```

> **[V2] Activity Stream Budget (Audit A6):** OIC limits visible activity stream entries per instance (typically 100–200). For the Employee domain with Phase 2, budget: 6 Phase 1 milestones + 6 Phase 2 milestones + 2–4 ESS poll entries + fault handler entries = ~18–20 planned entries. Remaining capacity (~180) handles RECORD_SKIP and ASSUP_SKIP_NO_MANAGER per-record entries — summarize these with counters rather than one entry per record for large files.

---

## 8. Notification Design Standard

### 8.1 Notification Parameter Mapping Reference

| Template Placeholder | Flow Variable | Populated From |
| --- | --- | --- |
| `{integrationName}` | `$Var_Integration_Name` | DVM lookup — `Name` column |
| `{environmentType}` | `$var_EnvironmentType` | DVM lookup — `EnvironmentType` column |
| `{runDate}` | `$var_RunDate` | Pre-computed in initialization block |
| `{fileName}` | `$var_InputFileName` | DVM lookup — `FileName` column |
| `{sourceDirectory}` | `$var_SourceDirectory` | DVM lookup — `DirectoryName` column |
| `{totalRecords}` | `$var_total_recordcount` | Phase 1 `getDataSetStatus` response |
| `{successRecords}` | `$var_sucess_recordcount` | Phase 1 `getDataSetStatus` response |
| `{errorRecords}` | `$var_Error_recordcount` | Phase 1 `getDataSetStatus` response |
| `{essJobId}` | `$var_EssRequestId` | Phase 1 `importAndLoadData` response |
| `{instanceId}` | OIC system variable | Auto-resolved by OIC notification runtime |
| `{errorReportUrl}` | `$var_UCM_ImportError_link` | `concat($var_Environment_Link, 'cs/idcplg?IdcService=GET_FILE&amp;dID=', $var_Document_Id)` |
| `{errorArchivePath}` | `$var_Error_Directory` | DVM lookup — `Error_Directory` column |
| `{faultCode}` | `$var_FaultCode` | Set in Level 2/3 fault handler |
| `{faultDetail}` | `$var_FaultDetail` | Set in Level 2/3 fault handler |
| `{failedStep}` | `$var_FailedStep` | Set in Level 2/3 fault handler |
| `{failedPhase}` | `$var_FailedPhase` | **[V3]** Set in Level 2/3 fault handler — `PHASE1` or `PHASE2` |
| `{phase2Status}` | `$var_HireStatus` | **[V3]** Phase 2 outcome router |
| `{phase2EssJobId}` | `$var_Phase2ESSJobId` | **[V3]** Phase 2 `importAndLoadData` response |
| `{phase2TotalRecords}` | `$var_Phase2_total_recordcount` | **[V3]** Phase 2 `getDataSetStatus` response |
| `{phase2SuccessRecords}` | `$var_Phase2_sucess_recordcount` | **[V3]** Phase 2 `getDataSetStatus` response |
| `{phase2ErrorRecords}` | `$var_Phase2_Error_recordcount` | **[V3]** Phase 2 `getDataSetStatus` response |
| `{phase2ErrorReportUrl}` | `$var_Phase2ErrorReportUrl` | **[V3]** `concat($var_Environment_Link, 'cs/idcplg?IdcService=GET_FILE&amp;dID=', $var_Phase2UCMErrorDocId)` |
| `{assignSupWritten}` | `$var_Phase2AssignSupCount` | **[V3]** Records written to AssignmentSupervisor.dat |
| `{assignSupSkipped}` | `$var_Phase2SkipCount` | **[V3]** Records skipped (no Manager_Person_Number) |

> **[V3] Variable naming note:** `$var_sucess_recordcount` (typo: missing 's') is the actual variable name in both the blueprint and the reference implementation. Changing it would require code changes across all integrations. The typo is retained for consistency but should be documented in the project glossary.

---

### 8.2 Standard Email Notification Templates

#### Template: NO_FILE

```text
Subject : [HCM Integration] [{environmentType}] {integrationName} — No Source File Found ({runDate})

Integration     : {integrationName}
Environment     : {environmentType}
Run Date/Time   : {runDate}
Expected File   : {fileName}
SFTP Directory  : {sourceDirectory}
OIC Instance ID : {instanceId}

No source file was present on SFTP. No records were processed.
```

#### Template: SUCCESS (Both Phases Succeeded)

```text
Subject : [HCM Integration] [{environmentType}] {integrationName} — Completed Successfully ({runDate})

Integration     : {integrationName}
Environment     : {environmentType}
Run Date/Time   : {runDate}
Source File     : {fileName}
OIC Instance ID : {instanceId}

── Phase 1: Worker / Assignment Load ──────────────────
ESS Job ID      : {essJobId}
Records Total   : {totalRecords}
Records Success : {successRecords}
Records Failed  : {errorRecords}
Phase 1 Status  : {phase1Status}

── Phase 2: Assignment Supervisor Load ────────────────
ESS Job ID      : {phase2EssJobId}
Records Written : {assignSupWritten}
Records Skipped (no manager) : {assignSupSkipped}
Phase 2 Success : {phase2SuccessRecords}
Phase 2 Errors  : {phase2ErrorRecords}
Phase 2 Status  : {phase2Status}
```

#### Template: PARTIAL (Phase 1 Partial Load)

```html
<html><body>
<p><strong>[HCM Integration] [{environmentType}] {integrationName} — PARTIAL ({runDate})</strong></p>
<table>
  <tr><td>Integration</td><td>{integrationName}</td></tr>
  <tr><td>Environment</td><td>{environmentType}</td></tr>
  <tr><td>OIC Instance ID</td><td>{instanceId}</td></tr>
  <tr><th colspan="2">Phase 1 — Worker / Assignment</th></tr>
  <tr><td>ESS Job ID</td><td>{essJobId}</td></tr>
  <tr><td>Total</td><td>{totalRecords}</td></tr>
  <tr><td>Success</td><td>{successRecords}</td></tr>
  <tr><td>Failed</td><td>{errorRecords}</td></tr>
</table>
<p>Phase 1 HCM Error Report: <a href="{errorReportUrl}">View Phase 1 import error details in UCM</a></p>
<p>Error log archived at SFTP: {errorArchivePath}</p>
</body></html>
```

#### Template: PHASE2_PARTIAL **[V3]** (Phase 1 Success but Phase 2 AssignmentSupervisor Errors)

```html
<html><body>
<p><strong>[HCM Integration] [{environmentType}] {integrationName} — PHASE 2 ERRORS: Supervisor Links Incomplete ({runDate})</strong></p>
<p><em>Employee records were loaded successfully (Phase 1). However, Assignment Supervisor records
failed partially or entirely (Phase 2). Affected employees have no manager link in Oracle HCM.</em></p>
<table>
  <tr><td>Integration</td><td>{integrationName}</td></tr>
  <tr><td>Environment</td><td>{environmentType}</td></tr>
  <tr><td>OIC Instance ID</td><td>{instanceId}</td></tr>
  <tr><th colspan="2">Phase 1 — Worker / Assignment (SUCCEEDED)</th></tr>
  <tr><td>ESS Job ID</td><td>{essJobId}</td></tr>
  <tr><td>Records Loaded</td><td>{successRecords} of {totalRecords}</td></tr>
  <tr><th colspan="2">Phase 2 — Assignment Supervisor</th></tr>
  <tr><td>ESS Job ID</td><td>{phase2EssJobId}</td></tr>
  <tr><td>Supervisor Records Written</td><td>{assignSupWritten}</td></tr>
  <tr><td>Skipped (no manager field)</td><td>{assignSupSkipped}</td></tr>
  <tr><td>Phase 2 Success</td><td>{phase2SuccessRecords}</td></tr>
  <tr><td>Phase 2 Failed</td><td>{phase2ErrorRecords}</td></tr>
</table>
<p>Phase 2 HCM Error Report: <a href="{phase2ErrorReportUrl}">View AssignmentSupervisor error details in UCM</a></p>
<p><strong>Action required:</strong> Review Phase 2 error report. Re-load AssignmentSupervisor records
after resolving the reported rejections.</p>
</body></html>
```

#### Template: ERROR

```html
<html><body>
<p><strong>[HCM Integration] [{environmentType}] {integrationName} — ERROR ({runDate})</strong></p>
<table>
  <tr><td>Integration</td><td>{integrationName}</td></tr>
  <tr><td>Environment</td><td>{environmentType}</td></tr>
  <tr><td>OIC Instance ID</td><td>{instanceId}</td></tr>
  <tr><td>ESS Job ID</td><td>{essJobId}</td></tr>
  <tr><td>Records Total</td><td>{totalRecords}</td></tr>
  <tr><td>Records Success</td><td>{successRecords}</td></tr>
  <tr><td>Records Failed</td><td>{errorRecords}</td></tr>
</table>
<p>HCM Error Report: <a href="{errorReportUrl}">View import error details in UCM</a></p>
<p>Error log archived at SFTP: {errorArchivePath}</p>
</body></html>
```

#### Template: FAULT

```text
Subject : [HCM Integration] [{environmentType}] {integrationName} — SYSTEM FAULT [{failedPhase}] ({runDate})

Integration     : {integrationName}
Environment     : {environmentType}
Run Date/Time   : {runDate}
OIC Instance ID : {instanceId}
Failed Phase    : {failedPhase}       ← [V3] PHASE1 or PHASE2
Failed Step     : {failedStep}
Fault Code      : {faultCode}
Fault Detail    : {faultDetail}

Action required: Review OIC instance {instanceId} in the monitoring console.
If failedPhase = PHASE2: Phase 1 employee data was loaded successfully.
Only AssignmentSupervisor records were not loaded.
```

---

### 8.3 HTML Email Attribute Rule — Mandatory

Every `href` attribute must use quoted values. The `&` character in URLs must be HTML-encoded as `&amp;`.

```html
<!-- CORRECT -->
<a href="{errorReportUrl}">View error report</a>

<!-- INCORRECT — not rendered as hyperlink by Outlook or Gmail -->
<a href={Content}>click here</a>
```

---

### 8.4 Notification Subject Prefix Convention

```text
[HCM Integration] [{environmentType}] {integrationName} — {outcome} ({runDate})
```

---

## 9. Audit and Traceability Standard

### 9.1 SFTP Audit Log **[V3 EXTENDED]**

Append one CSV record per integration run to the monthly audit file. For the Employee domain, one row captures both Phase 1 and Phase 2 outcomes.

**Path pattern:** `/Audit/{Domain}/audit_{YYYY-MM}.csv`

**CSV structure (extended for Phase 2):**

```csv
RunTimestamp,IntegrationCode,RiceId,SourceFile,Phase1TotalRecords,Phase1SuccessRecords,Phase1FailedRecords,Phase1ESSJobId,Phase1ESSStatus,Phase1FinalStatus,Phase1UCMErrorDocId,Phase2Status,Phase2AssignSupWritten,Phase2SkippedNoManager,Phase2TotalRecords,Phase2SuccessRecords,Phase2FailedRecords,Phase2ESSJobId,Phase2ESSStatus,Phase2UCMErrorDocId,OICInstanceId,FinalCombinedStatus,ArchivePath,DurationSeconds
2026-05-14T22:35:47,IFP_EMPLO,I-03,employees_20260514.csv,1200,1188,12,12345678,SUCCEEDED,PARTIAL,EMPLERR001,SUCCESS,47,3,47,47,0,98765432,SUCCEEDED,,IFP_EMPLO_XXXX,PARTIAL,/Outbound/error/,187
```

**Column definitions:**

| Column | Source |
| --- | --- |
| `RunTimestamp` | `fn:current-dateTime()` at integration start |
| `IntegrationCode` | Integration code constant |
| `RiceId` | `$Var_Rice_Id` |
| `SourceFile` | `$var_InputFileName` |
| `Phase1TotalRecords` | `$var_total_recordcount` |
| `Phase1SuccessRecords` | `$var_sucess_recordcount` |
| `Phase1FailedRecords` | `$var_Error_recordcount` |
| `Phase1ESSJobId` | Phase 1 `importAndLoadData` response |
| `Phase1ESSStatus` | Final `$HDLJobStatus` after Phase 1 WHILE exits |
| `Phase1FinalStatus` | `$var_import_status` |
| `Phase1UCMErrorDocId` | Phase 1 error log UCM doc ID |
| `Phase2Status` | `$var_HireStatus` (`SUCCESS`/`ERROR`/`PARTIAL`/`NO_RECORDS`/`SKIPPED`) |
| `Phase2AssignSupWritten` | `$var_Phase2AssignSupCount` |
| `Phase2SkippedNoManager` | `$var_Phase2SkipCount` |
| `Phase2TotalRecords` | Phase 2 `getDataSetStatus` total |
| `Phase2SuccessRecords` | Phase 2 `getDataSetStatus` load success |
| `Phase2FailedRecords` | Phase 2 `getDataSetStatus` load failed |
| `Phase2ESSJobId` | Phase 2 `importAndLoadData` response |
| `Phase2ESSStatus` | Final Phase 2 ESS terminal status |
| `Phase2UCMErrorDocId` | Phase 2 error log UCM doc ID |
| `OICInstanceId` | OIC system variable |
| `FinalCombinedStatus` | Intersection outcome per Section 4.9.3 table |
| `ArchivePath` | `$var_ArchiveDirectory` or `$var_Error_Directory` |
| `DurationSeconds` | Computed from start/end timestamps |

> For integrations with no Phase 2 (JOB, DEPT, NAMEMAIL), Phase 2 columns are written as empty strings. This keeps the CSV schema uniform across all domain integrations. **[V2] Concurrency Note (Audit A8):** Each domain integration writes to its own `/Audit/{Domain}/` directory. Concurrent manual re-runs of the same domain may cause audit file corruption depending on SFTP server locking behavior. Sequence manual re-runs.

---

### 9.2 End-to-End Traceability: Investigation Procedure **[V3 EXTENDED]**

**Investigation question:** Was employee PersonNumber 1234 synchronized to Oracle HCM, including supervisor link, on 2026-05-14?

| Step | Tool | Action |
| --- | --- | --- |
| 1 | OIC Monitoring | Search `Source_File = "employees_20260514.csv"` → find instance |
| 2 | OIC Monitoring | Check `Domain` = `EMPLOYEE` identifier and overall status |
| 3 | OIC Activity Stream | View Phase 1 milestones 1–6 for employee load trace |
| 4 | OIC Activity Stream | **[V3]** View Phase 2 milestones 2b–5b for supervisor load trace |
| 5 | Notification email | Read Phase 1 and Phase 2 record counts |
| 6 | Email Phase 1 link | Click `{errorReportUrl}` → UCM error report → search PersonNumber 1234 (Phase 1 failure check) |
| 7 | **[V3]** Email Phase 2 link | Click `{phase2ErrorReportUrl}` → UCM Phase 2 error report → search PersonNumber 1234 (supervisor failure check) |
| 8 | SFTP audit log | Cross-check `/Audit/EMPLOYEE/audit_2026-05.csv` — `Phase1FinalStatus`, `Phase2Status`, `Phase2ESSJobId` |
| 9 | **[V3]** HCM Console | Search ESS Job ID from `Phase2ESSJobId` audit column for HCM-side supervisor job details |

> **Silent record drops are eliminated** by the router branch completeness rule (Section 6.3) and the Phase 2 no-manager skip logging (Section 4.9.1). Every record either succeeds, is explicitly logged as skipped with a reason, or raises a named fault.

---

## 10. Deployment and Environment Management Checklist

### Pre-Deployment Checks

| # | Check | Method | Pass Criterion |
| --- | --- | --- | --- |
| 1 | Project warnings | Read `project_messages.json` | Zero `COMPOSER_ACTION_EMPTY_WARNING_ROUTE` entries |
| 2 | Design completion | Read `project.xml` | `projectHasWarnings = false`, `percentComplete = 100` |
| 3 | JCA endpoint scan | PowerShell scan (Section 2.2) | Zero files containing `<property name="endpointURL">` |
| 4 | DVM RICE_ID literals | Search `expr.properties` files | Zero `dvm:lookupValue(... 'RICE_ID', '{literal}' ...)` patterns |
| 5 | Environment type | Grep `all_vars.txt` for `var_Environment_type` | Value is DVM-sourced, not a literal string |
| 6 | ESS WHILE condition | Review WHILE expressions | `COMPLETED` is **not** in the WHILE continue condition; `maxPolls` counter present |
| 7 | Router completeness | Review all router branches | Zero empty `otherwise`/`else` branches |
| 8 | Notification HTML | Review all `notification_body.data` files | All `href` attributes quoted; `&` is `&amp;` |
| 9 | Business identifiers | Review OIC designer tracking panel | 3 identifiers configured per integration |
| 10 | Activity stream — Phase 1 | Count `ACTIVITY_STREAM_LOGGER` nodes (Phase 1) | Minimum 6 milestone entries; all valid JSON |
| 11 | Project descriptions | Review `projectDescription` in `project.xml` | Accurately describes the integration's purpose |
| 12 | Connection status | OIC Connections console | All 3 connections show `CONFIGURED` |
| 13 | **[V2]** ESS JCA naming | Review all `getESSJobStatus` JCA files | Consistent `referenceName = GetEssJobStatus` |
| 14 | **[V2]** Source file validation | Review post-download logic | File size > 0 check exists before FOR loop |
| 15 | **[V2]** DVM MaxPollCount cast | Review WHILE condition XPath | `number()` cast applied to `$var_MaxPollCount` |
| 16 | **[V3]** Phase 2 pipeline | Employee integration only — review Steps 10a–10g | Phase 2 FOR loop, gate check, and STAGE_ZIP present |
| 17 | **[V3]** Phase 2 activity stream | Count Phase 2 `ACTIVITY_STREAM_LOGGER` nodes | Milestones 2b–5b present; all valid JSON |
| 18 | **[V3]** Phase 2 DVM columns | Check `AssignSupStageBasePath`, `Phase2ReportAbsolutePath` | Both columns present and non-empty in DVM for EMPLO RICE_ID |
| 19 | **[V3]** AssignSupStageBasePath not hardcoded | Grep `all_vars.txt` for literal `/Stage/AssignSupHDL` | Zero hardcoded literals — must be DVM-sourced |
| 20 | **[V3]** Phase 2 notification template | Review notification node for PHASE2_PARTIAL | Template present; includes `{phase2ErrorReportUrl}` as quoted `href` |
| 21 | **[V3]** Audit log schema | Review SFTP audit write XPath | All Phase 2 columns present in CSV write expression |
| 22 | **[V3]** `var_HireStatus` initialized | Review initialization ASSIGNMENT block | `var_HireStatus = ""` initialized before any routing |
| 23 | **[V3]** Phase 2 fault codes | Review Phase 2 HCM CATCH block | `PHASE2_HDL_SUBMIT_FAIL`, `PHASE2_ESS_POLL_TIMEOUT` in fault handler |

### Post-Deployment Verification

| # | Verification | Pass Criterion |
| --- | --- | --- |
| 1 | Test run with HIRE sample file | Integration completes without OIC fault |
| 2 | Business identifiers visible | All 3 identifiers appear in OIC Monitoring instance list |
| 3 | Phase 1 activity stream populated | Milestones 1–6 present in instance activity stream |
| 4 | **[V3]** Phase 2 activity stream populated | Milestones 2b–5b present (if manager data in test file) |
| 5 | Notification received — correct template | SUCCESS template received when both phases succeed |
| 6 | **[V3]** PHASE2_PARTIAL notification test | Inject invalid manager PersonNumber in test file; confirm PHASE2_PARTIAL email received |
| 7 | SFTP audit log written — extended schema | One CSV row with all Phase 2 columns populated |
| 8 | Phase 1 source file archived | Source CSV moved to correct SFTP archive directory |
| 9 | Phase 1 UCM error report accessible | Phase 1 error report URL in notification opens UCM document |
| 10 | **[V3]** Phase 2 UCM error report accessible | Phase 2 error report URL in PHASE2_PARTIAL notification opens UCM document |
| 11 | **[V3]** Re-run safety test | Run integration twice with same file; no data corruption; audit log shows two rows |

---

## 11. Known Defect Register (Reference Implementation)

This register documents confirmed defects in the reference implementation. **[V3] D16–D24 are new findings from the V3 technical re-assessment.**

| ID | Severity | Integration | Defect | Root Cause | Standard Rule |
| --- | --- | --- | --- | --- | --- |
| D1 | Critical | IFP_EMPLO, IFR_EMPLOYEENA | 25+ empty router branches — silent record drops | Design incomplete | Section 6.3 |
| D2 | Critical | IFP_EMPLO | 7 BI Publisher JCA files hardcoded to DEV2 endpoint | JCA `endpointURL` overrides connection property | Section 2.2 |
| D3 | High | IFP_EMPLO | `var_Environment_type = "DEV"` hardcoded literal | Hardcoded dev artifact | Section 3.1 |
| D4 | High | All | FusionURL DVM lookup uses literal `'I-03'` not `$Var_Rice_Id` | Copy-paste error | Section 3.2 |
| D5 | High | All | `href={Content}` — unquoted HTML attribute | HTML template not tested | Section 8.3 |
| D6 | High | All | ESS WHILE includes `"COMPLETED"` as continue-polling status | Incorrect ESS status model | Section 5.1, 5.2 |
| D7 | Medium | IFP_EMPLO | Phase 2 outcomes not surfaced in notification or audit trail | No Phase 2 traceability design | Section 4.9, 7.2, 9.1 |
| D8 | Medium | IFP_EMPLO | O(n × 5) BI Publisher calls inside FOR loop | No pre-fetch design | Section 4.3 |
| D9 | Info | IFR_EMPLOYEENA | ESS polling JCA named `getCurrentESSStatus` instead of `GetEssJobStatus` | Inconsistent naming only | Section 5.3 |
| D10 | Low | IFP_JOB, IFR_EMPLO_DEPAR | `projectDescription` copy-pasted from Employee template | Missing review gate | Section 10, Item 11 |
| D11 | Low | IFR_EMPLOYEENA | JCA filename typo: `GetLoadImportStatuis_REQUEST.jca` | Typo; no naming validation | Section 1.2 |
| D12 | Low | IFR_EMPLOYEENA | Notification body typos: "GlobaleFault", "appural action" | No template review | Section 8.2 |
| D13 | Low | IFP_EMPLO, IFR_EMPLOYEENA | Error/success notification missing `{DateTime}` field | Inconsistent template | Section 8.2 |
| D14 | Info | All | Zero business identifiers configured | Feature not used | Section 7.1 |
| D15 | Info | All | Record counts not mapped to notification body | Variables computed but not mapped | Section 8.1 |
| D16 | **Critical** | IFP_EMPLO | **[V3]** Phase 2 `importAndLoadData` ESS job ID and outcome not written to SFTP audit log | Phase 2 audit trail completely absent | Section 9.1 |
| D17 | **Critical** | IFP_EMPLO | **[V3]** Phase 2 AssignmentSupervisor errors not surfaced in any notification email; Phase 1 SUCCESS + Phase 2 ERROR sends a SUCCESS notification | No PHASE2_PARTIAL notification template | Section 8.2, 4.9.3 |
| D18 | **High** | IFP_EMPLO | **[V3]** `HDLASDirectory` hardcoded as literal `"/Stage/AssignSupHDL"` in flow expression; bypasses DVM configuration management | Hardcoded path not in DVM | Section 3.1, 3.3 |
| D19 | **High** | IFP_EMPLO | **[V3]** No activity stream milestones for Phase 2 (Steps 10b–10g); Phase 2 UCM Doc ID, ESS Job ID, and dataset status are invisible in OIC monitoring | Phase 2 activity stream not designed | Section 7.2 |
| D20 | **High** | IFP_EMPLO | **[V3]** `var_HireStatus` gate check condition confirmed in code but Phase 2 scope not explicitly wrapped in a Level 1 HCM CATCH; Phase 2 HCM faults may propagate uncaught to Global CATCH_ALL without `failedPhase` context | Missing Phase 2 Level 1 CATCH scope | Section 6.1 |
| D21 | **Medium** | Both blueprints (V1, V2) | **[V3]** Section 4.1 Step 7 incorrectly specifies `Assignment.dat` for `ASG_CHANGE` and `GLB_TRANSFER`; reference implementation does not write Assignment for these actions | Blueprint spec not validated against implementation | Section 4.1, Design Decision Note B2 |
| D22 | **Medium** | IFP_EMPLO | **[V3]** Records with a valid action but blank `Manager_Person_Number` silently skip AssignmentSupervisor with no activity stream log entry | No ASSUP_SKIP_NO_MANAGER logging | Section 4.9.1 |
| D23 | **Medium** | Both blueprints (V1, V2) | **[V3]** Section 4.5 load order table listed ExternalIdentifier at position 9 after AssignmentSupervisor, implying a sequential dependency that does not exist; ExternalIdentifier is in Phase 1, not Phase 2 | Misleading numbering in load order table | Section 4.5 |
| D24 | **Low** | IFP_EMPLO | **[V3]** Phase 2 BIP error report path (`Phase2ReportAbsolutePath`) has no DVM column; reference code uses variable `var_ICS_AssignSupErrorReportZip_FileName` for the error zip but no BIP report path is defined for AssignmentSupervisor errors | Incomplete Phase 2 error report design | Section 4.9, 9.1 |

> **[V2] D9 Revision (Audit A1):** D9 was downgraded from Medium to Info. JCA artifact analysis confirmed all four integrations use identical HCM Cloud Adapter + ErpIntegrationService configuration. The only difference is the JCA reference name.

---

## 12. Architect Audit Register — V2 (A1–A12)

| ID | Severity | Section | Finding | Resolution in V2 |
| --- | --- | --- | --- | --- |
| A1 | Critical | 5.3, D9 | V1 claimed IFR_EMPLOYEENA used SOAP adapter for ESS polling (factually incorrect). All 4 use HCM adapter + ErpIntegrationService. | Section 5.3 rewritten; D9 downgraded to Info |
| A2 | High | 1.3 | Schedule-triggered integrations cannot be invoked synchronously. Coordinator requires REST-triggered domain integrations. | Section 1.3 updated; naming convention corrected to IFS_ |
| A3 | Medium | 3.1 | DVM `MaxPollCount` listed as Integer — all DVM columns are strings. | Column type corrected to String; `number()` cast requirement documented |
| A4 | High | 4.6 | V1 described XML accumulation in flow variable inside FOR loop — OIC does not support incremental variable append. | Section 4.6 rewritten with valid patterns |
| A5 | Medium | 4.3 | No BIP parameter size limit guidance. Oracle DB IN-clause limit is 1,000 items. | Size limit, chunking, and SQL design requirement added |
| A6 | Medium | 5.2, 7.2 | Per-iteration ESS poll logging saturates OIC activity stream. | Polling log strategy changed; activity stream budget guidance added |
| A7 | Medium | 4.3 | `$srcIdDoc` XPath access mechanism inside FOR loop unspecified. | Two implementation options documented with size constraints |
| A8 | Low | 9.1 | Concurrent SFTP audit log appends may cause corruption. | Concurrency note and sequencing recommendation added |
| A9 | Low | 5.2 | WHILE condition used uppercase `AND` — invalid XPath. | Corrected to lowercase `and` throughout |
| A10 | Low | 5.1 | `COMPLETED` claim was a positive assertion without citation. | Softened to "not documented"; version-verification guidance added |
| A11 | Medium | 4.1 | No integration-level re-run safety design. | Section 4.7 added with three mitigation strategies |
| A12 | Medium | 4.1 | No source file validation after download. | Section 4.8 added with validation checks |

---

## 13. Architect Audit Register — V3 (B1–B13) **[V3]**

All findings verified against extracted IAR artifacts (`all_vars.txt`, NXSD schemas, `expr.properties` files).

| ID | Severity | Section | Issue Type | Finding | Resolution in V3 |
| --- | --- | --- | --- | --- | --- |
| B1 | Critical | 4.1, 4.5 | Structural Omission | Two-phase HDL import (Phase 1: main objects; Phase 2: AssignmentSupervisor) was architecturally described in Section 4.5 but never represented as explicit pipeline steps in Section 4.1. Developers following Section 4.1 verbatim would build a single-phase integration. | Section 4.1 rewritten with explicit Steps 10a–10g; Section 4.5 restructured to show Phase 1 vs Phase 2; Section 4.9 added |
| B2 | High | 4.1 Step 7 | Spec vs. Implementation Conflict | Blueprint specified `Assignment.dat` for `ASG_CHANGE` and `GLB_TRANSFER`. Reference implementation does not write Assignment for these actions. | Section 4.1 Step 7 corrected to match implementation; Design Decision Note B2 added requiring team validation |
| B3 | High | 3.1, 3.3 | Configuration Management Violation | `HDLASDirectory = "/Stage/AssignSupHDL"` hardcoded as a literal flow expression in the reference implementation, bypassing DVM control and breaking the configuration management standard. | New DVM column `AssignSupStageBasePath` added to Section 3.1; initialization sequence updated in Section 3.3; Deployment checklist item 19 added |
| B4 | Critical | 7.2, 9.1 | Traceability Gap | Phase 2 (AssignmentSupervisor) ESS Job ID, outcome counts, UCM error doc ID, and final status are written to no persistent record. SFTP audit log and OIC activity stream are both silent on Phase 2. | Section 7.2 extended with Phase 2 milestones (2b–5b); Section 9.1 audit log schema extended with Phase 2 columns; Section 9.2 investigation procedure extended |
| B5 | Medium | 4.5 | Misleading Specification | Section 4.5 numbered ExternalIdentifier at position 9 after AssignmentSupervisor, implying ExternalIdentifier depends on or follows AssignmentSupervisor. ExternalIdentifier is in Phase 1 ZIP #1 and depends only on Worker. | Section 4.5 restructured to show Phase 1 and Phase 2 separately |
| B6 | High | 5.1, 5.2 | Code vs. Spec Discrepancy | Reference implementation includes `COMPLETED` in the WHILE **continue** condition, causing infinite polling if HCM returns this status, ultimately producing `ESS_POLL_TIMEOUT` instead of graceful completion. V2 blueprint correctly says to exclude COMPLETED but did not specify how to handle it if received. | Section 5.1 updated with explicit `COMPLETED` as defensive terminal; Section 5.2 POST-WHILE router extended with Branch 2 for COMPLETED |
| B7 | Critical | 8.2 | Notification Design Failure | Phase 1 SUCCESS + Phase 2 ERROR produces a SUCCESS notification. Operations team cannot detect supervisor load failures from email. | PHASE2_PARTIAL notification template added; Section 8.2 routing table defined in Section 4.9.3 |
| B8 | High | 4.9 | Design Gap | Records with a valid HIRE/ASG_CHANGE/REHIRE/GLB_TRANSFER action but blank `Manager_Person_Number` silently skip AssignmentSupervisor write with no activity stream log entry, audit record, or operator notification. | Section 4.9.1 mandates `ASSUP_SKIP_NO_MANAGER` activity stream log; `var_Phase2SkipCount` counter added; count surfaced in notification and audit log |
| B9 | Medium | 7.1 | Platform Constraint (documented) | OIC supports only 3 business identifier fields. ESS Job ID is not searchable in OIC Monitoring; reverse lookup from HCM ESS Job ID to OIC instance requires querying the SFTP audit log. | Section 7.1 note added; SFTP audit log retains both Phase 1 and Phase 2 ESS Job IDs; Section 9.2 investigation procedure references audit log for ESS Job ID lookup |
| B10 | High | 4.5, 4.1 | Load Order Correctness | ExternalIdentifier (position 9 in V1/V2) was placed after AssignmentSupervisor (position 8). ExternalIdentifier must be in Phase 1 because it depends on Worker/Person, not on AssignmentSupervisor. Submitting ExternalIdentifier after Phase 2 would add unnecessary latency and complexity. | Section 4.5 corrected; ExternalIdentifier shown as Phase 1 position 8 |
| B11 | Medium | 8.1, 9.1 | Variable Undocumented | `var_HireStatus` (Phase 2 outcome) was present in actual code (`expr.properties`) but not listed in Section 8.1 notification parameter table or Section 9.1 audit log columns. | `var_HireStatus` added to Section 8.1 as `{phase2Status}`; added to Section 9.1 as `Phase2Status` audit column; initialized in Section 3.3 |
| B12 | Medium | 4.1 Step 10 | Outcome Logic Ambiguity | Blueprint specified `ImportFailed=0 AND LoadFailed=0` as SUCCESS. Actual code uses LOAD-only counts: `total = success AND error = 0`. Import phase and Load phase are distinct HDL processing stages; outcome should be based on Load counts (object commitment). | Section 4.1 Step 10 outcome conditions rewritten to match actual code logic using LOAD counts |
| B13 | Low | 6.1 | Error Context Gap | Phase 2 HCM faults propagate to Global CATCH_ALL without a `failedPhase` label, making fault notifications indistinguishable between Phase 1 and Phase 2 failures. | `var_FailedPhase` variable added to Level 2 CATCH block (Section 6.1); FAULT notification template updated with `{failedPhase}` field; subject line includes `[PHASE1]` or `[PHASE2]` |

### Audit Evidence References (V3)

| Audit ID | Evidence Source |
| --- | --- |
| B1 | `HDLASFileName = "Worker.dat"`, `HDLASDirectory = "/Stage/AssignSupHDL"`, `LoadImportResponse_assignment` — confirmed in `all_vars.txt` lines 8, 26–27, 39. Phase 2 staging path exists in code with no corresponding pipeline step in V2 Section 4.1. |
| B2 | Router XPath for `$EachAssignment`: only `REHIRE` branch confirmed (`$EachAssignment/.../Action = "REHIRE"` in `all_vars.txt` line 195). Router XPath for `$EachWorkTerms`: `HIRE or ASG_CHANGE or REHIRE or GLB_TRANSFER` (line 171). No `$EachAssignment` ASG_CHANGE or GLB_TRANSFER expression found. |
| B3 | `TextExpression: "/Stage/AssignSupHDL"`, `XpathExpression: "/Stage/AssignSupHDL"` in `all_vars.txt` lines 26–27. No DVM column `AssignSupStageBasePath` exists in `Common_Utility_Lookup.dvm`. |
| B4 | `all_vars.txt` line 56: `var_ICS_AssignSupErrorReportZip_FileName` exists in code. No Phase 2 ESS Job ID, Phase 2 dataset counts, or Phase 2 UCM error doc ID appear in the audit write XPath expressions. |
| B6 | `all_vars.txt` lines 794–795: `$HDLJobStatus = "PAUSED" or $HDLJobStatus = "COMPLETED" or $HDLJobStatus = "Not Started" or $HDLJobStatus = "RUNNING" or $HDLJobStatus = "WAIT" or $HDLJobStatus = "READY"` — `COMPLETED` confirmed as continue-polling status in actual code. |
| B7 | `all_vars.txt` line 483: `$var_import_status = "SUCCESS" and $var_HireStatus="SUCCESS"` — gate check exists in code but output of Phase 2 error case is not routed to any notification template other than the existing SUCCESS/ERROR templates. |
| B8 | `all_vars.txt` line 1983: `($EachAssignSup/.../Action="HIRE" or "ASG_CHANGE" or "REHIRE" or "GLB_TRANSFER") and $EachAssignSup/.../Manager_Person_Number != ''` — no else-branch logging confirmed for blank manager case. |
| B12 | `all_vars.txt` lines at processors 13240 output_13242, 13523, 13528, 14793: actual router conditions use `$var_total_recordcount = $var_sucess_recordcount and $var_Error_recordcount = 0.0` (LOAD counts) not Import phase counts. |

---

## Revision History

| Version | Date | Author | Change Summary |
| --- | --- | --- | --- |
| 1.0 | 2026-05-14 | Integration Architecture Team | Initial finalized blueprint |
| 2.0 | 2026-05-14 | Integration Architecture Team | Architect audit applied — 12 findings (A1–A12) |
| 3.0 | 2026-05-15 | Integration Architecture Team | Technical re-assessment applied — 13 findings (B1–B13): two-phase pipeline explicitly specified (Sections 4.1, 4.5, 4.9); Phase 2 traceability framework added (Sections 7.2, 8.2, 9.1, 9.2); action-object matrix corrected; DVM schema extended; ESS COMPLETED handling corrected; PHASE2_PARTIAL notification template added; 9 new defects registered (D16–D24) |

---

> **Maintained by:** Integration Architecture Team
> **Review cadence:** Update on each major OIC platform version upgrade or when Oracle HCM Data Loader API changes are published
>
> *End of Blueprint V3*
