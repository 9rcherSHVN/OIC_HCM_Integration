# Design Inefficiency, Defects & Operational Observability Assessment

---

## Part A: Design Inefficiencies and Defects (Complete Catalogue)

This expands on the defects found in Goal 1 with additional evidence gathered from notification templates, tracking configurations, and variable expressions.

---

### D1 — CRITICAL: Empty Router Branches (Silent Record Loss)

**Integrations affected:** IFP_EMPLO (25+ instances), IFR_EMPLOYEENA (multiple)

**Evidence:** `project_messages.json` contains 25+ `COMPOSER_ACTION_EMPTY_WARNING_ROUTE` warnings in IFP_EMPLO. The project deploys with `projectHasWarnings=true`.

**Operational impact:** When an `Action` value arrives that matches a router condition whose branch has no action assigned, the record is discarded with zero trace — no log entry, no error, no count increment. IT support has no way to know a record was dropped. The daily reconciliation counts (`var_total_recordcount`, `var_sucess_recordcount`, `var_Error_recordcount`) will be understated by the number of silently dropped records, making discrepancy detection impossible.

---

### D2 — CRITICAL: Cross-Environment Endpoint Hardcoding in JCA Files

**Evidence (from direct JCA file scan):**

| JCA File | Integration | Hardcoded Endpoint |
|---|---|---|
| `CallImportErrorReport_REQUEST.jca` | IFP_EMPLO | `fa-eqcg-dev2-saasfaprod1` (DEV2) |
| `CallAssignSupImportErrorReport_REQUEST.jca` | IFP_EMPLO | `fa-eqcg-dev2-saasfaprod1` (DEV2) |
| `GetWRSourceSystemID_REQUEST.jca` | IFP_EMPLO | `fa-eqcg-dev2-saasfaprod1` (DEV2) |
| `GetWTSourceSystemID_REQUEST.jca` | IFP_EMPLO | `fa-eqcg-dev2-saasfaprod1` (DEV2) |
| `GetWASourceSystemID_REQUEST.jca` | IFP_EMPLO | `fa-eqcg-dev2-saasfaprod1` (DEV2) |
| `GetAssignmentSupervisor_REQUEST.jca` | IFP_EMPLO | `fa-eqcg-dev2-saasfaprod1` (DEV2) |
| `GetManagerSourceSystemID_REQUEST.jca` | IFP_EMPLO | `fa-eqcg-dev2-saasfaprod1` (DEV2) |
| `GetSourceSystemID_REQUEST.jca` | IFR_EMPLOYEENA | `fa-eqcg-saasfaprod1` (PROD) |
| `GetEmailSourceSystemID_REQUEST.jca` | IFR_EMPLOYEENA | `fa-eqcg-saasfaprod1` (PROD) |
| `CallImportErrorReport_REQUEST.jca` | IFP_JOB, IFR_EMPLO_DEPAR | `fa-eqcg-saasfaprod1` (PROD) |

**The same package contains DEV2 and PROD endpoints simultaneously.** This means when this IAR package is imported as-is to the production OIC instance, IFP_EMPLO's BI Publisher source system ID lookups will query DEV2 data — returning either no results or wrong source system IDs — causing HDL update failures or data corruption for all non-HIRE employee records.

---

### D3 — HIGH: Hardcoded `var_Environment_type = "DEV"` in IFP_EMPLO

**Evidence:** `all_vars.txt` — `TextExpression : "DEV"` assigned to `var_Environment_type`.

**Operational impact:** If `var_Environment_type` is used in any conditional logic or notification content, it always reports DEV regardless of which OIC instance the integration is running on. IT support receiving a notification showing "Environment: DEV" while investigating a production incident will have no way to confirm from the notification alone whether this is production data.

---

### D4 — HIGH: FusionURL DVM Lookup Uses Literal RICE_ID, Not Variable

**Evidence from `all_vars.txt`:**
```
XpathExpression : dvm:lookupValue('Common_Utility_Lookup','RICE_ID','I-03','FusionURL','')
```

