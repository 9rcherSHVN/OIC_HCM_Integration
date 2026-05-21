# Technical Design Blueprint: OIC–HCM Integration (Redesign)

---

## 1. Design Principles

Before any pattern, five governing principles:

1. **Environment-purity**: No endpoint URL, hostname, or environment identifier in any JCA or XSL file — ever. All environment-specific values are injected via OIC Connection properties (`%%PROPERTY` tokens) or OIC Lookups.
2. **Modularity over monoliths**: A flow that requires 75 JCA files and 60 XSL maps to maintain belongs in multiple coordinated integrations, not one.
3. **Explicit over implicit**: Every router branch has an explicit action — no silent data drops. Every fault has a structured handler — no naked catch-all with only an email.
4. **O(1) external calls per batch run, not O(n)**: Pre-fetch all external lookup data before any loop. Never call BI Publisher or HCM inside a per-record FOR loop.
5. **Structured state, not string flags**: Integration state (`var_import_status = "ERROR"`) must be a typed, structured business identifier — not a free-text variable that only appears in an email.

---

## 2. Architecture: Modular Hub-and-Spoke with a Coordinator

### What the current design does (flat, repetitive)
```
[Scheduler A] ──► [IFP_EMPLO: 139 Assignments, 75 JCAs, 2 FBDI chains] (monolith)
[Scheduler B] ──► [IFP_JOB: copy of same pipeline]
[Scheduler C] ──► [IFR_EMPLO_DEPAR: copy of same pipeline]
[Scheduler D] ──► [IFR_EMPLOYEENA: different ESS endpoint, no pattern alignment]
```
Consequence: the FBDI pipeline is duplicated 4 times. A bug fix must be applied 4 times. An environment property change must be applied 4 times.

### Proposed design: 1 Coordinator + 4 domain integrations + 3 shared sub-integrations

```
[COORDINATOR_HCM_DAILY]  ← Single scheduled integration, 22:15 America/Vancouver
        │
        ├──[1]── invoke SUB_IFP_JOB         (synchronous, wait for completion)
        ├──[2]── invoke SUB_IFR_EMPLO_DEPAR (synchronous, waits for [1] to finish)
        ├──[3]── invoke SUB_IFP_EMPLO       (synchronous, waits for [2] to finish)
        └──[4]── invoke SUB_IFR_EMPLOYEENA  (synchronous, waits for [3] to finish)

Each domain integration calls shared sub-integrations:
        SUB_FBDI_SUBMIT_AND_POLL     ← Reusable: UCM upload + importAndLoadData + ESS poll
        SUB_FETCH_SOURCE_SYSTEM_IDS  ← Reusable: bulk BIP lookup for all source system IDs
        SUB_GENERATE_ERROR_REPORT    ← Reusable: BIP error report + UCM archive + SFTP archive
```

**Why the coordinator pattern over time stagger:**
- If `IFP_JOB` runs long (large file, HDL backlog), `IFR_EMPLO_DEPAR` does not start until `IFP_JOB` actually completes — not just until the clock reaches 22:25.
- The coordinator holds a shared execution state. If any domain integration returns an error, the coordinator can halt subsequent integrations (preventing downstream data integrity issues) or continue and aggregate all outcomes into a single operational notification.
- OIC supports synchronous child integration invocation via the **Integration** invoke action. The child must be deployed as a callable integration (App-Driven Orchestration with REST or SOAP trigger), or use the Schedule-triggered integration with a REST wrapper.

---

## 3. Shared Sub-Integration: `SUB_FBDI_SUBMIT_AND_POLL`

The FBDI pipeline (UCM upload → importAndLoadData → ESS poll → getDataSetStatus) is **identical in all 4 current integrations**. It must exist once. This sub-integration accepts:

**Input parameters:**
- `zipStagePath` — OIC stage path of the prepared zip file
- `hdlDocName` — document name for UCM
- `processName` — identifies which domain for logging

**Internal steps:**

