# NinjaOne MCP Server

## Project Overview

An MCP (Model Context Protocol) server that wraps the NinjaOne RMM platform API, exposing 80 tools for device management, monitoring, patching, alerting, and more. Built with TypeScript, supports STDIO/HTTP/SSE transports.

- **Origin**: Third-party open-source repo (MIT license)
- **Runtime**: Node.js (ES modules)
- **MCP SDK**: `@modelcontextprotocol/sdk` v1.17.1
- **NinjaOne API version**: v2

## Quick Reference

```bash
npm run build        # Compile TypeScript
npm run dev          # Watch mode
npm start            # Run server (STDIO mode)
npm run start:http   # HTTP transport on port 3000
npm run start:sse    # SSE transport on port 3001
npm test             # Run basic connectivity tests (requires API credentials)
```

## Architecture

```
src/
  index.ts            # MCP server class, 80 tool definitions, tool routing (~1430 lines)
  ninja-api.ts        # NinjaOneAPI client: auth, HTTP requests, all API methods (~695 lines)
  transport/
    http.ts           # Express-based HTTP and SSE transport servers
```

### Key Classes

- **`NinjaOneMCPServer`** (`src/index.ts`) - MCP server orchestrator. Holds the `TOOLS` array, registers MCP request handlers, routes tool calls via `routeToolCall()`.
- **`NinjaOneAPI`** (`src/ninja-api.ts`) - Stateful API client. Manages OAuth2 tokens, regional endpoint detection, and all NinjaOne API calls.

### Request Flow

```
MCP Client → Transport (STDIO/HTTP/SSE) → Server.CallToolRequestSchema handler
  → routeToolCall(name, args) → NinjaOneAPI method → NinjaOne REST API v2
  → JSON response → MCP tool result
```

### Transport Modes

Selected via `MCP_MODE` env var (default: `stdio`):
- **stdio** - Standard for Claude Desktop / Claude Code integration
- **http** - REST endpoints on `HTTP_PORT` (default 3000): `/health`, `/info`, `/tools`
- **sse** - Server-Sent Events on `SSE_PORT` (default 3001): `/events` with 30s heartbeat

## Configuration

### Required Environment Variables

| Variable | Description |
|----------|-------------|
| `NINJA_CLIENT_ID` | OAuth2 client ID from NinjaOne |
| `NINJA_CLIENT_SECRET` | OAuth2 client secret from NinjaOne |

### Optional Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NINJA_BASE_URL` | (auto-detect) | Explicit API base URL |
| `NINJA_REGION` | (auto-detect) | Region key: `us`, `us2`, `eu`, `ca`, `oc` |
| `NINJA_BASE_URLS` | (all regions) | Comma-separated fallback URLs for auto-detect |
| `MCP_MODE` | `stdio` | Transport: `stdio`, `http`, `sse` |
| `HTTP_PORT` | `3000` | HTTP transport port |
| `SSE_PORT` | `3001` | SSE transport port |
| `LOG_LEVEL` | `info` | Logging level |
| `CORS_ORIGIN` | `*` | Allowed CORS origins |

### Region Auto-Detection

If neither `NINJA_BASE_URL` nor `NINJA_REGION` is set, the client tries all known regional endpoints in sequence until one succeeds with the provided credentials.

## Authentication

- **OAuth2 Client Credentials** flow
- Token endpoint: `{baseUrl}/ws/oauth/token`
- Scope: `monitoring management control`
- Tokens are cached with a 5-minute refresh buffer (re-authenticates 5 min before expiry)

## Tool Categories (80 tools)

