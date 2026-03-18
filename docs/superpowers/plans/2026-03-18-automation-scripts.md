# Automation Scripts & Job Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 6 new MCP tools for listing automation scripts, running scripts on devices, and tracking job/activity status.

**Architecture:** Flat addition to existing files following established patterns. New API methods in `ninja-api.ts`, new tool definitions and routing in `index.ts`. No new files.

**Tech Stack:** TypeScript, MCP SDK v1.17.1, NinjaOne API v2

**Spec:** `docs/superpowers/specs/2026-03-18-automation-scripts-design.md`

---

### Task 0: Create feature branch

- [ ] **Step 1: Create and switch to feature branch**

```bash
git checkout -b feature/automation-scripts
```

---

### Task 1: Add API methods to `ninja-api.ts`

**Files:**
- Modify: `src/ninja-api.ts:645-661` (append after `getDeviceSoftware` method, before closing `}`)

- [ ] **Step 1: Add the 6 new API methods**

Add the following methods to the `NinjaOneAPI` class, before the closing `}` on the last line:

```typescript
  // Automation Scripts

  async getAutomationScripts(lang?: string): Promise<any> {
    return this.makeRequest(`/v2/automation/scripts${this.buildQuery({ lang })}`);
  }

  async getDeviceScriptingOptions(id: number, lang?: string): Promise<any> {
    return this.makeRequest(`/v2/device/${id}/scripting/options${this.buildQuery({ lang })}`);
  }

  async runScriptOnDevice(
    id: number,
    type: string,
    scriptId?: number,
    actionUid?: string,
    parameters?: string,
    runAs?: string
  ): Promise<any> {
    const body: any = { type };
    if (type === 'SCRIPT' && scriptId !== undefined) body.id = scriptId;
    if (type === 'ACTION' && actionUid !== undefined) body.uid = actionUid;
    if (parameters !== undefined) body.parameters = parameters;
    if (runAs !== undefined) body.runAs = runAs;
    return this.makeRequest(`/v2/device/${id}/script/run`, 'POST', body);
  }

  // Jobs

  async getActiveJobs(jobType?: string, df?: string, lang?: string, tz?: string): Promise<any> {
    return this.makeRequest(`/v2/jobs${this.buildQuery({ jobType, df, lang, tz })}`);
  }

  async getDeviceActiveJobs(id: number, lang?: string, tz?: string): Promise<any> {
    return this.makeRequest(`/v2/device/${id}/jobs${this.buildQuery({ lang, tz })}`);
  }

  // Activities

  async getActivities(params: {
    class?: string;
    before?: string;
    after?: string;
    olderThan?: number;
    newerThan?: number;
    type?: string;
    status?: string;
    user?: string;
    seriesUid?: string;
    df?: string;
    pageSize?: number;
    lang?: string;
    tz?: string;
    sourceConfigUid?: string;
  }): Promise<any> {
    return this.makeRequest(`/v2/activities${this.buildQuery(params)}`);
  }
```

- [ ] **Step 2: Build and verify compilation**

Run: `npm run build`
Expected: Clean compilation with no errors

- [ ] **Step 3: Commit**

```bash
git add src/ninja-api.ts
git commit -m "feat: add API methods for automation scripts, jobs, and activities"
```

---

### Task 2: Add tool definitions to `index.ts` TOOLS array

**Files:**
- Modify: `src/index.ts:863` (insert before the closing `];` of the TOOLS array, after `query_backup_usage`)

- [ ] **Step 1: Add 6 tool definitions**

Insert the following before line 864 (`];`), after the `query_backup_usage` tool:

```typescript
  // Automation Script Tools
  {
    name: 'get_automation_scripts',
    description: 'List all available automation scripts with their IDs, names, languages, parameters, and variable definitions. Use this to discover script IDs and required variables before running scripts.',
    inputSchema: {
      type: 'object',
      properties: {
        lang: { type: 'string', description: 'Language code' }
      }
    }
  },
  {
    name: 'get_device_scripting_options',
    description: 'Get available scripts, built-in actions, and credentials for a specific device. Returns scripts filtered by device OS/architecture, plus available execution credentials. Use before run_script_on_device to discover what can be run and with which credentials.',
    inputSchema: {
      type: 'object',
      properties: {
        id: { type: 'number', description: 'Device identifier' },
        lang: { type: 'string', description: 'Language code' }
      },
      required: ['id']
    }
  },
  {
    name: 'run_script_on_device',
    description: 'Run a script or built-in action on a device. For type SCRIPT, provide scriptId. For type ACTION, provide actionUid. Use get_device_scripting_options first to discover available scripts, actions, and credentials. Track execution via get_device_active_jobs or get_activities with the returned job UID. IMPORTANT: Always confirm with the user before executing. Describe what script will be run, on which device, with what parameters, and the execution context (runAs). Scripts can perform destructive or irreversible actions on devices. Never run a script without explicit user approval.',
    inputSchema: {
      type: 'object',
      properties: {
        id: { type: 'number', description: 'Device identifier' },
        type: { type: 'string', enum: ['ACTION', 'SCRIPT'], description: 'Type of command to run' },
        scriptId: { type: 'number', description: 'Script ID (required when type is SCRIPT)' },
        actionUid: { type: 'string', description: 'Built-in action UUID (required when type is ACTION)' },
        parameters: { type: 'string', description: 'Serialized script/action parameters' },
        runAs: { type: 'string', description: 'Execution context / credential role (e.g. SYSTEM, LOGGED_ON_USER, LOCAL_ADMIN, DOMAIN_ADMIN)' }
      },
      required: ['id', 'type']
    }
  },

  // Job Tracking Tools
  {
    name: 'get_active_jobs',
    description: 'List all currently active/running jobs system-wide, including script executions. Filter by jobType or device filter.',
    inputSchema: {
      type: 'object',
      properties: {
        jobType: { type: 'string', description: 'Filter by job type' },
        df: { type: 'string', description: 'Device filter' },
        lang: { type: 'string', description: 'Language tag' },
        tz: { type: 'string', description: 'Time zone' }
      }
    }
  },
  {
    name: 'get_device_active_jobs',
    description: 'List currently active/running jobs for a specific device, including script executions.',
    inputSchema: {
      type: 'object',
      properties: {
        id: { type: 'number', description: 'Device identifier' },
        lang: { type: 'string', description: 'Language tag' },
        tz: { type: 'string', description: 'Time zone' }
      },
      required: ['id']
    }
  },

  // Activity Tracking Tools
  {
    name: 'get_activities',
    description: 'List activity log entries in reverse chronological order. Filter by seriesUid to track a specific script execution, or by type to filter for SCRIPTING activities. Returns status and result but full script stdout/stderr is only available in the NinjaOne dashboard.',
    inputSchema: {
      type: 'object',
      properties: {
        class: { type: 'string', enum: ['SYSTEM', 'DEVICE', 'USER', 'ALL'], description: 'Activity class filter (default: ALL)' },
        before: { type: 'string', description: 'Return activities before this date' },
        after: { type: 'string', description: 'Return activities after this date' },
        olderThan: { type: 'number', description: 'Return activities with ID less than this value' },
        newerThan: { type: 'number', description: 'Return activities with ID greater than this value' },
        type: { type: 'string', description: 'Activity type filter (e.g. SCRIPTING, ACTION, ACTIONSET)' },
        status: { type: 'string', description: 'Activity status filter' },
        user: { type: 'string', description: 'User filter' },
        seriesUid: { type: 'string', description: 'Filter by job/series UID to track a specific script execution' },
        df: { type: 'string', description: 'Device filter' },
        pageSize: { type: 'number', description: 'Limit number of results (min: 10, max: 1000, default: 200)' },
        lang: { type: 'string', description: 'Language tag' },
        tz: { type: 'string', description: 'Time zone' },
        sourceConfigUid: { type: 'string', description: 'Filter by source script config UID' }
      }
    }
  },
```

- [ ] **Step 2: Build and verify compilation**

Run: `npm run build`
Expected: Clean compilation with no errors

- [ ] **Step 3: Commit**

```bash
git add src/index.ts
git commit -m "feat: add tool definitions for automation scripts, jobs, and activities"
```

---

### Task 3: Add routing cases in `routeToolCall()`

**Files:**
- Modify: `src/index.ts:1208-1211` (insert new cases before the `default:` case in the switch statement)

- [ ] **Step 1: Add routing cases with input validation**

Insert the following cases before `default:` (line 1210):