```
1. HCM FileUpload          → upload zip to UCM, capture ContentId
2. HCM importAndLoadData   → trigger HDL, capture ESSRequestId
3. ASSIGNMENT              → initialize: pollCount=0, maxPolls=36 (6-minute max at 10s intervals)
4. WHILE loop:
     condition: HDLJobStatus ∉ {SUCCEEDED, ERROR, WARNING, CANCELLED, BLOCKED}
       AND pollCount < maxPolls
     body:
       WAIT 10s
       getESSJobStatus        → update HDLJobStatus
       ASSIGNMENT pollCount++ → increment counter
5. ROUTER (maxPolls exceeded?) → if yes: THROW fault ("ESS_POLL_TIMEOUT")
6. getDataSetStatus            → extract IMPORT/LOAD counts
7. RETURN structured output:   ESSStatus, ImportSuccess, ImportFailed, LoadSuccess, LoadFailed, ContentId
```

**Two critical fixes embedded here:**

- **Fix for D5 (WHILE loop with "COMPLETED"):** The correct ESS terminal statuses for HDL jobs are `SUCCEEDED`, `ERROR`, `WARNING`, `CANCELLED`, and `BLOCKED`. `COMPLETED` is not a valid HDL ESS terminal status — it is a generic ESS status for non-HDL jobs. Including it in the WHILE-continue condition means the loop continues when ESS returns COMPLETED, preventing exit. Removing it from the continue set is the fix.
- **Fix for infinite loop (no current safety valve):** The current implementation has no poll counter or timeout guard. If ESS never reaches a terminal status (hung job, network issue), the current WHILE loop will run forever, blocking the OIC worker thread indefinitely. The `maxPolls` counter above bounds this.

---

## 4. Source System ID Lookup — Bulk Pre-Fetch (Eliminating O(n×5) calls)

### Current problem
Inside the IFP_EMPLO FOR loop, for every non-HIRE record, the integration calls BI Publisher 5 times:
- `GetWRSourceSystemID` (WorkRelationship)
- `GetWTSourceSystemID` (WorkTerms)
- `GetWASourceSystemID` (Assignment)
- `GetAssignmentSupervisor`
- `GetManagerSourceSystemID`

For a file with 500 employee change records, this generates **2,500 synchronous SOAP calls** to BI Publisher within a single integration run. This is the primary scalability bottleneck.

### Redesign: Pre-fetch all source system IDs in a single bulk BIP call

**Step 1 — Collect all Person Numbers from the CSV before the loop:**

After `DownloadFile` and `ReadFile`, parse the staging CSV and build a comma-delimited list of all `Person_Number` values for non-HIRE records into a scalar variable `var_PersonNumberList`.

**Step 2 — One BI Publisher call for all IDs:**

Design a single parameterized BI Publisher report (`RPT_HCM_SourceSystemIDs`) that accepts `var_PersonNumberList` as an input parameter and returns a flat XML result set:

```xml
<SourceSystemIDs>
  <Person>
    <PersonNumber>1001</PersonNumber>
    <WR_SourceSystemId>PER_WR_1001</WR_SourceSystemId>
    <WT_SourceSystemId>PER_WT_1001</WT_SourceSystemId>
    <WA_SourceSystemId>PER_WA_1001</WA_SourceSystemId>
    <Manager_PersonNumber>2001</Manager_PersonNumber>
    <ManagerSourceSystemId>PER_WA_2001</ManagerSourceSystemId>
  </Person>
  ...
</SourceSystemIDs>
```

**Step 3 — Stage and read back:**

Write BIP result to OIC staging file → read into a stage variable. Inside the FOR loop, use an XPath lookup (or XSLT `key()` function) to find the pre-fetched `SourceSystemId` for the current `Person_Number` — no external call per iteration.

**Result:** 5 BI Publisher calls regardless of file size → O(1) complexity.

---

## 5. HDL File Generation — Structured XSL vs. Line-by-Line STAGE_WRITE

### Current approach
The current implementation writes HDL files using **53 individual STAGE_WRITE operations** (line-by-line) with header-flag variables (`Hire_Header = "True"/"False"`) to prevent duplicate section headers. This pattern is fragile:
- Any change to HDL field order requires finding and updating multiple STAGE_WRITE JCA files
- Header tracking variables can get out of sync across router branches
- Empty router branches silently skip writes, leaving orphaned header lines

### Proposed approach: XSL-driven XML-to-HDL transformation

**Step 1:** As records are read and validated, accumulate them into a structured in-memory XML document (OIC variable of type XML):

