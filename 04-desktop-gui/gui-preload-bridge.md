# Preload IPC Bridge: GUI Architecture Notes

## Overview

The `build/preload.js` file (small preload bridge) is the **single source of truth** for the Electron IPC bridge between the renderer process and the main process. It exposes two namespaces to the renderer via `contextBridge.exposeInMainWorld`: `window.electronAPI` (21 methods) for application functionality, and `window.__ERRTRACKER_IPC__` (6 methods) for error reporting via ErrTracker. All IPC calls follow strict patterns: `ipcRenderer.invoke` for request/response (15 methods), `ipcRenderer.on` for event listeners with cleanup functions (6 methods), and `ipcRenderer.send` for fire-and-forget ErrTracker messages (6 methods). The preload uses `contextIsolation: true` semantics: no direct `require('electron')` access is leaked to the renderer.

## File Map

| File | Size | Role |
|------|------|------|
| `build/preload.js` | small preload bridge | IPC bridge: contextBridge.exposeInMainWorld exposing electronAPI and __ERRTRACKER_IPC__ |

## Architecture

### Namespace Hierarchy

```
window.__ERRTRACKER_IPC__                          (ErrTracker error reporting bridge)
  └── ["errtracker-ipc"]                           (namespace key)
        ├── sendRendererStart()                → ipcRenderer.send
        ├── sendScope(scope)                   → ipcRenderer.send
        ├── sendEnvelope(envelope)             → ipcRenderer.send
        ├── sendStatus(status)                 → ipcRenderer.send
        ├── sendStructuredLog(log)             → ipcRenderer.send
        └── sendMetric(metric)                 → ipcRenderer.send

window.electronAPI                             (Primary application IPC bridge)
  ├── auth                                     (sub-namespace: 5 methods)
  │     ├── signIn()                           → ipcRenderer.invoke
  │     ├── signOut()                          → ipcRenderer.invoke
  │     ├── getAuthData()                      → ipcRenderer.invoke
  │     ├── onAuthSuccess(callback)            → ipcRenderer.on (listener)
  │     └── switchToOrganization(org)          → ipcRenderer.invoke
  ├── onNavigate(callback)                     → ipcRenderer.on (listener)
  ├── onAppError(callback)                     → ipcRenderer.on (listener)
  ├── onMcpOAuthCallback(callback)             → ipcRenderer.on (listener)
  ├── onReportBug(callback)                    → ipcRenderer.on (listener)
  ├── selectDirectory()                        → ipcRenderer.invoke
  ├── getAppVersion()                          → ipcRenderer.invoke
  ├── getLocalComputerId()                     → ipcRenderer.invoke
  ├── getRemoteAccessEnabled()                 → ipcRenderer.invoke
  ├── setRemoteAccessEnabled(enabled)          → ipcRenderer.invoke
  ├── notification                             (sub-namespace: 2 methods)
  │     ├── show(notification)                 → ipcRenderer.invoke
  │     └── setBadgeCount(count)               → ipcRenderer.invoke
  ├── readSessionFile(fileRef)                 → ipcRenderer.invoke
  ├── searchSessions(criteria)                 → ipcRenderer.invoke
  ├── window                                   (sub-namespace: 2 methods)
  │     ├── isFullscreen()                     → ipcRenderer.invoke
  │     └── onFullscreenChange(callback)       → ipcRenderer.on (listener)
  └── getBugReportLogs()                       → ipcRenderer.invoke
```

### IPC Call Pattern Summary

| Pattern | ipcRenderer Method | Direction | Return Type | Count |
|---------|-------------------|-----------|-------------|-------|
| Request/Response | `.invoke(channel, ...args)` | Renderer → Main → Renderer | Promise (resolved by ipcMain.handle) | 15 |
| Event Listener | `.on(channel, callback)` | Main → Renderer (push) | Unsubscribe function `() => void` | 6 |
| Fire-and-Forget | `.send(channel, ...args)` | Renderer → Main | void (no response) | 6 (ErrTracker) |

### Listener Cleanup Pattern

All `.on()` listeners follow the same cleanup pattern:
```js
// Generic pattern for all 6 event listeners:
onEventName: (callback) => {
  const handler = (event, data) => callback(data);
  ipcRenderer.on("channel:name", handler);
  return () => ipcRenderer.removeListener("channel:name", handler);
}
```

Each listener method wraps the user callback to strip the Electron `event` object (first arg) and pass only the `data` payload. The returned unsubscribe function removes the exact handler reference.