```typescript
      // Automation Scripts
      case 'get_automation_scripts':
        return this.api.getAutomationScripts(args.lang);
      case 'get_device_scripting_options':
        return this.api.getDeviceScriptingOptions(args.id, args.lang);
      case 'run_script_on_device': {
        if (args.type !== 'ACTION' && args.type !== 'SCRIPT') {
          throw new McpError(ErrorCode.InvalidParams, 'type must be ACTION or SCRIPT');
        }
        if (args.type === 'SCRIPT' && !args.scriptId) {
          throw new McpError(ErrorCode.InvalidParams, 'scriptId is required when type is SCRIPT');
        }
        if (args.type === 'ACTION' && !args.actionUid) {
          throw new McpError(ErrorCode.InvalidParams, 'actionUid is required when type is ACTION');
        }
        return this.api.runScriptOnDevice(
          args.id,
          args.type,
          args.scriptId,
          args.actionUid,
          args.parameters,
          args.runAs
        );
      }

      // Jobs
      case 'get_active_jobs':
        return this.api.getActiveJobs(args.jobType, args.df, args.lang, args.tz);
      case 'get_device_active_jobs':
        return this.api.getDeviceActiveJobs(args.id, args.lang, args.tz);

      // Activities
      case 'get_activities':
        return this.api.getActivities({
          class: args.class,
          before: args.before,
          after: args.after,
          olderThan: args.olderThan,
          newerThan: args.newerThan,
          type: args.type,
          status: args.status,
          user: args.user,
          seriesUid: args.seriesUid,
          df: args.df,
          pageSize: args.pageSize,
          lang: args.lang,
          tz: args.tz,
          sourceConfigUid: args.sourceConfigUid
        });
```

- [ ] **Step 2: Build and verify compilation**

Run: `npm run build`
Expected: Clean compilation with no errors

- [ ] **Step 3: Commit**

```bash
git add src/index.ts
git commit -m "feat: add routing for automation scripts, jobs, and activities tools"
```

---

### Task 4: Update CLAUDE.md and verify

**Files:**
- Modify: `CLAUDE.md` (update tool count and add new category to tool table)

- [ ] **Step 1: Update CLAUDE.md**

Update the tool count from 74 to 80 throughout the file. Update approximate line counts (`index.ts` from ~1327 to ~1430, `ninja-api.ts` from ~661 to ~695). Add these rows to the tool categories table:

```markdown
| Automation Scripts | 3 | `get_automation_scripts`, `get_device_scripting_options`, `run_script_on_device` |
| Job Tracking | 2 | `get_active_jobs`, `get_device_active_jobs` |
| Activity Tracking | 1 | `get_activities` |
```

- [ ] **Step 2: Full build and manual verification**

Run: `npm run build`
Expected: Clean compilation

Verify tool count by searching for tool names in the compiled output:
Run: `grep -c "name: '" dist/index.js`
Expected: 80

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with new automation scripts and job tracking tools"
```

---

### Task 5: Push and create PR

- [ ] **Step 1: Push feature branch**

```bash
git push -u origin feature/automation-scripts
```

- [ ] **Step 2: Create pull request**

```bash
gh pr create --title "Add automation scripts, job tracking, and activity tools" --body "$(cat <<'EOF'
## Summary
- Add 6 new MCP tools for automation script management and execution monitoring
- `get_automation_scripts` - List available scripts with variable definitions
- `get_device_scripting_options` - Get scripts/actions/credentials available for a device
- `run_script_on_device` - Execute scripts or built-in actions with safety confirmation
- `get_active_jobs` / `get_device_active_jobs` - Track running jobs
- `get_activities` - Query activity log with filtering by series UID, type, etc.

## Safety
- `run_script_on_device` tool description includes explicit LLM instruction to confirm with user before executing
- Input validation ensures correct type/scriptId/actionUid combinations

## Test plan
- [ ] Build compiles cleanly (`npm run build`)
- [ ] Verify 80 tools registered (was 74)
- [ ] Test `get_automation_scripts` returns script list
- [ ] Test `get_device_scripting_options` with a known device ID
- [ ] Test `run_script_on_device` with a safe read-only script
- [ ] Test `get_active_jobs` returns job list
- [ ] Test `get_activities` with type=SCRIPTING filter

## Spec
See `docs/superpowers/specs/2026-03-18-automation-scripts-design.md`

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```