| Category | Count | Examples |
|----------|-------|---------|
| Device Management | 5 | `get_devices`, `get_device`, `reboot_device`, `set_device_maintenance` |
| Device Search | 2 | `search_devices_by_name`, `find_windows11_devices` |
| Windows Services | 2 | `control_windows_service`, `configure_windows_service` |
| Patch Management | 4 | `scan_device_os_patches`, `apply_device_os_patches` |
| Organization Management | 7 | `get_organizations`, `create_organization`, `update_organization` |
| Location Management | 2 | `create_location`, `update_location` |
| Alert Management | 4 | `get_alerts`, `get_alert`, `reset_alert`, `get_device_alerts` |
| User Management | 6 | `get_end_users`, `create_end_user`, `get_technicians` |
| Role Management | 2 | `add_role_members`, `remove_role_members` |
| Contact Management | 5 | `get_contacts`, `create_contact`, `delete_contact` |
| Device Activities/Software | 2 | `get_device_activities`, `get_device_software` |
| Policy Management | 4 | `get_policies`, `get_device_policy_overrides`, `reset_device_policy_overrides` |
| System Info Queries | 6 | `query_antivirus_status`, `query_device_health`, `query_operating_systems` |
| Hardware Queries | 6 | `query_processors`, `query_disks`, `query_volumes`, `query_network_interfaces` |
| Software/Patch Queries | 6 | `query_software`, `query_os_patches`, `query_windows_services` |
| Custom Field Queries | 5 | `query_custom_fields`, `query_scoped_custom_fields` |
| Backup Queries | 1 | `query_backup_usage` |
| Automation Scripts | 3 | `get_automation_scripts`, `get_device_scripting_options`, `run_script_on_device` |
| Job Tracking | 2 | `get_active_jobs`, `get_device_active_jobs` |
| Activity Tracking | 1 | `get_activities` |
| Region Utilities | 2 | `list_regions`, `set_region` |

## API Limitations & Gotchas

- **Organization/Location DELETE**: Not available via API; must use NinjaOne dashboard.
- **Patch approval/rejection**: Only via dashboard or policies; no public API endpoint.
- **`nodeApprovalMode`**: Read-only after organization creation; `updateOrganization` intentionally ignores it.
- **End User `phone` field**: Read-only after creation; update attempts are silently ignored by the API.
- **`updateOrganization` endpoint**: Has a fallback from `/v2/organizations/{id}` to `/v2/organization/{id}` on 404 (API inconsistency workaround).
- **`search_devices_by_name`**: Client-side filtering - fetches up to 200 devices then filters locally by name.
- **`find_windows11_devices`**: Client-side - fetches devices, filters Windows nodes, then calls `getDevice()` individually (up to 50) to check OS version. Can be slow.
- **Maintenance mode timestamps**: Uses Unix epoch in **seconds** (not milliseconds). Start time is offset +5 seconds to avoid API processing delays.
- **Script output**: Full script stdout/stderr is only viewable in the NinjaOne dashboard. The API provides job status and activity results but not raw output.
- **`run_script_on_device`**: The tool description includes an LLM safety instruction requiring user confirmation before execution.
- **Query endpoints** (`/v2/queries/*`): All support `df` (device filter), `cursor` (pagination), and `pageSize` parameters.

## Development Guidelines

### Adding a New Tool

1. Add tool definition to the `TOOLS` array in `src/index.ts` with `name`, `description`, and `inputSchema`
2. Add the corresponding API method in `src/ninja-api.ts` (follow existing patterns)
3. Add the routing case in `routeToolCall()` in `src/index.ts`
4. Run `npm run build` to verify compilation

### Code Patterns

- All API methods return `Promise<any>` - responses are JSON-stringified in tool results
- Error handling: `routeToolCall()` wraps all calls in try/catch, converting to `McpError`
- Tool responses always use format: `{ content: [{ type: "text", text: JSON.stringify(result, null, 2) }] }`
- Use `buildQuery()` for URL query parameters, `pruneUndefined()` for request bodies
- TypeScript strict mode is enabled with additional checks (`exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`)

### Build

- TypeScript compiles `src/` to `dist/` (ES2022 target, ESNext modules)
- `noEmitOnError: true` - build fails on any type error
- Declaration files and source maps are generated

## File Reference

| File | Purpose |
|------|---------|
| `src/index.ts` | Server class, tool definitions, tool routing |
| `src/ninja-api.ts` | NinjaOne API client with OAuth2 and all endpoints |
| `src/transport/http.ts` | HTTP and SSE Express servers |
| `src/test.ts` | Basic connectivity test suite |
| `manifest.json` | MCPB distribution manifest |
| `mcp-config.json` | Claude Desktop config template |
| `server.json` | NPM/MCP registry metadata |
| `.env.example` | Environment variable template |
| `TOOLS.md` | Full tool documentation |
| `SETUP.md` | Setup and configuration guide |