The FusionURL lookup for constructing the UCM error report link uses the **literal string `'I-03'`** in several integrations — not the variable `$Var_Rice_Id`. If IFP_JOB or IFR_EMPLO_DEPAR have a different RICE_ID for their own DVM entry, this lookup will always return the URL from the Employee integration's DVM row, not from their own row. The UCM link in the notification email may point to the wrong Oracle Fusion environment for those integrations.

---

### D5 — HIGH: `{Content}` Placeholder in Notification HTML Uses Unquoted Attribute

**Evidence from all notification_body.data files:**
```html
<a href={Content}>click here</a>
```

The correct HTML for a dynamic hyperlink is `<a href="{Content}">click here</a>` — with quotes around the attribute value. Without quotes, virtually all email clients (Outlook, Gmail, Apple Mail) will reject this as malformed HTML and will **not render a clickable link**. IT support receiving this email sees the text "click here" but cannot click it. Accessing the UCM error report then requires manually navigating to UCM, knowing the document name from the plain text of the email, and finding the document through UCM search.

---

### D6 — HIGH: ESS Polling While-Loop Includes "COMPLETED" (Invalid Status) and Has No Timeout Guard

**Evidence:**
```
$HDLJobStatus = "PAUSED" or $HDLJobStatus = "COMPLETED" or $HDLJobStatus = "Not Started"
or $HDLJobStatus = "RUNNING" or $HDLJobStatus = "WAIT" or $HDLJobStatus = "READY"
```

Two issues:
1. `"COMPLETED"` is not a valid ESS job status for HCM Data Loader jobs (correct terminal statuses are `SUCCEEDED`, `ERROR`, `WARNING`, `CANCELLED`, `BLOCKED`). Its presence is harmless today only because that string is never actually returned.
2. There is no poll counter — if ESS never transitions to a terminal status (hung job, HDL service outage), the WHILE loop runs indefinitely, consuming the OIC worker thread until the OIC instance is manually aborted. No notification or alert is issued during the hanging loop.

---

### D7 — MEDIUM: Duplicate FBDI Submission Cycle in IFP_EMPLO (Uncoordinated)

IFP_EMPLO submits two separate HDL loads sequentially:
- **Load 1**: Worker/PersonName/WorkRelationship/WorkTerms/Assignment/ExternalIdentifier/PersonEmail
- **Load 2**: AssignmentSupervisor

If Load 1 succeeds but Load 2 fails, the notification for Load 2 failure has no reference to the successful Load 1. An IT support engineer receiving a "Load 2 failed" email has no information about whether the base employee record was loaded successfully — the two outcomes are reported independently with no cross-reference.

---

### D8 — MEDIUM: No Timeout or Retry on BI Publisher SOAP Calls

The 5 per-record BI Publisher lookups in IFP_EMPLO and the 2 pre-fetch lookups in IFR_EMPLOYEENA have no timeout configuration in their JCA files and no retry logic. A transient BI Publisher outage mid-loop throws the entire integration to the global CATCH_ALL, losing all context about which record triggered the failure and how many records had already been processed.

---

### D9 — LOW: Inconsistent ESS Polling Service Endpoint

IFR_EMPLOYEENA uses `ErpIntegrationService.getESSJobStatus` (WSDL: `erpIntegrationService`) for ESS polling, while IFP_EMPLO, IFP_JOB, and IFR_EMPLO_DEPAR use the HCM adapter's `getESSJobStatus`. Both call the same underlying Oracle ESS API, but via different service contracts. This makes the integrations harder to maintain consistently — a change in Oracle's ESS API would need to be addressed in two different service binding styles.

---

### D10 — LOW: Project Description Copy-Paste Errors

IFP_JOB and IFR_EMPLO_DEPAR both have `projectDescription = "This creates Employees in Oracle from Datawarehouse"` — copied from the Employee integration template. IFR_EMPLOYEENA description = "Fixed field mapping" (a commit note, not a description). None of these are operationally descriptive. A new IT support engineer reading the OIC console cannot identify what each integration does from its description.

---

### D11 — LOW: IFR_EMPLOYEENA JCA Typo

`GetLoadImportStatuis_REQUEST.jca` — "Statuis" instead of "Status". While harmless to runtime execution, it breaks the consistent naming convention used across all other JCA files and will cause confusion during future maintenance.

---

