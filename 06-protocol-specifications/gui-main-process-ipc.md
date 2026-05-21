# Main Process IPC Handlers: GUI Architecture Notes

## Overview

This report catalogs ALL `ipcMain.handle` and `ipcMain.on` channel registrations in the Electron main process bundle `main-process.js` (main process bundle). The main IPC handler registration occurs in the `LEt()` function (IPC registration region–242), which registers **15 `ipcMain.handle` channels** for request/response communication with the renderer. Additionally, the ErrTracker Electron SDK registers **5 `ipcMain.on` channels** (with 1 conditional) in the `Ont()` function (ErrTracker handler region) for fire-and-forget error reporting. The main process also pushes events to the renderer via `webContents.send()` on **6 channels** (navigate, app:error, auth:success, mcp-oauth-callback, app:reportBug, window:fullscreenChange). Total IPC channel count: **27 channels** (15 handle + 6 ErrTracker on + 6 renderer push), matching the 27 channels exposed in the preload bridge.

## File Map

| File | Size | Role |
|------|------|------|
| `build/main-process.js` | main process bundle (~1.4M bytes) | Electron main process: contains all ipcMain registrations |
| `build/preload.js` | small preload bridge | IPC bridge: contextBridge.exposeInMainWorld (cross-reference) |
| `build/session-search.js` | small helper chunk | Lazy-loaded session search module (used by session:searchSessions) |

## Architecture

### IPC Registration Topology

The IPC surface has three distinct layers:

```
Renderer Process                          Main Process
─────────────────                         ──────────────
window.electronAPI.*  ──invoke──>  ipcMain.handle (15 channels)  ──> Handler functions
                          │                                              ├── Auth (signIn, signOut, getAuthData, switchToOrganization)
                          │                                              ├── Dialog (selectDirectory)
                          │                                              ├── Notification (show, setBadgeCount)
                          │                                              ├── Session (readFile, searchSessions)
                          │                                              ├── Computer (getLocalId, getRemoteAccess, setRemoteAccess)
                          │                                              ├── Window (isFullscreen)
                          │                                              ├── App (getVersion)
                          │                                              └── Bug Report (getLogs)

window.electronAPI.*  <──on─────  webContents.send (6 channels)   <──  Event sources
                          │                                              ├── Deep link router (navigate, mcp-oauth-callback)
                          │                                              ├── Auth flow (auth:success)
                          │                                              ├── Error handler (app:error)
                          │                                              ├── Bug report trigger (app:reportBug)
                          │                                              └── Fullscreen events (window:fullscreenChange)

window.__ERRTRACKER_IPC__ ──send───>  ipcMain.on (5-6 channels)       ──> ErrTracker processing
                                                                           ├── errtracker-ipc.start
                                                                           ├── errtracker-ipc.scope
                                                                           ├── errtracker-ipc.envelope
                                                                           ├── errtracker-ipc.structured-log
                                                                           ├── errtracker-ipc.metric
                                                                           └── errtracker-ipc.status (conditional)
```

### Handler Registration Function: LEt()

The `LEt()` function (IPC registration region–242) is the single point of registration for all 15 `ipcMain.handle` channels. It is called during the main process bootstrap (`BEt()`) at the "ipc_and_protocol_setup" phase, after `app.whenReady()` and before window creation.

### ErrTracker Handler Registration Function: Ont()

The `Ont()` function (ErrTracker handler region) registers `ipcMain.on` handlers for ErrTracker IPC. It is called during ErrTracker SDK initialization (`Int()`) which occurs during the `Hee()` (ErrTracker init) call at module load time.

## Key Findings

### Finding 1: Complete 1:1 Preload-to-Main Channel Mapping

Every `ipcMain.handle` channel has a corresponding `ipcRenderer.invoke` call in preload.js, and vice versa. The 15 handle channels and 6 push channels map exactly to the 21 `electronAPI` methods exposed in the preload bridge. There are **zero orphaned channels**: no main-only invoke handlers without preload callers, and no preload invoke methods without main handlers.

### Finding 2: Auth Flow is Multi-Step with Both Handle and Push

The authentication system uses a hybrid IPC pattern:
- **Handle channels**: `auth:signIn` (initiate), `auth:getAuthData` (get current state), `auth:signOut` (logout), `auth:switchToOrganization` (switch org)
- **Push channel**: `auth:success` (pushed when OAuth flow completes via deep link)

The `auth:signIn` handler calls `rEt()` to initiate an OAuth flow via IdProvider. When the browser callback returns a deep link, the main process processes it in `sEt()` → `nEt()`, which then pushes `auth:success` to the renderer via `webContents.send("auth:success", ...)`.