## Key Findings

### Finding 1: Namespace is `electronAPI`, not `api`
The exposed namespace is **`window.electronAPI`** (via `contextBridge.exposeInMainWorld("electronAPI", {...})`), not the commonly assumed `window.api`. This is a critical detail for cross-referencing with the renderer bundle: search for `window.electronAPI` or `electronAPI.` in the renderer code.

### Finding 2: Structured Sub-Namespaces
The API is organized into logical sub-namespaces:
- **`auth`**: Authentication (signIn, signOut, getAuthData, onAuthSuccess, switchToOrganization)
- **`notification`**: Desktop notifications (show, setBadgeCount)
- **`window`**: Window state (isFullscreen, onFullscreenChange)

Top-level methods cover: navigation, errors, MCP OAuth, directory selection, app info, computer identity, remote access, sessions, and bug reporting.

### Finding 3: ErrTracker IPC Bridge (Separate Namespace)
A complete ErrTracker Electron integration is present, using `window.__ERRTRACKER_IPC__` with the namespace key `"errtracker-ipc"`. This uses `ipcRenderer.send` (fire-and-forget) rather than `invoke`, meaning the main process handles these messages via `ipcMain.on` without returning responses. The ErrTracker bridge also has its own `contextBridge.exposeInMainWorld("__ERRTRACKER_IPC__", ...)` call separate from the main electronAPI exposure.

### Finding 4: Payload Wrapping Convention
Some methods wrap single parameters into objects before sending:
- `setRemoteAccessEnabled(enabled)` → sends `{enabled: enabled}` 
- `notification.setBadgeCount(count)` → sends `{count: count}`

Other methods pass parameters directly:
- `auth.switchToOrganization(org)` → passes `org` directly
- `readSessionFile(fileRef)` → passes `fileRef` directly
- `searchSessions(criteria)` → passes `criteria` directly
- `notification.show(notification)` → passes `notification` directly

### Finding 5: Channel Naming Convention
All channel names follow a `domain:action` pattern using colon separator:
- `auth:signIn`, `auth:signOut`, `auth:getAuthData`, `auth:success`, `auth:switchToOrganization`
- `dialog:selectDirectory`
- `app:getVersion`, `app:error`, `app:reportBug`
- `computer:getLocalId`, `computer:getRemoteAccessEnabled`, `computer:setRemoteAccessEnabled`
- `notification:show`, `notification:setBadgeCount`
- `session:readFile`, `session:searchSessions`
- `window:isFullscreen`, `window:fullscreenChange`
- `mcp-oauth-callback` (exception: uses hyphens, not colon)
- `navigate` (exception: no separator, single word)

ErrTracker channels use dot separator: `errtracker-ipc.start`, `errtracker-ipc.scope`, etc.

### Finding 6: Internal Enum (Not Exposed)
A ErrTracker IPC mode enum is defined internally:
```js
var o;
(function(e) {
  e[e.Classic = 1] = "Classic";
  e[e.Protocol = 2] = "Protocol";
  e[e.Both = 3] = "Both";
})(o || (o = {}));
```
This enum is **not exposed** to the renderer: it's used internally by the ErrTracker helper functions.

## Code Examples

### preload.js Architecture Summary (~3KB minified)

The preload script follows a two-phase initialization pattern:

**Phase 1: ErrTracker IPC Bridge:**
```
helper(namespace) → creates URL/key builder for IPC channel naming
init(namespace)   → checks if already initialized, creates 6 send methods
                  → exposes window.__ERRTRACKER_IPC__ via contextBridge
```
Channels: `{ns}.start`, `{ns}.scope`, `{ns}.envelope`, `{ns}.status`, `{ns}.structured-log`, `{ns}.metric`

**Phase 2: Application IPC Bridge:**
```
contextBridge.exposeInMainWorld("electronAPI", {
  auth: { signIn, signOut, getAuthData, onAuthSuccess, switchToOrganization },
  onNavigate, onAppError, onMcpOAuthCallback, onReportBug,
  selectDirectory, getAppVersion, getLocalComputerId,
  getRemoteAccessEnabled, setRemoteAccessEnabled,
  notification: { show, setBadgeCount },
  readSessionFile, searchSessions,
  window: { isFullscreen, onFullscreenChange },
  getBugReportLogs
})
```

**IPC patterns:**
- Request/response: `ipcRenderer.invoke(channel)`: returns Promise
- Event listeners: `ipcRenderer.on(channel, handler)`: returns unsubscribe function
- Fire-and-forget: `ipcRenderer.send(channel, data)`: ErrTracker only