### D12 — LOW: Notification Body Inconsistencies Across Integrations

| Notification | IFP_EMPLO | IFP_JOB | IFR_EMPLO_DEPAR | IFR_EMPLOYEENA |
|---|---|---|---|---|
| Error email includes `{DateTime}` | No | Yes | Yes | No |
| Success email includes `{DateTime}` | No | Yes | Yes | No |
| Global fault body has typo | No | No | No | Yes ("GlobaleFault", "appural") |
| Raw `{errorMessage}` notification (no context) | No | No | No | Yes |
| Record counts in any email body | No | No | No | No |

IT support receives different levels of information depending on which integration generates the email. The absence of `{DateTime}` in IFP_EMPLO and IFR_EMPLOYEENA error emails is particularly problematic — if the email is delayed or arrives in a batch, there is no timestamp in the body to determine when the event occurred.

---

## Part B: Operational Monitoring Capability Assessment

The three operational scenarios from Goal 2:

---

### Scenario 1: Diagnosing Data Discrepancies Between Data Warehouse and Oracle HCM

**Question an IT support engineer needs to answer:** "Employee #1234 was transferred to a new department on Monday. Why does Oracle HCM still show the old department on Wednesday?"

**What the current implementation provides:**

| Step | Capability | Tool | Friction |
|---|---|---|---|
| 1 | Was the employee in Monday's source file? | SFTP archive | Requires SFTP access; need to know the archive path (from DVM) |
| 2 | Did the integration run on Monday? | OIC Monitoring — Integrations | No business identifiers → must visually scan all instances by date/time |
| 3 | Was the record processed? | OIC Instance Activity Stream | Only 1–2 activity stream entries exist; no per-record log |
| 4 | Did the record succeed or fail in HDL? | UCM error report | Requires UCM access; URL from email may not be clickable (`href={Content}` without quotes) |
| 5 | What was the HDL rejection reason? | UCM error report CSV | Requires downloading and reading the raw FBDI error CSV |
| 6 | Was it an empty router branch drop? | **No artifact available** | Silent drop — no record of this anywhere |

**Verdict:** An investigation into a single employee discrepancy requires access to 3 separate systems (OIC, SFTP, UCM), manual correlation between them, and a minimum of 30+ minutes of manual work — assuming the UCM link in the email is functional. If the record was silently dropped by an empty router branch (D1), the investigation hits a dead end: the integration shows "SUCCESS", the count matches, and there is no log of the dropped record anywhere.

---

### Scenario 2: Identifying and Tracing Integration Process Failures

**Question:** "The integration failed this morning. What failed, at what step, and why?"

**What the current implementation provides:**

The global CATCH_ALL sends one email per integration with:
- `{Instance_ID}` — allows the IT support engineer to look up the OIC instance in the monitoring console
- `{Error_Message}` — the raw OIC fault payload, which in practice is one of:
  - A Java exception stack trace (e.g., `java.net.SocketTimeoutException: connect timed out`) — readable but not structured
  - A SOAP fault XML envelope from Oracle HCM or BI Publisher — requires XML parsing to read
  - An OIC internal fault message — often references internal OIC processor IDs, not meaningful to IT support

**What the monitoring console shows for a failed instance:**
- Status: `FAILED`
- Duration: elapsed time
- Activity Stream: the most recent logged action before the fault — **but only if an `ACTIVITY_STREAM_LOGGER` node had been executed before the fault**. Since these integrations have only 1–2 activity stream entries total, a fault that occurs during the FOR loop iteration (e.g., during the 3rd of 500 BIP lookups) may have no activity stream entry between the DVM lookup and the failure.

**There is no fault classification.** Every failure — whether it is a network timeout, an HDL business validation rejection, an SFTP authentication failure, or an XPath null dereference — produces the same `FAILED` status in OIC monitoring and the same raw `{Error_Message}` in the notification email. IT support cannot determine the category of failure without reading the raw fault XML.

**There is no record of which step failed.** The OIC monitoring console shows the integration status (FAILED), the total execution duration, and the last completed step visible in the flow diagram — but the flow diagram for IFP_EMPLO has 300+ nodes; identifying the failed step requires the OIC console to visually highlight it, which requires the support engineer to open the instance in the OIC designer-level view.

