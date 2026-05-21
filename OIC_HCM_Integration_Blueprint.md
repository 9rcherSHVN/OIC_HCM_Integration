# Oracle Integration Cloud — Custom HCM Integration: Technical Design & Implementation Blueprint

| | |
| --- | --- |
| **Version** | 1.0 |
| **Date** | 2026-05-14 |
| **Audience** | Integration Developers, Solution Specialists |
| **Scope** | Custom HCM integrations using OIC App-Driven Orchestration with Oracle HCM Data Loader (HDL / FBDI) |
| **Status** | Finalized |

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

```Text
*{Type}_{Domain}_{Direction}_{Source}_{Version}*
```

| Token | Values | Meaning |
| --- | --- | --- |
| `Type` | `IFP` | Integration Flow Process — standalone, scheduled |
| `Type` | `IFS` | Integration Flow Sub — child, invoked by parent coordinator |
| `Domain` | `EMPLO`, `JOB`, `DEPT`, `NAMEMAIL` | HCM data domain |
| `Direction` | `FROM_DW` | From data warehouse source |
| `Version` | `01.00.0000` | major.minor.patch |

**Examples:**

```text
IFP_COORD_HCM_DAILY_01.00.0000      ← Coordinator (single scheduled entry point)
IFP_EMPLO_FROM_DW_01.00.0000        ← Employee domain integration
IFP_JOB_FROM_DW_01.00.0000          ← Job domain integration
IFS_FBDI_SUBMIT_POLL_01.00.0000     ← Shared FBDI sub-integration
IFS_BIP_SRCID_PREFETCH_01.00.0000   ← Shared source ID pre-fetch sub-integration
IFS_ERROR_REPORT_01.00.0000         ← Shared error report sub-integration
```

---
### 1.3 Orchestration Pattern: Coordinator + Domain + Shared Sub-Integrations

```text
[IFP_COORD_HCM_DAILY]                         ← Single scheduled coordinator
        │                                         (e.g., 22:00 America/Vancouver)
        │
        ├─[1]─ invoke IFP_JOB                    (synchronous — wait for actual completion)
        │
        ├─[2]─ invoke IFP_DEPT                   (synchronous — starts only after [1] finishes)
        │
        ├─[3]─ invoke IFP_EMPLO                  (synchronous — starts only after [2] finishes)
        │
        └─[4]─ invoke IFP_NAMEMAIL               (synchronous — starts only after [3] finishes)


Each domain integration calls shared sub-integrations:

  IFS_FBDI_SUBMIT_POLL    ← UCM upload + importAndLoadData + ESS poll + getDataSetStatus
  IFS_BIP_SRCID_PREFETCH  ← Bulk BI Publisher lookup (returns XML result set pre-loop)
  IFS_ERROR_REPORT        ← BIP error report + UCM upload + SFTP audit write
```
**Why coordinator over time-stagger:**

Time-stagger assumes each integration finishes within a fixed window. The coordinator pattern uses the actual completion signal — if an upstream domain integration runs long (large file, HDL queue backlog), downstream integrations wait. If a critical upstream integration fails, the coordinator can halt all subsequent integrations to prevent cascading partial loads.

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
| `ReportAbsolutePath` | String | BI Publisher FBDI error report absolute path |
| `FusionURL` | String | Oracle Fusion base URL — must include trailing `/` |
| `IT_Distribution_List1` | String | IT operations notification email distribution list |
| `EnvironmentType` | String | Runtime label: `DEV`, `UAT`, `PROD` |
| `HDLStageBasePath` | String | OIC stage file base path (e.g., `/Stage/Employee/`) |
| `MaxPollCount` | Integer | ESS polling iteration limit — default: `36` (= 6 minutes at 10s intervals) |

---

### 3.2 DVM Lookup Standard Pattern

All DVM lookups must use the integration flow variable `$Var_Rice_Id`. **Never use a literal RICE_ID string in any XPath expression.**

```xpath
<!-- CORRECT — uses the runtime variable -->
dvm:lookupValue('Common_Utility_Lookup', 'RICE_ID', $Var_Rice_Id, 'FusionURL', '')

<!-- INCORRECT — literal bypasses runtime RICE_ID variable; do not use -->
dvm:lookupValue('Common_Utility_Lookup', 'RICE_ID', 'I-03', 'FusionURL', '')
```
`$Var_Rice_Id` is set in the first ASSIGNMENT node immediately after the SCHEDULE_RECEIVE, before any DVM lookup executes.