### Finding 3: Session Search Uses Dynamic Module Loading

The `session:searchSessions` handler lazy-loads `session-search.js` via dynamic `require()`:
```js
const {runDroidSearch} = await Promise.resolve().then(() => require("./session-search.js"));
```
This is the only IPC handler that dynamically loads an external module. The search module (small helper chunk) is loaded on-demand when the first search request arrives.

### Finding 4: Bug Report Handler Reads and Tails Log Files

The `bugReport:getLogs` handler is the most complex handler: it:
1. Reads log files from two directories (OPENDROID_HOME logs dir + app.getPath("logs"))
2. For small logs (desktop-startups.log), reads entire file
3. For large logs (daemon-stderr.log, droid-log-single.log, console.log, desktop-droid.log), tails last 512KB
4. Returns a map of `{filename: content}` strings

### Finding 5: Notification Click Routes to Session

The `notification:show` handler creates a native notification with click handling that:
1. Restores the window (unminimize, show, focus)
2. If `sessionId` is provided in the notification payload, navigates to `/sessions/{sessionId}` after 100ms delay
3. Uses the `navigate` push channel for the routing

### Finding 6: No ipcMain.on for Application Channels

All application-level IPC uses `ipcMain.handle` (request/response). There are **zero** `ipcMain.on` registrations for application channels: the only `ipcMain.on` registrations are for ErrTracker's fire-and-forget messages. This confirms the architecture is fully request/response for app communication, with push events going renderer-bound only (main→renderer via `webContents.send`).

### Finding 7: ErrTracker ipcMain.on Handlers Support Both Protocol and Classic Modes

The ErrTracker IPC system (`Ont()` function) registers handlers for both `ipcMain.on` (Classic mode) AND custom protocol handlers (`re.protocol.registerSchemesAsPrivileged` + `protocol.handle` for Electron ≥25). The mode is controlled by `gl.Both` (value 3), meaning both paths are active simultaneously.

## Code Examples

### Main Process IPC Handler Summary

```text
registerMainProcessHandlers():
  app:getVersion -> return app version
  auth:signIn -> start OAuth flow, return success/error
  auth:getAuthData -> read token + profile, return auth state
  auth:switchToOrganization -> switch active organization
  auth:signOut -> stop daemon, clear auth state, clear error context
  dialog:selectDirectory -> open native directory picker
  notification:show -> show native notification and navigate on click
  notification:setBadgeCount -> set dock/taskbar badge count
  session:readFile -> read session transcript/settings
  session:searchSessions -> lazy-load search helper and query sessions
  computer:getLocalId -> return machine identifier
  computer:getRemoteAccessEnabled -> read preference
  computer:setRemoteAccessEnabled -> persist preference
  window:isFullscreen -> return current window fullscreen state
  bugReport:getLogs -> collect recent diagnostic logs
```

### ErrTracker ipcMain.on Registration Summary

```text
registerErrorReporterIpc(namespace):
  {ns}.start -> track renderer sender lifecycle
  {ns}.scope -> apply renderer error-reporting scope
  {ns}.envelope -> forward error/performance envelope
  {ns}.structured-log -> forward structured log payload
  {ns}.metric -> forward metric payload
  {ns}.status -> optional status reporting callback
```
### Renderer Push: webContents.send Calls

Push events are sent from various locations in the main process (not consolidated in one function):

```js
// Navigate push (deep link routing)
c.webContents.send("navigate", { path: `/sessions/${i}` });

// Error push (daemon crash, etc.)
i.webContents.send("app:error", { error: e });

// Auth success push (OAuth callback)
webContents.send("auth:success", { /* auth data */ });

// MCP OAuth callback push
webContents.send("mcp-oauth-callback", { code, state });

// Bug report trigger push
webContents.send("app:reportBug");

// Fullscreen change push (macOS only)
St.webContents.send("window:fullscreenChange", true/false);
```

## Theme / Style Tokens

Not applicable for this feature: this is the main process IPC handler analysis. Theme tokens are in `theme.css` (covered in gui-theme-css.md).

## IPC Channels

### ipcMain.handle Channels (15 channels: Request/Response, Renderer→Main→Renderer)

