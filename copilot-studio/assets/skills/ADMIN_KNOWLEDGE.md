# IBM BAW Admin — Skill Knowledge

This knowledge pack is derived from IBM BAW Skills and Bob Modes (Apache-2.0).
Source: https://github.com/ibmbpm/ibm-baw-skills-and-bob-modes

---

---
name: inspect-baw-processes
description: "Use when a BAW administrator or operator wants to find, investigate, or take action on running IBM BAW process instances: retry failed processes, suspend or resume instances, change due dates, or answer questions like \"what's broken right now?\", \"show me everything overdue in the Claims app\", or \"why did this instance fail?\" Covers the full Process Inspector workflow: searching by status/app/owner/date, drilling into instance details and business data, and executing lifecycle actions. Do not use for authoring new processes (generate-baw-bpmn), creating coach widgets (create-baw-widget), or building custom portals (create-baw-portal). Do not use for audit readiness, compliance checks, or questions about whether a process application is ready for an audit. Use audit-baw-readiness for those."
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Read
  - Edit
  - Bash
metadata:
  version: "1.0.0"
  display_name: Inspect BAW Processes
  short_description: "Find, investigate, and act on running IBM BAW process instances: retry failures, suspend, resume, and change due dates."
  author: IBM BAW Team
---
# Inspect BAW Processes

Search, inspect, and act on IBM Business Automation Workflow (BAW) process instances at runtime. This skill covers three categories of work: finding instances by filter, inspecting the details of a specific instance, and taking lifecycle actions (suspend, resume, retry, terminate, change due date).

One rule applies to all categories: **always resolve the instance ID by searching first**. The instance ID must always come from a search result, not from the user.

---

## Execution mode

At the start of the session, check your tool list once and pick a path:

- If you have tools for **process instance and task** operations → MCP path. Use the MCP operations described in this skill.
- If no such tools are visible → REST path (see `references/baw-rest.md`).

Stay on the chosen path for the entire session. Do not use a failed tool call as evidence of absence. Only absence from the tool list counts.

---

## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---

## Semantic operations

The capability logic in this skill uses transport-agnostic operation names. When you need to run one, look it up in the appropriate reference file. The following table describes each operation:

| Operation | What it does |
|---|---|
| `list_exposed_processes` | Get all process apps with their acronyms |
| `search_processes` | Filter instances by status, app, dates |
| `get_process_details` | Full metadata, business data, and tasks for one instance |
| `get_process_documents` | Documents attached to an instance |
| `suspend_process(id)` | Halt a running instance |
| `resume_process(id)` | Restart a suspended instance |
| `retry_process(id)` | Re-run a failed instance |
| `modify_process_due_date(id, date)` | Change the due date |
| `terminate_process(id)` | Permanently end an instance |

**Valid state transitions**

The following table shows the instance status that each operation requires:

| Operation | Only valid when instance status is |
|---|---|
| `suspend_process` | Active |
| `resume_process` | Suspended |
| `retry_process` | Failed |
| `modify_process_due_date` | Any |
| `terminate_process` | Any |

**Status vocabulary**

Translate user language to a status value before filtering:

| User says | Status value |
|---|---|
| stuck, paused, on hold | Suspended |
| broken, errored, failed | Failed |
| running late, overdue | Late |
| at risk | At_Risk |
| running, active | Active |
| done, finished | Completed |
| killed, stopped | Terminated |

---

## Category A: Finding instances

1. **Translate intent to filter parameters.** Map natural language to the status vocabulary above. If the user names a process app by full name, note that `search_processes_by_filter` requires an acronym, and go to step 2. If no app name was given, skip to step 3.

2. **Resolve app name to acronym.** Run `list_exposed_processes`. Find the entry whose app name matches what the user said and read its acronym (for example, `HSS`). If no match is found, tell the user and offer to search without the app filter.

3. **Run `search_processes_by_filter`** with the resolved parameters. All filters are optional. Omitting all filters returns all instances. Never call `search_processes` (advanced DSL) for natural-language requests.

4. **Present results** as a clean table: Instance ID, Name, Status, Due Date. Do not show raw JSON. If the list is empty, say so and suggest loosening the filter.

5. **Offer next steps:** inspect a specific instance, or take a lifecycle action. Wait for the user to choose.

---

## Category B: Inspecting a specific instance

1. **Get the instance ID.** Use results from Category A if available. Resolve the ID from search results. Asking the user to type a raw ID is a last resort.

2. **Run `get_process_details`.** Present the results in labeled sections:
   - **Instance Info:** name, status, due date, start time
   - **Business Data:** key/value list, not raw JSON
   - **Open Tasks:** task name, assigned to, due date

3. **Offer documents.** If the user mentioned documents, run `get_process_documents` immediately. Otherwise ask: "Would you also like to see documents attached to this instance?"

4. **Offer next steps:** offer applicable lifecycle actions based on the current status of the instance.

---

## Category C: Taking an action on an instance

1. **Confirm the instance ID.** If not already known, run Category A first. If the user described a process by name, search and present matches before acting.

2. **Check the instance is in the right state** by using the state transitions table. If the user asks to resume an Active instance, stop and say: "This instance is Active. It does not need to be resumed. Did you mean to suspend it?"

3. **Apply the right confirmation level:**
   - **Suspend:** Halts active work and might block people. Always ask: "This will suspend [name] ([id]), stopping any active tasks. Proceed?" Wait for confirmation.
   - **Retry:** First note: "Before retrying, make sure the underlying issue is resolved. Retrying against an unresolved problem fails again and might trigger duplicate side effects." Then ask: "Retry [name] ([id])?" Wait for confirmation.
   - **Resume:** Low risk and reversible. Confirmation is not needed.
   - **Modify due date:** No state change. Confirmation is not needed.
   - **Terminate:** Irreversible. Always ask explicitly with the instance name and ID. Do not proceed without a clear "yes."

4. **Run the operation** by using the appropriate reference file.

5. **Report the result** as one line: "Instance [name] ([id]) is now [state]." If the call fails with a state-conflict error (for example, 400), check the transitions table and explain what went wrong.

6. **Offer follow-up:**
   - After suspending: "Would you like to resume it later, or inspect its current tasks?"
   - After resuming: "Would you like to monitor its progress?"
   - After retrying: "Would you like to check its status in a moment?"
   - After changing due date: "Would you like to update any other instances?"

**Bulk actions:** Action operations work on one instance at a time. For bulk requests (for example, "retry all failed instances"), present the full list, confirm the count, then iterate one by one. Apply the confirmation level to the batch as a whole, not to each individual item.

---

## Disambiguation rules

**`search_processes_by_filter` vs `search_processes` (advanced DSL)**
Always call `search_processes_by_filter` first for all natural-language requests. It maps directly to the v1 BAW endpoint and is the reliable default. Call `search_processes` only when the user explicitly needs a complex boolean condition that `search_processes_by_filter` cannot express (for example, combining AND/OR across multiple fields). Never use `search_processes` as the default.

**App name vs acronym**
The user gives a full name, but the operation needs the acronym. Always resolve the acronym by calling `list_exposed_processes`. Never guess.

---

## Known limitations

**No owner/starter filter.** There is no way to filter instances by who started them. Explain this if the user asks.

**All action operations work on one instance at a time.** No bulk operation exists. Iterate for bulk requests.

**Non-federated environments.** In IBM Cloud Pak for Business Automation (CP4BA) federated deployments, instance results from the advanced search include a `system_id`, which you must pass to action operations. On standalone BAW, omit it. See `references/mcp.md` for details.

---

## Boundaries

In scope: searching instances by filter, inspecting instance details and documents, lifecycle actions (suspend, resume, retry, terminate, modify due date).

Out of scope (do not attempt):
- **Authoring or generating new processes.** Use `generate-baw-bpmn`.
- **Starting a new process instance.** Not covered by this skill.
- **Modifying process design or business objects.** Runtime inspection only.
- **Creating coach widgets or task portals.** Use `create-baw-widget` or `create-baw-portal`.
- **Packaging or deploying TWX toolkits.**

---

## Output format

**Search results:**

| Instance ID | Name | Status | Due Date |
|---|---|---|---|
| 15 | Standard Employee Requisition for... | Active | 2026-07-21T22:00:00Z |

**Instance details:** Labeled sections (Instance Info, Business Data, Open Tasks). Business data as a readable key/value list.

**Action confirmations:** "Instance [name] ([id]) is now [state]."

**Errors:** State what failed, the likely cause, and what the user can try next. Never show a raw stack trace or JSON blob.

---

## Example

**Input:** "Show me everything stuck or late in the Hiring Sample app, then suspend the oldest one."

1. "Stuck" → Suspended, "late" → Late. "Hiring Sample" needs an acronym. Run `list_exposed_processes` and find acronym `HSS`.
2. Run `search_processes(status=["Suspended","Late"], application="HSS")`. Get back instances 15, 22, 31.
3. Present the results as a table. Instance 15 has the earliest due date.
4. Ask: "Found 3 instances. Instance 15 (Standard Employee Requisition, due 2026-07-18) is most overdue. Suspend it?"
5. User confirms. Instance 15 is Active, so suspend is valid. Run `suspend_process("15")`.
6. Report: "Instance Standard Employee Requisition (15) is now Suspended."
7. Offer: "Would you like to suspend the others, resume this one later, or inspect any of them?"


---

---
name: troubleshoot-baw-logs
description: "Diagnoses IBM Business Automation Workflow (BAW) runtime failures by analyzing error logs, stack traces, and FFDC files. Identifies root causes, correlates entries across log types, and provides structured remediation steps. Use when the user: pastes BAW log content (SystemOut.log, messages.log, trace.log, FFDC files, or Liberty log lines); asks about a BAW error code (for example, BPMD0049E, CWTBG0019E, CWWKB*, CWLLG*); mentions a process instance in error or stuck state; says snapshot deployment or activation failed; or asks why a process/task/service failed. Also use when pointing to a local BAW log file, or when the MCP server is connected and they want get_process_instance_errors or get_baw_server_logs called. Trigger phrases: BAW error, FFDC, process is stuck, error state, task keeps failing, service call failed, stack trace, TWException, BPDInstance, messages.log, SystemOut.log, trace.log, BPMD, CWWKB, or any pasted log content with Liberty/BPM thread IDs and error codes."
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Read
  - Bash
metadata:
  version: "1.0.0"
  display_name: Troubleshoot BAW Logs
  short_description: Diagnose IBM BAW runtime failures from log files, stack traces, and FFDC files with root-cause analysis and remediation steps.
  author: IBM BAW Team
---
# Troubleshoot BAW Logs

Help developers and administrators diagnose IBM Business Automation Workflow (BAW) runtime failures quickly and accurately. The goal is to move from raw log data to a clear root cause with actionable next steps.

## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---

## Execution mode

At the start of the session, check your tool list once and pick a path:

- If you have `get_exposed_processes` in your tool list → Model Context Protocol (MCP) path. Use the MCP operations described in this skill.
- If `get_exposed_processes` is absent → REST path. Execute all REST calls directly using `execute_command` (curl). Do not ask the user to run curl commands themselves—run them for the user. See `references/baw-rest.md` for endpoint details and the REST fallback section below for the exact call sequence.

Stay on the chosen path for the entire session. Do not use a failed tool call as evidence of absence—only absence from the tool list counts.

Also read `references/BAW_CONNECTION.md` — it covers the no-MCP user prompt and how to retrieve `{BAW_BASE_URL}` from `.env` before making REST calls.

## How to detect which input mode applies

Read the user's message and pick the right starting point—modes are not mutually exclusive, use all available information.

**Live-log mode (MCP path only—try first when MCP server is connected)**—Call `get_baw_server_logs` to fetch the messages.log directly from the BAW server. This is only available on Liberty-based deployments. Check `available` in the response before proceeding:
- `available: true` → analyze `content` directly. Skip asking the user for log files.
- `available: false, error contains "404"` → endpoint not available on this server. Fall back to paste or file mode.
- `available: false, error contains "403"` → caller lacks admin rights. Tell the user and fall back.
- `available: false, error contains "empty response"` → server is not Liberty. Tell the user this API only works on Liberty and fall back.
- Any other error → tell the user what went wrong and fall back to paste or file mode.
- **No MCP available** → attempt live log retrieval through REST (`GET /ops/std/bpm/logs`) after establishing a session. See the REST fallback section. If that endpoint returns 404, 403, or an empty body, **stop immediately and ask the user**—do not probe any other endpoints.

**Paste mode**—The user has pasted raw log content (Liberty log lines, an FFDC snippet, a Java stack trace, a script error message). Analyze the text directly.

**File mode**—The user says "my log is at `/logs/SystemOut.log`" or similar, and you can read that path with `read_file`. Do so. If the path is inaccessible (remote server, no filesystem access), say so plainly and ask the user to paste the relevant section. Do not fail silently.

**Instance mode**—The user mentions a specific process instance ID (a number, or `BPDInstance.<number>`). When MCP is connected, call `get_process_instance_errors` first to retrieve structured error data, then combine it with whatever log content you have. When using REST, call `PUT /rest/bpm/wle/v1/process/errors?instanceIds={process_id}` to retrieve runtime error details—see the REST fallback section below for full details.

---

## Analysis steps

Before diving into any log content, read `references/BAW_LOG_FORMATS.md`. It covers the structure of every BAW log type, the `process/errors` API response fields, and the common error patterns table.

### Step 1: Orient yourself

Identify what you are looking at:
- Which log type(s) are present? (`messages.log`, `SystemOut.log`, `trace.log`, first failure data capture (FFDC) file, `process/errors` API response, or a mix)
- What is the failure surface? (Process instance stuck in error, failing script task, unreachable service, auth failure, DB issue, server startup failure)
- Is there a process instance ID in the text? (`BPDInstance.<number>`, `piid=<number>`)—capture it; you will use it for correlation.

### Step 2: Find the earliest causal error

Log files record *symptoms* repeatedly. Your job is to find the *cause*—the first error in the chain before the cascade began.

- **In Liberty logs (`messages.log` / `SystemOut.log`):** scan for `E` (ERROR) and `F` (FAILURE) level entries, sorted by timestamp. The earliest is usually the trigger. When you see `FFDC1015I`, note the filename it names—that file has the full stack.

  > **Important: do not conflate scheduler task IDs with process instance IDs.** BAW log messages contain many numeric IDs. A line like `CWLLG0181E: The following error occurred in the 360 task` refers to an **Event Manager scheduler task ID**, not a process instance. Only treat a numeric ID as a process instance reference when the log explicitly uses `BPDInstance.<number>`, `piid=<number>`, or the structured `instanceId` field from the `process/errors` API. Before attributing any log error to a specific instance, verify with a grep for `BPDInstance.<id>` in the log.
- **In FFDC files:** go straight to the `Stack Dump` section and read the `Caused by:` chain bottom-up—the deepest `Caused by:` is the root exception.
- **In `process/errors` API response:** `failedStepName` tells you where it broke; `errorMessage` tells you what the script said; `errorCode` gives you the BPM code to look up in `references/BAW_LOG_FORMATS.md`.
- **Cross-log correlation:** use the thread ID (8-digit hex, for example, `00000087`) to join entries across `messages.log`, `trace.log`, and the FFDC prologue. A single failing request stays on one thread.

### Step 3: Match against known patterns

Check the error code, exception class name, or key phrase against the Common error patterns table in `references/BAW_LOG_FORMATS.md` (section 7). If you find a match, use the "First Response Step" column as the starting point for your remediation advice.

If the error is not in the table, reason from first principles:
- Script errors → `failedScriptName` and `failedLine`; likely a null reference or type mismatch
- Service call failures → downstream endpoint unreachable or returned an unexpected response
- Authentication errors → Lightweight Directory Access Protocol (LDAP), OpenID Connect (OIDC), or token configuration
- DB errors (SQLCODE) → connection pool exhaustion or lock timeout
- Memory errors → large Business Object payloads or insufficient heap

### Step 4: Synthesize

Write the diagnosis report (see template below). Be specific—name the exact step, service, script, or configuration that failed. Do not just restate the error message; explain *why* it happened and *what to do next*.

---

## Using the MCP tools (MCP path)

Use the exact function names written below. Do not paraphrase, rename, or substitute a tool name.

### get_baw_server_logs (Liberty only)

Call this first whenever the MCP server is connected and the user has not already pasted log content. It fetches the live `messages.log` directly from BAW.

```
get_baw_server_logs(system_id=<optional>)
```

Always check `available` before using `content`. If `available` is false, read `error` and tell the user in plain language. An empty body means the server is not Liberty and this path will not work.

In the diagnosis report header, write the source exactly as:
```
**Log source(s):** live messages.log via `get_baw_server_logs`
```

### get_process_instance_errors

When you have a process instance ID, call **`get_process_instance_errors`**—this is the exact name of the MCP tool. Do not call any other function or invent a substitute name.

```
get_process_instance_errors(instance_ids=<numeric-id>)
```

1. Call with `instance_ids` set to the numeric instance ID (just the number—for example, `7743`, not `"BPDInstance.7743"`).
2. Use `failing_step`, `error_message`, `error_code`, and `exception_type` from the response to focus your log analysis.
3. If the response includes `failed_operations`, **do not assume the instance failed**. Check `get_process_details` first:
   - If `state` is `STATE_FINISHED` or `STATUS_COMPLETED` → the instance completed normally; there is no active runtime error record to retrieve. Report this accurately.
   - If `state` is `STATE_FAILED` or `STATUS_FAILED` → BAW may have removed the error record or was not serializable. Ask the user for log content and search for `BPDInstance.<id>` in the logs.
4. Follow up with `get_process_details` (MCP) or `GET /rest/bpm/wle/v1/process/{processId}` (REST) to see instance state, business data, and task history for the execution path leading to the failure.
5. When searching logs for a process instance, always grep explicitly for `BPDInstance.<id>`—never assume that a numeric ID appearing elsewhere in the log (for example, in a task scheduler message) refers to the process instance.