---

### 3.3 Initialization Sequence (Start of Every Integration)

```text
SCHEDULE_RECEIVE
  └─ ASSIGNMENT block (single node, multiple assignments):
       Var_Rice_Id          = "{literal RICE_ID for this integration}"
       Var_Integration_Name = dvm:lookupValue(..., 'Name', '')
       var_InputFileName    = dvm:lookupValue(..., 'FileName', '')
       var_SourceDirectory  = dvm:lookupValue(..., 'DirectoryName', '')
       var_ArchiveDirectory = dvm:lookupValue(..., 'ArchiveDirectory', '')
       var_Error_Directory  = dvm:lookupValue(..., 'Error_Directory', '')
       var_ReportPath       = dvm:lookupValue(..., 'ReportAbsolutePath', '')
       var_Environment_Link = dvm:lookupValue(..., 'FusionURL', '')
       var_EnvironmentType  = dvm:lookupValue(..., 'EnvironmentType', '')
       var_DistributionList = dvm:lookupValue(..., 'IT_Distribution_List1', '')
       var_HDLStageBase     = dvm:lookupValue(..., 'HDLStageBasePath', '')
       var_MaxPollCount     = dvm:lookupValue(..., 'MaxPollCount', '36')
```
> If any lookup returns empty string for a critical field (`FileName`, `DirectoryName`, `FusionURL`), the integration must THROW a `CONFIG_FAIL` fault immediately before any SFTP or HCM operation is attempted.

---

## 4. FBDI / HDL Pipeline Standard

### 4.1 End-to-End Pipeline Sequence

```text
 Step  1  SCHEDULE_RECEIVE
 Step  2  ASSIGNMENT: initialize all config variables from DVM (Section 3.3)
 Step  3  SFTP ListFile          → capture ItemCount
 Step  4  ROUTER: ItemCount > 0?
              No  → NOTIFY (NO_FILE template) → END
              Yes → continue
 Step  5  SFTP DownloadFile      → stage source CSV to {HDLStageBasePath}/source/
 Step  6  [If source IDs needed] IFS_BIP_SRCID_PREFETCH
                                  → bulk BIP lookup, returns XML result set
                                  → stage XML to {HDLStageBasePath}/srcids/
 Step  7  FOR each record in staged source CSV:
              STAGE_READ         → read record fields into loop variable
              ROUTER (Action?)   → branch per Action code:
                HIRE             → write: Worker, PersonName, PersonLegislative,
                                          WorkRelationship, WorkTerms, Assignment,
                                          ExternalIdentifier, PersonEmail (if present)
                REHIRE           → write: Worker, WorkRelationship, WorkTerms, Assignment
                ASG_CHANGE       → write: WorkRelationship, WorkTerms, Assignment
                GLB_TRANSFER     → write: WorkRelationship, WorkTerms, Assignment
                TERMINATION      → write: WorkRelationship (TerminationType required)
                [else/unknown]   → ACTIVITY_STREAM_LOGGER:
                                   {"step":"RECORD_SKIP","action":"{action}",
                                    "personNumber":"{personNumber}",
                                    "reason":"Action not applicable"}
 Step  8  STAGE_ZIP              → zip all HDL staging files
 Step  9  IFS_FBDI_SUBMIT_POLL   → UCM upload + importAndLoadData + ESS poll + getDataSetStatus
                                   returns: ESSStatus, ImportSuccess, ImportFailed,
                                            LoadSuccess, LoadFailed, UCMContentId, ESSJobId
 Step 10  ROUTER: outcome?
              All success (ImportFailed=0 AND LoadFailed=0)  → var_import_status = "SUCCESS"
              All failed  (ImportSuccess=0 AND LoadSuccess=0) → var_import_status = "ERROR"
              Partial (mixed)                                 → var_import_status = "PARTIAL"
              Zero records (total=0)                         → var_import_status = "NO_RECORDS"
 Step 11  IFS_ERROR_REPORT       → BIP error report + UCM upload + audit log write
 Step 12  SFTP MoveFile          → archive source CSV:
              SUCCESS/NO_RECORDS → ArchiveDirectory
              ERROR/PARTIAL      → Error_Directory
 Step 13  NOTIFY (outcome email)
 Step 14  [Global CATCH_ALL]    → see Section 6
```
---

