# Protocol: IPC & Daemon Communication

## Status: PARTIAL: Heavy RE coverage, minimal runtime confirmation

> **⚠️ GAP WARNING:** This protocol area has the MOST architecture notes data and the LEAST observed protocol samples of all 7 areas. IPC is a GUI-layer concern (Electron main↔renderer + daemon WebSocket) that leaves no direct traces in `~/.opendroid/` file artifacts. Runtime confirmation is indirect only (file existence implies IPC usage). Of the 27+ IPC channels and 35+ daemon RPC methods documented from RE, only ~6 have any indirect runtime confirmation. The remainder are analysis-only.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Electron IPC Channel Catalog](#electron-ipc-channel-catalog)
3. [Preload Bridge Mapping](#preload-bridge-mapping)
4. [Renderer IPC Client](#renderer-ipc-client)
5. [Daemon Communication Protocol](#daemon-communication-protocol)
6. [Discrepancies (Inferred vs Observed)](#discrepancies-re-vs-runtime)
7. [Gap Analysis](#gap-analysis)
8. [Open Questions](#open-questions)

---

## Architecture Overview

OpenDroid's IPC system has **three distinct communication layers**:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Renderer Process (React)                     │
│  window.electronAPI.*  (21 methods)                              │
│  window.__ERRTRACKER_IPC__ (6 methods)                               │
└──────────┬──────────────────┬────────────────────────────────────┘
           │ ipcRenderer       │ ipcRenderer
           │ .invoke/.on/.send │ .send (ErrTracker)
           ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Preload Bridge (preload.js)                   │
│  contextBridge.exposeInMainWorld("electronAPI", ...)             │
│  contextBridge.exposeInMainWorld("__ERRTRACKER_IPC__", ...)          │
└──────────┬──────────────────┬────────────────────────────────────┘
           │                   │
           ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                  Main Process (Electron)                          │
│  ipcMain.handle (15 channels)  ← request/response                │
│  ipcMain.on    (6 channels)    ← ErrTracker fire-and-forget          │
│  webContents.send (6 channels) ← push to renderer                │
│                                                                  │
│  Also manages:                                                   │
│  ├── Daemon process (child_process.spawn → opendroidd)             │
│  ├── Auth flow (IdProvider OAuth → deep link)                        │
│  ├── Native dialogs & notifications                              │
│  └── Log file collection                                         │
└──────────┬───────────────────────────────────────────────────────┘
           │ WebSocket (JSON-RPC 2.0)
           ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Daemon Process (opendroidd)                     │
│  Bun.serve() on localhost:31415                                   │
│  ├── JSON-RPC control (35+ methods)                              │
│  ├── Code-server proxy (/code-server/*)                          │
│  ├── SSH tunnel relay (/tunnel)                                  │
│  └── Session notification pub/sub                                │
└──────────────────────────────────────────────────────────────────┘
```

### Communication Pattern Summary

| Layer | Pattern | Mechanism | Channel Count |
|-------|---------|-----------|---------------|
| Renderer → Main (request) | Request/Response | `ipcRenderer.invoke` ↔ `ipcMain.handle` | 15 |
| Renderer → Main (ErrTracker) | Fire-and-Forget | `ipcRenderer.send` → `ipcMain.on` | 5–6 |
| Main → Renderer (push) | Event Push | `webContents.send` → `ipcRenderer.on` | 6 |
| GUI ↔ Daemon | JSON-RPC 2.0 | WebSocket on localhost | 35+ methods |
| CLI ↔ Daemon | JSON-RPC 2.0 | WebSocket on localhost | Same 35+ methods |

---

## Electron IPC Channel Catalog

### 1. ipcMain.handle Channels (15 channels: Request/Response)

All registered in `LEt()` function during bootstrap at "ipc_and_protocol_setup" phase.

| # | Channel Name | Direction | Request Payload | Response Payload | Runtime Confirmed? | Purpose |
|---|-------------|-----------|----------------|-----------------|-------------------|---------|
| 1 | `app:getVersion` | R→M→R | none | `string` (semver) | ❌ analysis-only | Returns app version |
| 2 | `auth:signIn` | R→M→R | none | `{success: boolean, error?: string}` | ❌ analysis-only | Initiates IdProvider OAuth flow |
| 3 | `auth:getAuthData` | R→M→R | none | `{user?: AuthUser, accessToken?: string}` | ❌ analysis-only | Returns current auth state |
| 4 | `auth:switchToOrganization` | R→M→R | `{organizationId: string}` | `{success: boolean, error?: string}` | ❌ analysis-only | Switches org context |
| 5 | `auth:signOut` | R→M→R | none | `{success: boolean, error?: string}` | ❌ analysis-only | Stops daemon, clears auth, signs out |
| 6 | `dialog:selectDirectory` | R→M→R | none | `{canceled: boolean, path?: string}` | ❌ analysis-only | Opens native directory picker |
| 7 | `notification:show` | R→M→R | `{title, body, sessionId?}` | void | ❌ analysis-only | Shows native notification with click routing |
| 8 | `notification:setBadgeCount` | R→M→R | `{count: number}` | void | ❌ analysis-only | Sets dock/taskbar badge |
| 9 | `session:readFile` | R→M→R | `{sessionId: string}` | `{messages, metadata, settings}` or `{error}` | ⚠️ Indirect | Reads session JSONL file |
| 10 | `session:searchSessions` | R→M→R | `{query: string}` | `{query, sessions[]}` or `{error}` | ⚠️ Indirect | Searches sessions (lazy-loads module) |
| 11 | `computer:getLocalId` | R→M→R | none | `string` (computer ID) | ⚠️ Indirect | Gets local computer identifier |
| 12 | `computer:getRemoteAccessEnabled` | R→M→R | none | `boolean` | ❌ analysis-only | Gets remote access preference |
| 13 | `computer:setRemoteAccessEnabled` | R→M→R | `{enabled: boolean}` | void | ❌ analysis-only | Sets remote access preference |
| 14 | `window:isFullscreen` | R→M→R | none | `boolean` | ❌ analysis-only | Returns fullscreen state |
| 15 | `bugReport:getLogs` | R→M→R | none | `{[filename: string]: string}` | ⚠️ Indirect | Collects log files for bug report |

### 2. ipcMain.on Channels (5–6 channels: ErrTracker Fire-and-Forget)

Registered by ErrTracker Electron SDK in `Ont()` function.

| # | Channel Name | Direction | Payload | Runtime Confirmed? | Purpose |
|---|-------------|-----------|---------|-------------------|---------|
| S1 | `errtracker-ipc.start` | R→M | none | ❌ analysis-only | Registers renderer sender ID |
| S2 | `errtracker-ipc.scope` | R→M | `scope object` (serialized) | ❌ analysis-only | Updates ErrTracker scope |
| S3 | `errtracker-ipc.envelope` | R→M | `envelope data` (serialized) | ❌ analysis-only | Sends error/perf envelope |
| S4 | `errtracker-ipc.structured-log` | R→M | `log data` (serialized) | ❌ analysis-only | Sends structured log |
| S5 | `errtracker-ipc.metric` | R→M | `metric data` (serialized) | ❌ analysis-only | Sends metric data |
| S6 | `errtracker-ipc.status` | R→M | `status data` (serialized) | ❌ analysis-only | Status update (conditional) |

> **Note:** S6 (`errtracker-ipc.status`) is conditionally registered based on `vee(e)` runtime check. May not always have a handler.

### 3. webContents.send Push Channels (6 channels: Main→Renderer)

| # | Channel Name | Direction | Payload | Runtime Confirmed? | Trigger |
|---|-------------|-----------|---------|-------------------|---------|
| P1 | `navigate` | M→R | `{path: string}` | ❌ analysis-only | Deep link routing, notification click |
| P2 | `app:error` | M→R | `{error: string}` | ❌ analysis-only | Daemon crash, spawn failure |
| P3 | `auth:success` | M→R | auth data object | ❌ analysis-only | OAuth deep link callback |
| P4 | `mcp-oauth-callback` | M→R | `{code, state}` | ❌ analysis-only | MCP server OAuth |
| P5 | `app:reportBug` | M→R | none (signal only) | ❌ analysis-only | Report bug trigger |
| P6 | `window:fullscreenChange` | M→R | `boolean` | ❌ analysis-only | macOS fullscreen change |

### Channel Naming Convention

All application channels follow `domain:action` pattern with colon separator:
- **Domains:** `auth`, `dialog`, `notification`, `session`, `computer`, `window`, `app`, `bugReport`
- **Exceptions:** `navigate` (no separator), `mcp-oauth-callback` (hyphens)
- **ErrTracker:** `errtracker-ipc.{action}` with dot separator (separate namespace)

---

## Preload Bridge Mapping

### window.electronAPI → ipcRenderer Mapping

The preload bridge (`build/preload.js`, small preload bridge) exposes **21 methods** via `contextBridge.exposeInMainWorld("electronAPI", {...})`:

| # | Preload Method | ipcRenderer Call | Channel Name | Returns |
|---|---------------|-----------------|--------------|---------|
| 1 | `auth.signIn()` | `.invoke` | `auth:signIn` | `Promise<{success, error?}>` |
| 2 | `auth.signOut()` | `.invoke` | `auth:signOut` | `Promise<{success, error?}>` |
| 3 | `auth.getAuthData()` | `.invoke` | `auth:getAuthData` | `Promise<{user?, accessToken?}>` |
| 4 | `auth.onAuthSuccess(cb)` | `.on` | `auth:success` | `() => void` (unsubscribe) |
| 5 | `auth.switchToOrganization(org)` | `.invoke` | `auth:switchToOrganization` | `Promise<{success, error?}>` |
| 6 | `onNavigate(cb)` | `.on` | `navigate` | `() => void` (unsubscribe) |
| 7 | `onAppError(cb)` | `.on` | `app:error` | `() => void` (unsubscribe) |
| 8 | `onMcpOAuthCallback(cb)` | `.on` | `mcp-oauth-callback` | `() => void` (unsubscribe) |
| 9 | `onReportBug(cb)` | `.on` | `app:reportBug` | `() => void` (unsubscribe) |
| 10 | `selectDirectory()` | `.invoke` | `dialog:selectDirectory` | `Promise<{canceled, path?}>` |
| 11 | `getAppVersion()` | `.invoke` | `app:getVersion` | `Promise<string>` |
| 12 | `getLocalComputerId()` | `.invoke` | `computer:getLocalId` | `Promise<string>` |
| 13 | `getRemoteAccessEnabled()` | `.invoke` | `computer:getRemoteAccessEnabled` | `Promise<boolean>` |
| 14 | `setRemoteAccessEnabled(enabled)` | `.invoke` | `computer:setRemoteAccessEnabled` | `Promise<void>` |
| 15 | `notification.show(notif)` | `.invoke` | `notification:show` | `Promise<void>` |
| 16 | `notification.setBadgeCount(count)` | `.invoke` | `notification:setBadgeCount` | `Promise<void>` |
| 17 | `readSessionFile(fileRef)` | `.invoke` | `session:readFile` | `Promise<SessionData>` |
| 18 | `searchSessions(criteria)` | `.invoke` | `session:searchSessions` | `Promise<SearchResult>` |
| 19 | `window.isFullscreen()` | `.invoke` | `window:isFullscreen` | `Promise<boolean>` |
| 20 | `window.onFullscreenChange(cb)` | `.on` | `window:fullscreenChange` | `() => void` (unsubscribe) |
| 21 | `getBugReportLogs()` | `.invoke` | `bugReport:getLogs` | `Promise<LogMap>` |

### window.__ERRTRACKER_IPC__ → ipcRenderer Mapping

| # | Method | ipcRenderer Call | Channel Name |
|---|--------|-----------------|--------------|
| S1 | `["errtracker-ipc"].sendRendererStart()` | `.send` | `errtracker-ipc.start` |
| S2 | `["errtracker-ipc"].sendScope(scope)` | `.send` | `errtracker-ipc.scope` |
| S3 | `["errtracker-ipc"].sendEnvelope(envelope)` | `.send` | `errtracker-ipc.envelope` |
| S4 | `["errtracker-ipc"].sendStatus(status)` | `.send` | `errtracker-ipc.status` |
| S5 | `["errtracker-ipc"].sendStructuredLog(log)` | `.send` | `errtracker-ipc.structured-log` |
| S6 | `["errtracker-ipc"].sendMetric(metric)` | `.send` | `errtracker-ipc.metric` |

### Listener Cleanup Pattern

All `.on()` listeners follow this pattern: returns unsubscribe function:
```typescript
type Listener<T> = (callback: (data: T) => void) => () => void;

// Implementation:
onEventName: (callback) => {
  const handler = (event, data) => callback(data); // strips Electron event
  ipcRenderer.on("channel:name", handler);
  return () => ipcRenderer.removeListener("channel:name", handler);
}
```

### Payload Wrapping Inconsistency

| Method | Param | How Sent | Notes |
|--------|-------|----------|-------|
| `setRemoteAccessEnabled(enabled)` | boolean | `{enabled: enabled}` | Wrapped in object |
| `notification.setBadgeCount(count)` | number | `{count: count}` | Wrapped in object |
| `auth.switchToOrganization(org)` | object | `org` directly | NOT wrapped |
| `notification.show(notif)` | object | `notif` directly | NOT wrapped |
| `readSessionFile(fileRef)` | any | `fileRef` directly | NOT wrapped |
| `searchSessions(criteria)` | any | `criteria` directly | NOT wrapped |

> ⚠️ **Inconsistency:** Single primitive params are wrapped, but object params are passed directly. OpenDroid should standardize this.

### Channel Count Verification

| Category | Preload | Main Process | Match |
|----------|---------|-------------|-------|
| `ipcMain.handle` ↔ `ipcRenderer.invoke` | 15 | 15 | ✅ 1:1 |
| `ipcMain.on` ↔ `ipcRenderer.send` (ErrTracker) | 6 | 5–6 | ✅ (status conditional) |
| `webContents.send` ↔ `ipcRenderer.on` (push) | 6 | 6 | ✅ 1:1 |
| **Total** | **27** | **26–27** | ✅ Match |

---

## Renderer IPC Client

### Call Sites

The renderer (`renderer-main.js`, large renderer bundle) calls **8 of 21** electronAPI methods directly in the main bundle. The remaining 13 are in lazy-loaded route chunks.

#### Directly Called Methods (8/21 = 38%)

| # | Method | Channel | Call Site | Pattern |
|---|--------|---------|-----------|---------|
| 1 | `auth.getAuthData()` | `auth:getAuthData` | `ivo()` init + onAuthSuccess + getAccessToken | One-shot invoke (×3 sites) |
| 2 | `auth.signIn()` | `auth:signIn` | `ivo()` signIn closure | One-shot, user-triggered |
| 3 | `auth.signOut()` | `auth:signOut` | `ivo()` signOut closure | One-shot, user-triggered |
| 4 | `auth.switchToOrganization(org)` | `auth:switchToOrganization` | `ivo()` switchToOrganization closure | One-shot, user-triggered |
| 5 | `auth.onAuthSuccess(cb)` | `auth:success` | `ivo()` useEffect | Streaming listener |
| 6 | `onNavigate(cb)` | `navigate` | `ivo()` useEffect | Streaming listener |
| 7 | `getAppVersion()` | `app:getVersion` | `MBs()` (lazy cached) | One-shot, singleton cached |
| 8 | `onAppError(cb)` | `app:error` | Likely secondary chunks | Streaming listener |

#### Methods in Secondary Chunks (13/21 = 62%)

| # | Method | Channel | Hypothesized Location |
|---|--------|---------|----------------------|
| 1 | `onMcpOAuthCallback(cb)` | `mcp-oauth-callback` | MCP OAuth route component |
| 2 | `onReportBug(cb)` | `app:reportBug` | Bug report modal |
| 3 | `selectDirectory()` | `dialog:selectDirectory` | Settings/repository config |
| 4 | `getLocalComputerId()` | `computer:getLocalId` | Droid computers settings |
| 5 | `getRemoteAccessEnabled()` | `computer:getRemoteAccessEnabled` | Remote access settings |
| 6 | `setRemoteAccessEnabled(e)` | `computer:setRemoteAccessEnabled` | Remote access settings |
| 7 | `notification.show(n)` | `notification:show` | Notification service |
| 8 | `notification.setBadgeCount(c)` | `notification:setBadgeCount` | Dock badge handler |
| 9 | `readSessionFile(ref)` | `session:readFile` | Session viewer component |
| 10 | `searchSessions(criteria)` | `session:searchSessions` | Session search component |
| 11 | `window.isFullscreen()` | `window:isFullscreen` | Window controls |
| 12 | `window.onFullscreenChange(cb)` | `window:fullscreenChange` | Window controls |
| 13 | `getBugReportLogs()` | `bugReport:getLogs` | Bug report modal |

### Auth Token Flow

The desktop auth flow has a specific token propagation pattern:

```
1. User clicks Sign In
2. Renderer → electronAPI.auth.signIn() → ipcRenderer.invoke("auth:signIn")
3. Main process → rEt() → opens browser for IdProvider OAuth
4. Browser OAuth → deep link callback → opendroid://auth/...
5. Main process → sEt() → nEt() → processes OAuth result
6. Main process → webContents.send("auth:success", authData)
7. Renderer → onAuthSuccess callback → electronAPI.auth.getAuthData()
8. Renderer → gets accessToken from response
9. API client → attaches Authorization: Bearer <token> header to all fetch() calls
```

### Platform Detection

The renderer supports dual platform (web + desktop) via `vu()` helper:
- Desktop: `window.electronAPI` required, throws custom error if missing
- Web: `window.electronAPI` undefined, uses cookie-based auth as fallback
- Conditional: `window.electronAPI?.getAppVersion()` with optional chaining

---

## Daemon Communication Protocol

### Network Configuration

| Config | Value |
|--------|-------|
| Default Host | `localhost` |
| Production Port | `31415` |
| Development Port | `41832` |
| Computer Port | `8080` |
| Ping Interval | `30000ms` (30s) |
| Protocol | JSON-RPC 2.0 over WebSocket |
| Runtime | Bun.serve() |

### JSON-RPC 2.0 Message Format

#### Request (Client → Daemon)

```typescript
{
  type: "request",           // Discriminator
  jsonrpc: "2.0",            // JSON-RPC version
  opendroidApiVersion: string, // OpenDroid API version
  id: string,                // UUID v4 for correlation
  method: "daemon.*",        // RPC method name
  params: { ... },           // Method-specific parameters
  _meta?: {
    traceparent?: string,    // OpenTelemetry trace propagation
    tracestate?: string
  }
}
```

#### Response (Daemon → Client)

```typescript
{
  type: "response",
  jsonrpc: "2.0",
  opendroidApiVersion: string,
  id: string,                // Matches request ID
  result?: { ... },          // On success
  error?: {                  // On failure
    code: number,            // -32600 to -32603 (JSON-RPC) or -32001 (auth)
    message: string,
    data?: any
  }
}
```

#### Notification (Daemon → Client, Unsolicited)

```typescript
{
  jsonrpc: "2.0",
  opendroidApiVersion: string,
  type: "notification",
  method: "daemon.session_notification",
  params: {
    sessionId: string,
    notification: { ... }    // Session events, terminal data, permissions
  }
}
```

#### Connection Status Notification

```typescript
{
  type: "notification",
  method: "daemon.connection_status",
  params: {
    isDroidCLIInPath: boolean,
    droidCLIVersion: string,
    homedir: string,
    platform: string
  }
}
```

### Connection Lifecycle

```
1. CONNECT: Client opens WebSocket to ws://localhost:{port}
2. AUTHENTICATE: Client sends daemon.authenticate with JWT or API key
   ├── Token verified via w1A() (JWT/ApiKey validation)
   ├── On success: ConnectionManager caches AuthenticatedConnection
   └── On failure: Error -32001 returned
3. REQUEST/RESPONSE: JSON-RPC request/response with UUID correlation
   └── Pending requests tracked in Map<id, {resolve, reject}>
4. NOTIFICATIONS: Daemon pushes session events to subscribed clients
   ├── Session notifications (stream chunks, tool results)
   ├── Terminal data/exit events
   └── Permission/ask_user prompts requiring GUI response
5. PING/PONG: Server sends ping every 30s, client responds with pong
6. CLOSE: WebSocket closed (code 1000 = normal, 1001 = going away)
   ├── Terminals scheduled for cleanup with grace period
   └── Droid sessions scheduled for cleanup
```

### Authentication

Dual authentication support:
- **JWT token**: For browser/GUI clients
- **API key**: For CLI clients
- Both verified via `w1A()` which calls the OpenDroid API

### Daemon RPC Method Catalog (35+ methods)

#### Session Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `daemon.initialize_session` | Client→Daemon | Create new droid session |
| `daemon.load_session` | Client→Daemon | Load existing session |
| `daemon.add_user_message` | Client→Daemon | Add user message to session |
| `daemon.interrupt_session` | Client→Daemon | Interrupt running session |
| `daemon.kill_worker_session` | Client→Daemon | Kill worker session |
| `daemon.list_opened_sessions` | Client→Daemon | List active sessions |
| `daemon.list_available_sessions` | Client→Daemon | List all available sessions |
| `daemon.update_session_settings` | Client→Daemon | Update session config |
| `daemon.search_sessions` | Client→Daemon | Search session history |
| `daemon.archive_session` | Client→Daemon | Archive a session |
| `daemon.unarchive_session` | Client→Daemon | Unarchive a session |

#### MCP Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `daemon.toggle_mcp_server` | Client→Daemon | Enable/disable MCP server |
| `daemon.authenticate_mcp_server` | Client→Daemon | Start MCP OAuth flow |
| `daemon.cancel_mcp_auth` | Client→Daemon | Cancel MCP auth flow |
| `daemon.clear_mcp_auth` | Client→Daemon | Clear MCP auth tokens |
| `daemon.add_mcp_server` | Client→Daemon | Add MCP server config |
| `daemon.remove_mcp_server` | Client→Daemon | Remove MCP server |
| `daemon.list_mcp_registry` | Client→Daemon | List available MCP servers |
| `daemon.list_mcp_tools` | Client→Daemon | List MCP tools |
| `daemon.toggle_mcp_tool` | Client→Daemon | Enable/disable MCP tool |

#### Terminal Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `daemon.create_terminal` | Client→Daemon | Create PTY terminal |
| `daemon.write_terminal_data` | Client→Daemon | Write to terminal stdin |
| `daemon.resize_terminal` | Client→Daemon | Resize terminal |
| `daemon.close_terminal` | Client→Daemon | Close terminal |
| `daemon.list_terminals` | Client→Daemon | List active terminals |

#### File Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `daemon.list_files` | Client→Daemon | List files in directory |
| `daemon.search_files` | Client→Daemon | Search file contents |

#### Automation Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `daemon.list_automations` | Client→Daemon | List automations |
| `daemon.run_automation` | Client→Daemon | Execute automation |
| `daemon.pause_automation` | Client→Daemon | Pause running automation |
| `daemon.resume_automation` | Client→Daemon | Resume paused automation |
| `daemon.get_automation_history` | Client→Daemon | Get automation run history |
| `daemon.get_automation_visual` | Client→Daemon | Get automation visual |
| `daemon.create_automation` | Client→Daemon | Create new automation |
| `daemon.update_automation_model` | Client→Daemon | Update automation model |
| `daemon.rename_automation` | Client→Daemon | Rename automation |
| `daemon.delete_automation` | Client→Daemon | Delete automation |

#### Git Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `daemon.get_git_diff` | Client→Daemon | Get git diff |
| `daemon.git_push` | Client→Daemon | Push to remote |
| `daemon.create_pr` | Client→Daemon | Create pull request |

#### Other Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `daemon.validate_working_directory` | Client→Daemon | Validate working dir |
| `daemon.get_default_settings` | Client→Daemon | Get default settings |
| `daemon.start_code_server` | Client→Daemon | Start code-server instance |
| `daemon.submit_bug_report` | Client→Daemon | Submit bug report |
| `daemon.get_semantic_diff` | Client→Daemon | Get semantic diff |
| `daemon.save_semantic_diff` | Client→Daemon | Save semantic diff |
| `daemon.generate_semantic_diff` | Client→Daemon | Generate semantic diff |
| `daemon.save_semantic_diff_cache` | Client→Daemon | Save diff cache |

### Daemon Auto-Spawn Protocol

When a client needs the daemon but it's not running:

```
1. TCP probe: Check if daemon port is reachable
2. If not running:
   a. Spawn `droid daemon --droid-path <path> [--debug] --port <port>`
   b. Detached child process with stdio: "ignore"
   c. CWD: user homedir
3. Poll port readiness with exponential backoff
4. Return true when daemon is ready
```

### Single-Port Multiplexing

All traffic flows through one port, demultiplexed by URL path:
- `/` → JSON-RPC control channel (WebSocket)
- `/code-server/*` → Code-server HTTP + WS proxy
- `/tunnel` or `?tunnel=<port>` → SSH tunnel relay

---

## Discrepancies (Inferred vs Observed)

| # | Area | Architecture Notes Says | Observed Samples Show | Resolution |
|---|------|-----------------|-------------------|------------|
| 1 | **IPC traceability** | 27 channels documented from static analysis notes | No IPC traces in observed protocol samples (`~/.opendroid/` is file-based, no IPC logs) | IPC is GUI-layer only; observed protocol samples cannot confirm. Mark all channels as analysis-only except indirect inferences. |
| 2 | **computer:getLocalId** | Returns string (computer ID) | `computer.json` exists with `computerId: "9eb6be95-..."`: confirms ID exists | ⚠️ Indirect confirmation: computer.json file implies this IPC channel is functional |
| 3 | **session:readFile** | Reads session JSONL files | Session JSONL files exist in `~/.opendroid/sessions/` with documented format | ⚠️ Indirect confirmation: session files exist and must be read somehow |
| 4 | **session:searchSessions** | Searches sessions with lazy-loaded module | `sessions-index.json` exists with 52 sessions: implies search capability | ⚠️ Indirect confirmation: search infrastructure exists |
| 5 | **bugReport:getLogs** | Reads and tails daemon log files | Observed protocol samples mentions `daemon-stderr.log`, `droid-log-single.log` in bug report context | ⚠️ Indirect confirmation: log files exist and would be collected |
| 6 | **Daemon process** | Spawned as `opendroidd` via child_process.spawn | `daemon-stderr.log` and `daemon-startups.log` referenced in architecture notes | ⚠️ Indirect confirmation: daemon log files exist |
| 7 | **Auth flow** | IdProvider OAuth with deep link callback | No runtime traces (auth tokens in memory, not persisted to files) | Cannot confirm from observed protocol samples. analysis-only. |
| 8 | **Notification system** | Native notifications with session click routing | No runtime traces | Cannot confirm from observed protocol samples. analysis-only. |
| 9 | **IPC channel count** | Exactly 27 channels (15 handle + 6 on + 6 push) | N/A | RE is definitive for count: preload.js is small preload bridge and fully analyzed |

---

## Gap Analysis

### Channels with Runtime Confirmation

| Channel | Confirmation Type | Evidence |
|---------|------------------|----------|
| `session:readFile` | Indirect | Session JSONL files exist in observed protocol samples with documented format |
| `session:searchSessions` | Indirect | `sessions-index.json` exists with 52 entries |
| `computer:getLocalId` | Indirect | `computer.json` exists with `computerId` field |
| `bugReport:getLogs` | Indirect | Log files referenced in RE (`daemon-stderr.log`, etc.) |
| Daemon WebSocket | Indirect | Daemon log files exist in observed protocol samples |

### Channels with NO Runtime Confirmation (RE-Only)

**All 15 ipcMain.handle channels**: No runtime IPC traces possible (in-memory only)

**All 6 webContents.send push channels**: No runtime event traces possible

**All 5–6 ErrTracker channels**: No runtime traces (ErrTracker is fire-and-forget)

**All 35+ daemon RPC methods**: No runtime traces of JSON-RPC messages (WebSocket frames are transient)

### Why Observed Protocol Samples Cannot Confirm IPC

| Reason | Explanation |
|--------|-------------|
| IPC is in-memory | Electron IPC messages (`ipcRenderer.invoke`, `ipcMain.handle`, `webContents.send`) exist only in process memory: they leave no file artifacts |
| WebSocket is transient | JSON-RPC messages over WebSocket are ephemeral network frames: not logged to disk |
| Observed protocol samples is file-based | `~/.opendroid/` contains file artifacts (JSON, JSONL, Markdown): it does NOT contain IPC traces, WebSocket frames, or network logs |
| Auth tokens in memory | Authentication tokens from `auth:getAuthData` are used in-memory for API calls, not persisted to files |
| Daemon logs are separate | The daemon's own logs (`daemon-stderr.log`) are about daemon lifecycle, not individual IPC messages |

### What CAN Be Inferred from Observed Protocol Samples

1. **Session system works** → `session:readFile` and `session:searchSessions` must be functional
2. **Computer registration works** → `computer:getLocalId` must be functional (computer.json exists)
3. **Daemon runs** → Auto-spawn and WebSocket must work (daemon log files exist)
4. **Auth works** → Sessions have `providerLock` and tokens are used for API calls
5. **Config loading works** → Settings and config files are read/written correctly

---

## Open Questions

1. **[INFERRED]** What is the exact `opendroidApiVersion` value sent in JSON-RPC messages? RE shows the field exists but the actual version string is dynamic.
2. **[INFERRED]** Does the `errtracker-ipc.status` channel actually get registered in production? It's conditional on `vee(e)`.
3. **[INFERRED]** What happens when the daemon is unreachable? The `app:error` push channel is documented but the exact error handling flow is unclear.
4. **[INFERRED]** The `session:readFile` response schema `{messages, metadata, settings}`: how does this map to the runtime session JSONL format? The RE shows the handler calls `$Et(r)` but the exact parsing logic wasn't fully deobfuscated.
5. **[INFERRED]** How does the MCP OAuth flow work end-to-end? `mcp-oauth-callback` push channel exists but the full flow needs more analysis.
6. **[INFERRED]** What is the `decompSessionType: "worker"` parameter for worker sessions in `daemon.initialize_session`? This connects to the mission system but the exact parameters are unknown.
7. **[INFERRED]** The daemon's `DroidRequestHandler` is 2197 lines: the largest single module. How many total RPC methods does it handle? RE found 35+ but the exact count is uncertain due to minification.

---

## Summary Statistics

| Metric | Count | Source |
|--------|-------|--------|
| Electron IPC channels (total) | 27 | RE: preload.js + main process |
| ipcMain.handle channels | 15 | RE: LEt() function |
| ipcMain.on channels (ErrTracker) | 5–6 | RE: Ont() function |
| webContents.send push channels | 6 | RE: scattered in main process |
| Preload methods exposed | 21 | RE: preload.js |
| ErrTracker preload methods | 6 | RE: preload.js |
| Renderer direct call sites | 8/21 | RE: renderer-main.js |
| Daemon RPC methods | 35+ | RE: 3754.js + 3755.js |
| Terminal RPC methods | 5 | RE: VWL handler |
| Channels with runtime confirmation | ~5 | Indirect inference only |
| Channels analysis-only | ~57 | No runtime traces possible |

---

## Implementation Notes

### Electron IPC

1. **Handler registration**: Use a single `registerIpcHandlers()` function called after `app.whenReady()`: mirrors the `LEt()` pattern
2. **Channel naming**: Adopt `domain:action` convention with colon separator
3. **Error handling**: Standardize on `{success: boolean, error?: string}` for all handlers
4. **Listener cleanup**: Return unsubscribe functions from all `.on()` listeners
5. **Payload wrapping**: Standardize: either always wrap single params or never wrap
6. **Context isolation**: Always use `contextBridge.exposeInMainWorld`: never leak `require('electron')`

### Daemon Protocol

1. **Replace Bun.serve()** with Node.js `http` + `ws` or `fastify-websocket`
2. **Preserve JSON-RPC 2.0**: the message format is transport-agnostic
3. **Implement daemon process manager**: auto-spawn with port probe + exponential backoff
4. **Authentication adapter**: replace OpenDroid API auth with target platform's auth
5. **Terminal multiplexer**: replace Bun PTY APIs with `node-pty`
6. **Single-port multiplexing**: keep the URL-path-based demultiplexing pattern

### Critical Invariants

1. Every `ipcMain.handle` channel must have a 1:1 match in preload's `ipcRenderer.invoke`
2. Every `webContents.send` push must have a corresponding `ipcRenderer.on` listener in preload
3. Daemon WebSocket requires authentication before any RPC method is accepted
4. Session notifications are pushed to ALL authenticated connections (fan-out)
5. Terminal cleanup has a grace period after WebSocket disconnect (allows reconnection)