In the diagnosis report header, always record the exact call you made, for example:
```
**Log source(s):** `get_process_instance_errors(instance_ids=7743)` via workflow-runtime-mcp-server
```
If the tool call failed or was unavailable, record that too:
```
**Log source(s):** `get_process_instance_errors(instance_ids=7743)` — call failed (tool unavailable)
```
This is the audit record. It must be accurate regardless of whether the call succeeded.

---

### get_process_details

When you need instance state, business variables, and task history for a known process instance ID, call `get_process_details`:

```
get_process_details(
    process_instance_id=<numeric-id>,
    task_limit=100,
    task_offset=0,
    system_id=<optional>
)
```

1. Use `meta_data.state` (for example, `STATE_FAILED` and `STATE_FINISHED`) to determine whether the instance is still in an error state before reporting.
2. Use `meta_data.status` (`executionState`) as the human-readable label for the report.
3. Strip any `business_data.variables` entries—they are process variables from the response; use them to understand the instance's data context at the time of failure.
4. Each entry in `tasks` corresponds to one task instance. `state` and `status` on the task reveal whether that step completed, failed, or is still pending.
5. If `tasks` returns exactly 100 entries, there may be more—re-call with `task_offset=100` and continue until fewer than `task_limit` entries are returned.

In the diagnosis report header, record the call:
```
**Log source(s):** `get_process_details(process_instance_id=7743, task_limit=100, task_offset=0)` via workflow-runtime-mcp-server
```

---

## REST fallback (no MCP)

When the MCP server is not connected, run all REST calls yourself by using `curl` with `execute_command`. Do not give the user curl commands to run. Use `-k` to allow self-signed certificates and `-s` for silent output. BAW servers commonly use self-signed certs on localhost; always include `-k`.

### Establish connection first

Before any REST call, follow `references/CREDENTIAL_HANDLING.md` to resolve `{BAW_BASE_URL}`, `{BAW_USERNAME}`, and `{BAW_PASSWORD}` from `.env`. Once credentials are resolved, execute the login call:

```bash
curl -k -s -X POST "{BAW_BASE_URL}/ops/system/login" \
  -H "Authorization: Basic <base64(BAW_USERNAME:BAW_PASSWORD)>" \
  -H "Content-Type: application/json" \
  -d '{"refresh_groups": false}'
```

On HTTP 201, parse `csrf_token` from the JSON response body and capture the `Set-Cookie` headers for `LtpaToken2` and `JSESSIONID`. Store these as `{BAW_CSRF}` and `{BAW_SESSION}` for all subsequent `/ops/…` calls.

To capture cookies with curl, add `-c /tmp/baw_cookies.txt` to the login call and `-b /tmp/baw_cookies.txt` to subsequent calls.

The WLE v1 endpoint (`/rest/bpm/wle/v1/process/errors`) uses HTTP Basic auth per request—it does not require the session or cross-site request forgery (CSRF) token.

### Retrieve process instance runtime errors

Execute directly—no session required for this endpoint:

```bash
curl -k -s -X PUT \
  "{BAW_BASE_URL}/rest/bpm/wle/v1/process/errors?instanceIds={process_id}" \
  -H "Authorization: Basic <base64(username:password)>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

You can pass multiple comma-separated IDs in `instanceIds`. Parse the JSON response and extract:
- `data.runtimeErrors[]`—array of per-instance error objects, each containing:
  - `instanceId`—the process instance ID
  - `errorMessage`—the exception message (maps to `error_message` in MCP response)
  - `errorCode`—BPM error code (maps to `error_code`)
  - `exceptionType`—Java exception class (maps to `exception_type`)
  - `failedStepName`—the activity that failed (maps to `failing_step`)
  - `timestamp`—when the error occurred
- `data.failedOperations[]`—instance IDs whose error info could not be retrieved

Apply the same interpretation logic as the MCP path: if `runtimeErrors` is empty and the ID appears in `failedOperations`, BAW may have removed the error record—ask the user for log content. If `failedStepName` is present, use it as the failing step for log correlation (grep for `BPDInstance.<id>`).

In the diagnosis report header, record the call made:
```
**Log source(s):** `PUT /rest/bpm/wle/v1/process/errors?instanceIds=7743` via REST (curl)
```

### Retrieve process instance details

After retrieving runtime errors, fetch the full instance state, business data, and tasks. No session is required—this endpoint uses HTTP Basic auth per request:

```bash
curl -k -s \
  "{BAW_BASE_URL}/rest/bpm/wle/v1/process/{process_id}?parts=all&taskLimit=100&taskOffset=0" \
  -H "Authorization: Basic <base64(username:password)>"
```

Parse the JSON response and extract from `data`:
- `piid` → process instance ID (confirm it matches what you queried)
- `executionState` → human-readable status (for example, `Active`, `Completed`)
- `state` → machine-readable state constant (for example, `STATE_FAILED`, `STATE_FINISHED`)
- `creationTime`, `lastModificationTime`, `dueDate` → timeline context
- `variables` → business data; strip all keys starting with `@` (for example, `@metadata`) before presenting
- `tasks[]` → each task entry has `tkiid`, `name`, `displayName`, `status`, `state`, `assignedTo`, `dueTime`

See `references/baw-rest.md`—`/rest/bpm/wle/v1/process/{processId}`—for the full field list and field mapping to MCP response fields.

Apply the same state interpretation as the MCP path:
- `state: STATE_FINISHED` or `executionState: Completed` → instance completed; no active error to report.
- `state: STATE_FAILED` → instance is in an error state; combine with error data from the previous step.

If `tasks` returns exactly `taskLimit` entries, re-call with `taskOffset` incremented by `taskLimit` until fewer are returned.

In the diagnosis report header, record the call:
```
**Log source(s):** `GET /rest/bpm/wle/v1/process/7743?parts=all&taskLimit=100&taskOffset=0` via REST (curl)
```

### Retrieve event manager tasks for an instance

If the user reports a task that failed silently or a repeating error (which in BAW often manifests as a stuck Event Manager task), execute:

> **Note:** `GET /ops/std/bpm/event_manager_tasks` requires an authenticated session (CSRF token + session cookie) and admin rights. It works on both Liberty and traditional BAW deployments. Only `GET /ops/std/bpm/logs` is Liberty-only.

```bash
curl -k -s \
  "{BAW_BASE_URL}/ops/std/bpm/event_manager_tasks?states=on_hold&process_id={process_id}&optional_parts=message" \
  -H "BPMCSRFToken: {BAW_CSRF}" \
  -b /tmp/baw_cookies.txt
```

The `message` optional part returns the serialized event payload, which may contain the exception text and stack trace. This is particularly useful for failures in timer events, receive tasks, and integration services.

### Retrieve live server logs

`GET /ops/std/bpm/logs` is available on Liberty-based BAW deployments only. Execute after establishing a session:

```bash
curl -k -s \
  "{BAW_BASE_URL}/ops/std/bpm/logs" \
  -H "BPMCSRFToken: {BAW_CSRF}" \
  -b /tmp/baw_cookies.txt
```

- HTTP 200 with a non-empty body → the response body is the raw `messages.log` content. Write it to a temp file (for example, `/tmp/baw_messages.log`) and read with `read_file` using a line range (last 200 lines first).
- HTTP 404 → the endpoint is not available on this BAW release. **Stop. Do not try any other endpoints.** Ask the user immediately (see prompt below).
- HTTP 403 → caller lacks admin rights. **Stop. Do not try any other endpoints.** Tell the user and ask (see prompt below).
- Empty body → the server is not Liberty. **Stop. Do not try any other endpoints.** Tell the user and ask (see prompt below).

When any of the above non-200 conditions occur, ask immediately:

> "The server log endpoint is not available on this server (`<HTTP status>`). Could you paste the relevant section of `messages.log` (or `SystemOut.log`), or give me a local file path I can read?"

Record the source accurately in the report:
```
**Log source(s):** `GET /ops/std/bpm/logs` via REST (curl) — pasted on fallback
```

---

## Diagnosis report template

Always produce a report in this structure. Keep it scannable—headings and bullets rather than long paragraphs.

```
## BAW Failure Diagnosis

**Instance / Context:** [process instance ID if known, or "N/A"]
**Log source(s):** [exact source — use the precise MCP tool call if one was made, e.g.
  `get_process_instance_errors(instance_ids=7743)` via workflow-runtime-mcp-server,
  `get_baw_server_logs` via workflow-runtime-mcp-server,
  pasted SystemOut.log excerpt,
  FFDC file defaultServer_XXXXXXXX_...txt,
  or a combination. Never invent a tool name.]
**Failure surface:** [one sentence: what failed, where, when]

