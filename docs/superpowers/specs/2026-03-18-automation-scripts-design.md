# Automation Scripts & Job Tracking - Design Spec

**Date:** 2026-03-18
**Status:** Approved

## Overview

Add 6 new MCP tools to support listing and running automation scripts on devices, plus job and activity tracking for monitoring script execution results.

## Approach

Flat addition to existing files (`src/ninja-api.ts` and `src/index.ts`), matching all current codebase patterns. No new files, no refactoring.

## Files Modified

- `src/ninja-api.ts` - Add 6 new API methods
- `src/index.ts` - Add 6 tool definitions to `TOOLS` array + 6 routing cases in `routeToolCall()`

## New Tools

### 1. `get_automation_scripts`

- **API:** `GET /v2/automation/scripts`
- **Parameters:**
  - `lang` (optional string) - Language code
- **Returns:** Array of AutomationScript objects containing: id, name, description, language, architecture, operatingSystems, scriptParameters, scriptVariables (with id, name, type, required, defaultValue, valueList), active, createdBy, lastUpdatedBy, createdOn, updatedOn
- **Description:** "List all available automation scripts with their IDs, names, languages, parameters, and variable definitions. Use this to discover script IDs and required variables before running scripts."

### 2. `get_device_scripting_options`

- **API:** `GET /v2/device/{id}/scripting/options`
- **Parameters:**
  - `id` (required number) - Device identifier
  - `lang` (optional string) - Language code
- **Returns:** DeviceScriptingOptions object containing:
  - `categories` - Array of {id, name, internal}
  - `scripts` - Array of {type (ACTION|SCRIPT), id, uid, name, language, description, architecture, categoryId}
  - `credentials` - {roles: string[], credentials: [{id, name, type}]}
- **Description:** "Get available scripts, built-in actions, and credentials for a specific device. Returns scripts filtered by device OS/architecture, plus available execution credentials. Use before run_script_on_device to discover what can be run and with which credentials."

### 3. `run_script_on_device`

- **API:** `POST /v2/device/{id}/script/run`
- **Parameters:**
  - `id` (required number) - Device identifier
  - `type` (required enum: ACTION | SCRIPT) - Type of command to run
  - `scriptId` (optional number) - Script ID (required when type is SCRIPT)
  - `actionUid` (optional string) - Built-in action UUID (required when type is ACTION)
  - `parameters` (optional string) - Serialized parameters for execution
  - `runAs` (optional string) - Execution context / credential role (e.g. SYSTEM, LOGGED_ON_USER, LOCAL_ADMIN, DOMAIN_ADMIN)
- **Note on `scriptVariables`:** The Swagger spec does not include `scriptVariables` in the request body schema. The documented body fields are: `type`, `id`, `uid`, `parameters`, `runAs`. Script variable values should be serialized into the `parameters` string field. If the live API accepts `scriptVariables` as an additional body field, this can be added later.
- **Parameter name mapping:** The MCP tool uses `scriptId` and `actionUid` to avoid collision with the device `id` parameter. The routing code must map `args.scriptId` to body field `id` and `args.actionUid` to body field `uid`.
- **Input validation:** The routing code should validate: (a) `type` is ACTION or SCRIPT, (b) when type is SCRIPT, `scriptId` is provided, (c) when type is ACTION, `actionUid` is provided. Return `McpError(InvalidParams)` on failure.
- **Returns:** Job status information. Note: the Swagger spec does not define a response schema (`*/*: {}`), so the response shape should be verified empirically. It likely includes a job UID for tracking.
- **Description:** "Run a script or built-in action on a device. For type SCRIPT, provide scriptId. For type ACTION, provide actionUid. Use get_device_scripting_options first to discover available scripts, actions, and credentials. Track execution via get_device_active_jobs or get_activities with the returned job UID. IMPORTANT: Always confirm with the user before executing. Describe what script will be run, on which device, with what parameters, and the execution context (runAs). Scripts can perform destructive or irreversible actions on devices. Never run a script without explicit user approval."

### 4. `get_active_jobs`

- **API:** `GET /v2/jobs`
- **Parameters:**
  - `jobType` (optional string) - Filter by job type
  - `df` (optional string) - Device filter
  - `lang` (optional string) - Language tag
  - `tz` (optional string) - Time zone