```xml
<HCMPayload>
  <Workers>
    <Worker action="MERGE"><PersonNumber>1001</PersonNumber>...</Worker>
  </Workers>
  <WorkRelationships>
    <WorkRelationship action="MERGE">...</WorkRelationship>
  </WorkRelationships>
</HCMPayload>
```

This is one cumulative ASSIGNMENT per record appending to the XML structure.

**Step 2:** After the loop, apply a **single XSL transformation** that converts the entire `HCMPayload` XML to the flat HDL pipe-delimited format, writing one unified HDL file:

```xslt
<xsl:for-each select="Workers/Worker">
  METADATA|Worker|SourceSystemOwner|SourceSystemId|EffectiveStartDate|...
  MERGE|HRC_SQLLOADER|<xsl:value-of select="concat('PER_', PersonNumber)"/>|...
</xsl:for-each>
```

The XSL emits the `METADATA` header row once per section automatically — no header-flag variable needed, no risk of duplicate headers.

**Benefits:**
- HDL field mapping lives in 1 XSL file per object type, not 53 JCA stage-write files
- Section headers are generated by the XSL, eliminating the header-flag pattern
- Testable independently using any XSLT processor
- A field addition requires editing one XSL, not hunting through dozens of JCA files

---

## 6. Configuration & Environment Management

### Current problems
- `endpointURL` in 7 JCA files hardcoded to `fa-eqcg-dev2-saasfaprod1` (DEV2) while the package is deployed to production
- `var_Environment_type = "DEV"` hardcoded in assignment (IFP_EMPLO)
- SFTP default path `/Outbound/GL/` (finance path) in FTP adapter JCA

### Blueprint: two-layer configuration

**Layer 1 — OIC Connection Properties (for connection endpoints)**

All connection properties that vary by environment use the `%%PROPERTY_NAME` token, which OIC resolves at deploy time from the OIC instance configuration. **No JCA file contains any URL, hostname, or port.** The target environment is configured once per OIC instance, not per integration.

- `IFP_OIC_HCM_CONNECTI`: `targetWSDLURL = %%IFP_OIC_HCM_CONNECTI_targetWSDLURL` ✓ (already correct)
- `IFP_OIC_REPORT_SERVICE`: `targetWSDLURL = %%IFP_OIC_REPORT_SERVICE_targetWSDLURL` ✓ (already correct at connection level — the JCA `endpointURL` override must be removed)
- `IFP_OIC_SFTP`: `Host = %%IFP_OIC_SFTP_Host`, `Port = %%IFP_OIC_SFTP_Port` ✓ (already correct)

**Removing the JCA `endpointURL` override** in `CallImportErrorReport_REQUEST.jca`, `GetWRSourceSystemID_REQUEST.jca`, etc. causes the SOAP adapter to use the connection-level `targetWSDLURL` at runtime — which is already environment-parameterized.

**Layer 2 — DVM for runtime configuration (already used — extend it)**

The `Common_Utility_Lookup.dvm` is the correct mechanism for file names, directory paths, report paths, and distribution lists. Extend it with two additional columns:

| New Column | Purpose |
|---|---|
| `MaxPollCount` | ESS polling iteration limit (currently hardcoded to whatever the WHILE implies) |
| `HDLStageBasePath` | OIC stage base path — removes hardcoded `/Stage/Employee`, `/Stage/HDL`, etc. |

Remove `var_Environment_type` from integration code entirely. If environment tagging is needed in notifications or logs, derive it from `FusionURL` (already in DVM) at runtime.

---

## 7. Content-Based Router — Explicit Else Branch Enforcement

### Current problem
25+ routers in IFP_EMPLO and IFR_EMPLOYEENA have empty branches (`COMPOSER_ACTION_EMPTY_WARNING_ROUTE`). Records matching those branches are dropped silently.

### Blueprint rule: every router branch must have one of three explicit actions

| Branch type | Required action |
|---|---|
| Known, handled action (HIRE, ASG_CHANGE, etc.) | Normal processing flow |
| Known, intentionally skipped action | ACTIVITY_STREAM_LOGGER: `"Skipped: Action={action}, Person={person_number}"` |
| Unknown/unexpected action | THROW fault with message: `"Unexpected Action code {action} for Person {person_number} — not processed"` |

This means after redesign: **zero `COMPOSER_ACTION_EMPTY_WARNING_ROUTE` warnings** at deploy time. The OIC project health check must show 0 warnings before any deployment.