### 4.2 Shared Sub-Integration: `IFS_FBDI_SUBMIT_POLL`

This sub-integration encapsulates the entire FBDI submission chain. It is invoked by every domain integration after the HDL zip is ready.

**Input parameters:**

| Parameter | Type | Description |
| --- | --- | --- |
| `zipStagePath` | String | OIC stage path of the prepared HDL zip |
| `hdlDocumentName` | String | UCM document name for the upload |
| `processLabel` | String | Domain label for activity stream logging |

**Internal sequence:**

```text
1. HCM FileUpload (UCM)      → upload zip; capture UCMContentId
2. HCM importAndLoadData     → trigger HDL ESS job; capture ESSRequestId
3. ASSIGNMENT: pollCount = 0; maxPolls = {input maxPollCount}
4. WHILE loop (see Section 5 for full standard)
5. ROUTER: maxPolls exceeded? → THROW ESS_POLL_TIMEOUT
6. HCM getDataSetStatus      → extract ImportSuccess, ImportFailed, LoadSuccess, LoadFailed
7. RETURN structured output:
     ESSStatus, ImportSuccess, ImportFailed, LoadSuccess, LoadFailed, UCMContentId, ESSJobId
```
---

### 4.3 Shared Sub-Integration: `IFS_BIP_SRCID_PREFETCH`

Eliminates the O(n × k) per-record BI Publisher call pattern by performing **one bulk BIP call before the FOR loop**.

**Problem it solves:** Without this sub-integration, each non-HIRE record requires individual BIP calls inside the loop to retrieve existing HCM source system IDs (`WorkRelationship`, `WorkTerms`, `Assignment`, `AssignmentSupervisor`, `Manager`). For a 500-record file, this generates 2,500 sequential SOAP calls.

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
2. STAGE_WRITE               → write XML result to {HDLStageBasePath}/srcids/srcids.xml
3. RETURN: stagePath of the written file
```
**Inside the domain integration FOR loop**, after reading each record, use an XPath key lookup to retrieve the pre-fetched IDs:

```xpath
$srcIdDoc/SourceSystemIDs/Person[PersonNumber = $currentPersonNumber]/WR_SourceSystemId
```
**Result:** 1 BIP call per run regardless of file size. O(1) complexity.

---

### 4.4 HDL File Format Standard

**File naming convention:**

```text
Source CSV (downloaded)   : {FileName}_{YYYYMMDD}.csv
HDL data file             : {Domain}_{YYYYMMDD}_{HHmmss}.dat
HDL zip (upload to UCM)   : {Domain}_HDL_{YYYYMMDD}.zip
Error log text            : {Domain}_Error_{YYYYMMDD}.log
Error log zip             : {Domain}_Error_{YYYYMMDD}.zip
Monthly audit log (SFTP)  : /Audit/{Domain}/audit_{YYYY-MM}.csv
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

---

### 4.5 HDL Object Dependency Load Order

HDL requires objects in a specific dependency order within the zip file. **Objects submitted out of order will cause HDL to fail with a dependency error.**

```text
Load Order    HDL Object
──────────    ──────────────────────
     1        Worker
     2        PersonName
     3        PersonLegislativeData
     4        PersonEmail
     5        WorkRelationship
     6        WorkTerms
     7        Assignment
     8        AssignmentSupervisor   ← Submit in a separate second HDL load
     9        ExternalIdentifier
```
> **AssignmentSupervisor** must be submitted in a **separate second `importAndLoadData` call**, executed only after the first load (Steps 1–7 above) has fully completed with `SUCCEEDED` or `WARNING` status. A failed first load must block the AssignmentSupervisor submission.

---

### 4.6 XSL-Driven HDL Generation (Preferred for 3+ HDL Object Types)

For complex integrations with multiple HDL object types, use XSL transformation to generate HDL files from an in-memory XML payload. This eliminates the header-flag variable pattern and the risk of duplicate METADATA headers.

**Step 1 — Accumulate records into a structured XML variable during the FOR loop:**