## Theme / Style Tokens

Not applicable for this feature: preload.js contains no CSS or theme-related code.

## IPC Channels

### Complete Channel Catalog: `window.electronAPI` (21 channels)

| # | Method Path | ipcRenderer Call | Channel Name | Params | Payload Shape | Direction |
|---|------------|-----------------|--------------|--------|---------------|-----------|
| 1 | `electronAPI.auth.signIn()` | `.invoke` | `auth:signIn` | 0 | Request: none; Response: auth result | R→M→R |
| 2 | `electronAPI.auth.signOut()` | `.invoke` | `auth:signOut` | 0 | Request: none; Response: void | R→M→R |
| 3 | `electronAPI.auth.getAuthData()` | `.invoke` | `auth:getAuthData` | 0 | Request: none; Response: auth data object | R→M→R |
| 4 | `electronAPI.auth.onAuthSuccess(cb)` | `.on` | `auth:success` | cb(data) | Push: auth success data (stripped event) | M→R |
| 5 | `electronAPI.auth.switchToOrganization(org)` | `.invoke` | `auth:switchToOrganization` | 1: org | Request: org identifier; Response: result | R→M→R |
| 6 | `electronAPI.onNavigate(cb)` | `.on` | `navigate` | cb(data) | Push: navigation target (stripped event) | M→R |
| 7 | `electronAPI.onAppError(cb)` | `.on` | `app:error` | cb(data) | Push: error data (stripped event) | M→R |
| 8 | `electronAPI.onMcpOAuthCallback(cb)` | `.on` | `mcp-oauth-callback` | cb(data) | Push: OAuth callback data (stripped event) | M→R |
| 9 | `electronAPI.onReportBug(cb)` | `.on` | `app:reportBug` | cb() | Push: no payload, signal only | M→R |
| 10 | `electronAPI.selectDirectory()` | `.invoke` | `dialog:selectDirectory` | 0 | Request: none; Response: directory path | R→M→R |
| 11 | `electronAPI.getAppVersion()` | `.invoke` | `app:getVersion` | 0 | Request: none; Response: version string | R→M→R |
| 12 | `electronAPI.getLocalComputerId()` | `.invoke` | `computer:getLocalId` | 0 | Request: none; Response: computer ID | R→M→R |
| 13 | `electronAPI.getRemoteAccessEnabled()` | `.invoke` | `computer:getRemoteAccessEnabled` | 0 | Request: none; Response: boolean | R→M→R |
| 14 | `electronAPI.setRemoteAccessEnabled(enabled)` | `.invoke` | `computer:setRemoteAccessEnabled` | 1: boolean | Request: `{enabled: boolean}`; Response: result | R→M→R |
| 15 | `electronAPI.notification.show(notif)` | `.invoke` | `notification:show` | 1: object | Request: notification object; Response: result | R→M→R |
| 16 | `electronAPI.notification.setBadgeCount(count)` | `.invoke` | `notification:setBadgeCount` | 1: number | Request: `{count: number}`; Response: void | R→M→R |
| 17 | `electronAPI.readSessionFile(fileRef)` | `.invoke` | `session:readFile` | 1: fileRef | Request: file reference; Response: file content | R→M→R |
| 18 | `electronAPI.searchSessions(criteria)` | `.invoke` | `session:searchSessions` | 1: criteria | Request: search criteria; Response: session list | R→M→R |
| 19 | `electronAPI.window.isFullscreen()` | `.invoke` | `window:isFullscreen` | 0 | Request: none; Response: boolean | R→M→R |
| 20 | `electronAPI.window.onFullscreenChange(cb)` | `.on` | `window:fullscreenChange` | cb(data) | Push: fullscreen state (stripped event) | M→R |
| 21 | `electronAPI.getBugReportLogs()` | `.invoke` | `bugReport:getLogs` | 0 | Request: none; Response: logs data | R→M→R |

### ErrTracker IPC Channels: `window.__ERRTRACKER_IPC__["errtracker-ipc"]` (6 channels)