---

## 8. Error Handling — Structured Fault Hierarchy

### Current problem
The current global CATCH_ALL sends one email with `{Error_Message}` and `{Instance_ID}`. This is the only artifact of any unhandled fault. There is no error classification, no retry strategy, and no structured error data for operational triage.

### Blueprint: three-level fault handling

```
Level 1: Operation-level CATCH (around each major step)
   ├─ Catch: SFTP_FAULT
   │    Action: log to activity stream, archive input file to Error_Directory, NOTIFY (SFTP error)
   ├─ Catch: HCM_FAULT (UCM upload, importAndLoadData)
   │    Action: log to activity stream, NOTIFY (HCM connectivity error), THROW to Level 2
   └─ Catch: BIP_FAULT (BI Publisher calls)
        Action: log to activity stream, NOTIFY (BIP lookup error), THROW to Level 2

Level 2: Scope-level CATCH (around the FOR loop / FBDI chain)
   └─ Catch: any fault from Level 1 rethrows
        Action: set var_FaultCode, var_FaultDetail, var_FailedStep → THROW to Level 3

Level 3: Global CATCH_ALL
   └─ Action:
        1. ACTIVITY_STREAM_LOGGER: structured JSON {faultCode, faultDetail, failedStep, personNumber, instanceId, timestamp}
        2. SFTP MoveFile: source file → Error_Directory (ensure file is always archived regardless of failure point)
        3. NOTIFICATION: structured email with faultCode + faultDetail + failedStep + UCM link (if available)
        4. Set integration instance tracking variable: Status=FAILED, ErrorCode=var_FaultCode
```

This replaces the current pattern where a SOAP timeout on one BIP call produces only `{Error_Message}` in an email with no context about which step, which person, or which HCM object failed.

---

## 9. Observability — Business Identifiers and Structured Logging

### Current state
OIC monitoring shows each integration instance as an opaque row with a timestamp and status. There are no configured business identifiers. To investigate a problem for a specific employee, you must open each instance and read through the activity stream — if one exists.

### Blueprint: configure 3 business identifiers per integration

OIC allows up to 3 business identifiers per integration, which become searchable fields in the OIC monitoring console.

| Identifier | Source | Benefit |
|---|---|---|
| `Source_FileName` | DVM `FileName` lookup result | Search all runs for a specific source file |
| `Run_Date` | Schedule trigger timestamp (date only) | Search all runs for a given business day |
| `Integration_Code` | Static (e.g., `"IFP_EMPLO"`) | Filter monitoring by integration domain |

With this in place, an IT support engineer can open OIC Monitoring → Integrations → filter by `Source_FileName = "employees_2026-05-14.csv"` and immediately see the specific instance, its status, and its activity stream.

### Structured Activity Stream entries (per integration)

Replace or supplement the current single `ACTIVITY_STREAM_LOGGER` with structured log entries at key milestones:

```json
{"step":"FILE_CHECK",    "status":"FILE_FOUND",    "fileName":"employees_2026-05-14.csv", "recordCount":342}
{"step":"HDL_SUBMITTED", "status":"PENDING",        "essRequestId":"12345678", "contentId":"UCMCS-9999"}
{"step":"ESS_POLL",      "status":"SUCCEEDED",      "essRequestId":"12345678", "pollCount":4}
{"step":"HDL_RESULT",    "status":"PARTIAL",        "importSuccess":338, "importFailed":4, "loadSuccess":338}
{"step":"ERROR_REPORT",  "status":"UPLOADED",       "errorLogContentId":"UCMCS-10001"}
{"step":"COMPLETED",     "status":"PARTIAL_SUCCESS","archivePath":"/Outbound/success/employees_2026-05-14.csv"}
```

This gives any IT support engineer a full execution trace in OIC's native monitoring UI — no email triage needed to understand what happened.

---

## 10. Notification Redesign — Actionable, Not Just Informational

### Current problem
Notification bodies are free-text templates with minimal context. The `{Content}` link placeholder may not render as a hyperlink. The error email body contains `{errorMessage}` with no surrounding context. IFR_EMPLOYEENA has a typo ("appural action").

### Blueprint: standardized notification schema per outcome type