```xml
<HCMPayload>
  <Workers>
    <Worker action="MERGE">
      <PersonNumber>1001</PersonNumber>
      <EffectiveStartDate>2026-01-01</EffectiveStartDate>
      <LastName>Smith</LastName>
      ...
    </Worker>
  </Workers>
  <WorkRelationships>
    <WorkRelationship action="MERGE">...</WorkRelationship>
  </WorkRelationships>
</HCMPayload>
```
**Step 2 — After the FOR loop, apply one XSL per HDL section:**

```xslt
<!-- Worker section: emits METADATA once, then one MERGE line per Worker element -->
<xsl:if test="count(/HCMPayload/Workers/Worker) &gt; 0">
METADATA|Worker|SourceSystemOwner|SourceSystemId|EffectiveStartDate|PersonNumber|...
  <xsl:for-each select="/HCMPayload/Workers/Worker">
MERGE|HRC_SQLLOADER|<xsl:value-of select="concat('PER_', PersonNumber)"/>|<xsl:value-of select="EffectiveStartDate"/>|<xsl:value-of select="PersonNumber"/>|...
  </xsl:for-each>
</xsl:if>
```
**Step 3 — One STAGE_WRITE per HDL section** writes the XSL output to the staging file.

**Benefits over line-by-line STAGE_WRITE:**
- METADATA header emitted automatically only when records exist for that object type
- All field mappings for an object type in one XSL file — one-place maintenance
- No header-flag variables (`Hire_Header = "True"/"False"`) required
- XSL file is independently testable with any XSLT processor

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

> **`COMPLETED` is not a valid ESS terminal status for HCM Data Loader jobs.** Do not include it in any WHILE loop condition. It is returned by the Oracle ESS service only for non-HDL job categories.

---

### 5.2 Standard WHILE Loop Implementation

```text
ASSIGNMENT:  pollCount = 0
ASSIGNMENT:  maxPolls  = $var_MaxPollCount   ← from DVM (default 36 = 6 min at 10s intervals)
ASSIGNMENT:  HDLJobStatus = "Not Started"

WHILE condition:
  (
    $HDLJobStatus = "Not Started"  or
    $HDLJobStatus = "READY"        or
    $HDLJobStatus = "WAIT"         or
    $HDLJobStatus = "RUNNING"      or
    $HDLJobStatus = "PAUSED"
  )
  AND ($pollCount < $maxPolls)

  WHILE body:
    WAIT: 10 seconds
    HCM getESSJobStatus            → update $HDLJobStatus from response
    ACTIVITY_STREAM_LOGGER:
      {"step":"ESS_POLL","pollCount":"{pollCount}","status":"{HDLJobStatus}","jobId":"{essJobId}"}
    ASSIGNMENT: pollCount = $pollCount + 1

POST-WHILE:
  ROUTER: Did loop exit due to timeout?
    Condition: $pollCount >= $maxPolls
               AND ($HDLJobStatus = "Not Started" or "READY" or "WAIT" or "RUNNING" or "PAUSED")
      Yes → THROW fault: code=ESS_POLL_TIMEOUT,
                         detail="ESS Job {essJobId} did not reach terminal status after {pollCount} polls"
      No  → continue to getDataSetStatus
```
---

### 5.3 ESS Service Endpoint Consistency Rule

All HCM FBDI integrations must use the **HCM Cloud Adapter** native `getESSJobStatus` operation.

Do **not** use `ErpIntegrationService.getESSJobStatus` via the SOAP adapter — this introduces a second service binding for the same function, creates inconsistency, and requires separate WSDL maintenance if Oracle changes the ESS API contract.

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
│    ACTIVITY_STREAM_LOGGER: {step, essJobId, faultCode, faultDetail}
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
     THROW  → re-raise to Level 3

─────────────────────────────────────────────────────────────────

Level 3: Global CATCH_ALL
│
└─ Actions (executed in order):
     1. ACTIVITY_STREAM_LOGGER:
           {"step":"GLOBAL_FAULT","instanceId":"{instanceId}",
            "faultCode":"{var_FaultCode}","faultDetail":"{var_FaultDetail}",
            "failedStep":"{var_FailedStep}","timestamp":"{timestamp}"}
     2. SFTP MoveFile → Error_Directory
           (ensures source file is always archived regardless of failure point)
     3. SFTP AUDIT WRITE
           (append error/fault record to monthly audit log — see Section 9)
     4. NOTIFY: FAULT template
     5. END