---

### Scenario 3: Tracking and Auditing Transaction Record Status

**Question:** "Show me the status of every employee record processed in this week's batch. Which ones succeeded? Which failed? What are the failure reasons?"

**What the current implementation provides:**

| Capability | Available? | Evidence |
|---|---|---|
| Total records in source file | Indirectly — `var_total_recordcount` exists | Not surfaced in notification body; only used for routing decision |
| Records successfully imported | Indirectly — `var_sucess_recordcount` exists | Not surfaced in notification body |
| Records failed import | Indirectly — `var_Error_recordcount` exists | Not surfaced in notification body |
| Per-record processing status | **No** | No per-record log in OIC, staging, or external store |
| Per-record failure reason | **No (for silent drops)** | Only HDL rejections appear in the BI Publisher error CSV |
| ESS Job ID for cross-referencing | **No** | Not in notification email |
| Run-date searchable instances in OIC | **No** | No business identifiers configured |
| Historical audit trail (more than OIC retention period) | **No** | No external audit log |

**Confirmed from the notification body template:**
```html
<p>The Employee Integration is completed with Error or warning.</p>
FileName: {fileName}
InstanceID: {instanceID}
```
The notification body does not include `var_total_recordcount`, `var_sucess_recordcount`, or `var_Error_recordcount`. IT support receives an email saying "completed with error" but does not know from the email alone whether 1 record failed out of 1,000 or all 1,000 records failed.

**Confirmed from project.xml:** `trackingFields` = NONE and `businessIdentifiers` = NONE across all 4 integrations. OIC monitoring instance list shows only: timestamp, status (SUCCESS/FAILED), and duration. There is no searchable field for source file name, run date, domain (Employee/Job/Department), or employee ID.

---

## Part C: Observability Gap Summary

| Operational Need | Provided | Gap |
|---|---|---|
| Know which integration ran on a given date | OIC monitoring (date filter) | No business identifier for run date or file name — cannot search, only scroll |
| Know the outcome of a specific integration run | OIC instance status (SUCCESS/FAILED) | Binary — no contextual outcome (partial success with 12 failures out of 500 records) |
| Know record counts (total / success / error) | Variables exist internally | **Not in notification email. Not in OIC business identifiers. Not logged.** |
| Know which specific records failed | BI Publisher HDL error CSV (in UCM) | Requires UCM access; UCM link in email is broken (unquoted `href`); no record for silent drops |
| Know what pipeline step failed on a fault | OIC FAILED instance, last flow node | Only 1–2 activity stream entries — pre-failure step may not be logged |
| Know the fault category (network, validation, HDL, XPath) | Raw `{Error_Message}` | No classification; raw SOAP/Java fault payload only |
| Search OIC monitoring by source file name | **Not available** | No business identifier configured |
| Search OIC monitoring by employee ID | **Not available** | No per-record business identifier |
| Audit trail beyond OIC instance retention period | **Not available** | No external log, database, or persistent audit store |
| SLA breach detection (integration running too long) | **Not available** | No alerting on duration; no ESS poll timeout notification |
| Cross-integration correlation for same employee | **Not available** | No shared correlation ID across IFP_EMPLO + IFR_EMPLOYEENA runs |
| Distinguish DEV from PROD in notifications | Partially — `{SUBJECT_PARAM_1}` may include env name | `var_Environment_type = "DEV"` hardcoded — always reports DEV |

---

## Part D: Compliance and Audit Dimension

HR data integrations are subject to data governance requirements. The current implementation cannot answer:

- "Who changed employee 1234's data, when, and from which system?" — No integration-side audit record exists.
- "What was the state of this record when it was sent to Oracle HCM?" — Staging files are transient and are not archived at the OIC layer; only the CSV archive on SFTP remains.
- "Was this HCM data load authorized and who triggered it?" — The integration is scheduled and auto-triggered; there is no approval workflow or authorization record in the integration layer.

These are not implementation bugs, but they represent risks for any regulated HR data compliance framework (SOX, GDPR, PIPEDA depending on jurisdiction).

---


# Recommended Improvements

*All recommendations are grounded in confirmed OIC capabilities and documented defects from Goals 1 and 2. Recommendations are prioritized by operational impact.*