**No-file notification:**
```
Subject: [HCM Integration] {IntegrationName} — No File Found ({RunDate})
Body:    Integration: {IntegrationName}
         Run Date: {RunDate}
         SFTP Directory: {DirectoryName}
         Expected File: {FileName}
         OIC Instance ID: {InstanceID}
         No source file was present. No data was processed.
```

**Error/Partial notification:**
```
Subject: [HCM Integration] {IntegrationName} — {Status} ({RunDate})
Body:    Integration: {IntegrationName}
         Run Date: {RunDate}
         Source File: {FileName}
         OIC Instance ID: {InstanceID}
         Import — Success: {ImportSuccess} | Failed: {ImportFailed}
         Load   — Success: {LoadSuccess}   | Failed: {LoadFailed}
         Error Report (UCM): <URL>
         Error Report (SFTP Archive): {ErrorArchivePath}
```

**Fault notification:**
```
Subject: [HCM Integration] {IntegrationName} — SYSTEM FAULT ({RunDate})
Body:    Integration: {IntegrationName}
         OIC Instance ID: {InstanceID}
         Failed Step: {FailedStep}
         Fault Code: {FaultCode}
         Fault Detail: {FaultDetail}
         Timestamp: {Timestamp}
         Action required: Review OIC instance {InstanceID} in monitoring console.
```

Subjects are prefixed with `[HCM Integration]` to allow email rule filtering in operations mailboxes.

---

## 11. ESS Polling — Correct Status Set and Bounded Wait

Accurate OIC Oracle ESS terminal statuses for an HDL job:

| ESS Status | Meaning | WHILE loop action |
|---|---|---|
| `RUNNING` | In progress | Continue polling |
| `WAIT` | Queued, not started | Continue polling |
| `READY` | Pending execution | Continue polling |
| `PAUSED` | Waiting on sub-step | Continue polling |
| `Not Started` | Initial state before pickup | Continue polling |
| `SUCCEEDED` | Completed successfully | **Exit loop** |
| `ERROR` | Completed with errors | **Exit loop** |
| `WARNING` | Completed with warnings | **Exit loop** |
| `CANCELLED` | Job was cancelled | **Exit loop** |
| `BLOCKED` | Dependent job blocking | **Exit loop** (notify operations) |

**`COMPLETED` is not in either set.** It is not returned by `getESSJobStatus` for HDL-type ESS jobs. Including it in the WHILE-continue condition (as all 4 current integrations do) adds a dead condition that can never match. It is harmless today only because the status is never actually returned — but it masks a misunderstanding of the ESS status model.

**Corrected WHILE condition:**
```
$HDLJobStatus = "RUNNING"
or $HDLJobStatus = "WAIT"
or $HDLJobStatus = "READY"
or $HDLJobStatus = "PAUSED"
or $HDLJobStatus = "Not Started"
```

---

## 12. Summary: Current vs. Blueprint

| Dimension | Current | Blueprint |
|---|---|---|
| Integration count | 4 independent, flat | 1 coordinator + 4 domain + 3 shared sub-integrations |
| FBDI pipeline | Duplicated 4 times (copy-paste) | 1 reusable `SUB_FBDI_SUBMIT_AND_POLL` |
| Source system ID lookup | O(n×5) BIP calls inside loop | O(1) bulk BIP pre-fetch before loop |
| HDL file generation | 53 line-by-line STAGE_WRITEs + header flags | XSL-driven XML→HDL transformation |
| Endpoint management | 7 JCA files hardcoded to DEV2 URL | All via `%%PROPERTY` at connection level — zero JCA hardcoding |
| Empty router branches | 25+ silent data drops | Every branch explicitly logs or throws a named fault |
| ESS poll exit condition | Includes incorrect "COMPLETED" status | Correct terminal status set; bounded by maxPolls counter |
| Error handling | 1 global catch-all → 1 email | 3-level structured fault hierarchy with fault classification |
| Monitoring/search | No business identifiers, no structured log | 3 business identifiers + JSON activity stream entries at each milestone |
| Schedule dependency | Time-stagger (10–20 min gap, no guard) | Coordinator: sequential invocation, actual-completion guard |
| Notifications | Free-text, inconsistent, typos | Standardized schema per outcome type, correct placeholder usage |
| OIC project warnings | 25+ at deploy time | Zero warnings required before deployment |