### Root cause
[One to three sentences. Name the exact exception class or error code, the failing step or service,
and the immediate trigger — e.g. "A `TWException: Cannot read property 'amount' of undefined`
was thrown at line 3 of the 'ValidateExpenseData' script task because the `orderData` variable
was null when the process reached that step."]

### Evidence
- [Key log line or API field that confirms the root cause — quote it]
- [Any corroborating entry — e.g. the FFDC1015I pointer in messages.log]

### Likely contributing factors
- [Why the variable was null / why the service was unreachable / etc.]

### Remediation steps
1. [Immediate fix — the specific change to make]
2. [Verification — how to confirm the fix worked]
3. [Optional: longer-term hardening — guard clause, error boundary, retry logic]

### If this recurs
[One sentence on where to look first — e.g. "Enable `WLE.wle_bpd=all` trace before reproducing
to capture the exact variable state at the failing step."]
```

Omit sections that do not apply. Add a "What I cannot determine from this log" section when there is a critical gap in the evidence.

---

## File mode assumptions

When reading a log file via `read_file`:
- Large files (> 500 lines): read the last 200 lines first—errors are typically near the end. Expand earlier if needed.
- If the user says "the error happened around 2pm", use timestamps to jump to that window.
- State which lines you analyzed so the user knows what you saw.

If the file is inaccessible (remote server, missing path, requires credentials), say so and ask the user to paste the relevant section.

---

## What this skill does not do

- Replace Process Admin Console (PAC) or the WebPD debugger for stepping through live process execution.
- Cover all log types exhaustively—focus on what is in front of you and say so when evidence is limited.


---

---
name: compare-baw-versions
description: "Use when an IBM Business Automation Workflow (BAW) developer or administrator wants to compare two versions (snapshots or branches) of a process application. Exports both versions and produces a structured diff by artifact type: process steps, gateways, variables, business objects, services, scripts, and configuration. Flags artifacts that changed in both versions as conflict hints so the developer knows what to reconcile manually in BAW Process Designer. Use when the user says 'what changed between snapshots', 'compare two versions', 'diff my branches', 'what changed in this release', 'show me what was added or removed', or 'help me understand what to merge'. Also use proactively when a developer describes divergent branches needing reconciliation. Applies under both BAW Author and BAW Admin modes. Do not use for generating processes (generate-baw-bpmn), running tests (generate-baw-tests), or inspecting live instances (inspect-baw-processes)."
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Read
  - Edit
  - Bash
metadata:
  version: "1.0.0"
  display_name: Compare BAW Versions
  short_description: Diff two BAW process app snapshots or branches by artifact type and flag conflicting changes for manual reconciliation.
  author: IBM BAW Team
---
> **Important: REST only. This skill does not support MCP.** All BAW interactions use the BAW REST API exclusively, through HTTP calls to `{BAW_BASE_URL}`. Do not call any MCP tools at any point, whether or not an MCP server is connected. This skill has no MCP dependency. If you are about to use an MCP tool, stop and use the REST equivalent from `references/baw-rest.md` instead.

> **Important: always export the TWX file through REST, never through MCP.** Even if an Authoring MCP server is connected and its `export_solution` tool is visible in your tool list, do not call it for this skill. Always download the TWX file with `GET {BAW_BASE_URL}/ops/std/bpm/containers/{container}/versions/{version}/export?use_enhanced_filenames=true`. This rule applies even when MCP is fully connected. Do not mention the MCP export tool to the user. Do not reason that MCP is more reliable. REST is the required transport for this operation.

# Compare BAW versions

Compare two versions of an IBM Business Automation Workflow (BAW) process application and produce a structured, human-readable diff. The output tells the developer exactly what changed. It also flags the artifacts that changed in both versions, so the developer knows what needs careful manual reconciliation in BAW Process Designer.

This skill does not perform merges. BAW has no merge API, and generating or modifying a TWX file for import back into BAW is out of scope. The skill makes the diff clear enough that the developer can merge confidently.

---

## Execution mode

This skill operates in REST mode only. All calls use the BAW REST API documented in `references/baw-rest.md`.

---

## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---

## Semantic operations

| Operation | Description |
|---|---|
| `list containers` | `GET /ops/std/bpm/containers` lists all process applications. |
| `list versions` | `GET /ops/std/bpm/containers/{container}/versions` lists the snapshots for a given application. |
| `export version` | `GET /ops/std/bpm/containers/{container}/versions/{version}/export?use_enhanced_filenames=true` downloads a TWX file. |

All three operations require an active session. Establish a session first, as described in the discovery steps.

---

## Guided discovery

Work through these steps in order. Do not announce step labels to the user. Ask the questions naturally.

### Step 1: Identify the process application

If the user named the application, call `list containers` and find the match by name. If the name is ambiguous or missing, present a short numbered list from the response and ask which application the user means. Confirm the acronym before you proceed. Do not guess an acronym.

---

### Step 2: Identify the two versions to compare

Call `list versions` for the identified application. Present the available snapshots in a concise table with their names, acronyms, activation status, and creation dates.

Ask the user which two to compare:
> *"Which two versions would you like to compare? You can refer to them by name or acronym."*

Label them Version A (older or base) and Version B (newer or target). By convention, the diff shows what changed going from A to B. If the user does not specify which version is older, order them by creation date and confirm the order with the user.

If the user says "latest" or "current" for one side, use the most recently activated snapshot.

---

### Step 3: Export both versions

For each version, call `export version` with `use_enhanced_filenames=true`. The call returns a binary `.twx` file, which is a ZIP archive. Extract both archives into memory.

If a REST export fails:
- On 403, tell the user that they might not have permission to export this snapshot. Export usually requires administrator access or project read permission.
- On 404, show the snapshot acronym that you used and ask the user to confirm it.
- On other errors, show the status code and the response body, and ask how the user wants to proceed.

If the REST export fails twice, offer the manual fallback:
> *"If you already have the `.twx` files for both versions on your machine, you can share the file paths and I'll work from those directly."*

Accept local file paths, read them as ZIP archives, and process them the same way as the REST exports.

---

### Step 4: Analyze and produce the diff

Read `references/DIFF_OUTPUT.md` for the exact output structure before you start the analysis.

Parse the extracted XML files from both TWX archives. Organize your analysis by artifact type, and work through each category in turn. Classify every change for each artifact as one of the following:

- **Added**: exists in B but not in A.
- **Removed**: exists in A but not in B.
- **Modified**: exists in both but differs.
- **Conflict hint**: the same artifact name has different content in A and B. This set needs the most attention during manual reconciliation.

> **Note on conflict hints:** This skill runs a two-way diff. Without a common ancestor, it cannot detect true three-way merge conflicts. A conflict hint means that the artifact differs between the two versions, and the developer must choose which version to keep or blend them manually. Present conflict hints prominently and separately from straightforward one-sided changes.

Compare these artifact types in this order:

1. **Processes (BPMN)**: steps and activities (added, removed, reordered), gateways (condition changes), sequence flows, lanes and swimlanes, subprocess boundaries.
2. **Services and service flows**: integration services, decision services, and human services that were added, removed, or changed.
3. **Business objects**: parameters, field types, field additions and removals, nested business object changes.
4. **Variables**: process variables, exposed process values (EPVs), and private variables, with their type changes, additions, and removals.
5. **Scripts**: inline script bodies that changed. Show a compact diff of the script content, not raw XML.
6. **Configuration**: environment variables, team bindings, and server configurations.

Skip a category entirely, without mentioning it, if it has zero changes. Show only the categories where something changed.

---

### Step 5: Present the diff and save the report file

Follow the output format in `references/DIFF_OUTPUT.md` exactly. Lead with a summary header, then expand each artifact type that has changes. Present conflict hints in their own clearly marked section at the top, before the other diff categories.

Write the report to a file at the same time that you present it in chat. Do not wait for the user to ask. Save it as:
`<app-acronym>-<versionA-acronym>-vs-<versionB-acronym>-diff.md`

After you save the file, tell the user:
> *"I've also saved the full report to `{filename}`. Let me know if you'd like a JSON version as well."*

If the diff has zero conflict hints, close with:
> *"The changes above are informational only. I have not modified either version. To apply changes, open the target version in BAW Process Designer and make the edits manually."*

If the diff has conflict hints, close with:
> *"I found N conflict(s): artifacts that differ on both sides and need a decision when you reconcile. Want me to walk you through each one so you can decide which version to keep?"*

If the user agrees, proceed to Step 6.

---

### Step 6: Resolve conflicts interactively (only if the user agrees)

Work through each conflict hint one at a time. For each conflict, present a clean side-by-side comparison and ask for a decision:

```
─────────────────────────────────────────
Conflict 1 of N: [Artifact Type] — [Artifact Name]
─────────────────────────────────────────
  Version A ([snapshot name]):
    [human-readable description of Version A's value/state]

  Version B ([snapshot name]):
    [human-readable description of Version B's value/state]

→ What would you like to do?
  (A) Keep Version A's version
  (B) Keep Version B's version
  (C) Blend them — describe what you want
  (S) Skip for now
```

Wait for the user's response before you move to the next conflict. Do not batch the conflicts.

Handle each response as follows:
- **(A) or (B):** Record the decision and move to the next conflict.
- **(C) Blend:** Ask the user to describe what the blended result should look like. Capture the description as the decision, and move on.
- **(S) Skip:** Mark the conflict as "to decide later" and move on without recording a decision.

After the user addresses all conflicts, produce a reconciliation checklist. See `references/DIFF_OUTPUT.md` for the format. Present the checklist in chat and save it to a file at the same time:
`<app-acronym>-<versionA-acronym>-vs-<versionB-acronym>-reconciliation.md`

The checklist covers:
1. Every conflict decision that the user made, with what to change and the target value.
2. All one-sided additions and removals from the diff. These are safe to apply without a decision, so list them as action items.
3. Any skipped conflicts, flagged clearly as still needing a decision.

Close with:
> *"This checklist is your guide for reconciling the two versions manually in BAW Process Designer. I have not changed either version. You need to apply all edits in the designer."*

---

## Important boundaries

- **Read-only.** This skill only exports and reads. It never imports, modifies, or writes back to BAW.
- **No merge generation.** Do not generate a merged TWX file or suggest importing a generated artifact. If the user asks you to perform the merge, explain that BAW has no merge API and direct the user to reconcile manually in Process Designer by using the diff output.
- **Two-way diff only.** Without a common ancestor snapshot, true merge conflict detection is not possible. Conflict hints surface same-artifact differences. They are not guaranteed to be conflicts in the strict three-way-merge sense.
- **Authoring environments only for export.** The export API requires an authoring environment: Workflow Center for traditional BAW, or Business Automation Studio for containerized CP4BA. Workflow Server (runtime-only) environments do not support export.
- **Process applications only.** The same API can export toolkits, but comparing toolkit dependencies is out of scope for this skill.

---

## Disambiguation

**Compare versus document:** If the user wants documentation or a summary of a single version instead of a diff between two versions, direct the user to `baw-project-doc` or offer to describe one version at a time. This skill requires exactly two versions.

**Compare versus audit:** If the user wants a compliance or audit readiness analysis instead of a version diff, this is the wrong skill. Direct the user to `audit-baw-readiness` (design-time scorecard analysis against six internal control domains) or `process-compliance-violation-detection` (policy and regulatory compliance checks against external standards such as SOX, GDPR, and ISO).

**Same branch versus different branches:** The skill supports both. Snapshot-to-snapshot comparisons on the same branch are the most common case (for example, v1.0 versus v2.0). Cross-branch comparisons (for example, the `main` tip versus `feature/new-approval-flow`) work the same way. The skill exports a specific snapshot from each branch.


---

---
name: audit-baw-readiness
description: "Evaluates an IBM Business Automation Workflow (BAW) process application's design for compliance and audit readiness — scores six internal design-time control domains (traceability, separation of duties, access control, SLA/timeliness, data completeness, error handling) and produces a Markdown scorecard with specific gaps and remediation steps. Use when someone asks if a process application is ready for an audit, checks internal control design readiness, or wants to score design-time controls. This skill inspects process design — not running instances and not external regulatory policies. For evaluating process models against external policies and regulations (SOX, GDPR, ISO) with remediation paths, use process-compliance-violation-detection. Do not use for live process instances (inspect-baw-processes) or authoring processes (generate-baw-bpmn)."
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Read
  - Edit
  - Bash
metadata:
  version: "1.0.0"
  display_name: BAW Audit Readiness
  short_description: Score a BAW process application across six compliance domains and produce a gap-and-remediation scorecard.
  author: IBM BAW Team
---
> **⚠️ REST only — MCP is not supported by this skill.** All BAW interactions use the BAW REST API exclusively via HTTP calls to `{BAW_BASE_URL}`. Do not call any MCP tools at any point — regardless of whether an MCP server is connected or not. This skill has no MCP dependency. If you find yourself reaching for an MCP tool, stop and use the REST equivalent from `references/baw-rest.md` instead.

> **⚠️ Hard rule — TWX export is always REST, never MCP.** Even if an Authoring MCP server is connected and its `export_solution` tool is visible in your tool list, do not call it — ever — for this skill. The TWX must always be downloaded via `GET {BAW_BASE_URL}/ops/std/bpm/containers/{container}/versions/{version}/export?use_enhanced_filenames=true`. This applies even when MCP is fully connected. Do not mention the MCP export tool to the user. Do not reason that MCP is "more reliable" — REST is the required transport for this operation, always.

# Audit BAW readiness

Inspect an IBM BAW process application's exported TWX artifact and produce an **audit readiness scorecard** — a scored gap analysis across six compliance domains that tells the user exactly what needs to be fixed before an audit begins.

This skill is **read-only**. It exports and analyzes — it never imports, modifies, or writes back to BAW.

---

## Execution mode

This skill operates in **REST mode only**. All calls use the BAW REST API documented in `references/baw-rest.md`.

---

## Semantic operations

| Operation | What it does |
|---|---|
| `login` | `POST /ops/system/login` — obtain CSRF token and session |
| `list containers` | `GET /ops/std/bpm/containers` — list all process apps |
| `list versions` | `GET /ops/std/bpm/containers/{container}/versions` — list snapshots for an app |
| `export version` | `GET /ops/std/bpm/containers/{container}/versions/{version}/export?use_enhanced_filenames=true` — download the TWX |

All calls after login require `BPMCSRFToken` header and session cookie.

---

## Audit domains

Read `references/audit-domains.md` before scoring. It defines all six domains, the evidence to look for in the TWX artifacts, and the scoring criteria (Pass / Partial / Fail) for each.

The six domains are:

1. **Traceability** — decisions are captured with who, what, and when
2. **Separation of Duties** — request and approval steps are in separate lanes/teams
3. **Access Control** — lanes are bound to named teams, not open to all users
4. **SLA / Timeliness** — critical activities have timer events or escalation paths
5. **Data Completeness** — business objects capture the fields an auditor would need
6. **Error Handling** — risky service tasks have boundary error events or recovery paths

---

## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---

## Steps

### Step 1 — Identify the process application

If the user named the app, call `list containers` and find the match by name. If the name is ambiguous or not given, present a short numbered list and ask which app they mean. Confirm the acronym before proceeding — never guess one.

---

### Step 2 — Identify the version to assess

Call `list versions` for the identified app. Present the available snapshots in a concise table (Name, Acronym, Created, Active).

If the user doesn't specify a version, use the most recently activated snapshot and confirm:
> *"I'll assess the latest snapshot: [name] ([acronym]). Is that right?"*

---

### Step 3 — Export the TWX

Call `export version` with `use_enhanced_filenames=true`. This returns a binary `.twx` file (a ZIP archive). Extract the archive in memory.

If export fails:
- **403:** tell the user they may not have sufficient permissions — admin access is typically required for export.
- **404:** the snapshot acronym may be wrong — show what was used and ask them to confirm.
- **Other errors:** show the status code and ask how they'd like to proceed.

If export fails persistently, offer the manual fallback:
> *"If you already have the `.twx` file for this version, you can share the file path and I'll work from that directly."*

Accept local file paths and read them as ZIPs — process them identically to the REST export.

---

### Step 4 — Analyze the artifacts

Parse the extracted XML files from the TWX archive. The TWX is a ZIP of XML files representing process definitions, business objects, service flows, and configuration. Extract:

- **Process definitions** — activities, gateways, sequence flows, swim lanes, boundary events (timers, errors), and their types
- **Business objects** — field names and types from BO definition files
- **Team/lane bindings** — which lanes are assigned to which teams
- **Variables** — process variables and their types
- **Service tasks** — automated steps that call external systems

Present a brief **artifact inventory** before scoring:
```
## Artifact inventory — [Process App Name] ([snapshot])
- Processes found: [N]
- Activities: [N] total ([n] human tasks, [n] service tasks, [n] gateways, [n] events)
- Lanes: [list of lane names]
- Business objects: [list of BO names with field count]
- Timer boundary events: [list, or "none"]
- Error boundary events: [list, or "none"]
```

---

### Step 5 — Score each audit domain

Read `references/audit-domains.md` for the full scoring criteria. For each domain:

1. Apply the evidence checklist against what was found in Step 4.
2. Assign a score: **Pass**, **Partial**, or **Fail**.
3. For any Partial or Fail, record:
   - The specific artifact (activity name, lane name, BO field name)
   - The missing control or gap
   - A concrete remediation recommendation

Do not assign Pass unless all criteria for that domain are met. Do not assign Fail unless no criteria are met — use Partial when some evidence is present but gaps remain.

---

### Step 6 — Write the scorecard

**Overall rating** (compute before writing — do not include this legend in the output file):
- **Pass** — all 6 domains Pass or N/A
- **Needs Attention** — 1 or more Partial, no Fail
- **Not Ready** — 1 or more Fail

Write the scorecard to `reports/[app-acronym]-audit-readiness.md`. Create the `reports/` directory if it does not exist.

**Scorecard format:**

```markdown
# Audit Readiness Scorecard — [Process App Name]
**Snapshot:** [snapshot name] ([acronym])
**Assessed:** [date]
**Overall:** [Pass / Needs Attention / Not Ready]

## Summary

| Domain | Score | Gaps |
|---|---|---|
| Traceability | ✅ Pass | — |
| Separation of Duties | ⚠️ Partial | 1 gap |
| Access Control | ❌ Fail | 2 gaps |
| SLA / Timeliness | ✅ Pass | — |
| Data Completeness | ⚠️ Partial | 1 gap |
| Error Handling | ⚠️ Partial | 2 gaps |

## Gap Details

### [Domain Name]
**Score:** ⚠️ Partial

| # | Artifact | Missing Control | Recommendation |
|---|---|---|---|
| 1 | Task "Submit Request" | Approval step missing from a separate lane | Add a "Manager Approval" task in a dedicated approver lane |

### [Next Domain]
...

## Passing Domains

List each domain that scored Pass with one sentence confirming what evidence was found.
```

After writing the file, tell the user:
> *"Scorecard written to `reports/[filename].md`. [N] gap(s) found across [domains with gaps]. Would you like me to walk through the remediation recommendations?"*

---

## Boundaries

In scope: exporting a snapshot, parsing its XML artifacts, scoring the six audit domains, writing the Markdown scorecard.

Out of scope — do not attempt:
- **Calling any MCP tools.** This skill uses REST only.
- **Starting, modifying, or inspecting live process instances.** Use `inspect-baw-processes`.
- **Authoring new processes or fixing gaps directly.** Use `generate-baw-bpmn`.
- **Evaluating against external regulations and compliance policies (SOX, GDPR, ISO, corporate standards).** Use `process-compliance-violation-detection`.
- **Importing or writing back to BAW.** Read-only.
- **Scoring domains beyond the six defined in `references/audit-domains.md`.** Do not invent new domains.
- **Producing a certified audit artifact for external submission** — this is a gap analysis, not a compliance certificate.

---

## Output format

**Artifact inventory:** always present before scoring so the user can see what was found.

**Scorecard:** written to `reports/` as Markdown. Always confirm the file path after writing.

**Errors:** state what failed, the likely cause, and what the user can try next. Never show raw XML or a stack trace.


---

---
name: version-lifecycle-manager
description: >-
  Manages IBM Business Automation Workflow (BAW) Process App and Toolkit
  version lifecycle via the Operations REST API — handles state-changing operations
  including installing applications, managing snapshots (activating, deactivating,
  deleting), and reading, setting, or synchronizing environment variables — no Process
  Admin Console needed. Use when a BAW administrator wants to perform lifecycle
  actions or state changes. Do NOT use for read-only queries, including snapshot metadata,
  process-app listings, and container inventories — use version-inspector for those.
  Supports BAW on-premises, BAW on Cloud, and CP4BA (IBM Cloud Pak for Business
  Automation).
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Bash
metadata:
  version: "1.0.0"
  display_name: BAW Version Lifecycle Manager
  short_description: Activate, deactivate, install, and delete BAW snapshots and manage environment variables via the Operations REST API.
  author: IBM BAW Team
---
# Version Lifecycle Manager

Manage IBM BAW Process App and Toolkit version lifecycles using the BAW Operations REST API. This skill installs applications, activates, deactivates, and deletes snapshots, verifies snapshot state, and manages environment variables — all via documented public REST endpoints with no Process Admin Console required.

> **Companion skill:** To inspect a snapshot's metadata (version name, status, creation date, dependencies) before taking a lifecycle action, use the `version-inspector` skill. The two skills are designed to work together — inspect first, act here.

## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---

Before performing any operation:
1. Identify the **container** (Process App or Toolkit acronym/ID) and, where applicable, the **snapshot version** (snapshot acronym/ID).
2. State the **intended operation** clearly and explain its effect.
3. For **deactivation, deletion, or environment-variable changes on a live snapshot**, require explicit user confirmation before proceeding.

Read `references/BAW_SNAPSHOT_API.md` for the exact API details, parameter formats, authentication, deployment-specific base URLs, and response schemas.

## Deployment types

BAW is deployed in three forms — the base URL and authentication differ, but the API path structure is the same:

| Deployment | Base URL pattern | Auth |
|---|---|---|
| BAW on-premises (root mount) | `https://<host>:9443` (Ops API directly at `/ops/...`) | Basic auth or LTPA token |
| BAW on-premises (prefixed) | `https://<host>:9443/rest`, `/bas`, or `/baw` | Basic auth or LTPA token |
| BAW on Cloud | `https://<tenant>.baw.ibmcloud.com` | API key / OAuth |
| CP4BA (Cloud Pak) | `https://<cpd-host>/bas` | ZenAPI key or OAuth |

> **Version requirement:** The `/ops/std/bpm/...` Operations REST API (used for activate, deactivate, delete, verify, and env-var endpoints) requires **BAW 18.0.0.0 or later**. It is not available on IBM BPM 8.x (released before 2018). If the user appears to be on BPM 8.x, state this clearly — see the version detection note below.

The default on-premises port is **9443** (HTTPS). Port 9080 (HTTP) usually redirects to 9443. If the user provides only a hostname with no port, probe port 9443 first. If the user provides only a hostname with no context root, probe the root `""` (no prefix), `/rest`, `/bas`, then `/baw` on port 9443 in that order using `POST <base-url>/ops/system/login` or container probe — use the first that returns HTTP 200, 201, 401, or 403 (401/403 means the root exists but the credentials or auth method were rejected, which is still a valid root).

**Version detection:** After finding the context root, probe the Operations API with a lightweight call:
```bash
curl -sk -o /dev/null -w "%{http_code}" \
  "<base-url>/ops/std/bpm/containers?containerType=PA" \
  -H "Authorization: Basic <credentials>"
```
- If it returns **200, 401, or 403** → Operations API is present; proceed normally. A 403 on CP4BA typically means Basic auth is not accepted — switch to a Zen Bearer token (see CP4BA auth note below).
- If it returns **404** but `GET <base-url>/bpm/wle/v1/processApps` returns **200** → the server is likely IBM BPM 8.x. Stop and tell the user:
  > "The Operations REST API (`/ops/std/bpm/...`) is not available on this server. It requires IBM Business Automation Workflow 18.0 or later, but this server appears to be IBM BPM 8.x. Snapshot lifecycle management through this skill is not supported on BPM 8.x. Use the Process Admin Console on that server to manage snapshots manually."

Never hard-code or invent a URL. The base URL is read from `.env` via `references/CREDENTIAL_HANDLING.md`.

## Step-by-step workflow

### Step 1 — Gather context

Collect everything needed before constructing an API call. Ask for missing values in a single question rather than one at a time:

- **BAW base URL** — read from `.env` via `references/CREDENTIAL_HANDLING.md`; ask only if missing
- **Container** — Process App or Toolkit acronym (e.g. `MYAPP`) or numeric container ID
- **Snapshot version** — snapshot acronym (e.g. `0.9.3`) or snapshot ID (tip: if unsure, list snapshots first — see `references/BAW_SNAPSHOT_API.md` section 8; snapshots are nested in the `installedSnapshots` array of the container response)
- **Operation** — install / verify / activate / deactivate / delete / read-env-vars / set-env-vars / sync-env-vars / list-versions

For **install**, also collect the `.twx` source. Prefer the REST export API over a
manually supplied file — see the Installing section below for the acquisition sequence.
For **set-env-vars**, also collect the variable name–value pairs to write.
If the user only gave a Process App *name* but not an acronym, ask for the acronym or offer to retrieve it by listing containers first.

### Step 2 — Identify the target and state the operation

Before making any API call, present a clear summary:

```
Container:  MYAPP (Process App)
Snapshot:   0.9.3
Operation:  Activate
Effect:     Snapshot 0.9.3 will become the active (default) version served to new process instances.
```

For deactivation, deletion, or env-var changes on an active snapshot, include the lifecycle-management warning from the relevant section below.

### Step 3 — Confirm destructive or disruptive operations

**Read-only operations** (verify, list versions, read env vars) — present the operation summary and wait for the user to acknowledge before calling the API.

**Install / activate / set env vars** — low risk; present the operation summary and ask the user to confirm before calling the API.

**Sync env vars** ⚠️ — requires confirmation when targeting an active snapshot:
> "Syncing environment variables will push changes to the running snapshot. Any service or process that reads these variables will see the new values immediately. Ready to proceed?"

**Deactivation** ⚠️ — requires confirmation:
> "Deactivating a snapshot removes it from the active execution path. Any running process instances on this snapshot continue to completion, but no new instances will start on it. Ready to proceed?"

**Deletion** 🚨 — requires confirmation with a stronger warning:
> "Deleting a snapshot is permanent and cannot be undone. All process instance history tied exclusively to this snapshot may be affected. Confirm you have archived or migrated any relevant instance data before continuing. Ready to proceed?"

Do not make the API call until the user replies affirmatively for any of the above.

### Step 4 — Execute the API call

Construct and execute the call using the endpoint details from `references/BAW_SNAPSHOT_API.md`.

Provide the exact `curl` command (or equivalent) so the user can audit what will be sent:

```bash
# Example — activate
curl -X POST \
  "https://baw.example.com/bas/std/bpm/containers/MYAPP/versions/0.9.3/activate" \
  -H "Authorization: Basic <base64-encoded-credentials>" \
  -H "Content-Type: application/json"
```

Then execute the call if you have the `execute_command` tool available, or ask the user to run it and paste the response.

### Step 5 — Report the result

Report the outcome clearly. Structure:

```
Operation:  Activate snapshot 0.9.3 on MYAPP
Status:     ✅ Success  (HTTP 200)
Message:    <any message returned by the API>
```

If the API returns a warning or error:
- Surface the full response body — don't filter or summarize it.
- Explain what the error code means in plain language (see `references/BAW_SNAPSHOT_API.md` for common errors).
- Suggest a corrective action where applicable.

---

## Lifecycle management best practices

Include the relevant practice note whenever it applies to the requested operation.

### Installing a process app or toolkit

The install endpoint accepts a `.twx` file. Obtain it using the first path that
succeeds:

**Path 1 — REST export from the source server (preferred).**

Ask for the source server URL, container acronym, snapshot version, and credentials only
when the install operation is confirmed — do not ask earlier.

1. Obtain a CSRF token:
   ```bash
   CSRF=$(curl -sk -X POST "<SOURCE_BAW_BASE_URL>/ops/system/login" \
     -H "Authorization: Basic <base64(user:pass)>" \
     -H "Content-Type: application/json" \
     -d '{"refresh_groups": false}' \
     | python3 -c 'import sys,json; print(json.load(sys.stdin).get("csrf_token",""))')
   ```
   On CP4BA clusters, Basic auth returns 403 — exchange credentials for a Bearer token
   first via `POST https://<cpd-host>/icp4d-api/v1/authorize`, then use
   `Authorization: Bearer $TOKEN` on the login call and on the export call below.

2. Download the TWX:
   ```bash
   curl -sk \
     -H "Authorization: Basic <base64(user:pass)>" \
     -H "BPMCSRFToken: $CSRF" \
     "<SOURCE_BAW_BASE_URL>/ops/std/bpm/containers/<container>/versions/<version>/export?use_enhanced_filenames=true" \
     -o /tmp/<container>_<version>.twx
   ```
   Verify the downloaded file is non-empty. A non-200 response or empty file means the
   export failed — report the error and fall through to Path 2.

**Path 2 — locally supplied file (fallback).**

Offer this only after Path 1 has visibly failed:
> *"If you already have the `.twx` file on your machine, provide its path and I'll use
> that directly."*

If the user provides a file path, use it as-is in the install call below.

**Install call (after obtaining the file by either path):**

Pass the `.twx` file as a multipart upload to the install endpoint on the **target**
server. The target server credentials and CSRF token are obtained independently using
the same login sequence above (target may differ from source).

- If a container with the same acronym already exists on the target server, BAW may
  create a new snapshot within it rather than a brand-new container — verify with the
  list endpoint afterward.
- After install, use the verify endpoint to confirm BAW registered the snapshot and
  check its initial status before activating.

### Verifying a snapshot
- Use verify proactively after install and before activation to confirm the snapshot is present and its metadata matches expectations.
- The verify endpoint returns the snapshot's current lifecycle state; use it to detect if a snapshot was accidentally deleted or never deployed.

### Activating a snapshot
- Only one snapshot can be active at a time per container. Activating a new version implicitly deactivates the currently active one.
- Verify the snapshot was successfully tested on a non-production environment before activating on production.
- Consider scheduling activation during a maintenance window if the process app is under heavy load.

### Deactivating a snapshot
- Running instances on the deactivated snapshot continue until they complete. Check the verify endpoint (`GET /ops/std/bpm/containers/{container}/versions/{version}`) to see the active instance count before deactivating if there is any concern about in-flight work.
- Do not deactivate the only snapshot of a Process App unless you intend to disable it entirely.

### Environment variables
- Read env vars first (`GET`) before making changes so you have a baseline to roll back to if needed.
- Setting env vars (`POST`) writes the values to the snapshot's configuration but does not push them into the running environment; use the sync endpoint afterward to apply them.
- Syncing env vars (`POST .../sync`) applies the stored configuration to the live snapshot. Warn the user if the snapshot is currently active, since the change is immediate.
- Keep a record of env var values externally (e.g. in a secrets manager or version-controlled config file) — BAW does not maintain a history of previous values.

### Deleting a snapshot
- Deletion is irreversible. Export or archive the snapshot (`.twx` file via **Workflow Center** (formerly Process Center) for traditional BAW, or **Business Automation Studio** for CP4BA) before deleting if there is any chance the version will be needed again.
- You cannot delete the currently active snapshot without first deactivating it.
- BAW may reject the deletion if there are active process instances on the snapshot; resolve those first.
- On CP4BA, namespace-level RBAC may require additional authorization for deletion.

---

## Boundaries

**In scope:** Installing, verifying, activating, deactivating, and deleting BAW snapshots; reading, setting, and synchronizing environment variables; listing containers and snapshot versions — all via the Operations REST API; explaining lifecycle-management implications.

**Out of scope — redirect explicitly:**
- **Read-only snapshot queries, metadata inspection, and container listings:** use `version-inspector`.
- **Exporting or backing up a snapshot (.twx):** use the authoring environment UI — **Workflow Center** for traditional BAW or **Business Automation Studio** for CP4BA containerized. This skill only administers snapshots already present on the server or being installed via the install endpoint.
- **Migrating running instances from one snapshot to another:** that is a separate administration task using the BAW Process Admin Console migration tooling.
- **Installing or upgrading BAW itself:** not covered here.
- **Generating BPMN processes or Business Objects:** use `generate-baw-bpmn` or `generate-baw-business-objects`.

---

## Output format

For every operation, always produce (in this order):

1. **Operation summary** — container, snapshot (where applicable), intended action, and effect
2. **Warning / confirmation prompt** (sync env vars, deactivation, and deletion only)
3. **Curl command** — the exact API call that will be (or was) made, with placeholders replaced by the real values
4. **Result** — HTTP status, response body, and a plain-language interpretation
5. **Next steps** (optional) — what the administrator should verify or do after the operation

For **env var operations**, include a table of the variables read or written alongside the result.

For **install**, include a follow-up suggestion to verify the snapshot using the verify endpoint.

---

## Examples

**Example 1 — Deactivate a snapshot**

Input: "Deactivate snapshot v2.1 of our HRAPP process app on https://baw.corp.example.com/bas. My username is <your-username> and password is <your-password>."

1. Container: `HRAPP`, snapshot: `v2.1`, operation: Deactivate.
2. Present summary and ⚠️ deactivation warning; wait for confirmation.
3. On confirmation, construct and execute:
   ```bash
   curl -X POST \
     "https://baw.corp.example.com/bas/std/bpm/containers/HRAPP/versions/v2.1/deactivate" \
     -H "Authorization: Basic <base64-encoded-credentials>" \
     -H "Content-Type: application/json"
   ```
4. Report HTTP status, response body, and note that running instances on `v2.1` complete normally but no new instances start on that version.

**Example 2 — Install and verify**

Input: "Install snapshot from /tmp/SUPPLYAPP_v3.twx to https://baw.prod.example.com/bas, then verify it landed."

1. Construct the install call (POST to `/ops/std/bpm/containers/install` with the `.twx` file path).
2. After install, automatically follow up with a verify call (GET `/ops/std/bpm/containers/SUPPLYAPP/versions/v3`) and present the snapshot status.
3. Suggest running an env vars sync if the snapshot has environment configuration to carry over from the previous version.

**Example 3 — Set and sync environment variables**

Input: "Set DB_URL to jdbc:db2://prod-db:50000/BAWDB and DB_POOL_SIZE to 20 on snapshot v3.0 of SUPPLYAPP."

1. Summarize the variables to be written: two key–value pairs on SUPPLYAPP v3.0.
2. Call POST `.../env_vars` to store the values.
3. Ask: "Would you like me to sync these variables to the live snapshot now? This will apply them immediately."
4. On confirmation, call POST `.../env_vars/sync` and report the result.

## Reference files

- **`references/BAW_SNAPSHOT_API.md`** — full API reference: all endpoints, parameters, authentication, common errors, and response schemas for all deployment types. Read this before constructing any API call.


---

---
name: version-inspector
description: >-
  Inspect IBM BAW snapshots and browse deployed containers. Handles read-only
  queries, including snapshot metadata (version name, lifecycle status, creation
  date, toolkit dependencies), process-app listings, and container inventories.
  Works for both Process App and Toolkit types on BAW on-premises, BAW on Cloud,
  and CP4BA. Use whenever an administrator wants to inspect, view, or browse
  versions or snapshots. This skill is read-only and does not change state.
  For state-changing lifecycle operations (installing applications, activating,
  deactivating, deleting snapshots, or setting/syncing environment variables),
  use version-lifecycle-manager.
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Bash
metadata:
  version: "1.0.0"
  display_name: BAW Version Inspector
  short_description: Browse and inspect BAW snapshot metadata — version names, lifecycle status, creation dates, and toolkit dependencies.
  author: IBM BAW Team
---
# Version Inspector

Retrieve and present a plain-language summary of a BAW snapshot's current state — version name, lifecycle status, creation date, in-flight task count, and toolkit dependencies — using the public Operations REST API. No write operations are performed. Works for both Process App and Toolkit container types.

**Companion skill:** The `version-lifecycle-manager` skill handles lifecycle actions (activate, deactivate, delete, env var management) — mention this after presenting the summary.


## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---

## Entry point — choosing the right path

Before calling any API, determine how much the user already knows:

| User knows | Start at |
|---|---|
| Container acronym **and** snapshot version | Retrieval workflow below |
| Container acronym but **not** the snapshot version | Snapshot picker section below |
| Neither container nor snapshot | Container browser section below |

---

## Container browser — user does not know which Process App or Toolkit

Use this path when the user says things like "what process apps are deployed?", "show me all toolkits", "list everything on this server", "what's installed?", or does not provide a container name.

Call:

```bash
curl -sk -X GET \
  "{BAW_BASE_URL}/ops/std/bpm/containers?optional_parts=versions" \
  -H "Authorization: Basic <base64(BAW_USERNAME:BAW_PASSWORD)>" \
  -H "BPMCSRFToken: {BAW_CSRF}" \
  -b /tmp/baw_cookies.txt
```

The `optional_parts=versions` parameter returns containers **and** their snapshots in a single call. Use `type=PA` or `type=TK` to filter to Process Apps or Toolkits only; omit to return both.

Present results in two labelled tables — Process Apps first, then Toolkits (`"toolkit": true`):

**Process Apps**

| Name | Acronym | Created |
|------|---------|---------|
| HR Process Application | HRAPP | 2025-06-10 |

**Toolkits**

| Name | Acronym | Created |
|------|---------|---------|
| Shared Utilities | SHAREDTK | 2025-03-01 |

> `container` in the response is the acronym to use in all subsequent API calls. `container_name` is the display name.

After presenting the list, ask: "Which one would you like to inspect? I can show you its snapshots."

---

## Snapshot picker — user knows the container but not the snapshot

Use this path when the user provides a container acronym (or name) but not a snapshot version — or after selecting a container from the browser above.

```bash
curl -sk -X GET \
  "{BAW_BASE_URL}/ops/std/bpm/containers/<CONTAINER>/versions" \
  -H "Authorization: Basic <base64(BAW_USERNAME:BAW_PASSWORD)>" \
  -H "BPMCSRFToken: {BAW_CSRF}" \
  -b /tmp/baw_cookies.txt
```

Present that container's snapshots as a table:

| Acronym | Name | Status | Created |
|---------|------|--------|---------|
| v2.0 | HR Process v2.0 | Active ✅ | 2025-06-10 |
| v1.9 | HR Process v1.9 | Inactive ⏸ | 2025-03-01 |

Then ask: "Which snapshot would you like to inspect in detail?"

---

## Retrieval workflow — known container and snapshot

### Step 1 — Call the Operations API

**Method:** `GET`  
**Path:** `/ops/std/bpm/containers/{container}/versions/{version}`

```bash
curl -sk -X GET \
  "{BAW_BASE_URL}/ops/std/bpm/containers/<CONTAINER>/versions/<VERSION>" \
  -H "Authorization: Basic <base64(BAW_USERNAME:BAW_PASSWORD)>" \
  -H "BPMCSRFToken: {BAW_CSRF}" \
  -b /tmp/baw_cookies.txt
```

This is a read-only call — no confirmation is needed before making it. Show the exact `curl` command in the response.

### Step 2 — Enrich toolkit dependencies (if needed)

The Operations API `version` schema does not include a `toolkitDependencies` field. To retrieve dependency data, make a second call to the WLE endpoint.

> **WLE context root:** If `<WLE_BASE_URL>` prefix is already known or was established earlier in the session (e.g. `/rest` or `""`), reuse it directly. **Do not probe if the prefix is already known.**

**Probe only once if the WLE context root is unknown:**
If the WLE context root has not been determined yet, test prefixes (`/rest`, `""`, `/baw`, `/bas`) once and cache the working prefix for all subsequent calls in the session:

```bash
for prefix in "/rest" "" "/baw" "/bas"; do
  code=$(curl -sk -o /dev/null -w "%{http_code}" \
    "https://<host>:9443${prefix}/bpm/wle/v1/processApps?includeToolkitProcessApps=true" \
    -H "Authorization: Basic <base64(user:pass)>" \
    -H "BPMCSRFToken: <csrf_token>")
  echo "prefix='${prefix}' -> HTTP $code"
done
```

Use the first prefix that returns HTTP 200 as `<WLE_BASE_URL>` for the calls below and reuse it across all subsequent queries.

**For Process Apps:**
```bash
curl -sk -X GET \
  "<WLE_BASE_URL>/bpm/wle/v1/processApps?includeToolkitProcessApps=true" \
  -H "Authorization: Basic <base64(user:pass)>" \
  -H "BPMCSRFToken: <csrf_token>"
```
Locate the container by `shortName`, then find the target snapshot in `installedSnapshots` by `acronym`. Look for toolkit references in `toolkits`, `dependencies`, or `usedToolkitSnapshots` (field name varies by BAW version).

**For Toolkits:**
```bash
curl -sk -X GET \
  "<WLE_BASE_URL>/bpm/wle/v1/toolkit?includeSnapshots=true" \
  -H "Authorization: Basic <base64(user:pass)>" \
  -H "BPMCSRFToken: <csrf_token>"
```
Locate the toolkit by `shortName`, then find the target snapshot in `installedSnapshots`.

If no dependency data is found from either source, report "No toolkit dependencies found for this snapshot." — do not infer or guess.

### Step 3 — Present the Snapshot Summary

Use this fixed block format. Fill every field; use `—` for fields not returned by the API.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Snapshot Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Container      : <container> (Process App | Toolkit)
Snapshot name  : <version_name>
Snapshot ID    : <version>
Status         : Active ✅  |  Inactive ⏸  |  Unknown
Created        : YYYY-MM-DD HH:mm UTC
Branch         : <branch_name>  (or —)
Default        : Yes | No
State          : <status>  (e.g. New, Validated, Released)

Dependencies
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
| Toolkit name     | Acronym | Snapshot |
|------------------|---------|----------|
| System Data      | TWSYS   | 1.0      |

(or: No toolkit dependencies found for this snapshot.)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Field mapping (Operations API → summary block):**

| Block field | API field | Notes |
|---|---|---|
| Container | `container` + `toolkit` flag | `toolkit: true` → "Toolkit", `false` → "Process App" |
| Snapshot name | `version_name` | Human-readable name |
| Snapshot ID | `version` | Acronym — use in subsequent API calls |
| Status | `active: true` → Active ✅ · `active: false` → Inactive ⏸ · field absent → Unknown | |
| Created | `creation_date` | Format as `YYYY-MM-DD HH:mm UTC` |
| Branch | `branch_name` | From Operations API response |
| Default | `is_default` | `true` / `false` |
| State | `status` | Development lifecycle state |
| Dependencies | WLE enrichment (Step 2) | Not in Operations API response |

### Step 4 — Offer next steps

End with: **Next steps:** To activate, deactivate, delete this snapshot, or manage its environment variables, use the `version-lifecycle-manager` skill — or ask me to do it now.

---

## Boundaries

**In scope:** Read-only snapshot metadata queries, listing process apps and toolkits, container inventory discovery, toolkit dependency inspection.

**Out of scope — redirect explicitly:**
- **State-changing lifecycle operations:** Installing applications, activating/deactivating snapshots, deleting snapshots, and setting/syncing environment variables → use `version-lifecycle-manager`.
- **Runtime process instance operations:** Inspecting or acting on running/failed instances → use `inspect-baw-processes`.
- **Generating BPMN processes or Business Objects:** → use `generate-baw-bpmn` or `generate-baw-business-objects`.

---

## Error handling

| HTTP | Likely cause | What to say |
|---|---|---|
| `401` | Invalid credentials | "Authentication failed — check the username/password or API key and try again." |
| `403` | Insufficient permissions or wrong auth type for CP4BA | "Access denied. Follow `references/CREDENTIAL_HANDLING.md` to check credentials and permissions. On-premises: ensure the account is in the `tw_admins` group." |
| `400 CWTBG0651E` | Missing or invalid CSRF token | "CSRF token missing or expired. Re-read `references/CREDENTIAL_HANDLING.md` to obtain a fresh token and include it as the `BPMCSRFToken` header." |
| `400 CWTBG0624E` | Container acronym not found | "The container `<acronym>` was not found — acronyms are case-sensitive. Offer to list containers to confirm the exact acronym." |
| `400 CWTBG0646E` | Snapshot acronym not found | "The snapshot `<version>` was not found in container `<acronym>`. Offer to list snapshots to confirm the exact acronym." |
| `404` | Operations API not present on server | "This server may be IBM BPM 8.x, which does not support the Operations REST API. Use the Process Admin Console to inspect snapshots manually." |

Surface the full raw API response body alongside the plain-language explanation — do not filter it.

---

## Examples

**Example 1 — Process App snapshot**

Input: "Show me the details for snapshot v2.0 of HRAPP."

1. Authenticate via `references/CREDENTIAL_HANDLING.md` (read from `.env`).
2. Call `GET {BAW_BASE_URL}/ops/std/bpm/containers/HRAPP/versions/v2.0` with authorization and CSRF headers.
3. Probe the WLE context root (try `/rest`, `""`, `/baw`, `/bas`) to find the prefix that returns HTTP 200, then call `GET <WLE_BASE_URL>/bpm/wle/v1/processApps?includeToolkitProcessApps=true` to retrieve toolkit dependencies.
4. Render the Snapshot Summary block.
5. Offer lifecycle action next steps via `version-lifecycle-manager`.

**Example 2 — CP4BA cluster**

Input: "I want to inspect some versions from our CP4BA environment."

1. Authenticate via `references/CREDENTIAL_HANDLING.md` (read from `.env`).
2. Call `GET {BAW_BASE_URL}/ops/std/bpm/containers?optional_parts=versions` with authorization and CSRF headers.
3. Present containers table and ask which one to inspect.

**Example 3 — Toolkit snapshot**

Input: "Get metadata for snapshot 1.5 of the SHAREDTK toolkit."

1. Authenticate via `references/CREDENTIAL_HANDLING.md` (read from `.env`).
2. Call `GET {BAW_BASE_URL}/ops/std/bpm/containers/SHAREDTK/versions/1.5` with authorization and CSRF headers. Note `toolkit: true` in the response — show container type as "Toolkit" in the summary.
3. Call `GET /bpm/wle/v1/toolkit?includeSnapshots=true` and locate SHAREDTK to find 1.5 for dependency data.
4. Render the Snapshot Summary block.
5. Offer lifecycle action next steps.

**Example 4 — User does not know the container name at all**

Input: "I'm not sure what process apps we have deployed. Can you show me?"

1. Authenticate via `references/CREDENTIAL_HANDLING.md` (read from `.env`).
2. Call `GET {BAW_BASE_URL}/ops/std/bpm/containers?optional_parts=versions` and present all containers as labelled tables (Process Apps and Toolkits).
3. Ask: "Which one would you like to inspect? I can show you its snapshots."
4. Once the user picks a container, present its snapshots from the `branches[].versions[]` already returned, or call `GET /ops/std/bpm/containers/{container}/versions` if snapshots weren't included.
5. Once the user picks a snapshot, proceed with the Snapshot Summary workflow (Steps 1–4).

**Example 5 — User knows the container but not the snapshot**

Input: "I want to check the status of our current INVOICING process app but I'm not sure what the snapshot name is."

1. Authenticate via `references/CREDENTIAL_HANDLING.md` (read from `.env`).
2. Call `GET /ops/std/bpm/containers/INVOICING/versions` and present its snapshots as a picker table (Acronym, Name, Status, Created).
3. Ask the user to confirm which snapshot to inspect.
4. Then proceed with the Snapshot Summary workflow.

## Reference files

- **`references/BAW_SNAPSHOT_API.md`** — Authentication patterns, CSRF token acquisition, base URLs by deployment type, full response schemas for the Ops API and WLE endpoints, and common error codes.


---

---
name: baw-container-inventory
description: "Inspect the container inventory of a live IBM BAW server — list all Process Apps and Toolkits, count them, search by name or acronym, view metadata, and confirm whether a specific container exists. Use when a BAW administrator wants to know what is installed on a server without opening Workflow Center or BAStudio. Trigger phrases: 'list process apps', 'list toolkits', 'what containers are installed', 'how many process apps', 'does this app exist', 'find container', 'show me the toolkits', 'container inventory', 'what is installed in BAW', 'show container details', 'look up acronym'. This skill is read-only — it queries and reports. For snapshot lifecycle operations use version-lifecycle-manager; for running process instances use inspect-baw-processes."
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Read
  - Bash
metadata:
  version: "1.0.0"
  display_name: BAW Container Inventory
  short_description: List and search all Process Apps and Toolkits installed on a live IBM BAW server without opening Workflow Center.
  author: IBM BAW Team
---

> **⚠️ REST only — MCP is not supported by this skill.** All BAW interactions use the BAW REST API exclusively via HTTP calls to `{BAW_BASE_URL}`. Do not call any MCP tools at any point — regardless of whether an MCP server is connected or not. Never tell the user to connect an MCP server; this skill does not use one. If you find yourself reaching for an MCP tool, stop and use the REST equivalent from `references/baw-rest.md` instead.

# BAW Container Inventory

Inspect the container inventory of a live IBM BAW server — list all Process Apps and Toolkits, count them, search by name or acronym, view metadata, and confirm whether a specific container exists. No Workflow Center (traditional BAW) or Business Automation Studio (CP4BA containerized) access required.

This skill is **read-only**. It queries and reports — it never creates, archives, or deletes containers.

---

## Execution mode

This skill operates in **REST mode only**. All calls use the BAW REST API documented in `references/baw-rest.md`.


## Semantic operations

| Operation | What it does |
|---|---|
| `login` | `POST /ops/system/login` — obtain CSRF token and session |
| `list containers` | `GET /ops/std/bpm/containers` — list all containers; supports `type`, `ids`, `offset`, `size`, `optional_parts` |
| `get container` | `GET /ops/std/bpm/containers/{container}` — fetch one container by acronym; supports `optional_parts=branches` or `optional_parts=versions` |

All calls after login require `BPMCSRFToken` header and session cookie.

---


## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---
## Steps

---

### Step 1 — Understand what the user wants

Before fetching anything, identify which of these the user is asking for:

| Request type | API call | Output |
|---|---|---|
| List all containers | `GET /ops/std/bpm/containers` (no filter) | Two tables (Process Apps, Toolkits) + count summary derived from the response |
| List only Process Apps | `GET /ops/std/bpm/containers?type=PA` | One table of Process Apps + count derived from the response |
| List only Toolkits | `GET /ops/std/bpm/containers?type=TK` | One table of Toolkits + count derived from the response |
| Total count (PA + TK + combined) | `GET /ops/std/bpm/containers` (no filter) | Count summary line only — no table. Partition response: `toolkit: false` → PA count, `toolkit: true` → TK count |
| Count only Process Apps | `GET /ops/std/bpm/containers?type=PA` | Count line only — no table: "X Process Apps" |
| Count only Toolkits | `GET /ops/std/bpm/containers?type=TK` | Count line only — no table: "X Toolkits" |
| Search by name or acronym | `GET /ops/std/bpm/containers` (no filter) | Match client-side after a single fetch |
| Existence check | `GET /ops/std/bpm/containers` (no filter) | Search result in memory — no second call needed |
| Basic metadata for a specific container | `GET /ops/std/bpm/containers` (no filter) | Standard response includes all metadata fields |
| Branches or versions for a specific container | `GET /ops/std/bpm/containers/{ACRONYM}?optional_parts=branches` | Single targeted call only when explicitly requested |

**Do not repeat an identical REST call** within the same request. If you have already fetched all containers and the cached response already covers the user's request, re-use it — do not call the same endpoint with the same parameters again.

---

### Step 2 — Fetch containers

Each curl snippet below is shown in both macOS/Linux and Windows form. Use the form that matches the platform. On Windows always use `cmd /c "curl ..."` — never call `curl` directly from PowerShell.

#### Fetching all pages — mandatory for list and count requests

The API returns a maximum of 100 items per call. Always paginate to retrieve the full inventory before processing. Use `size=100` to minimise round-trips. Repeat with increasing `offset` until the `next` field in the response is `null` — that is the last page. Save each page's response to a file, then merge all pages into a single JSON array for the script to process.

**Page fetch — macOS / Linux** (increment `offset` by 100 each time):
```bash
curl -sk -X GET "{BAW_BASE_URL}/ops/std/bpm/containers?size=100&offset=0" \
  -u "{BAW_USER}:{BAW_PASS}" \
  -H "BPMCSRFToken: {BAW_CSRF}" \
  -o /tmp/baw_page0.json
```

**Page fetch — Windows** (increment `offset` by 100 each time):
```powershell
cmd /c "curl -sk -X GET ""{BAW_BASE_URL}/ops/std/bpm/containers?size=100&offset=0"" -u {BAW_USER}:{BAW_PASS} -H ""BPMCSRFToken: {BAW_CSRF}"" -o %TEMP%\baw_page0.json"
```

If `next` is not `null` in the response, fetch the next page (`offset=100`, `offset=200`, …) saving each to its own file. When `next` is `null`, all pages are collected.

**With type filter** (`PA` or `TK`) — same pagination rule, add `?type=PA` or `?type=TK`:
```bash
curl -sk -X GET "{BAW_BASE_URL}/ops/std/bpm/containers?type=PA&size=100&offset=0" \
  -u "{BAW_USER}:{BAW_PASS}" \
  -H "BPMCSRFToken: {BAW_CSRF}" \
  -o /tmp/baw_page0.json
```

**Single container by acronym** (no pagination):
```bash
# macOS / Linux
curl -sk -X GET "{BAW_BASE_URL}/ops/std/bpm/containers/{ACRONYM}" \
  -u "{BAW_USER}:{BAW_PASS}" \
  -H "BPMCSRFToken: {BAW_CSRF}"
```
```powershell
# Windows
cmd /c "curl -sk -X GET ""{BAW_BASE_URL}/ops/std/bpm/containers/{ACRONYM}"" -u {BAW_USER}:{BAW_PASS} -H ""BPMCSRFToken: {BAW_CSRF}"""
```

Append `?optional_parts=branches` or `?optional_parts=versions` when branches or versions are needed. Replace `{ACRONYM}` with the actual container acronym. Read the single-container call response directly — no script needed.

---

### Step 2b — Process the response with the script

**Do not count or format containers by reading the raw JSON yourself.** Use `scripts/process_containers.py` for all list, count, and search operations. The script counts deterministically and formats the Markdown tables — this avoids the miscounting that occurs when an LLM reads large JSON arrays.

The script accepts one or more page files as arguments and merges them automatically. There is no separate merge step.

**Run the script** with the appropriate `--mode`, passing all page files as arguments:

| User request | Mode flag |
|---|---|
| List all containers (PA + TK) | `--mode list` |
| List only Process Apps | `--mode pa` |
| List only Toolkits | `--mode tk` |
| Count (any combination) | `--mode count` |

**Single page (most common case):**
```bash
# macOS / Linux
python3 scripts/process_containers.py /tmp/baw_page0.json --mode list
```
```powershell
# Windows
python3 scripts/process_containers.py $env:TEMP\baw_page0.json --mode list
```

**Multiple pages** — pass all files as space-separated arguments:
```bash
# macOS / Linux
python3 scripts/process_containers.py /tmp/baw_page0.json /tmp/baw_page1.json --mode list
```
```powershell
# Windows
python3 scripts/process_containers.py $env:TEMP\baw_page0.json $env:TEMP\baw_page1.json --mode list
```

Use `$env:TEMP` on Windows (PowerShell environment variable syntax) — never use `%TEMP%` inside a Python command or script argument, as Python does not expand Windows environment variables.

Present the script's stdout directly to the user. Do not recount, reformat, or summarise the numbers yourself — the script output is the authoritative result.

**For name/acronym searches and existence checks:** run with `--mode list`, then filter the script output rows by the search term. Do not re-read the raw JSON file to search.

**For basic metadata** on a specific container: call `GET /containers/{ACRONYM}` directly (no script needed) and present the returned fields as a labelled block.

#### Name-based searches

After running the script in `--mode list`, search the output rows for the user's term. Match case-insensitively against both the Name and Acronym columns. If no match is found, say so clearly.

---

### Step 3 — Present results

Present the output from `scripts/process_containers.py` directly. The script produces correctly formatted Markdown tables and accurate counts — do not reformat or recount anything.

#### Output format by mode

**`--mode list`** — two headed tables followed by a count summary:
```
### Process Apps

| Name                     | Acronym |
|--------------------------|---------|
| Hiring Sample            | HSS     |
| Old Invoicing (Archived) | OIA     |

### Toolkits

| Name        | Acronym |
|-------------|---------|
| System Data | SYSDATA |

2 Process Apps, 1 Toolkit (3 total)
```

**`--mode count`** — count summary line only, no tables:
```
46 Process Apps, 13 Toolkits (59 total)
```

**`--mode pa`** or **`--mode tk`** — single table for that type plus its count.

#### Large result sets

If the total exceeds 25 containers, ask the user whether they want the full output in chat or written to a `.md` file. Default to file. Write the script output to the file as-is.

#### Existence check
Answer directly first, then provide details:
> *"✅ **Hiring Sample** (HSS) exists — it is a **Process App**."*
> *"❌ No container named 'Legacy Payroll' was found."*

If not found, list available containers so the user can check for a similar name.

#### Basic metadata view (single container)
Use a labelled block — not a one-line table row:
```
**Name:** Hiring Sample
**Acronym:** HSS
**Type:** Process App
**ID:** 2066.a26f1…
**Description:** Sample process for hiring workflow
**Created:** 2024-03-15
**Created by:** admin
```

#### Branches or versions
When `optional_parts=branches` or `optional_parts=versions` is requested for a specific container, present the data as a nested table under the container header. Keep it readable — do not dump the raw JSON.

---

### Step 4 — Errors

| Situation | What to say |
|---|---|
| 401 on any call after login | "Your session may have expired. Let me re-authenticate." Re-read `references/CREDENTIAL_HANDLING.md` and re-establish the session. |
| 403 | "You may not have sufficient permissions for this operation. Check that your account has the BAW admin role." |
| 404 on `get container` | "Container '[acronym]' was not found on the server. Confirm the acronym is correct." |
| Network or timeout error | State what failed and ask: "Would you like me to retry, or would you prefer to check the server URL?" |

Never display raw JSON, stack traces, or internal error objects to the user.

---

## Boundaries

**In scope:**
- Listing all containers (Process Apps and Toolkits)
- Total counts and PA vs TK breakdown counts
- Searching and filtering by name or acronym
- Displaying basic metadata per container
- Confirming whether a specific container exists
- Showing branches or versions for a specific container when requested

**Out of scope — redirect to the appropriate skill:**
- Installing, activating, deactivating, or deleting snapshots → `version-lifecycle-manager`
- Inspecting or acting on running process instances → `inspect-baw-processes`
- Auditing a process app for compliance → `audit-baw-readiness`
- Generating BPMN processes or Business Objects → `generate-baw-bpmn`, `generate-baw-business-objects`
- Calling any MCP tools — this skill is REST only

---

## Output format summary

| Output type | Format |
|---|---|
| Container list | Markdown table: Name, Acronym, Type |
| Count | Inline summary: "X Process Apps, Y Toolkits (Z total)" |
| Existence check | Direct yes/no answer with name, acronym, and type |
| Basic metadata | Labelled key/value block |
| Branches / versions | Nested table under container header |
| Large result sets | File output (`.md`) when > 25 items or when the output contains extensive metadata |

Never output raw JSON. Never expose internal field names like `toolkit`, `container_name`, or `creator_user_id` as labels — always use human-readable equivalents.


---

---
name: baw-deployment-manager
description: Deploy a process app or toolkit snapshot into a target IBM BAW (Business Automation Workflow) environment, list available process apps and toolkits with their snapshots, check available workflow servers, and monitor installation progress. Uses the BAW REST API to list containers, check workflow servers, POST a snapshot installation request, and poll the returned queue URL until the operation completes or fails. Use whenever the user wants to deploy, install, or promote a snapshot to a workflow environment, list process apps or toolkits, browse available snapshots, check deployment status, monitor an in-progress install, check available servers, list workflow servers, or when phrases like "deploy snapshot", "install process app", "promote to environment", "list process apps", "what toolkits are available", "show me the snapshots", "deployment status", "monitor deployment", "check install progress", "available servers", "workflow servers", or "what servers can I deploy to" appear. Also triggers on CP4BA.
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Bash
metadata:
  version: "1.0.0"
  display_name: BAW Deployment Manager
  short_description: Deploy a BAW process app or toolkit snapshot to a target environment and monitor the installation to completion.
  author: IBM BAW Team
---
<!-- baw-rest-ops: baw-ops.26.0.0.json: /std/bpm/containers, /std/bpm/containers/{container}/versions/{version}/install, /std/bpm/servers/workflow, /system/queue/{id} -->

> **⚠️ REST only — this skill does not support MCP.** All BAW interactions use the BAW REST API exclusively via HTTP calls using `curl`. Do not call any MCP tools at any point — regardless of whether an MCP server is connected or not. Never tell the user to connect an MCP server; this skill does not use one. If you find yourself reaching for an MCP tool, stop and use the REST calls described in this skill instead.

# BAW Deployer

Deploy a BAW process app or toolkit snapshot to a target environment and monitor the installation to completion. Optionally list available containers and snapshots first if the user does not know what is on the server.

## Execution mode

This skill operates in **REST mode only**. All calls use the BAW REST API directly via `curl`.

Before making any REST call, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically. Do this once per session only.

This skill makes up to three REST calls:

1. **List** *(optional)* — `GET /std/bpm/containers` — discover available process apps, toolkits, and their snapshots
2. **Install** — `POST /std/bpm/containers/{container}/versions/{version}/install`
3. **Poll** — `GET <queueUrl>` (URL returned directly by the install response)


## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---
## Step 0 — List available containers (optional)

Perform this step when the user does not know the container acronym or available snapshot versions, or when they ask to "list process apps", "show toolkits", "what can I deploy", or similar.

This step uses the `baw_url`, `username`, and `password` inputs. If the target environment is a CP4BA Zen deployment (e.g. `https://<cpd-host>/<instance_name>`), obtain a Zen Bearer token and CSRF token first (see Authentication section below).

### Authentication & Endpoints

Depending on the environment topology:

1. **Traditional BAW / Workflow Center (tWAS):**

   **Step A — Obtain CSRF token** via `POST /bpm/system/login`. This is **required** — every `/ops/` and `/bpm/` API call needs a `BPMCSRFToken` header or it returns `403 CWTBG0651E`:
   ```bash
   CSRF=$(curl -s -k -u "<username>:<password>" -X POST \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d '{"refresh-groups": false, "requested-lifetime": 7200}' \
     "<baw_url>/bpm/system/login" \
     | python3 -c 'import sys, json; print(json.load(sys.stdin).get("csrf_token",""))')
   ```

   **Step B — Call the Operations REST API** with Basic Auth and CSRF token:
   ```bash
   curl -s -k \
     -u "<username>:<password>" \
     -H "Accept: application/json" \
     -H "BPMCSRFToken: $CSRF" \
     "<baw_url>/ops/std/bpm/containers" \
     -w "\n__HTTP_STATUS__:%{http_code}"
   ```

   > **Note:** The CSRF token expires after 7200 seconds (2 hours). If calls start returning `403 CWTBG0651E` mid-session, re-run Step A to refresh it.

2. **Cloud Pak for Business Automation (CP4BA / Zen Runtime):**

   **Step A — Obtain Zen Bearer token** via `/v1/preauth/validateAuth` (use the CPD host, not the instance prefix):
   ```bash
   TOKEN=$(curl -s -k -u "<username>:<password>" \
     "https://<cpd_host>/v1/preauth/validateAuth" \
     | python3 -c 'import sys, json; print(json.load(sys.stdin).get("accessToken",""))')
   ```

   **Step B — Obtain CSRF token** via `POST /ops/system/login`. This is **required** — every `/ops/` call needs a `BPMCSRFToken` header or it returns `403 CWTBG0651E`. The `baw_url` here includes the instance prefix (e.g. `https://<cpd-host>/bas` or `https://<cpd-host>/baw-bawins1`):
   ```bash
   CSRF=$(curl -s -k -X POST \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{}' \
     "<baw_url>/ops/system/login" \
     | python3 -c 'import sys, json; print(json.load(sys.stdin).get("csrf_token",""))')
   ```

   **Step C — Call the Operations REST API** with both headers:
   ```bash
   curl -s -k \
     -H "Accept: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -H "BPMCSRFToken: $CSRF" \
     "<baw_url>/ops/std/bpm/containers" \
     -w "\n__HTTP_STATUS__:%{http_code}"
   ```

   > **Note:** The CSRF token expires after 7200 seconds (2 hours). If calls start returning `403 CWTBG0651E` mid-session, re-run Step B to refresh it.

### Response Shapes

**Schema A — Traditional BAW (`processAppsList`):**
```json
{
  "status": "200",
  "data": {
    "processAppsList": [
      {
        "ID": "2066.b1e4...",
        "shortName": "BILLPAY",
        "name": "Bill Payment",
        "defaultVersion": "Main",
        "installedSnapshots": [
          {
            "name": "2024.04.1",
            "ID": "...",
            "acronym": "2024041",
            "active": true
          }
        ]
      }
    ]
  }
}
```

**Schema B — CP4BA / Operations API (`containers` array):**
```json
{
  "containers": [
    {
      "container": "BILLPAY",
      "container_name": "Bill Payment",
      "toolkit": false,
      "id": "2066.b1e4...",
      "branches": []
    },
    {
      "container": "SYSBPMUI",
      "container_name": "UI Toolkit",
      "toolkit": true,
      "id": "2066.c2f5..."
    }
  ]
}
```

> **Note:** For Schema B, query snapshots and versions per container using `GET <baw_url>/ops/std/bpm/containers/{container}/versions`.

### Key fields mapping

| Field (Schema A / Schema B) | Meaning | Used as |
|---|---|---|
| `shortName` / `container` | Container acronym | `container` input for deployment |
| `name` / `container_name` | Human-readable name | display only |
| `toolkit` (`true`/`false`) | Toolkit flag | group into Process Apps vs Toolkits |
| `installedSnapshots[].name` / `version_name` | Snapshot name | `version` input for deployment |
| `installedSnapshots[].acronym` / `version` | Snapshot acronym | alternative `version` input |

### Presenting the results

Format the output as a table or grouped list, separating toolkits and process apps:

```
Process Applications
  BILLPAY  "Bill Payment"
    Snapshots: 2024.04.1 (active), 2024.03.0

Toolkits
  SYSBPMUI "UI Toolkit"
    Snapshots: 8.6.0.0 (active)
```

After displaying the list, prompt:

> "Which container and snapshot would you like to deploy, and to which target environment?"

### Error handling

| HTTP | Likely cause |
|---|---|
| 401 / 403 | Credentials rejected or insufficient permissions |
| 404 | The `/std/bpm/containers` endpoint is unavailable on this server version |
| 500 | Internal server error — share the body for diagnosis |

---

## Step 0b — List registered Workflow Servers (optional, use before deployment)

Use this step to discover the valid `server` value required by the install endpoint on some BAW/CP4BA configurations. If the install in Step 1 returns error `CWTBG0011E` ("mandatory parameter 'server' is not specified") or `CWTBG0701E` ("Server X does not exist"), run this first. Also run proactively when the user asks "what servers are available", "which servers can I deploy to", or similar.

Try both endpoints in order — use whichever returns a successful response.

### Option A — Traditional BAW (`/ops/std/bpm/servers/workflow`)

Obtain a CSRF token first (see Step 0 authentication), then:

```bash
curl -s -k \
  -u "<username>:<password>" \
  -H "Accept: application/json" \
  -H "BPMCSRFToken: $CSRF" \
  "<baw_url>/ops/std/bpm/servers/workflow" \
  -w "\n__HTTP_STATUS__:%{http_code}"
```

**Response shape:**

```json
{
  "servers": [
    {
      "server_name": "dev",
      "server_type": "TEST",
      "online": true,
      "environment_name": "Development"
    }
  ]
}
```

Key fields:

| Field | Meaning |
|---|---|
| `server_name` | The value to pass as the `server` query parameter in the install call |
| `server_type` | `TEST` or `PRODUCTION` = deployable target; `PLAYBACK` = authoring server, skip |
| `online` | If `false`, the server is unreachable — do not deploy to it |
| `environment_name` | Human-readable environment label |

### Option B — CP4BA Operations API (`/ops/std/bpm/servers/workflow`)

Use this endpoint when the BAW environment is a CP4BA Zen deployment (e.g. `https://<cpd-host>/bas/`). Obtain a Zen Bearer token and CSRF token first (see Step 0 authentication), then:

```bash
curl -s -k \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "BPMCSRFToken: $CSRF" \
  "<baw_url>/ops/std/bpm/servers/workflow" \
  -w "\n__HTTP_STATUS__:%{http_code}"
```

For example, with the environment `https://cpd-demo-project.apps.hs.cp.fyre.ibm.com/bas`:

```bash
TOKEN=$(curl -s -k -u "<username>:<password>" \
  "https://cpd-demo-project.apps.hs.cp.fyre.ibm.com/v1/preauth/validateAuth" \
  | python3 -c 'import sys, json; print(json.load(sys.stdin).get("accessToken",""))')

CSRF=$(curl -s -k -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{}' \
  "https://cpd-demo-project.apps.hs.cp.fyre.ibm.com/bas/ops/system/login" \
  | python3 -c 'import sys, json; print(json.load(sys.stdin).get("csrf_token",""))')

curl -s -k \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "BPMCSRFToken: $CSRF" \
  "https://cpd-demo-project.apps.hs.cp.fyre.ibm.com/bas/ops/std/bpm/servers/workflow" \
  -w "\n__HTTP_STATUS__:%{http_code}"
```

**Response shape:**

```json
{
  "servers": [
    {
      "server_name": "dev",
      "server_type": "TEST",
      "online": true,
      "environment_name": "Development"
    },
    {
      "server_name": "production",
      "server_type": "PRODUCTION",
      "online": true,
      "environment_name": "Production"
    }
  ]
}
```

Key fields:

| Field | Meaning |
|---|---|
| `server_name` | The value to pass as the `server` query parameter in the install call |
| `server_type` | `TEST` or `PRODUCTION` = deployable target; `PLAYBACK` = authoring server, skip |
| `online` | If `false`, the server is unreachable — do not deploy to it |
| `environment_name` | Human-readable environment label |

### Presenting results

Present all non-`PLAYBACK` / non-offline servers in a table:

```
Available Workflow Servers
  Name         Type        Status   Environment
  dev          TEST        Online   Development
  production   PRODUCTION  Online   Production
```

If more than one deployable server exists, ask the user which one to deploy to before proceeding.

### If both endpoints return 404

Neither server-listing path is available. Instruct the user to find the server name in one of these places:

- **Process Admin Console** → Server Admin → Overview → "Server Name" field
- **Traditional BAW (Workflow Center):** Workflow Center → Process Center → Admin → Workflow Servers
- **CP4BA containerized (Business Automation Studio):** Business Automation Studio → Workflow Servers (or check with your CP4BA administrator)

---

## Required inputs for deployment

Collect these before making any REST call. Use `ask_followup_question` for anything missing:

| Input | Notes |
|---|---|
| `baw_url` | Base URL of the target BAW/CP4BA server, e.g. `https://baw.example.com/bas` or `https://cpd.example.com/bas/`. Strip any trailing slash before building URLs. |
| `username` | BAW user with deployment authority |
| `password` | Password — treat as sensitive; do not log or echo it |
| `container` | Process app or toolkit acronym (container ID), e.g. `BILLPAY` |
| `version` | Snapshot ID or acronym, e.g. `2024.04.1` or `TIP` |
| `target_env` | Deployment environment name, e.g. `Production`, `QA` (passed as the `install_to` query parameter) |
| `server` | *(CP4BA/some configs)* Registered Workflow Server name, e.g. `dev`. Required when the server returns `CWTBG0011E`. Discover using Step 0b. |

Optional inputs:

| Input | Default | Notes |
|---|---|---|
| `force_offline` | `false` | Set `true` to force offline install |
| `poll_interval_sec` | `10` | Seconds between status checks |
| `timeout_sec` | `600` | Stop polling after this many seconds and report timeout |

## Step 1 — Trigger the installation

Build the URL:

- **Traditional BAW / Workflow Center:**
  ```
  POST {baw_url}/std/bpm/containers/{container}/versions/{version}/install
       ?install_to={target_env}
       [&server={server}]       (required on some BAW configurations)
       [&offline=true]          (only when force_offline is true)
  ```
- **CP4BA Zen Runtime (Operations REST):**
  ```
  POST {baw_url}/ops/std/bpm/containers/{container}/versions/{version}/install
       ?install_to={target_env}
       [&offline=true]          (only when force_offline is true)
  ```

Set headers:
- `Content-Type: application/json`
- `Accept: application/json`
- Authorization + CSRF (**required on both tWAS and CP4BA**):
  - Traditional BAW: `-u "<username>:<password>"` + `-H "BPMCSRFToken: $CSRF"` (token from `POST /bpm/system/login`)
  - CP4BA Zen: `-H "Authorization: Bearer $TOKEN"` + `-H "BPMCSRFToken: $CSRF"` (token from `POST /ops/system/login`)

Use `execute_command` with `curl` to make the request. Acquire the CSRF token first (see Step 0 authentication). Example:

```bash
# Traditional BAW (CSRF must already be set — see Step 0 authentication)
curl -s -k -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -u "<username>:<password>" \
  -H "BPMCSRFToken: $CSRF" \
  "<baw_url>/ops/std/bpm/containers/<container>/versions/<version>/install?install_to=<target_env>&server=<server>" \
  -w "\n__HTTP_STATUS__:%{http_code}"

# CP4BA Zen (TOKEN and CSRF must already be set — see Step 0 authentication)
curl -s -k -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "BPMCSRFToken: $CSRF" \
  "<baw_url>/ops/std/bpm/containers/<container>/versions/<version>/install?install_to=<target_env>&server=<server>" \
  -w "\n__HTTP_STATUS__:%{http_code}"
```

Parse the response body (JSON) and the HTTP status code.

### Install response handling

**Success (HTTP 200 or 202):**

The response body contains a `status_url` (or equivalent) field — a fully qualified URL pointing to the queue item for this job, for example:

```json
{
  "status": "Queued",
  "status_url": "https://baw.example.com/bas/system/queue/a1b2c3d4-..."
}
```

Extract the `status_url` value verbatim. Do **not** reconstruct the URL from only the queue ID — use the `status_url` exactly as the response returns it. Proceed to Step 2.

**Synchronous success (HTTP 200 with `status: "Completed"`):**

Report success immediately and skip polling.

**Error (4xx / 5xx):**

Report the HTTP status code and the full response body to the user. Common causes:

| Code | Likely cause |
|---|---|
| 400 | Invalid container ID, version, or environment name |
| 401 / 403 | Credentials rejected or insufficient permissions |
| 404 | Container or version not found on this server |
| 409 | Snapshot already installed in this environment |
| 500 | Internal server error — share the body for diagnosis |

Do not proceed to polling if the install POST fails.

## Step 2 — Poll for completion

Poll the `status_url` that Step 1 returns at the configured interval until the status reaches a terminal state or the timeout elapses.

For **CP4BA** the queue endpoint also requires `BPMCSRFToken`. The `status_url` is a fully-qualified URL to `<baw_url>/ops/system/queue/<id>` — use the same `$TOKEN` and `$CSRF` variables:

```bash
# Traditional BAW
curl -s -k \
  -H "Accept: application/json" \
  -u "<username>:<password>" \
  "<status_url>" \
  -w "\n__HTTP_STATUS__:%{http_code}"

# CP4BA Zen
curl -s -k \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "BPMCSRFToken: $CSRF" \
  "<status_url>" \
  -w "\n__HTTP_STATUS__:%{http_code}"
```

The queue item response looks like:

```json
{
  "queueId": "a1b2c3d4-...",
  "status": "Running",
  "percentComplete": 45,
  "message": "Installing artifacts..."
}
```

### Terminal states

| `status` value | Meaning | Action |
|---|---|---|
| `Completed` | Install succeeded | Report success with any message from the response |
| `Failed` | Install failed | Report failure with `message` and `errorDetails` from the response |
| `Cancelled` | Install cancelled | Report cancellation |

Continue polling for all other status values (`Queued`, `Running`, etc.).

### Progress reporting

After each poll, report to the user:

```
[HH:MM:SS] Status: Running — 45% complete — "Installing artifacts..."
```

Use `date` in the shell to get the current time, or derive it from elapsed seconds since the install started.

### Timeout

If `timeout_sec` elapses without reaching a terminal state, report:

> ⚠️ **Deployment timed out** after {timeout_sec}s. The last known status was `{status}` ({percentComplete}% complete).
> The operation may still be in progress on the server. You can check manually using:
> `GET <status_url>`

## Step 3 — Final report

When the status reaches a terminal state, present a summary:

```
✅ Deployment complete
   Container : BILLPAY
   Version   : 2024.04.1
   Target    : Production
   Duration  : 1m 23s
   Message   : Successfully installed.
```

For failures:

```
❌ Deployment failed
   Container : BILLPAY
   Version   : 2024.04.1
   Target    : Production
   Duration  : 0m 47s
   Error     : <message from response>
   Details   : <errorDetails from response, if present>
```

## Resuming a monitor session

If the user already has a queue URL from a previous install (the server returned a queue URL but this session did not capture it), skip Step 1 and go directly to Step 2 with the provided URL. Ask:

> "Do you have the queue status URL from the install response? If so I can monitor it directly."

## Security notes

- Never reveal the password in any output — this covers both command stdout and any curl commands shown to the user. Always display credentials as `-u "<username>:<password>"` (placeholder form) or `-u "$BAW_USER:$BAW_PASS"` (env var form), never substituting the real password value.
- If the server uses a self-signed certificate, append `-k` to curl commands and warn the user that `-k` disables certificate validation.

## Boundaries

**In scope:**
- Listing process apps and toolkits with their available snapshots via `GET …/containers`
- Triggering a snapshot installation via `POST …/install`
- Monitoring deployment progress via the queue URL
- Reporting installation success or failure

**Out of scope — redirect clearly:**
- Exporting or creating snapshots — use **Workflow Center** (traditional BAW) or **Business Automation Studio** (CP4BA containerized) directly
- Migrating data or process instances — not covered by this API
- Setting up environment configuration (data sources, LDAP) — administrative task outside this skill
- Deploying via TWX file import — different workflow; this skill uses the container/version REST route only


---

---
name: baw-endpoint-validation
description: "Use when a BAW administrator wants to validate a process application before installing it on a new server — asking what new server to target, collecting source and target endpoints, scanning the TWX export for hardcoded references to the old server, and generating a report of findings. Supports both traditional BAW (reads BPMConfig .properties files or runs BPMConfig -export) and CP4BA containerized deployments on OpenShift (reads the ICP4ACluster custom resource via oc get). Trigger phrases: 'install process app on new server', 'check for hardcoded endpoints', 'validate TWX for new environment', 'endpoint validation', 'what needs to change to move to new server', 'BPMConfig export', 'scan TWX for old server references', 'run BPMConfig', 'validate TWX on CP4BA', 'CP4BA endpoint validation', 'oc get ICP4ACluster'. Do not use for runtime process inspection (use inspect-baw-processes) or BPMN authoring (use generate-baw-bpmn)."
license: Apache-2.0
compatibility: "IBM BAW 22.x and later; IBM CP4BA 22.x and later"
allowed-tools:
  - Read
  - Edit
  - Bash
metadata:
  version: "1.0.0"
  display_name: BAW Endpoint Validation
  short_description: Scan a BAW process app TWX export for hardcoded server endpoints and generate a migration report before moving to a new environment.
  author: IBM BAW Team
---
# BAW Endpoint Validation

Identify what needs to change when a BAW process application is installed on a new server. Supports both deployment types:

| Deployment type | How source endpoints are collected |
|---|---|
| **Traditional BAW** (on-premises / VM with WebSphere) | `BPMConfig -export` or a user-provided `.properties` file |
| **CP4BA containerized** (OpenShift / Kubernetes) | `oc get ICP4ACluster` YAML or a user-provided CR YAML snippet |

Steps for both paths:
1. Detect deployment type (traditional BAW or CP4BA containerized).
2. Collect source environment endpoints (Option A/B for traditional, Option C for CP4BA).
3. Collect target environment endpoints (same options).
4. Scan the TWX export for hardcoded references pointing to the old server.
5. Generate a report of findings with a summary of the intended endpoint changes.

The TWX scan is driven by `scripts/scan_twx_endpoints.py`.


## Credentials

Never prompt the user for credentials. Credentials are only needed when making a REST call.
When a REST call is about to be made, read `references/CREDENTIAL_HANDLING.md` and follow it — it will read credentials from `.env` automatically, and only prompt the user if a value is missing from that file. Do this once per session only.

---

## Key constraint

Bob derives all environment information from exported BPMConfig properties files, values the user provides directly, or commands run by Bob via `execute_command`. Paths come from the user or are discovered automatically (profiles directory). Do not invent paths that are not discoverable.

## Offline promotion context

`BPMConfig -export` requires the **deployment manager (or stand-alone server) to be running**. Export the properties file before taking any server offline.

The TWX scan (Steps 4–5) is entirely file-based and safe to run with the server stopped.

The actual offline promotion step uses the `BPMInstallOfflinePackage` wsadmin command, which installs a snapshot onto a Workflow Server not connected to Workflow Center. This skill validates the TWX *before* that step — it does not perform the promotion itself.

## Deployment type detection

**Perform this check before any other step.** Determine whether the user is on a traditional BAW deployment or a CP4BA containerized deployment.

### Signals that indicate CP4BA containerized

- User mentions **CP4BA**, **Cloud Pak for Business Automation**, **OpenShift**, **OCP**, **Kubernetes**, **k8s**, or **pods**
- User mentions `icp4adeploy` or a `ICP4ACluster` custom resource
- User mentions `cpd-cli`, `oc` (OpenShift CLI), or `kubectl`
- User provides a hostname ending in a cluster domain (e.g. `.svc.cluster.local`, `.apps.<cluster>`)
- User mentions that `BPMConfig` is not found or unavailable
- A provided BAW install root path does not exist when tested via `execute_command` (the directory is absent — this is normal inside a CP4BA container where no traditional BAW filesystem is present)
- `oc version --client` or `kubectl version --client` succeeds when run via `execute_command` (indicates Bob is running inside or alongside an OpenShift/Kubernetes environment)

### Signals that indicate traditional BAW

- User provides a `.properties` file exported by `BPMConfig`
- User mentions WebSphere, Deployment Manager (dmgr), Network Deployment, or a profiles directory
- User describes an on-premises or VM-based installation
- User provides a BAW install root path **AND** confirms they are working directly on that server (e.g. SSH'd into the BAW host, not a cluster node)

> **Note:** Install root path alone is **not** sufficient to conclude traditional BAW. An admin managing a migration from traditional BAW to CP4BA will often state the old install root as the source, while their current machine has `oc` and no BAW filesystem. Always confirm the working environment before attempting `BPMConfig`.

### When the deployment type is ambiguous

Ask using `ask_followup_question`:

> "Is your BAW environment a traditional on-premises installation (running on VMs with WebSphere), or a CP4BA containerized deployment on OpenShift/Kubernetes?"

With options:
- "Traditional BAW on-premises / VM installation"
- "CP4BA containerized deployment on OpenShift or Kubernetes"

### If CP4BA containerized is confirmed

Do not invoke any BPMConfig steps. **Do not attempt to access any BAW install root path** — on CP4BA there is no BAW filesystem accessible outside of pods, and `oc exec` into pods is not required. All endpoint data comes from the `ICP4ACluster` custom resource and OpenShift routes. Proceed with **Option C** (see Step 1, Option C below).

### Mixed scenario — migrating from traditional BAW to CP4BA

When the user mentions **both** an old on-premises BAW install root (the source) **and** a CP4BA/OpenShift target, the deployment types are different per role:

| Role | Deployment type | How to collect endpoints |
|---|---|---|
| **Source** | Traditional BAW | `BPMConfig -export` on the old server, or user provides `.properties` file |
| **Target** | CP4BA containerized | `oc get ICP4ACluster` on the target cluster |

In this case: use Option B for source, Option C for target. Confirm this split with the user before proceeding.

## Input requirements

Collect all three items before proceeding. Do not assume which environment (source or target) a block belongs to — always read the user's explicit labelling in their message.

| # | Input | Traditional BAW | CP4BA containerized |
|---|---|---|---|
| 1 | **Source endpoints** | User-provided `.properties` file path, or run `BPMConfig -export` | User-provided CR YAML snippet, or run `oc get ICP4ACluster` |
| 2 | **Target endpoints** | User-provided `.properties` file path, or stated hostname/URL | User-provided CR YAML snippet, or stated hostname/URL |
| 3 | **TWX file** | Prefer to download via the export API (see Step 4 — Acquire TWX). If the user already has a `.twx` path on disk, use it directly — no export needed. |

**Ownership rule — do not assume ownership of a block.** If the user says "my target is `https://host:9443/` [block]", the block is the **target**. Misassigning source/target will produce an inverted endpoint map and incorrect findings.

Ask using `ask_followup_question` for any item not yet clearly provided with an explicit source/target label.

**Tool scope** — this skill uses exactly four tools: `read_file` (read properties, CR YAML, and TWX files), `write_file` (write the endpoint validation report in Step 5), `execute_command` (run BPMConfig/oc discovery in Steps B1–B5/C1 and the scan script in Step 4c), and `ask_followup_question` (collect missing inputs). No other tools are used.

**Automatic execution** — `execute_command` is restricted to read-only discovery and export operations. Discovery commands (`python`, `find`, `BPMConfig -export`, `oc get`) and the scan script run immediately without a confirmation prompt. Any command that modifies, deletes, or overwrites files on the server requires a pause and explicit user approval before running. Pause on read-only operations only if a command fails.

**`read_file` is for explicit file paths only — never for filesystem exploration.** Only use `read_file` on paths the user has explicitly provided (e.g. a `.properties` file path, a CR YAML file, a `.twx` path) or that this skill has written itself. Do not use `read_file` to probe whether a directory exists, search for binaries, or list directory contents — use `execute_command` for all discovery.

**If `execute_command` reports that a BAW install root does not exist** (e.g. "no such file or directory") — do not simply ask for a different path. First run the following probe to determine what environment Bob is actually on:

```bash
oc version --client 2>/dev/null || kubectl version --client 2>/dev/null || echo "no-k8s-cli"
```

- If `oc` or `kubectl` responds → this machine is part of a CP4BA/OpenShift environment. `BPMConfig` does not exist here. Discard the install root. Tell the user: *"The path `<install_root>` does not exist on this machine — this appears to be a CP4BA/OpenShift environment where `BPMConfig` is not available. I'll collect configuration from the `ICP4ACluster` custom resource instead."* Proceed to **Option C**.
- If the probe returns `no-k8s-cli` → this machine has neither BAW nor Kubernetes tooling. Tell the user: *"The path `<install_root>` is not accessible from this machine. Please either run the BPMConfig discovery commands on the BAW server and paste the output here, or provide an already-exported `.properties` file."*

## Steps

### Step 1 — Collect environment info

**Before reading any files, identify which environment each piece of input belongs to** by reading the user's explicit labels in their message. Common patterns:

- "my source install is in `C:\baw`" → `C:\baw` is the **source** install root
- "my target is `https://host:9443/` [properties block]" → the properties block is the **target**
- "this .properties is for the target server" → block is **target**
- A bare properties block with no label → **ask** which environment it belongs to before proceeding

Once ownership is established, determine how to obtain each:

**Traditional BAW — source environment** — choose the first that applies:
- User provided a `.properties` **file path** → read it with `read_file`
- User provided an **install root path** (e.g. `C:\baw`) → run `BPMConfig -export` (Option B)
- User pasted properties **inline and labelled it as source** → parse the pasted content
- None of the above → ask: "Should I run `BPMConfig -export` against your source server, or do you have an exported `.properties` file?"

**Traditional BAW — target environment** — choose the first that applies:
- User provided a `.properties` **file path** → read it with `read_file`
- User pasted properties **inline and labelled it as target** → parse the pasted content
- User provided only a **hostname/URL** → ask: "Do you have a BPMConfig `.properties` file for the target server, or should I use just the hostname you provided?"
- None of the above → ask for it

**CP4BA containerized — source environment** — choose the first that applies:
- User pasted a CR YAML snippet **labelled as source** → parse it (Option C)
- User provided a namespace and CR name → run `oc get ICP4ACluster` (Option C)
- None of the above → ask: "Should I run `oc get ICP4ACluster` against your source cluster, or do you have the CR YAML?"

**CP4BA containerized — target environment** — choose the first that applies:
- User pasted a CR YAML snippet **labelled as target** → parse it (Option C)
- User provided a target namespace/CR name → run `oc get ICP4ACluster` on the target cluster (Option C)
- User provided only a **hostname/URL** → use it directly as the target; no CR needed
- None of the above → ask for the target hostname or CR YAML

#### Option A — User provides existing `.properties` file (traditional BAW)
Read it directly with `read_file`. No further questions needed for the source.

#### Option B — Run `BPMConfig -export` directly

**Ask for the BAW install root if it is not already in the conversation:**

> "What is the BAW install root directory on the source server? (e.g. `/opt/IBM/BAW` on Linux or `C:\IBM\BAW` on Windows)"

Once provided, discover everything else automatically. **Do not assume a fixed directory layout** — BAW installations vary; search the tree before asking anything.

**Step B1 — Locate BPMConfig**

Check the standard location first: `<install_root>\bin\BPMConfig.bat` (Windows) or `<install_root>/bin/BPMConfig.sh` (Linux). If not found there, search the entire tree using Python:

```python
python -c "
import os, sys
root = sys.argv[1]
name = 'BPMConfig.bat' if os.name == 'nt' else 'BPMConfig.sh'
for dirpath, dirnames, filenames in os.walk(root):
    if name in filenames:
        print(os.path.join(dirpath, name))
" "<install_root>"
```

If multiple matches are found, prefer the one whose path contains `\bin\` or `/bin/`.

**If BPMConfig is not found anywhere in the tree** — do not silently fail. Display:

> ⚠️ **BPMConfig not found under `<install_root>`.**
> `BPMConfig` ships only with traditional BAW (WebSphere-based). It is not present on CP4BA containerized deployments.
>
> - If this is a **traditional BAW** installation, please confirm the correct install root — `BPMConfig.bat` / `BPMConfig.sh` should be under `<install_root>/bin/`.
> - If this is a **CP4BA containerized deployment** on OpenShift/Kubernetes, `BPMConfig` is unavailable by design. I'll collect configuration from the `ICP4ACluster` custom resource instead.

Ask using `ask_followup_question`:
- "This is traditional BAW — provide the correct install root"
- "This is a CP4BA containerized deployment — proceed with ICP4ACluster CR"

If the user confirms CP4BA, stop Option B entirely and proceed to **Option C**.

**Step B2 — Locate the profiles directory**

Once `BPMConfig` is found, its parent `bin\` directory gives the real install root. Look for `profiles\` as a sibling of `bin\`. If that doesn't exist, search the tree using Python (depth-limited to 4 levels):

```python
python -c "
import os, sys
root = sys.argv[1]
for dirpath, dirnames, dirfiles in os.walk(root):
    depth = dirpath[len(root):].count(os.sep)
    if depth >= 4:
        dirnames[:] = []
        continue
    if 'profiles' in dirnames:
        print(os.path.join(dirpath, 'profiles'))
" "<real_install_root>"
```

If still not found, search for `BPMConfig.bat`/`.sh` inside any `config\cells` directory as an alternative sign of a profile root — or look directly for directories containing a `config\cells` subdirectory.

**Step B3 — Discover the profile name**

List child directories inside the discovered `profiles\` directory. If exactly one exists, use it. If multiple exist, ask the user to choose.

**Step B4 — Discover the DE name**

Check `<profile>\config\cells\*\bpmconfig\` for `.properties` files. The DE name is the filename without extension. If exactly one file exists, use it. If multiple exist, ask.

As a fallback, search the entire profile root using Python:

```python
python -c "
import os, sys
root = sys.argv[1]
for dirpath, dirnames, filenames in os.walk(root):
    if 'bpmconfig' in dirpath:
        for f in filenames:
            if f.endswith('.properties'):
                print(os.path.join(dirpath, f))
" "<profile_root>"
```

**Step B5 — Output directory and export**

Use the system temp directory:
```
python -c "import tempfile; print(tempfile.gettempdir())"
```

Then run the export. On Linux/AIX:
```bash
<real_install_root>/bin/BPMConfig.sh -export -profile <DmgrProfile> -de <DE_name> -outputDir <temp_dir>
```
On Windows:
```
<real_install_root>\bin\BPMConfig.bat -export -profile <DmgrProfile> -de <DE_name> -outputDir <temp_dir>
```
The `-de` option can be omitted when only one DE exists in the cell. The command produces `<DE_name>.properties` in `<temp_dir>`.

After the export completes, read the output file immediately.

#### Option C — Read the `ICP4ACluster` custom resource (CP4BA containerized)

**Step C0 — Check whether `oc` is available**

Before attempting any `oc` command, run a client-only version check (no cluster connection required):

```bash
oc version --client
```

- **If the command succeeds** — `oc` is present. Ask for the namespace and CR name if not already in the conversation (default CR name: `icp4adeploy`), then run:

  ```bash
  oc get ICP4ACluster icp4adeploy -n <namespace> -o yaml
  ```

  If the command fails with an authentication or connection error (not a "command not found" error), the user is not logged in. Tell them:

  > ⚠️ **`oc` is installed but you are not logged in to a cluster.**
  > Please run `oc login <cluster-api-url>` and try again, or paste the CR YAML directly.

- **If `oc version --client` fails** — `oc` is not installed on this machine. Do not attempt further `oc` commands. Tell the user:

  > ⚠️ **The OpenShift CLI (`oc`) is not available on this machine.**
  > Please run the following command on a machine with cluster access and paste the output here:
  >
  > ```bash
  > oc get ICP4ACluster icp4adeploy -n <your-namespace> -o yaml
  > ```
  >
  > Alternatively, if you have access to the OpenShift web console, navigate to **Administration → CustomResourceDefinitions → ICP4ACluster → Instances** and copy the YAML from there.

  Wait for the user to paste the YAML before continuing.

> **Note:** `kubectl` can substitute for `oc` for read-only CR access when the user is logged in via `kubeconfig`. If `oc` is absent but `kubectl` is present, run `kubectl get ICP4ACluster icp4adeploy -n <namespace> -o yaml` instead.

> **Security note:** Store `{BAW_CSRF}` and `{BAW_SESSION}` in memory for this conversation only. Do not write them to any file. Ensure all calls use HTTPS.

Parse the YAML output — the key fields to extract are:

**Workflow hostname (BAW external URL)**
- `spec.baw_configuration[*].hostname` — external hostname for each BAW instance

**Deployment hostname suffix (applies to all routes if `hostname` is not set per-component)**
- `spec.shared_configuration.sc_deployment_hostname_suffix` — wildcard suffix (e.g. `apps.mycluster.example.com`)
  - If set, the BAW route hostname is `<baw-instance-name>.<sc_deployment_hostname_suffix>`

**Databases**
- `spec.datasource_configuration.dc_baw_datasource[*].dc_database_type` — database type
- `spec.datasource_configuration.dc_baw_datasource[*].dc_oracle_server_name` or equivalent hostname field
- `spec.datasource_configuration.dc_baw_datasource[*].database_port`

**Namespace / cluster**
- `metadata.namespace` — the namespace this CR lives in (used to identify the cluster)

If the `hostname` field is absent, construct the BAW URL as `https://<cr-name>-baw-server.<sc_deployment_hostname_suffix>` or ask the user for the actual route:

```bash
oc get route -n <namespace> | grep baw
```

Build a **Source Endpoints Map** from whichever fields are present. Include every distinct hostname found.

### Step 2 — Parse source configuration

**Traditional BAW path** — use `read_file` to read the source `.properties` file. Extract:

**Server / topology**
- `bpm.de.node.#.hostname` — managed node hostname(s)
- `bpm.dmgr.hostname` — deployment manager hostname (ND topology only)

**Databases** (for each `bpm.de.db.#.*` block)
- `bpm.de.db.#.hostname` — database server hostname
- `bpm.de.db.#.portNumber` — database port
- `bpm.de.db.#.databaseName` — database name
- `bpm.de.db.#.name` — logical DB label (e.g. `ProcessServerDb`)

**Install paths**
- `bpm.de.node.#.installPath` — BAW installation path on each node
- `bpm.de.caseManager.networkSharedDirectory` — Case Manager shared directory

**CP4BA containerized path** — parse the CR YAML obtained in Option C. Extract the fields listed in Option C above.

Build a **Source Endpoints Map**: a deduplicated list of every distinct hostname, IP address, and (for traditional BAW only) file path found.

### Step 3 — Capture target environment

**Traditional BAW:**
1. **Explicit `.properties` path already provided** — read it with `read_file` and extract the same fields as Step 2.
2. **No target `.properties` provided** — ask using `ask_followup_question`:
   > "Do you have a BPMConfig `.properties` file exported from the target server? If so, please provide the path. Otherwise I'll use the hostname/URL you've already stated."
3. **User confirms no file** — fall back to verbal values already in the conversation (hostname, port, protocol). Extract the hostname and port from any URL provided; infer HTTPS if port is 9443.

**CP4BA containerized:**
1. **User provided a target CR YAML or namespace** — run `oc get ICP4ACluster` on the target cluster (Option C) and extract the same fields.
2. **User provided only a target hostname/URL** — use it directly. Note the BAW URL pattern on CP4BA containers is `https://<baw-route-hostname>/WorkflowCenter` or `/ProcessPortal`.
3. **No target info provided** — ask: "What is the hostname or namespace of your target CP4BA cluster?"

Build the **Target Endpoints Map** from whichever source is used.

### Step 4 — Acquire the TWX

Use the first path that succeeds. If the user already stated a `.twx` file path earlier in the conversation, skip to Step 5 directly using that path.

**Path 1 — Export API (preferred)**


#### Step 4a — Resolve container and version acronyms

If the user provided an acronym and snapshot, skip to Step 4b. Otherwise:

1. **List containers:**
   ```
   GET {BAW_BASE_URL}{CTX}/ops/std/bpm/containers
   ```
   Match `container_name` against the app name the user gave. Record the `container` field as `{CONTAINER}`.

2. **List versions:**
   ```
   GET {BAW_BASE_URL}{CTX}/ops/std/bpm/containers/{CONTAINER}/versions
   ```
   Pick the most recent non-archived version. Record its `version` field as `{VERSION}`. If there are multiple snapshots, show the user a brief table and ask which to export.

#### Step 4b — Export the TWX

```
GET {BAW_BASE_URL}{CTX}/ops/std/bpm/containers/{CONTAINER}/versions/{VERSION}/export?use_enhanced_filenames=true
```

Include headers:
```
Authorization: Basic <base64(user:pass)>
BPMCSRFToken: {BAW_CSRF}
Cookie: {BAW_SESSION}
Accept: application/octet-stream, */*
```

The `Accept: application/octet-stream, */*` header is **required** — omitting it causes a `406 Not Acceptable` error.

The response is a binary `.twx` file. Get the system temp directory:

```python
python -c "import tempfile; print(tempfile.gettempdir())"
```

Save as `<temp_dir>/<CONTAINER>_<VERSION>.twx`. Use that path as `<absolute_path_to_twx>` throughout Steps 5–6.

If the export returns 403/404, re-check `{CONTAINER}` and `{VERSION}` via the listing calls in Step 4a.

**Path 2 — Local file (fall back after Path 1 has visibly failed)**

If the REST export fails or errors more than once, offer:
> *"If you already have the `.twx` file for this app on your machine, you can point me to it and I'll work from that directly."*

Use the provided path directly as `<absolute_path_to_twx>`. Do not ask pre-emptively — only after a real failure.

### Step 5 — Run TWX endpoint scan

The script path is `<skill_dir>/scripts/scan_twx_endpoints.py` where `<skill_dir>` is the directory containing this SKILL.md.

Obtain the system temp directory once and reuse it for all temp files in Steps 4a–4d:
```
python -c "import tempfile; print(tempfile.gettempdir())"
```

Delete the source map immediately after the scan, regardless of success or failure.

**Step 4a — Write the source map**

Write the source map to `<temp_dir>/_baw_source_map.json`. Include every distinct hostname and path value from the Source Endpoints Map. Example:

```json
{
  "old-node-hostname": "node-hostname",
  "old-db-hostname": "db-hostname",
  "C:/baw/BUILT/deploy2/AppServer": "source-install-path"
}
```

Include `localhost` if the source environment genuinely uses it (e.g. a single-node install where all services run on the same host). Do **not** include values like `localhost` that map to themselves unless the source properties actually contain `localhost` — only include terms that are meaningful to search for.

**Step 4b — Derive the report output path**

```
<twx_dir>/<twxfilename> - reports/
```
where `<twx_dir>` is the directory containing the TWX and `<twxfilename>` is the full TWX filename including the `.twx` extension (e.g. `ClaimsApp.twx`). Create this directory:
```
python -c "import os; os.makedirs(r'<report_dir>', exist_ok=True)"
```

**Step 4c — Run the scan**

Use `--json` for structured output.

On Windows:
```
python "<skill_dir>\scripts\scan_twx_endpoints.py" --twx "<absolute_path_to_twx>" --source-map-file "<temp_dir>\_baw_source_map.json" --json
```

On Linux/AIX:
```bash
python "<skill_dir>/scripts/scan_twx_endpoints.py" \
  --twx "<absolute_path_to_twx>" \
  --source-map-file "<temp_dir>/_baw_source_map.json" --json
```

After the run (success or failure), delete the temp source map:
```
python -c "import os; os.remove(r'<temp_dir>/_baw_source_map.json')"
```

The `--json` flag causes the script to print a single JSON object to stdout. Parse this object — it has the shape `{"findings": [...], "files_scanned": N, "binary_files_skipped": N, "source_terms_searched": [...]}`. Each finding has `file`, `line`, `matched_pattern`, `severity`, and `context_snippet` fields. Capture this JSON and use it in Step 4d.

If the script errors, diagnose first: verify the TWX path, check `python --version` (3.6+ required), confirm the TWX is not password-protected.

**Step 4d — Resolve artifact names from the TWX**

For every unique `objects/*.xml` path in the findings, extract the artifact's human-readable name and type from inside the TWX. Write the following to `<temp_dir>/_baw_resolve_names.py` and run it:

```python
import zipfile, re, json, sys

twx_path = sys.argv[1]
files_arg = sys.argv[2]  # comma-separated list of TWX-internal paths to resolve

targets = [f.strip() for f in files_arg.split(',') if f.strip().startswith('objects/')]

results = {}
with zipfile.ZipFile(twx_path) as z:
    for entry in targets:
        try:
            content = z.read(entry).decode('utf-8', errors='replace')
            # Extract root element tag and name attribute from first ~800 chars
            head = content[:800]
            tag_match = re.search(r'<(\w+)\s', head)
            name_match = re.search(r'\bname="([^"]+)"', head)
            tag = tag_match.group(1) if tag_match else 'unknown'
            name = name_match.group(1) if name_match else entry
            results[entry] = {'tag': tag, 'name': name}
        except Exception as e:
            results[entry] = {'tag': 'unknown', 'name': entry, 'error': str(e)}

print(json.dumps(results))
```

Run it:
```
python "<temp_dir>/_baw_resolve_names.py" "<absolute_path_to_twx>" "<comma-separated unique objects/*.xml paths from findings>"
```

Delete the temp script after running:
```
python -c "import os; os.remove(r'<temp_dir>/_baw_resolve_names.py')"
```

The output is a JSON object mapping each TWX internal path to `{"tag": "<xml-element-name>", "name": "<artifact-name>"}`. Use the following tag-to-type mapping to produce a human-readable artifact type:

| XML tag | Artifact type in Workflow Center / Studio |
|---|---|
| `process` (processType=12) | Service Flow |
| `process` (processType=11) | Heritage Human Service |
| `process` (processType=10 or other) | Service Flow |
| `bpd` | Process (BPD) |
| `externalActivity` | REST operation (child of External Service) |
| `environmentVariableSet` | Environment Variables |
| `teamworks` wrapping `process` with no processType | Service |
| any other | BAW Artifact |

To distinguish Service Flow from Heritage Human Service, check for `<processType>12</processType>` in the XML content (processType 12 = Service Flow, 11 = Heritage Human Service). If processType is absent or ambiguous, label it "Service".

For `files/*` entries (not `objects/*.xml`), do not attempt name resolution — label them as **"Auto-generated binding file"** and note they are regenerated when the parent artifact is saved.

### Step 6 — Generate report

Write the report using `write_file` to `<report_dir>/endpoint-validation-report.md`. Use this structure:

```markdown
# BAW Endpoint Validation Report

## Environment Transition Summary
| | Value |
|---|---|
| Process App TWX | <filename> |
| Deployment type | Traditional BAW / CP4BA containerized |
| Source config | <path or "ICP4ACluster CR — namespace/name"> |
| Target config | <path, CR, or "stated verbally"> |
| Scan Date | <today's date> |
| Files scanned | <N> |
| Binary files skipped | <N> |

## Source → Target Endpoint Mapping
| Property | Source Value | Target Value | Status |
|---|---|---|---|
| bpm.de.node.1.hostname | old-host | new-host | ✅ Changed |
...

## TWX Hardcoded Reference Findings

### 🔴 Critical (<count>)

| # | Artifact Name | Type | Step / Variable | Line | Context | Where to fix |
|---|---|---|---|---|---|---|
| 1 | External service | External Service | External service operation1 | 99 | `urlTemplate="http://localhost:9080/MyAPI/rest/v1"` | **Traditional BAW:** Workflow Designer → Process App Settings → Servers → MyAPI. **CP4BA (Studio):** Business Automation Studio → open process app → Process App Settings → Servers → MyAPI |
...

### 🟡 Warning (<count>)

| # | Artifact Name | Type | Step / Variable | Line | Context | Notes |
|---|---|---|---|---|---|---|
...

### ℹ️ Info (<count>)
<count> additional lower-priority matches. No direct action required.

## Recommended Actions

### Blockers (must resolve before install)

1. **<short title>** — <what to do and where, using plain navigation text only>
   - <sub-step if needed>

### Warnings (review before go-live)

1. **<short title>** — <what to check>

## Version Compatibility Note
*(include only if source and target fix pack versions differ)*

| | Source | Target |
|---|---|---|
| Fix pack | `<version>` | `<version>` |
| Product type | <type> | <type> |
| Environment | <env> | <env> |

> **Blocker/Warning:** <one-sentence plain-text explanation>

## Overall Readiness
✅ Ready to promote / ⚠️ Review required / 🔴 Blockers must be resolved
```

**Report writing rules:**

- **Artifact Name** column: use the `name` value from Step 4d. For `files/*` entries use "Auto-generated binding file".
- **Type** column: use the mapped type from the tag-to-type table in Step 4d.
- **Step / Variable** column: for `<script>` findings, extract the nearest `name="…"` attribute in the 25 lines before the match (this is the step name). For `environmentVariableSet`, use the `envVar name` attribute. For `externalActivity`, use the operation name. Leave blank if not determinable.
- **Where to fix** column: use the table in Step 7 to select the right navigation path; include the artifact name so the developer knows exactly what to open. For inline scripts, add "⚠️ Cannot fix via Process Admin Console — requires code change in Workflow Designer."
- **Group duplicate entries**: if the same artifact + line context appears twice (e.g. line 84 and 216 in the same file, both containing the same script), merge them into a single row with a note like "(2 occurrences — fixing in Designer updates both)".
- Do not expose internal UUID paths in the report body. They may appear in a collapsed `<details>` footnote at most.
- **Hyperlinks must resolve.** Only include a hyperlink in the report if the URL is taken directly from the verified References section at the bottom of this SKILL.md. Do not construct version-specific IBM Docs URLs that are not in the References section — they cannot be verified without a search. Navigation paths (e.g. `Workflow Designer → Process App Settings → Servers`) are always written as plain text; a link is optional and only added when a verified URL is available.

### Step 7 — Present summary and fix guidance

After writing the report, give the user a plain-language summary inline covering:
- How many critical references were found and in which files
- What the endpoint changes are (old → new, per property)
- Whether the process app is ready to install on the new server

For each Critical finding, tell the user exactly where to fix it in BAW 26.0.x:

| Finding type | Where to fix (BAW 26.0.x) |
|---|---|
| REST server URL hardcoded in `<urlTemplate>` or `urlTemplate` attribute | **Traditional BAW:** In **Workflow Designer**, open the process app → **Process App Settings** → **Servers** tab → find the REST server entry → update **Host name** and **Port**. **CP4BA (Studio):** Business Automation Studio → open process app → **Process App Settings** → **Servers** tab → update the entry. |
| REST server host/port needs updating post-deployment (without re-importing TWX) | In the **Process Admin Console** → **Installed Apps** → click the overflow menu of the snapshot → **App Details** → **Servers** tab → update **Hostname**, **Port**, and **Secure Server** on each REST server binding. (ref: https://www.ibm.com/docs/en/baw/26.0.x?topic=properties-changing-server-settings-in-process-admin-console) |
| Hardcoded URL in a service flow script (`<script>` tag) | **Traditional BAW:** In **Workflow Designer**, navigate to the service flow → **Script** tab of the script task → update the hardcoded URL. **CP4BA (Studio):** Business Automation Studio → open process app → open the service flow → script task → **Script** tab. Consider replacing with a server variable bound via Process App Settings instead. |
| Hardcoded URL in a client-side human service (coach view script) | **Traditional BAW:** In **Workflow Designer**, open the Coach or client-side human service → find the view with the inline script → **Script** tab. **CP4BA (Studio):** Business Automation Studio → open process app → open the coach view → **Script** tab. |
| Hostname in a variable default value (`<defaultValue>`) | **Traditional BAW:** In **Workflow Designer**, open the service or process → **Variables** tab → find the variable → update the default value. **CP4BA (Studio):** Business Automation Studio → open process app → open the artifact → **Variables** tab. |
| Hardcoded portal link in notification / task activity documentation | **Traditional BAW:** In **Workflow Designer**, open the process → click the notification or task activity → edit the **Documentation** or **Notification** field. **CP4BA (Studio):** Business Automation Studio → open process app → open the process → click the activity → edit **Documentation** / **Notification** field. |
| JDBC URL in a service SQL call | In the **WebSphere Integrated Solutions Console (ISC)** on the target server → **Resources → JDBC → Data Sources** → update the data source URL. This is a server-side change, not made in Designer. |
| REST server URL hardcoded in TWX (CP4BA target) | REST server hostname/port can be updated post-deployment in the **Process Admin Console** → **Installed Apps** → overflow menu → **App Details** → **Servers** tab. On CP4BA containers the Process Admin Console is available at `https://<baw-route-hostname>/ProcessAdmin`. |
| Environment endpoint needs updating at cluster level (CP4BA) | Edit `spec.baw_configuration[*].hostname` or the external service configuration in the `ICP4ACluster` CR and re-apply with `oc apply -f`. Server-level endpoint changes (e.g. database host) must be made in the CR, not in Designer. Consult your CP4BA platform administrator. |

Do not offer to generate XML patches, apply configuration, or edit files inside the TWX — that is out of scope.

## Disambiguation rules

**No live BAW API calls**
REST calls are only used to: (1) authenticate via `POST {CTX}/ops/system/login`, and (2) download the TWX via `GET {CTX}/ops/std/bpm/containers/{container}/versions/{version}/export`. Do not make any other REST calls to the BAW runtime. All environment analysis is file-based.

**No TWX path provided and no server access**
If the user cannot connect to the BAW server and has no `.twx` file on disk, explain how to export based on their deployment:

- **Traditional BAW (Workflow Center):** Open Workflow Center → navigate to the process app → find the snapshot → click **Export** → select **.twx** → save the file. The `.twx` option is for Workflow Center import/export; use `.zip` only for installing directly to a runtime environment. (ref: https://www.ibm.com/docs/en/baw/26.0.x?topic=snapshots-importing-exporting-projects-toolkits)
- **CP4BA containerized (Business Automation Studio):** Open Business Automation Studio → **Business automations** → open the process app → select the snapshot → overflow menu → **Export** → save the `.twx` file.

**Multiple deployment environments**
If the source properties file contains multiple `bpm.de.*` blocks, extract endpoints from all blocks and report them separately by DE number.

**`localhost` in source properties**
`localhost` in a single-node (Express/stand-alone) source environment is valid — include it in the source map so references to `http://localhost:9080/...` are caught in the TWX scan.

**`localhost` findings in the TWX — BAW-internal vs external**
Not all `localhost` findings are blockers. Apply this classification when reporting:

| URL pattern | Classification | Rationale |
|---|---|---|
| `http://localhost:.../portal/...` | ⚠️ Warning | BAW portal — resolves to the local server. Valid for stand-alone. Only a concern if HTTPS is required. |
| `http://localhost:.../teamworks/...` | ⚠️ Warning | BAW internal endpoint — same reasoning. |
| `http://localhost:.../rest/bpm/...` | ⚠️ Warning | BAW REST API — resolves locally. |
| `<defaultValue>` or variable binding with `"host":"localhost"` | ⚠️ Warning | Likely a BAW server variable, not an external service. |
| `http://localhost:.../<custom-app>/...` | 🔴 Critical — conditional | External REST API (e.g. `MyAPI`, `OrderAPI`). Whether `localhost` is valid depends on whether the API is co-located on the target BAW server. Flag as conditional blocker and ask the user to confirm deployment topology. |
| `urlTemplate` or `<url>` pointing to `localhost` with a non-BAW context root | 🔴 Critical — conditional | Same as above. |

When in doubt, flag as conditional and explain the two scenarios clearly rather than asserting it is definitely broken.

## Known limitations

**CP4BA containerized — `oc` / `kubectl` access required** — Collecting source endpoints from a CP4BA deployment requires the OpenShift CLI (`oc`) or `kubectl` to be installed and logged in, or the user must supply the `ICP4ACluster` CR YAML manually. If neither is available, the skill pauses and asks for a paste rather than failing silently. The skill does **not** support `BPMConfig` on containerized deployments — that tool is absent by design on CP4BA.

**Encrypted property values (traditional BAW)** — BPMConfig exports mask passwords with `{xor}` encoding. The scan does not attempt to decrypt these; the skill excludes them from the endpoint map.

**Binary or compiled artifacts inside TWX** — some TWX files contain compiled JavaScript or minified CSS where pattern matching may miss encoded references. The scan reports binary files skipped; flag this in the report if relevant.

**Script requires Python 3.6+** — `scripts/scan_twx_endpoints.py` uses only stdlib (`zipfile`, `xml.etree.ElementTree`, `re`, `json`, `argparse`). No pip install required.

## References

- IBM Docs — BPMConfig command-line utility: https://www.ibm.com/docs/en/baw/26.0.x?topic=utilities-bpmconfig-command-line-utility
- IBM Docs — Configuration properties for the BPMConfig command: https://www.ibm.com/docs/en/baw/26.0.x?topic=utility-configuration-properties-bpmconfig-command
- IBM Docs — Specifying a REST server in Process App Settings: https://www.ibm.com/docs/en/baw/26.0.x?topic=service-specifying-rest-server
- IBM Docs — Changing REST server settings in Process Admin Console: https://www.ibm.com/docs/en/baw/26.0.x?topic=properties-changing-server-settings-in-process-admin-console
- IBM Docs — BPMInstallOfflinePackage command: https://www.ibm.com/docs/en/baw/26.0.x?topic=scripting-bpminstallofflinepackage
- IBM Docs — Importing and exporting projects and toolkits from Workflow Center: https://www.ibm.com/docs/en/baw/26.0.x?topic=snapshots-importing-exporting-projects-toolkits
- IBM Docs — CP4BA custom resource pattern structure: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployment-installing-capabilities-in-openshift-console
- IBM Docs — BAW on containers — verifying installation (ConfigMap/route URLs): https://www.ibm.com/docs/en/baw/23.0.x?topic=deployment-verifying-installation
- IBM Docs — Moving from BAW traditional to CP4BA: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=cloud-business-automation-workflow
- `references/BPMCONFIG_REFERENCE.md` — full properties key reference