```
---

### 6.2 Fault Code Reference

Every thrown fault must include a `faultCode` from the following standardized set:

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

Configure exactly **3 business identifiers** on every HCM integration via the OIC Designer tracking panel. These appear as searchable columns in OIC Monitoring → Integration Instances.

| Position | Label | XPath Expression | Example Value |
| --- | --- | --- | --- |
| 1 | `Source_File` | `$var_InputFileName` | `employees_20260514.csv` |
| 2 | `Run_Date` | `fn:format-dateTime(fn:current-dateTime(),'[Y0001]-[M01]-[D01]')` | `2026-05-14` |
| 3 | `Domain` | Literal constant per integration | `EMPLOYEE`, `JOB`, `DEPARTMENT`, `NAME_EMAIL` |

**Verification procedure after deployment:**
1. Trigger a test run
2. Open OIC Monitoring → Integration Instances
3. Confirm all 3 identifiers appear as columns with correct values for the test instance
4. Confirm `Source_File` search returns only the matching instance

---

### 7.2 Mandatory Activity Stream Entries

Add an `ACTIVITY_STREAM_LOGGER` node at each of the following 6 milestone points. All messages must be **valid JSON strings**.

| # | Trigger Point | Required JSON Fields |
| --- | --- | --- |
| 1 | After SFTP ListFile | `step`, `status` (`FOUND`/`NOT_FOUND`), `file`, `recordCount` |
| 2 | After `importAndLoadData` | `step`, `essJobId`, `ucmDocId`, `hdlZipFile` |
| 3 | After ESS WHILE exits | `step`, `essJobId`, `essStatus`, `pollCount`, `elapsedSeconds` |
| 4 | After `getDataSetStatus` | `step`, `importSuccess`, `importFailed`, `loadSuccess`, `loadFailed` |
| 5 | After error report UCM upload | `step`, `errorLogDocId`, `errorReportUrl` |
| 6 | At integration end (before NOTIFY) | `step`, `finalStatus`, `archivePath`, `instanceId`, `timestamp` |

**Example — Step 3:**

```json
{
  "step": "ESS_COMPLETE",
  "essJobId": "12345678",
  "essStatus": "SUCCEEDED",
  "pollCount": "4",
  "elapsedSeconds": "42"
}
```
> **Do not use free-text activity stream messages.** JSON format enables log parsing by monitoring tools and OIC support teams.

---

## 8. Notification Design Standard

### 8.1 Notification Parameter Mapping Reference

All notification parameters are resolved at runtime from OIC flow variables via the notification action's parameter mapping.

| Template Placeholder | Flow Variable | Populated From |
| --- | --- | --- |
| `{integrationName}` | `$Var_Integration_Name` | DVM lookup — `Name` column |
| `{environmentType}` | `$var_EnvironmentType` | DVM lookup — `EnvironmentType` column |
| `{runDate}` | `$var_RunDate` | `fn:format-dateTime(fn:current-dateTime(),'[Y0001]-[M01]-[D01] [H01]:[m01]:[s01]')` |
| `{fileName}` | `$var_InputFileName` | DVM lookup — `FileName` column |
| `{sourceDirectory}` | `$var_SourceDirectory` | DVM lookup — `DirectoryName` column |
| `{totalRecords}` | `$var_total_recordcount` | `getDataSetStatus` response |
| `{successRecords}` | `$var_sucess_recordcount` | `getDataSetStatus` response |
| `{errorRecords}` | `$var_Error_recordcount` | `getDataSetStatus` response |
| `{essJobId}` | `$var_EssRequestId` | `importAndLoadData` response |
| `{instanceId}` | OIC system variable | Auto-resolved by OIC notification runtime |
| `{errorReportUrl}` | `$var_UCM_ImportError_link` | `concat($var_Environment_Link, 'cs/idcplg?IdcService=GET_FILE&amp;dID=', $var_Document_Id)` |
| `{errorArchivePath}` | `$var_Error_Directory` | DVM lookup — `Error_Directory` column |
| `{faultCode}` | `$var_FaultCode` | Set in Level 2/3 fault handler |
| `{faultDetail}` | `$var_FaultDetail` | Set in Level 2/3 fault handler |
| `{failedStep}` | `$var_FailedStep` | Set in Level 2/3 fault handler |

---

### 8.2 Standard Email Notification Templates

#### Template: NO_FILE

```text
Subject : [HCM Integration] [{environmentType}] {integrationName} — No Source File Found ({runDate})
To      : {distributionList}
From    : {fromAddress}