- **Returns:** Array of Job objects containing: uid, deviceId, message, createTime, updateTime, sourceType, jobStatus (START_REQUESTED|STARTED|IN_PROCESS|COMPLETED|CANCEL_REQUESTED|CANCELLED), jobResult (SUCCESS|FAILURE|UNSUPPORTED|UNCOMPLETED), jobType, data, subject, userId
- **Description:** "List all currently active/running jobs system-wide, including script executions. Filter by jobType or device filter."

### 5. `get_device_active_jobs`

- **API:** `GET /v2/device/{id}/jobs`
- **Parameters:**
  - `id` (required number) - Device identifier
  - `lang` (optional string) - Language tag
  - `tz` (optional string) - Time zone
- **Returns:** Array of Job objects for that device
- **Description:** "List currently active/running jobs for a specific device, including script executions."

### 6. `get_activities`

- **API:** `GET /v2/activities`
- **Parameters:**
  - `class` (optional string, enum: SYSTEM | DEVICE | USER | ALL, default: ALL) - Activity class filter
  - `before` (optional string) - Activities before this date
  - `after` (optional string) - Activities after this date
  - `olderThan` (optional number) - Activities older than this activity ID
  - `newerThan` (optional number) - Activities newer than this activity ID
  - `type` (optional string) - Activity type filter (e.g. SCRIPTING)
  - `status` (optional string) - Activity status filter
  - `user` (optional string) - User filter
  - `seriesUid` (optional string) - Filter by job/series UID to track specific script execution
  - `df` (optional string) - Device filter
  - `pageSize` (optional number, min: 10, max: 1000, default: 200) - Limit number of results
  - `lang` (optional string) - Language tag
  - `tz` (optional string) - Time zone
  - `sourceConfigUid` (optional string) - Filter by source script config UID
- **Returns:** Wrapper object with `lastActivityId` (int64, pagination cursor for subsequent requests) and `activities` (array of Activity objects containing: id, activityTime, deviceId, severity, priority, seriesUid, activityType, statusCode, status, activityResult (SUCCESS|FAILURE|UNSUPPORTED|UNCOMPLETED), sourceConfigUid, sourceName, subject, userId, message, type, data). Return the raw API response (matching existing codebase pattern).
- **Description:** "List activity log entries in reverse chronological order. Filter by seriesUid to track a specific script execution, or by type to filter for SCRIPTING activities. Returns status and result but full script stdout/stderr is only available in the NinjaOne dashboard."

## Safety

The `run_script_on_device` tool description includes an explicit instruction for the LLM to always confirm with the user before executing, describing the script, target device, parameters, and execution context. This is enforced at the tool description level as LLM behavioral guidance, in addition to whatever approval UI the MCP client provides.

## Intended LLM Workflow

```
1. get_automation_scripts or get_device_scripting_options
   -> Discover script ID, variable definitions, available credentials

2. Present findings to user, confirm which script to run with what parameters

3. run_script_on_device (returns job UID)

4. get_device_active_jobs or get_activities(seriesUid=jobUid)
   -> Check jobStatus / activityResult for completion and success/failure
```

## API Limitations

- **No script output retrieval:** Full script stdout/stderr is only viewable in the NinjaOne dashboard. The API provides job status, result (SUCCESS/FAILURE), and activity log messages, but not the raw output.
- **Timestamps:** All timestamps are Unix epoch in seconds (double precision), consistent with existing tools.
- **Script variables:** The `scriptVariables` field seen in PolicyConditionScript schemas is not part of the `run_script_on_device` request body per the Swagger spec. Variable values should be passed via the `parameters` string.
- **No pagination on `get_automation_scripts`:** The endpoint returns all scripts at once with no pagination parameters. Large tenants may see large responses.
- **`olderThan`/`newerThan` Swagger inconsistency:** The Swagger descriptions for these `get_activities` parameters appear swapped (names contradict descriptions). Implementation should use the parameter names at face value: `olderThan` = activities with ID less than the given value, `newerThan` = activities with ID greater than the given value.

## Implementation Pattern

Each tool follows the existing pattern:

1. **Tool definition** in `TOOLS` array with `name`, `description`, `inputSchema`
2. **API method** in `NinjaOneAPI` class using `makeRequest()` and `buildQuery()`
3. **Routing case** in `routeToolCall()` that extracts args, calls the API method, and returns `{ content: [{ type: "text", text: JSON.stringify(result, null, 2) }] }`

Error handling uses the existing try/catch in `routeToolCall()` which converts exceptions to `McpError`.