---

## Priority 1 — Fix Before Next Production Run

### R1: Remove All Hardcoded Endpoint URLs from JCA Files (D2)

**Affected files:** 7 JCA files in IFP_EMPLO (all pointing to `fa-eqcg-dev2-saasfaprod1`), plus `CallImportErrorReport` in IFP_JOB and IFR_EMPLO_DEPAR.

**Action:** Remove the `<property name="endpointURL">` element from each affected JCA file. With this element absent, the OIC SOAP adapter uses the connection-level `targetWSDLURL` at runtime — which is already parameterized via `%%IFP_OIC_REPORT_SERVICE_targetWSDLURL`. No logic change required. This is a one-line deletion per JCA file.

**Verification:** After removing, re-import the IAR to a DEV OIC instance, configure the connection property to the DEV endpoint, run the integration against a test file, confirm BI Publisher lookups return valid data.

---

### R2: Fix Empty Router Branches — No Silent Record Drops (D1)

**Affected integrations:** IFP_EMPLO (25+ branches), IFR_EMPLOYEENA (multiple).

**Action:** For each `COMPOSER_ACTION_EMPTY_WARNING_ROUTE` branch, determine intent and assign one of:
- **Intentional skip** → Add an `ACTIVITY_STREAM_LOGGER` node: `"SKIPPED: Action={action}, PersonNumber={person_number}, Reason=not applicable for this object type"`
- **Unexpected data** → Add a `THROW` fault: `"Unexpected Action code [{action}] for PersonNumber [{person_number}] — record not processed"`

**Acceptance criterion:** OIC integration health check shows `projectHasWarnings=false` at deploy time. Zero `COMPOSER_ACTION_EMPTY_WARNING_ROUTE` entries.

---

### R3: Fix the UCM Error Report Hyperlink in All Notification Emails (D5)

**Root cause (confirmed from artifacts):** The notification body template contains:
```html
<a href={Content}>click here</a>
```
The `{Content}` placeholder resolves at runtime to:
```
https://{FusionURL}/cs/idcplg?IdcService=GET_FILE&dID={docID}
```
Two problems in the resulting HTML:
1. The `href` attribute value is unquoted — HTML specification requires quoting when the value contains `&`, `=`, or `/`
2. The `&` between query parameters is not HTML-encoded as `&amp;`

Combined, Outlook (which renders HTML email via the Word engine) will not render this as a clickable hyperlink.

**Fix:**
```html
<a href="{FusionURL}cs/idcplg?IdcService=GET_FILE&amp;dID={DocumentId}">Click here to view the HCM import error report</a>
```
Pass `FusionURL` and `DocumentId` as separate notification parameters, and embed `&amp;` as a literal string in the template body (not in the runtime value).

---

## Priority 2 — High-Impact Operational Improvements

### R4: Configure Business Identifiers on All 4 Integrations (OIC Native Feature — Zero Code)

**What this is:** OIC allows up to 3 business identifiers per integration, configured in the OIC Designer under the integration's tracking settings. At runtime, OIC evaluates the mapped XPath expressions and stores the results as searchable columns in the Monitoring console.

**Confirmation:** The `trackingFields` and `businessIdentifiers` elements in all 4 `project.xml` files are confirmed empty — this feature is completely unused.

**Recommended identifiers for all integrations:**

| Identifier Label | Source Expression | Purpose |
|---|---|---|
| `Source_File` | `$InputFileName` (from DVM lookup) | Search all runs that processed a specific source file |
| `Run_Date` | `fn:format-dateTime(fn:current-dateTime(),'[Y0001]-[M01]-[D01]')` | Search all runs for a given business date |
| `Integration_Domain` | Literal constant (e.g., `"EMPLOYEE"`, `"JOB"`, `"DEPARTMENT"`, `"NAME_EMAIL"`) | Filter monitoring view by HR domain |

**Result:** IT support can open OIC Monitoring → search by `Source_File = "employees_20260514.csv"` and immediately see the specific instance, its outcome, and activity stream — instead of scrolling through all instances sorted by timestamp.

---

### R5: Add Structured Activity Stream Logging at 6 Key Milestones