Integration     : {integrationName}
Environment     : {environmentType}
Run Date/Time   : {runDate}
Expected File   : {fileName}
SFTP Directory  : {sourceDirectory}
OIC Instance ID : {instanceId}

No source file was present on SFTP. No records were processed.
This notification is informational. If a file was expected, verify the data warehouse
export process.
```
#### Template: SUCCESS

```text
Subject : [HCM Integration] [{environmentType}] {integrationName} — Completed Successfully ({runDate})

Integration     : {integrationName}
Environment     : {environmentType}
Run Date/Time   : {runDate}
Source File     : {fileName}
OIC Instance ID : {instanceId}
ESS Job ID      : {essJobId}
Records Total   : {totalRecords}
Records Success : {successRecords}
Records Failed  : {errorRecords}
```
#### Template: ERROR or PARTIAL (HTML)

```html
<html>
<body>
<p><strong>[HCM Integration] [{environmentType}] {integrationName} — {status} ({runDate})</strong></p>
<table>
  <tr><td>Integration</td><td>{integrationName}</td></tr>
  <tr><td>Environment</td><td>{environmentType}</td></tr>
  <tr><td>Run Date/Time</td><td>{runDate}</td></tr>
  <tr><td>Source File</td><td>{fileName}</td></tr>
  <tr><td>OIC Instance ID</td><td>{instanceId}</td></tr>
  <tr><td>ESS Job ID</td><td>{essJobId}</td></tr>
  <tr><td>Records Total</td><td>{totalRecords}</td></tr>
  <tr><td>Records Success</td><td>{successRecords}</td></tr>
  <tr><td>Records Failed</td><td>{errorRecords}</td></tr>
</table>
<p>HCM Error Report: <a href="{errorReportUrl}">View import error details in UCM</a></p>
<p>Error log also archived at SFTP: {errorArchivePath}</p>
</body>
</html>
```
#### Template: FAULT

```text
Subject : [HCM Integration] [{environmentType}] {integrationName} — SYSTEM FAULT ({runDate})

Integration     : {integrationName}
Environment     : {environmentType}
Run Date/Time   : {runDate}
OIC Instance ID : {instanceId}
Failed Step     : {failedStep}
Fault Code      : {faultCode}
Fault Detail    : {faultDetail}

Action required: Review OIC instance {instanceId} in the monitoring console.
The source file has been moved to the error archive directory.
```
---

### 8.3 HTML Email Attribute Rule — Mandatory

Every `href` attribute in any HTML notification template **must** use quoted values. The `&` character in URLs must be HTML-encoded as `&amp;`.

```html
<!-- CORRECT: quoted attribute, & encoded -->
<a href="{errorReportUrl}">View error report</a>