| # | Channel Name | Handler Region | Request Payload | Response Payload | Preload Method | Purpose |
|---|-------------|----------------|-----------------|------------------|----------------|---------|
| 1 | `app:getVersion` | IPC handlers | none | `string` (semver) | `electronAPI.getAppVersion()` | Returns app version |
| 2 | `auth:signIn` | IPC handlers | none | `{success: boolean, error?: string}` | `electronAPI.auth.signIn()` | Initiates IdProvider OAuth flow |
| 3 | `auth:getAuthData` | IPC handlers | none | `{user?: object, accessToken?: string}` | `electronAPI.auth.getAuthData()` | Returns current auth state |
| 4 | `auth:switchToOrganization` | IPC handlers | `{organizationId: string}` | `{success: boolean, error?: string}` | `electronAPI.auth.switchToOrganization(org)` | Switches org context |
| 5 | `auth:signOut` | IPC handlers | none | `{success: boolean, error?: string}` | `electronAPI.auth.signOut()` | Stops daemon, clears auth, signs out |
| 6 | `dialog:selectDirectory` | IPC handlers | none | `{canceled: boolean, path?: string}` | `electronAPI.selectDirectory()` | Opens native directory picker |
| 7 | `notification:show` | IPC handlers | `{title: string, body: string, sessionId?: string}` | void | `electronAPI.notification.show(notif)` | Shows native notification |
| 8 | `notification:setBadgeCount` | IPC handlers | `{count: number}` | void | `electronAPI.notification.setBadgeCount(count)` | Sets dock/taskbar badge |
| 9 | `session:readFile` | IPC handlers | `{sessionId: string}` | `{messages, metadata, settings}` or `{error}` | `electronAPI.readSessionFile(fileRef)` | Reads session JSONL file |
| 10 | `session:searchSessions` | IPC handlers | `{query: string}` | `{query, sessions[]}` or `{error}` | `electronAPI.searchSessions(criteria)` | Searches sessions (lazy-loads session-search.js) |
| 11 | `computer:getLocalId` | IPC handlers | none | `string` (computer ID) | `electronAPI.getLocalComputerId()` | Gets local computer identifier |
| 12 | `computer:getRemoteAccessEnabled` | IPC handlers | none | `boolean` | `electronAPI.getRemoteAccessEnabled()` | Gets remote access preference |
| 13 | `computer:setRemoteAccessEnabled` | IPC handlers | `{enabled: boolean}` | void | `electronAPI.setRemoteAccessEnabled(enabled)` | Sets remote access preference |
| 14 | `window:isFullscreen` | IPC handlers | none | `boolean` | `electronAPI.window.isFullscreen()` | Returns fullscreen state |
| 15 | `bugReport:getLogs` | IPC handlers | none | `{[filename: string]: string}` | `electronAPI.getBugReportLogs()` | Collects log files for bug report |

### ipcMain.on Channels (5–6 channels: Fire-and-Forget, Renderer→Main, ErrTracker Only)

| # | Channel Name | Line | Payload | Purpose | Preload Method |
|---|-------------|------|---------|---------|----------------|
| S1 | `errtracker-ipc.start` | 182 | none | Registers renderer sender ID for session tracking | `__ERRTRACKER_IPC__["errtracker-ipc"].sendRendererStart()` |
| S2 | `errtracker-ipc.scope` | 182 | `scope object` (serialized) | Updates ErrTracker scope from renderer | `__ERRTRACKER_IPC__["errtracker-ipc"].sendScope(scope)` |
| S3 | `errtracker-ipc.envelope` | 182 | `envelope data` (serialized) | Sends error/performance envelope to ErrTracker | `__ERRTRACKER_IPC__["errtracker-ipc"].sendEnvelope(envelope)` |
| S4 | `errtracker-ipc.structured-log` | 182 | `log data` (serialized) | Sends structured log entry via ErrTracker | `__ERRTRACKER_IPC__["errtracker-ipc"].sendStructuredLog(log)` |
| S5 | `errtracker-ipc.metric` | 182 | `metric data` (serialized) | Sends metric data via ErrTracker | `__ERRTRACKER_IPC__["errtracker-ipc"].sendMetric(metric)` |
| S6 | `errtracker-ipc.status` | 182 | `status data` (serialized) | Sends status update (conditional on `vee(e)` check) | `__ERRTRACKER_IPC__["errtracker-ipc"].sendStatus(status)` |

### Main→Renderer Push Channels (6 channels: webContents.send)