Add an `ACTIVITY_STREAM_LOGGER` node at each of the following points. The log message should be a structured JSON string:

| Milestone | Placement | Log Content |
|---|---|---|
| **1. File Check** | After SFTP ListFile | `{"step":"FILE_CHECK","status":"FOUND","file":"{fileName}","recordCount":"{count}"}` |
| **2. HDL Submitted** | After importAndLoadData | `{"step":"HDL_SUBMIT","essJobId":"{essRequestId}","ucmDocId":"{contentId}","file":"{fileName}"}` |
| **3. ESS Completed** | After WHILE exits | `{"step":"ESS_COMPLETE","essJobId":"{essRequestId}","essStatus":"{HDLJobStatus}","pollCount":"{n}"}` |
| **4. Dataset Status** | After getDataSetStatus | `{"step":"DATASET_STATUS","importSuccess":"{n}","importFailed":"{n}","loadSuccess":"{n}","loadFailed":"{n}"}` |
| **5. Error Report** | After error log UCM upload | `{"step":"ERROR_REPORT","errorLogDocId":"{docId}","ucmUrl":"{fullUrl}"}` |
| **6. Completed** | Final step before end | `{"step":"COMPLETED","status":"{var_import_status}","archivePath":"{archivePath}","instanceId":"{oicInstanceId}"}` |

These 6 structured entries give IT support a complete execution trace in the OIC Monitoring console for any instance, without needing to open any external system.

---

### R6: Add Record Counts to All Notification Email Bodies

**Confirmed gap:** `var_total_recordcount`, `var_sucess_recordcount`, `var_Error_recordcount` are computed and used for routing decisions but are **not included in any notification body** across all 4 integrations.

**Fix — add to all error/partial/success notification bodies:**
```
Records Processed  : {totalRecords}
Records Succeeded  : {successRecords}
Records Failed     : {errorRecords}
ESS Job ID         : {essJobId}
OIC Instance ID    : {instanceId}
Run Date/Time      : {dateTime}
Error Report (UCM) : {errorReportUrl}
```

Pass `totalRecords`, `successRecords`, `errorRecords`, `essJobId`, and `dateTime` as additional notification parameters mapped from the corresponding flow variables.

**Impact:** IT support can triage severity from the email alone — "12 of 1,200 records failed" vs "1,200 of 1,200 records failed" require very different escalation responses.

---

### R7: Bound the ESS Polling WHILE Loop and Fix the Status Condition

**Two changes in one:**

**Change 1 — Remove "COMPLETED" from the continue condition.** Confirmed: `COMPLETED` is not a valid ESS terminal status for HCM Data Loader jobs. The correct terminal statuses are `SUCCEEDED`, `ERROR`, `WARNING`, `CANCELLED`, and `BLOCKED`. Remove `$HDLJobStatus = "COMPLETED"` from the WHILE condition in all 4 integrations.

**Change 2 — Add a poll counter with a maximum:**
```
Initialize: pollCount = 0, maxPolls = 36  (36 × 10s = 6 minutes max wait)

WHILE condition:
  ($HDLJobStatus = "RUNNING" or $HDLJobStatus = "WAIT" or $HDLJobStatus = "READY"
   or $HDLJobStatus = "PAUSED" or $HDLJobStatus = "Not Started")
  AND ($pollCount < $maxPolls)

Inside loop body (after WAIT):
  pollCount = pollCount + 1

After WHILE exits — add ROUTER:
  If $pollCount >= $maxPolls AND $HDLJobStatus still in-progress set:
    THROW fault "ESS_POLL_TIMEOUT: ESS Job {essJobId} did not reach terminal status after {maxPolls} polls"
```

**Why maxPolls = 36:** Oracle HCM Data Loader jobs for typical payroll-sized files complete within 1–3 minutes. A 6-minute timeout catches genuinely hung jobs while accommodating large file runs. Adjust per actual SLA.

---

### R8: Remove `var_Environment_type = "DEV"` Hardcoding (D3)

**Action:** Replace the hardcoded assignment `"DEV"` with a DVM lookup — add a new column `EnvironmentType` to `Common_Utility_Lookup.dvm`. Each RICE_ID row gets the correct value (`DEV2`, `UAT`, `PROD`) per environment. The OIC integration deployed to production would reference the production DVM which returns `PROD`.