<!-- INCORRECT: unquoted attribute — not rendered as hyperlink by Outlook or Gmail -->
<a href={Content}>click here</a>
```text
The `errorReportUrl` placeholder value is pre-constructed in the flow as:
```xpath
concat($var_Environment_Link, 'cs/idcplg?IdcService=GET_FILE&amp;dID=', $var_Document_Id)
```
---

### 8.4 Notification Subject Prefix Convention

All notification subjects must be prefixed with `[HCM Integration]` to enable email rule filtering in IT operations mailboxes:

```text
[HCM Integration] [{environmentType}] {integrationName} — {outcome} ({runDate})
```
This allows operations staff to create inbox rules that:
- Route all HCM integration emails to a dedicated folder
- Flag `SYSTEM FAULT` subjects for immediate escalation
- Filter by `[PROD]` vs `[DEV]` environment

---

## 9. Audit and Traceability Standard

### 9.1 SFTP Audit Log (Mandatory for All Domain Integrations)

At the end of every integration run — including fault paths via the global CATCH_ALL — append one CSV record to the monthly audit file on SFTP.

**Path pattern:**
```text
/Audit/{Domain}/audit_{YYYY-MM}.csv
```
**CSV structure:**

```
RunTimestamp,IntegrationCode,RiceId,SourceFile,TotalRecords,SuccessRecords,FailedRecords,ESSJobId,ESSStatus,UCMErrorDocId,OICInstanceId,FinalStatus,ArchivePath,DurationSeconds
2026-05-14T22:35:47,IFP_EMPLO,I-03,employees_20260514.csv,1200,1188,12,12345678,SUCCEEDED,EMPLERR20260514,IFP_EMPLO_XXXX,PARTIAL,/Outbound/error/,127
```
**Column definitions:**

| Column | Source |
| --- | --- |
| `RunTimestamp` | `fn:current-dateTime()` at integration start |
| `IntegrationCode` | Integration code constant |
| `RiceId` | `$Var_Rice_Id` |
| `SourceFile` | `$var_InputFileName` |
| `TotalRecords` | `$var_total_recordcount` |
| `SuccessRecords` | `$var_sucess_recordcount` |
| `FailedRecords` | `$var_Error_recordcount` |
| `ESSJobId` | Captured from `importAndLoadData` response |
| `ESSStatus` | Final `$HDLJobStatus` after WHILE exits |
| `UCMErrorDocId` | `$var_Document_Id` (error log UCM doc ID) |
| `OICInstanceId` | OIC system variable |
| `FinalStatus` | `$var_import_status` |
| `ArchivePath` | `$var_ArchiveDirectory` or `$var_Error_Directory` |
| `DurationSeconds` | Computed from start/end timestamps |

**Retention:** Audit files are retained indefinitely. Files rotate monthly (new file per month). The integration never deletes audit files.

---

### 9.2 End-to-End Traceability: Investigation Procedure

With the full standard implemented, an IT support engineer can answer *"Was employee PersonNumber 1234 synchronized to Oracle HCM on 2026-05-14?"* in under 5 minutes:

| Step | Tool | Action |
| --- | --- | --- |
| 1 | OIC Monitoring | Search `Source_File = "employees_20260514.csv"` → find instance directly |
| 2 | OIC Monitoring | Check instance status and `Domain` = `EMPLOYEE` identifier |
| 3 | OIC Activity Stream | View 6 structured JSON milestone entries for full pipeline trace |
| 4 | Notification email | Read `{totalRecords}`, `{successRecords}`, `{errorRecords}` for summary counts |
| 5 | Email hyperlink | Click `{errorReportUrl}` → UCM error report → search for PersonNumber 1234 |
| 6 | UCM error report | Read exact HDL rejection reason for the employee (if applicable) |
| 7 | SFTP audit log | Cross-check `/Audit/EMPLOYEE/audit_2026-05.csv` for independent audit record |

> Silent record drops are **eliminated** by the router branch completeness rule (Section 6.3). Every record either succeeds, is explicitly logged as skipped, or raises a named fault.

---

## 10. Deployment and Environment Management Checklist

Run this checklist on every IAR before import to any OIC environment (DEV, UAT, PROD).

### Pre-Deployment Checks

| # | Check | Method | Pass Criterion |
| --- | --- | --- | --- |
| 1 | Project warnings | Read `project_messages.json` | Zero `COMPOSER_ACTION_EMPTY_WARNING_ROUTE` entries |
| 2 | Design completion | Read `project.xml` | `projectHasWarnings = false`, `percentComplete = 100` |
| 3 | JCA endpoint scan | PowerShell scan (Section 2.2) | Zero files containing `<property name="endpointURL">` |
| 4 | DVM RICE_ID literals | Search `expr.properties` files | Zero `dvm:lookupValue(... 'RICE_ID', '{literal}' ...)` patterns |
| 5 | Environment type | Grep `all_vars.txt` for `var_Environment_type` | Value is DVM-sourced, not a literal string |
| 6 | ESS WHILE condition | Review WHILE expressions | `COMPLETED` does not appear; `maxPolls` counter is present |
| 7 | Router completeness | Review all router branches | Zero empty `otherwise`/`else` branches |
| 8 | Notification HTML | Review all `notification_body.data` files | All `href` attributes are quoted; `&` is `&amp;` |
| 9 | Business identifiers | Review OIC designer tracking panel | 3 identifiers configured per integration |
| 10 | Activity stream | Count `ACTIVITY_STREAM_LOGGER` nodes | Minimum 6 per integration; all messages are valid JSON |
| 11 | Project descriptions | Review `projectDescription` in `project.xml` | Accurately describes the integration's purpose |
| 12 | Connection status | OIC Connections console | All 3 connections show `CONFIGURED` |

### Post-Deployment Verification

| # | Verification | Pass Criterion |
| --- | --- | --- |
| 1 | Test run with sample file | Integration completes without OIC fault |
| 2 | Business identifiers visible | All 3 identifiers appear in OIC Monitoring instance list |
| 3 | Activity stream populated | All 6 milestone entries present in instance activity stream |
| 4 | Notification received | Email received with correct subject prefix, record counts, and clickable UCM link |
| 5 | SFTP audit log written | One CSV row appended to `/Audit/{Domain}/audit_{YYYY-MM}.csv` |
| 6 | Source file archived | Source CSV moved to correct SFTP archive directory |
| 7 | UCM error report accessible | Error report URL in notification email opens the UCM document |

---

## 11. Known Defect Register (Reference Implementation)

This register documents confirmed defects found in the reference implementation (`IFP_EMPLO_FROM_DATAW_TO_ORACL` v01.00.0011 and related integrations). Use this as a test baseline and a lesson-learned reference when building new integrations.

| ID | Severity | Integration | Defect | Root Cause | Standard Rule |
| --- | --- | --- | --- | --- | --- |
| D1 | Critical | IFP_EMPLO, IFR_EMPLOYEENA | 25+ empty router branches — silent record drops | Design incomplete; branches added without actions | Section 6.3 |
| D2 | Critical | IFP_EMPLO | 7 BI Publisher JCA files hardcoded to DEV2 endpoint | JCA `endpointURL` overrides connection property | Section 2.2 |
| D3 | High | IFP_EMPLO | `var_Environment_type = "DEV"` hardcoded literal | Hardcoded dev artifact never removed | Section 3.1 |
| D4 | High | All | FusionURL DVM lookup uses literal `'I-03'` not `$Var_Rice_Id` | Copy-paste, variable not substituted | Section 3.2 |
| D5 | High | All | `href={Content}` — unquoted HTML attribute; hyperlink not rendered | HTML template not tested in Outlook | Section 8.3 |
| D6 | High | All | ESS WHILE includes `"COMPLETED"` (invalid HDL status); no poll timeout guard | Incorrect ESS status model; no timeout design | Section 5.1, 5.2 |
| D7 | Medium | IFP_EMPLO | Dual FBDI submission outcomes reported separately; no cross-reference | Independent notification design | Section 4.1 |
| D8 | Medium | IFP_EMPLO | O(n × 5) BI Publisher calls inside FOR loop — scalability risk | No pre-fetch design pattern | Section 4.3 |
| D9 | Medium | IFR_EMPLOYEENA | Uses `ErpIntegrationService` for ESS polling; other 3 use HCM adapter | Developed independently, inconsistent adapter choice | Section 5.3 |
| D10 | Low | IFP_JOB, IFR_EMPLO_DEPAR | `projectDescription` copy-pasted from Employee integration template | Missing review gate | Section 10, Item 11 |
| D11 | Low | IFR_EMPLOYEENA | JCA filename typo: `GetLoadImportStatuis_REQUEST.jca` | Typo; no naming validation | Section 1.2 |
| D12 | Low | IFR_EMPLOYEENA | Notification body typos: "GlobaleFault", "appural action" | No template review; no test email step | Section 8.2 |
| D13 | Low | IFP_EMPLO, IFR_EMPLOYEENA | Error/success notification missing `{DateTime}` field | Inconsistent template copy from base | Section 8.2 |
| D14 | Info | All | Zero business identifiers configured | Feature not used | Section 7.1 |
| D15 | Info | All | Record counts (`totalRecords`, `successRecords`, `errorRecords`) not in notification body | Variables computed but not mapped to notification | Section 8.1 |

---

*End of Blueprint*

---

> **Maintained by:** Integration Architecture Team  
> **Review cadence:** Update on each major OIC platform version upgrade or when Oracle HCM Data Loader API changes are published