| # | Channel Name | Trigger Location | Payload | Direction | Preload Listener |
|---|-------------|-----------------|---------|-----------|-----------------|
| P1 | `navigate` | Deep link router, notification click | `{path: string}` | M→R | `electronAPI.onNavigate(cb)` |
| P2 | `app:error` | Daemon crash, spawn failure, startup error | `{error: string}` | M→R | `electronAPI.onAppError(cb)` |
| P3 | `auth:success` | OAuth deep link callback completion | auth data object | M→R | `electronAPI.auth.onAuthSuccess(cb)` |
| P4 | `mcp-oauth-callback` | MCP server OAuth deep link | `{code, state}` | M→R | `electronAPI.onMcpOAuthCallback(cb)` |
| P5 | `app:reportBug` | Report bug trigger | none (signal only) | M→R | `electronAPI.onReportBug(cb)` |
| P6 | `window:fullscreenChange` | macOS fullscreen enter/exit | `boolean` | M→R | `electronAPI.window.onFullscreenChange(cb)` |

## Integration Points

### Cross-System Dependencies

1. **Preload Bridge (gui-preload-bridge.md)**: Every `ipcMain.handle` channel maps 1:1 to a `ipcRenderer.invoke` call in preload.js. Every `ipcMain.on` ErrTracker channel maps to a `ipcRenderer.send` call. Every `webContents.send` push channel maps to an `ipcRenderer.on` listener in preload. Cross-reference confirms **zero orphaned channels** in either direction.

2. **Main Process Core (gui-main-process-core.md)**: The `LEt()` function is called during the `BEt()` bootstrap at the "ipc_and_protocol_setup" phase (IPC handlers region). The `app:getVersion` handler uses `re.app.getVersion()`. The `auth:signOut` handler calls `kEt()` (stop daemon) and `tEt()` (clear ErrTracker user). The `notification:show` handler creates native notifications with tray-icon.png.

3. **Session Search Module (gui-main-process-helpers.md)**: The `session:searchSessions` handler dynamically loads `session-search.js` via `require()`. The loaded module exports `runDroidSearch(query)` which performs the actual search. This is the only lazy-loaded module in the IPC surface.

4. **Daemon Manager (gui-main-process-daemon.md)**: The `auth:signOut` handler stops the daemon via `kEt()` before signing out. The `bugReport:getLogs` handler reads daemon log files. Daemon errors trigger `app:error` push to renderer.

5. **Auth Flow (IdProvider)**: The `auth:signIn` handler calls `rEt()` to open browser for IdProvider OAuth. The callback returns via deep link (`opendroid://` protocol), processed by `sEt()` → `nEt()`, which pushes `auth:success` to renderer.

6. **ErrTracker Integration**: The 5–6 `ipcMain.on` channels are registered by the ErrTracker Electron SDK's `Ont()` function. They handle scope updates, error envelopes, structured logs, and metrics from the renderer process. The `errtracker-ipc.start` channel also tracks renderer webContents IDs for session management.

### Cross-Mission References

- **Auth channels** → Connect to CLI/TUI auth flow (Terminal UI section, Tool & Agent section scope)
- **Computer ID / Remote Access** → Connects to infrastructure (Infrastructure section scope)
- **Daemon WebSocket** → Connects to opendroid.exe (Infrastructure section scope)
- **Session search** → May use daemon API (Infrastructure section scope)
- These are documented as boundary points; analysis of the other side is not in 04-desktop-gui scope.

### Preload ↔ Main Channel Count Comparison

| Category | Preload Count | Main Count | Match |
|----------|--------------|------------|-------|
| `ipcMain.handle` ↔ `ipcRenderer.invoke` | 15 | 15 | ✅ 1:1 |
| `ipcMain.on` ↔ `ipcRenderer.send` (ErrTracker) | 6 | 5–6 (status conditional) | ✅ |
| `webContents.send` ↔ `ipcRenderer.on` (push) | 6 | 6 | ✅ 1:1 |
| **Total** | **27** | **26–27** | ✅ Match |

The only uncertainty is `errtracker-ipc.status` which is conditionally registered based on `vee(e)`: a runtime check for whether status reporting is enabled in the ErrTracker client configuration. This channel may or may not have a registered handler depending on configuration.

### Main-Only Channels (Not Exposed via Preload)

**None (application-level).** All `ipcMain.handle` channels have corresponding preload `ipcRenderer.invoke` calls. All `ipcMain.on` channels have corresponding preload `ipcRenderer.send` calls. All `webContents.send` push channels have corresponding preload `ipcRenderer.on` listeners. There are zero main-only application channels that lack preload exposure.