Alternatively, derive environment from `FusionURL` at runtime: if `FusionURL` contains `dev`, assign `DEV2`; if it contains `saasfaprod` without `dev`, assign `PROD`.

---

### R9: Add an SFTP-Based Per-Run Audit Log

**What to implement:** After every integration run completes (success, partial, or error), write a single-line CSV record to a dedicated SFTP audit path (`/Audit/{domain}/audit_{YYYY-MM}.csv`):

```
RunDate,RunTime,IntegrationCode,SourceFile,TotalRecords,SuccessRecords,FailedRecords,ESSJobId,UCMDocId,OICInstanceId,Status
2026-05-14,22:35:47,IFP_EMPLO,employees_20260514.csv,1200,1188,12,12345678,EMPLERR20260514,IFP_EMPLO_XXXX,PARTIAL
```

**Why SFTP:** OIC instance data is subject to OIC platform retention policies (typically 30–90 days depending on storage tier). The SFTP audit log persists indefinitely, is accessible by both integration teams and compliance teams, and requires no additional infrastructure beyond the existing SFTP connection.

**Implementation:** Add one `STAGE_WRITE` (audit line) + one SFTP `WriteFile` (append mode) at the end of each integration's main flow, after the final status is determined.

---

### R10: Standardize IFR_EMPLOYEENA ESS Polling to Use HCM Adapter (D9)

**Current:** `IFR_EMPLOYEENA` uses `ErpIntegrationService.getESSJobStatus` via the SOAP adapter.
**Other 3 integrations:** Use the HCM Cloud Adapter's native `getESSJobStatus` operation.

**Action:** Refactor IFR_EMPLOYEENA to use the HCM adapter for ESS polling, matching the pattern of the other 3 integrations. This eliminates the dependency on a separate `ErpIntegrationService` WSDL binding and ensures consistent behavior if Oracle changes the ESS API schema.

---

### R11: Correct the FusionURL DVM Lookup Hardcoded RICE_ID (D4)

**Confirmed from artifacts:** The FusionURL lookup expression in IFP_EMPLO uses the literal `'I-03'` instead of the variable `$Var_Rice_Id`:
```xpath
dvm:lookupValue('Common_Utility_Lookup','RICE_ID','I-03','FusionURL','')
```

**Fix:** Replace the literal `'I-03'` with the variable `$Var_Rice_Id` in all FusionURL and EnvironmentLink lookup expressions across all integrations. This ensures each integration uses its own DVM row for the Oracle Fusion base URL.

---

## Priority 3 — Quality and Maintainability

### R12: Fix All Notification Body Content Issues

| Issue | Fix |
|---|---|
| IFR_EMPLOYEENA global fault: "GlobaleFault", "appural action" | Correct to "Global Fault", "appropriate action" |
| IFR_EMPLOYEENA raw `{errorMessage}` notification with no context | Replace with structured fault body template matching other integrations |
| IFP_EMPLO and IFR_EMPLOYEENA error emails missing `{DateTime}` | Add `DateTime` parameter to match IFP_JOB and IFR_EMPLO_DEPAR pattern |
| IFP_JOB and IFR_EMPLO_DEPAR description copy-pasted from Employee template | Rewrite descriptions to accurately describe each integration's purpose |
| IFR_EMPLOYEENA: `GetLoadImportStatuis_REQUEST.jca` typo | Rename to `GetLoadImportStatus_REQUEST.jca` (requires JCA file rename + project reference update) |

---

### R13: Dual FBDI Submission in IFP_EMPLO — Cross-Reference in Notifications

**Current:** Load 1 (Worker/Assignment) and Load 2 (AssignmentSupervisor) send separate notifications. A Load 2 failure email has no reference to Load 1's outcome.

**Fix:** Capture Load 1 outcome variables (`var_import_status`, ESS Job ID, counts) before Load 2 executes. In the Load 2 notification, include a "Previous load (Worker/Assignment) outcome" section alongside the AssignmentSupervisor outcome. Both loads share the same OIC Instance ID, providing a correlation key.

---