| # | Method Path | ipcRenderer Call | Channel Name | Params | Direction |
|---|------------|-----------------|--------------|--------|-----------|
| S1 | `__ERRTRACKER_IPC__["errtracker-ipc"].sendRendererStart()` | `.send` | `errtracker-ipc.start` | 0 | R→M |
| S2 | `__ERRTRACKER_IPC__["errtracker-ipc"].sendScope(scope)` | `.send` | `errtracker-ipc.scope` | 1: scope object | R→M |
| S3 | `__ERRTRACKER_IPC__["errtracker-ipc"].sendEnvelope(envelope)` | `.send` | `errtracker-ipc.envelope` | 1: envelope | R→M |
| S4 | `__ERRTRACKER_IPC__["errtracker-ipc"].sendStatus(status)` | `.send` | `errtracker-ipc.status` | 1: status | R→M |
| S5 | `__ERRTRACKER_IPC__["errtracker-ipc"].sendStructuredLog(log)` | `.send` | `errtracker-ipc.structured-log` | 1: log | R→M |
| S6 | `__ERRTRACKER_IPC__["errtracker-ipc"].sendMetric(metric)` | `.send` | `errtracker-ipc.metric` | 1: metric | R→M |

### Summary Counts

| Category | Count |
|----------|-------|
| **electronAPI invoke methods** | 15 |
| **electronAPI on methods (listeners)** | 6 |
| **electronAPI total** | **21** |
| **ErrTracker send methods** | 6 |
| **Grand total IPC channels** | **27** |

## Integration Points

### Cross-System Dependencies

1. **Main Process Handlers** (gui-main-process-ipc): The main process bundle (`build/main-process.js`, main process bundle) must contain `ipcMain.handle()` registrations for all 15 invoke channels and `ipcMain.on()` for the 6 ErrTracker send channels. Cross-reference with `gui-main-process-ipc` feature.

2. **Daemon WebSocket** (gui-main-process-daemon): Event push channels (`auth:success`, `navigate`, `app:error`, `mcp-oauth-callback`, `app:reportBug`, `window:fullscreenChange`) likely originate from or are triggered by daemon messages received via WebSocket in the main process. Cross-reference with `gui-main-process-daemon` feature.

3. **Renderer IPC Client** (gui-renderer-ipc-client): The renderer bundle (`renderer-main.js`, large renderer bundle) calls `window.electronAPI.*` methods. Every method listed here should have at least one caller in the renderer. Cross-reference with `gui-renderer-ipc-client` feature.

4. **Auth Flow**: `auth:signIn` → `auth:signOut` → `auth:getAuthData` → `auth:success` (push) → `auth:switchToOrganization` suggests a multi-step auth flow with both request/response and event-driven patterns.

5. **MCP OAuth**: The `mcp-oauth-callback` channel indicates MCP (Model Context Protocol) server OAuth flow: main process receives OAuth callback and pushes result to renderer.

6. **Session System**: `session:readFile` and `session:searchSessions` are the only session-related IPC channels, suggesting session data is managed by the main process/daemon and accessed via these two methods.

### Cross-Mission References

- **Auth channels** → may connect to CLI/TUI auth flow (Terminal UI section, Tool & Agent section scope)
- **Computer ID / Remote Access** → connects to infrastructure (Infrastructure section scope)
- **Daemon WebSocket** → connects to opendroid.exe (Infrastructure section scope)
- These are documented here as boundary points; analysis of the other side is not in 04-desktop-gui scope.

## Implementation Notes

### Porting the IPC Bridge

1. **Namespace mapping**: The current namespace is `window.electronAPI`. When porting, decide whether to keep this name (for compatibility) or rename (e.g., `window.opendroid`). If renaming, all renderer references must be updated.

2. **Channel name convention**: The `domain:action` pattern (e.g., `auth:signIn`, `session:readFile`) is clean and extensible. Adopt this convention for OpenDroid's IPC channels. The colon separator clearly distinguishes domain from action.

3. **Listener cleanup pattern**: The preload's pattern of returning unsubscribe functions is best practice. Ensure OpenDroid's preload follows the same pattern: wrapping callbacks to strip the `event` parameter and returning a cleanup function that calls `removeListener`.

4. **ErrTracker integration**: If OpenDroid uses different error reporting (or none), the entire ErrTracker IPC section can be removed. The `__ERRTRACKER_IPC__` exposure is independent of `electronAPI` and won't affect application functionality.

5. **Payload wrapping inconsistency**: Note that `setRemoteAccessEnabled` and `setBadgeCount` wrap single params in objects (`{enabled}`, `{count}`), while other single-param methods pass directly. Standardize this in OpenDroid: either always wrap or always pass directly.

6. **Missing IPC surface**: The preload exposes relatively few channels (21). For a full IDE-like application, expect to need additional channels for: file system operations, git operations, terminal/pty management, extension host communication. Design the channel namespace to accommodate future growth.