**Note:** There is one additional internal `ipcMain.on` channel from the `electron-store` library (not exposed via preload's `contextBridge`):
- `electron-store-get-data` (DaemonManager region): Synchronous IPC (`sendSync`/`returnValue` pattern) used by electron-store to provide config paths to renderer-side store instances. This is third-party infrastructure, not part of the application IPC surface.

## Implementation Notes

### Porting the IPC Handler System

1. **Handler registration pattern**: The `LEt()` function pattern: a single function that registers all handlers at bootstrap: is clean and auditable. Port this as a `registerIpcHandlers()` function called during app initialization after `app.whenReady()`.

2. **Channel name convention**: The `domain:action` pattern (`auth:signIn`, `session:readFile`, etc.) is well-structured. Adopt this convention for OpenDroid. The colon separator clearly separates the domain from the action.

3. **Payload destructuring**: Handlers receive payloads as the second argument to the async callback. For `invoke` handlers, the first arg is the `IpcMainInvokeEvent`, the second is the payload. For `on` handlers, the first arg is the `IpcMainEvent`, the second is the data. Ensure OpenDroid follows this convention.

4. **Error handling pattern**: Most handlers return `{success: boolean, error?: string}` on failure. This is a good pattern: the renderer can check `success` and display `error` if needed. Port this pattern consistently (some handlers like `notification:show` silently fail, which should be made consistent).

5. **Dynamic module loading**: The `session:searchSessions` handler uses `await require("./session-search.js")` to lazy-load the search module. This is a good pattern for optional/heavy features. Consider using this for expensive operations in OpenDroid.

6. **Log file collection**: The `bugReport:getLogs` pattern of tailing large files (512KB max) is useful for diagnostic features. Port this with configurable file paths for OpenDroid's log structure.

7. **Notification click routing**: The notification click→window restore→navigate pattern is valuable for session-aware notifications. Port this for OpenDroid's notification system.

8. **ErrTracker integration**: The entire ErrTracker IPC subsystem can be replaced with OpenDroid's error reporting (or removed entirely). The `errtracker-ipc.*` channels and `__ERRTRACKER_IPC__` preload namespace are independent of the application IPC and can be cleanly separated.

9. **Session file reading**: The JSONL parsing with validation and metadata extraction is domain-specific but the pattern of reading structured files with error recovery is reusable. Adapt for OpenDroid's session format.

10. **Remote access configuration**: The electron-store-based preference storage (`computer-remote-access` store with zod validation) is a good pattern for persistent settings. Port with OpenDroid's validation schema.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| main-process.js | ErrTracker handler region | `Ont(e, t, n)` | ErrTracker ipcMain.on handler registration: 5–6 channels |
| main-process.js | ErrTracker handler region | `kee(e, t)` | ErrTracker scope handler: applies renderer scope data |
| main-process.js | ErrTracker handler region | `Uee(e, t, n, r)` | ErrTracker envelope handler: processes error/perf data |
| main-process.js | ErrTracker handler region | `Fee(e, t, n, r)` | ErrTracker structured log handler |
| main-process.js | ErrTracker handler region | `Gee(e, t, n, r)` | ErrTracker metric handler |
| main-process.js | IPC registration region | `LEt()` | Main IPC handler registration: all 15 ipcMain.handle channels |
| main-process.js | IPC handlers region | `rEt()` | Auth: initiate OAuth flow (called by auth:signIn handler) |
| main-process.js | IPC handlers region | `jLe(r)` | Auth: switch organization (called by auth:switchToOrganization) |
| main-process.js | IPC handlers region | `kEt()` | Daemon: stop daemon (called by auth:signOut handler) |
| main-process.js | IPC handlers region | `tDe()` | Auth: clear auth tokens (called by auth:signOut handler) |
| main-process.js | IPC handlers region | `tEt()` | ErrTracker: clear user context (called by auth:signOut handler) |
| main-process.js | IPC handlers region | `z9()` | Auth: get access token |
| main-process.js | IPC handlers region | `X9()` | Auth: get user profile |
| main-process.js | IPC handlers region | `$Et(r)` | Session: read JSONL file handler implementation |
| main-process.js | IPC handlers region | `MEt(r)` | Session: search sessions handler implementation |
| main-process.js | IPC handlers region | `vot()` | Computer: get local ID |
| main-process.js | IPC handlers region | `bEt()` | Computer: get remote access enabled from store |
| main-process.js | IPC handlers region | `NEt(r)` | Computer: set remote access enabled in store |
| main-process.js | IPC handlers region | `$c()` | Window: get current BrowserWindow reference |
| main-process.js | IPC handlers region | `XT(e)` | Error: send error to renderer via app:error push channel |
| main-process.js | IPC handlers region | `vS(e)` | Deep link router: parses URL and routes to handler |