7. **Context isolation**: The preload correctly uses `contextBridge.exposeInMainWorld` without leaking `require('electron')`. Ensure OpenDroid's preload maintains this security boundary.

8. **Sub-namespace structure**: The `auth`, `notification`, `window` sub-namespaces provide good organization. Consider extending this pattern in OpenDroid with additional sub-namespaces like `git`, `terminal`, `extensions`, `workspace`.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `build/preload.js` | Line 1 | `"use strict"` | Module mode |
| `build/preload.js` | Line 1 | `n = require("electron")` | Electron import |
| `build/preload.js` | Line 2-5 | Enum `o`: Classic/Protocol/Both | ErrTracker IPC mode enum (internal, not exposed) |
| `build/preload.js` | Line 6-12 | `c(e)`: errtracker namespace helper | Creates ErrTracker IPC namespace config (createUrl, urlMatches, createKey) |
| `build/preload.js` | ErrTracker setup section | `s(e="errtracker-ipc")`: ErrTracker init | Initializes ErrTracker IPC, exposes `__ERRTRACKER_IPC__` via contextBridge |
| `build/preload.js` | ErrTracker init section | `s()` call | ErrTracker initialization executed immediately |
| `build/preload.js` | ErrTracker init section | `contextBridge.exposeInMainWorld("__ERRTRACKER_IPC__", ...)` | ErrTracker IPC bridge exposure |
| `build/preload.js` | electronAPI section | `contextBridge.exposeInMainWorld("electronAPI", {...})` | Main application IPC bridge exposure |
| `build/preload.js` | electronAPI section | `electronAPI.auth.signIn` | invoke → `auth:signIn` |
| `build/preload.js` | electronAPI section | `electronAPI.auth.signOut` | invoke → `auth:signOut` |
| `build/preload.js` | electronAPI section | `electronAPI.auth.getAuthData` | invoke → `auth:getAuthData` |
| `build/preload.js` | electronAPI section | `electronAPI.auth.onAuthSuccess` | on → `auth:success`, returns unsubscribe |
| `build/preload.js` | electronAPI section | `electronAPI.auth.switchToOrganization` | invoke → `auth:switchToOrganization`, 1 param |
| `build/preload.js` | electronAPI section | `electronAPI.onNavigate` | on → `navigate`, returns unsubscribe |
| `build/preload.js` | electronAPI section | `electronAPI.onAppError` | on → `app:error`, returns unsubscribe |
| `build/preload.js` | electronAPI section | `electronAPI.onMcpOAuthCallback` | on → `mcp-oauth-callback`, returns unsubscribe |
| `build/preload.js` | electronAPI section | `electronAPI.onReportBug` | on → `app:reportBug`, returns unsubscribe, no data |
| `build/preload.js` | electronAPI section | `electronAPI.selectDirectory` | invoke → `dialog:selectDirectory` |
| `build/preload.js` | electronAPI section | `electronAPI.getAppVersion` | invoke → `app:getVersion` |
| `build/preload.js` | electronAPI section | `electronAPI.getLocalComputerId` | invoke → `computer:getLocalId` |
| `build/preload.js` | electronAPI section | `electronAPI.getRemoteAccessEnabled` | invoke → `computer:getRemoteAccessEnabled` |
| `build/preload.js` | electronAPI section | `electronAPI.setRemoteAccessEnabled` | invoke → `computer:setRemoteAccessEnabled`, wraps `{enabled}` |
| `build/preload.js` | electronAPI section | `electronAPI.notification.show` | invoke → `notification:show`, 1 param |
| `build/preload.js` | electronAPI section | `electronAPI.notification.setBadgeCount` | invoke → `notification:setBadgeCount`, wraps `{count}` |
| `build/preload.js` | electronAPI section | `electronAPI.readSessionFile` | invoke → `session:readFile`, 1 param |
| `build/preload.js` | electronAPI section | `electronAPI.searchSessions` | invoke → `session:searchSessions`, 1 param |
| `build/preload.js` | electronAPI section | `electronAPI.window.isFullscreen` | invoke → `window:isFullscreen` |
| `build/preload.js` | electronAPI section | `electronAPI.window.onFullscreenChange` | on → `window:fullscreenChange`, returns unsubscribe |
| `build/preload.js` | electronAPI section | `electronAPI.getBugReportLogs` | invoke → `bugReport:getLogs` |
