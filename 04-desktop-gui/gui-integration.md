# IPC Topology & Cross-System Integration: GUI Architecture Notes

## Overview

This report is the **final synthesis** of 04-desktop-gui's Electron GUI architecture research, combining findings from four prior IPC-focused analyses into a unified cross-layer catalog. The OpenDroid desktop application uses a three-layer IPC architecture: (1) `preload.js` exposes 27 channels via `contextBridge.exposeInMainWorld`, (2) the main process (`main-process.js`) registers 15 `ipcMain.handle` handlers + 6 `webContents.send` push channels + 5–6 ErrTracker `ipcMain.on` handlers, and (3) the renderer (`renderer-main.js`) consumes these channels through `window.electronAPI`. In addition, the renderer communicates **directly** with the `opendroid.exe` daemon via WebSocket (bypassing the main process entirely), while the main process acts only as a daemon lifecycle manager (spawn, health-poll, restart). This report documents the complete channel inventory, cross-layer consistency checks, orphan analysis, discrepancies, and at least one concrete end-to-end daemon→main→renderer event flow.

## File Map

| File | Size | Role | Prior Report |
|------|------|------|-------------|
| `build/preload.js` | small preload bridge | IPC bridge: `contextBridge.exposeInMainWorld` | `gui-preload-bridge.md` |
| `build/main-process.js` | main process bundle | Electron main process: all `ipcMain` handlers, daemon manager | `gui-main-process-ipc.md`, `gui-main-process-daemon.md`, `gui-main-process-core.md` |
| `build/session-search.js` | small helper chunk | Lazy-loaded session search module | `gui-main-process-helpers.md` |
| `renderer/main_window/assets/renderer-main.js` | large renderer bundle | Main renderer bundle: React app, IPC client calls | `gui-renderer-ipc-client.md`, `gui-renderer-core.md` |
| `renderer/main_window/assets/renderer-chunk-a.js` | 471KB | Secondary chunk: Mammoth docx extractor | `gui-renderer-secondary-chunks.md` |
| `renderer/main_window/assets/renderer-chunk-b.js` | 92KB | Secondary chunk: Mermaid diagram renderer | `gui-renderer-secondary-chunks.md` |

## Architecture

### Complete IPC Topology (Three Layers)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RENDERER PROCESS                               │
│                        (renderer-main.js, large renderer bundle)                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  window.electronAPI  ←── IPC Bridge ──→  Main Process               │   │
│  │                                                                     │   │
│  │  ┌─ auth.signIn()        invoke ──► ipcMain.handle("auth:signIn")   │   │
│  │  ├─ auth.signOut()       invoke ──► ipcMain.handle("auth:signOut")  │   │
│  │  ├─ auth.getAuthData()   invoke ──► ipcMain.handle("auth:getAuthData")│  │
│  │  ├─ auth.switchToOrganization() invoke ──► ipcMain.handle("auth:switchToOrganization")│
│  │  ├─ selectDirectory()    invoke ──► ipcMain.handle("dialog:selectDirectory")│
│  │  ├─ getAppVersion()      invoke ──► ipcMain.handle("app:getVersion")│  │
│  │  ├─ getLocalComputerId() invoke ──► ipcMain.handle("computer:getLocalId")││
│  │  ├─ getRemoteAccessEnabled() invoke ──► ipcMain.handle("computer:getRemoteAccessEnabled")│
│  │  ├─ setRemoteAccessEnabled() invoke ──► ipcMain.handle("computer:setRemoteAccessEnabled")│
│  │  ├─ notification.show()  invoke ──► ipcMain.handle("notification:show")│ │
│  │  ├─ notification.setBadgeCount() invoke ──► ipcMain.handle("notification:setBadgeCount")│
│  │  ├─ readSessionFile()    invoke ──► ipcMain.handle("session:readFile")│ │
│  │  ├─ searchSessions()     invoke ──► ipcMain.handle("session:searchSessions")││
│  │  ├─ window.isFullscreen() invoke ──► ipcMain.handle("window:isFullscreen")││
│  │  ├─ getBugReportLogs()   invoke ──► ipcMain.handle("bugReport:getLogs")│  │
│  │  │                                                                 │   │
│  │  ├─ auth.onAuthSuccess(cb)  ◄──on── ipcMain.push "auth:success"    │   │
│  │  ├─ onNavigate(cb)          ◄──on── ipcMain.push "navigate"        │   │
│  │  ├─ onAppError(cb)          ◄──on── ipcMain.push "app:error"       │   │
│  │  ├─ onMcpOAuthCallback(cb)  ◄──on── ipcMain.push "mcp-oauth-callback"│  │
│  │  ├─ onReportBug(cb)         ◄──on── ipcMain.push "app:reportBug"   │   │
│  │  └─ window.onFullscreenChange(cb) ◄──on── ipcMain.push "window:fullscreenChange"││
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  WebSocket Client  ←── DIRECT ──→  opendroid.exe Daemon                │   │
│  │  (ws://localhost:31415 or ws://localhost:41832)                    │   │
│  │                                                                     │   │
│  │  JSON-RPC: daemon.* methods (50+)                                   │   │
│  │  Push events: daemon.session_notification, daemon.request_permission│   │
│  │               daemon.ask_user, daemon.relay.status_changed          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ contextBridge
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MAIN PROCESS                                   │
│                        (main-process.js, main process bundle)                           │
│                                                                             │
│  ┌─ ipcMain.handle()  (15 channels) ── LEt() function, IPC registration region–242       │
│  ├─ ipcMain.on()       (5–6 ErrTracker channels) ── Ont() function, ErrTracker handler region  │
│  ├─ webContents.send() (6 push channels) ── scattered in main code        │
│  │                                                                         │
│  ├─ DaemonManager (eEt class) ── spawn, health poll, restart             │
│  │   ├── child_process.spawn("opendroid.exe", ["daemon", ...])               │
│  │   ├── fetch("http://localhost:{port}/health") ── HTTP polling          │
│  │   └── Exponential backoff restart (2s, 4s, 8s), max 3 attempts       │
│  │                                                                         │
│  └─ CSP configures ws://localhost:31415, ws://localhost:41832             │
│      (renderer↔daemon WebSocket: main process does NOT use WebSocket)    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ child_process.spawn
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DAEMON PROCESS                                 │
│                         (opendroid.exe / opendroid binary)                          │
│                                                                             │
│  ├─ HTTP server  (port 31415 prod / 41832 dev)                            │
│  ├─ WebSocket server (same ports)                                         │
│  ├─ JSON-RPC API (50+ methods: session, terminal, MCP, git, automation)   │
│  └─ Push events to renderer via WebSocket                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Communication Patterns Summary

| Pattern | Mechanism | Direction | Count | Layer |
|---------|-----------|-----------|-------|-------|
| Request/Response | `ipcRenderer.invoke` ↔ `ipcMain.handle` | Renderer → Main → Renderer | 15 | Preload ↔ Main |
| Event Push | `webContents.send` ↔ `ipcRenderer.on` | Main → Renderer | 6 | Main → Preload |
| Fire-and-Forget | `ipcRenderer.send` ↔ `ipcMain.on` | Renderer → Main | 6 (ErrTracker) | Preload ↔ Main |
| Direct Data | WebSocket JSON-RPC | Renderer ↔ Daemon | 50+ methods | Renderer ↔ Daemon |
| Lifecycle | `child_process.spawn` + HTTP | Main ↔ Daemon |: | Main ↔ Daemon |

## Key Findings

### Finding 1: Unified IPC Channel Count: 27 Total Channels

The complete IPC surface across all three layers consists of **27 channels**:

| Category | Preload | Main Process | Renderer (confirmed) | Match |
|----------|---------|-------------|---------------------|-------|
| `invoke` ↔ `handle` | 15 | 15 | 8 direct + 7 hypothesized | ✅ 1:1 |
| `on` (push) ↔ `send` | 6 | 6 | 2 direct + 4 hypothesized | ✅ 1:1 |
| ErrTracker `send` ↔ `on` | 6 | 5–6 | 0 (ErrTracker SDK internal) | ✅ |
| **Total** | **27** | **26–27** | **8 direct + 11 hypothesized** | ✅ Consistent |

### Finding 2: Zero Orphaned Main↔Preload Channels

Every `ipcMain.handle` channel has a corresponding `ipcRenderer.invoke` in preload.js. Every `webContents.send` push channel has a corresponding `ipcRenderer.on` listener in preload.js. Every ErrTracker `ipcMain.on` channel has a corresponding `ipcRenderer.send` in preload.js. There are **zero orphaned channels** at the main↔preload boundary.

### Finding 3: 13 Preload Methods Not Found in Main Renderer Bundle (Hypothesized in Secondary Chunks)

The renderer's main bundle (`renderer-main.js`) directly calls only **8 of 21** `electronAPI` methods. The remaining 13 are **not true orphans**: they are hypothesized to exist in lazy-loaded secondary chunks:

| # | Missing Method | Channel | Likely Location |
|---|---------------|---------|-----------------|
| 1 | `onAppError(cb)` | `app:error` | Error boundary component (secondary chunk or inline) |
| 2 | `onMcpOAuthCallback(cb)` | `mcp-oauth-callback` | MCP OAuth route component |
| 3 | `onReportBug(cb)` | `app:reportBug` | Bug report modal |
| 4 | `selectDirectory()` | `dialog:selectDirectory` | Settings / repository config page |
| 5 | `getLocalComputerId()` | `computer:getLocalId` | Computer settings |
| 6 | `getRemoteAccessEnabled()` | `computer:getRemoteAccessEnabled` | Remote access settings |
| 7 | `setRemoteAccessEnabled(e)` | `computer:setRemoteAccessEnabled` | Remote access settings |
| 8 | `notification.show(n)` | `notification:show` | Notification service |
| 9 | `notification.setBadgeCount(c)` | `notification:setBadgeCount` | Dock badge handler |
| 10 | `readSessionFile(ref)` | `session:readFile` | Session viewer component |
| 11 | `searchSessions(criteria)` | `session:searchSessions` | Session search component |
| 12 | `window.isFullscreen()` | `window:isFullscreen` | Window controls |
| 13 | `window.onFullscreenChange(cb)` | `window:fullscreenChange` | Window controls |

**Evidence for hypothesis:**
- TanStack Router with code-splitting is confirmed in `gui-renderer-core.md`
- Settings, sessions, and notifications are separate route components
- The 471KB (`renderer-chunk-a.js`) and 92KB (`renderer-chunk-b.js`) chunks contain feature-specific code
- `gui-renderer-secondary-chunks.md` confirms lazy-loading patterns

### Finding 4: Main Process is NOT in the Daemon Data Path

Contrary to initial assumptions, the main process does **not** use WebSocket to communicate with the daemon. The architecture is:
- **Main ↔ Daemon**: `child_process.spawn` (lifecycle), `fetch()` HTTP health polling, FD3 pipe (liveness signal)
- **Renderer ↔ Daemon**: WebSocket JSON-RPC (direct, bypassing main process)

The CSP's `connect-src: ws://localhost:31415, ws://localhost:41832` is for the **renderer's** WebSocket client, not the main process. This is a critical architectural insight: the main process manages the daemon's lifecycle but never proxies its data.

### Finding 5: Three Documented Discrepancies

**Discrepancy A: Payload Wrapping Inconsistency**
Two single-parameter invoke methods wrap their argument in an object, while all others pass directly:
- `setRemoteAccessEnabled(enabled)` → sends `{enabled: enabled}` ✅ wrapped
- `notification.setBadgeCount(count)` → sends `{count: count}` ✅ wrapped
- `auth.switchToOrganization(org)` → passes `org` directly ❌ not wrapped
- `readSessionFile(fileRef)` → passes `fileRef` directly ❌ not wrapped

**Explanation:** Likely historical evolution: the two wrapped methods were added later when the convention changed, or the wrapper provides type safety / future-proofing for these specific calls.

**Discrepancy B: `errtracker-ipc.status` Conditionally Registered**
The ErrTracker `ipcMain.on` handler for `errtracker-ipc.status` is only registered if `vee(e)` returns true (status reporting enabled in ErrTracker config). This means:
- Preload always exposes `sendStatus()`
- Main process may or may not have a handler
- If called when no handler exists, the send is silently dropped (fire-and-for-forget)

**Explanation:** This is by design: ErrTracker's status reporting is optional and configurable. No functional impact.

**Discrepancy C: `onAppError` Renderer Call Site Not Confirmed**
The `app:error` push channel has both preload exposure (`electronAPI.onAppError(cb)`) and main process push logic (`XT()` function). However, the renderer-side call site for `window.electronAPI.onAppError()` was **not definitively found** in the main bundle analysis. It is either:
1. In a secondary chunk (error boundary component)
2. In an unanalyzed portion of the large renderer bundle bundle
3. Registered but never called (dead code: unlikely, given the main process actively pushes this channel)

**Verdict:** Most likely hypothesis #1: error boundaries are typically separate components that would be code-split.

## Code Examples

### Example 1: Complete Three-Layer Channel Registration for `auth:signIn`

**Layer 1: Preload (`build/preload.js`):**
```js
n.contextBridge.exposeInMainWorld("electronAPI", {
  auth: {
    signIn: () => n.ipcRenderer.invoke("auth:signIn")
    // ...
  }
});
```

**Layer 2: Main Process (`build/main-process.js`, IPC handlers region):**
```js
re.ipcMain.handle("auth:signIn", async () => {
  Be("Handling auth:signIn request");
  try {
    return await rEt(), { success: !0 };
  } catch (n) {
    return En("Failed to initiate OAuth flow:", { error: n }),
           { success: !1, error: n.message };
  }
});
```

**Layer 3: Renderer (`renderer-main.js`, ~line 10565):**
```js
const n = async () => {
  if (!window.electronAPI) throw new Error("electronAPI is not available");
  e({isLoading: true});
  await window.electronAPI.auth.signIn();
};
```

### Example 2: Push Channel Three-Layer Registration for `auth:success`

**Layer 1: Preload (`build/preload.js`):**
```js
auth: {
  onAuthSuccess: e => {
    const r = (i, t) => e(t);
    return n.ipcRenderer.on("auth:success", r),
           () => n.ipcRenderer.removeListener("auth:success", r);
  }
}
```

**Layer 2: Main Process (`build/main-process.js`, scattered):**
```js
// After OAuth deep link callback processing:
webContents.send("auth:success", { user: { /* auth data */ } });
```

**Layer 3: Renderer (`renderer-main.js`, ~line 10565):**
```js
const h = u.auth.onAuthSuccess(async ({user: b}) => {
  hr("Received auth success from main process", {userId: b?.id});
  try {
    const y = await u.auth.getAuthData();
    e({user: y.user, isLoading: false});
  } catch (y) {
    oee(y, "Failed to get auth data after auth success");
    e({user: void 0, isLoading: false});
  }
});

// Cleanup on unmount:
return () => { h(); g(); };
```

### Example 3: End-to-End Daemon Crash → Main → Renderer Event Flow

This is the canonical daemon→main→renderer flow:

**Step 1: Daemon crashes unexpectedly:**
```js
// In DaemonManager (eEt class), line ~218:
handleProcessExit(code, signal) {
  if (this.isShuttingDown) return;  // Ignore expected shutdowns
  this.state = pi.Stopped;
  this.process = null;

  // Push error to renderer
  XT(`Daemon process exited unexpectedly (code: ${code}, signal: ${signal})`);

  // Schedule restart with exponential backoff
  this.scheduleRestart();
}
```

**Step 2: Main process pushes error to renderer (`XT()` function):**
```js
// Line ~218 in main-process.js:
function XT(e, t = 5, n = 500) {
  let r = 0;
  const s = () => {
    const i = $c();  // Get current BrowserWindow
    if (i && !i.isDestroyed()) {
      if (i.webContents.isLoading()) {
        r < t && (r++, setTimeout(s, n));  // Retry if page loading
      } else {
        i.webContents.send("app:error", { error: e });
      }
    } else {
      r < t && (r++, setTimeout(s, n));  // Retry if no window yet
    }
  };
  s();
}
```

**Step 3: Preload receives and strips event object:**
```js
// build/preload.js:
onAppError: e => {
  const r = (i, t) => e(t);  // Strip Electron event, pass only data
  return n.ipcRenderer.on("app:error", r),
         () => n.ipcRenderer.removeListener("app:error", r);
}
```

**Step 4: Renderer displays error (hypothesized in error boundary/secondary chunk):**
```js
// Hypothesized call site (not found in main bundle, likely in secondary chunk):
const unsubscribe = window.electronAPI.onAppError(({error}) => {
  showErrorToast(error);
});
```

**Flow summary:**
```
opendroid.exe crashes
    ↓ (child_process exit event)
MainProcess.handleProcessExit()
    ↓ (calls XT())
webContents.send("app:error", {error: "Daemon process exited unexpectedly..."})
    ↓ (Electron IPC)
preload.js: ipcRenderer.on("app:error", handler)
    ↓ (strips event, calls callback)
renderer: onAppError(callback) → UI error toast
```

## Theme / Style Tokens

Not applicable for this feature: IPC topology analysis contains no CSS or theme-related code.

## IPC Channels

### Unified Channel Catalog

#### A. Request/Response Channels (`invoke` ↔ `handle`): 15 channels

| # | Channel Name | Direction | Preload Method | Main Handler | Renderer Call Site | Payload Shape |
|---|-------------|-----------|---------------|-------------|-------------------|---------------|
| 1 | `auth:signIn` | R→M→R | `auth.signIn()` | `LEt()` IPC handlers region | `ivo()` signIn closure | Req: none; Resp: `{success, error?}` |
| 2 | `auth:signOut` | R→M→R | `auth.signOut()` | `LEt()` IPC handlers region | `ivo()` signOut closure | Req: none; Resp: `{success, error?}` |
| 3 | `auth:getAuthData` | R→M→R | `auth.getAuthData()` | `LEt()` IPC handlers region | `ivo()` init + getAccessToken + onAuthSuccess cb | Req: none; Resp: `{user?, accessToken?}` |
| 4 | `auth:switchToOrganization` | R→M→R | `auth.switchToOrganization(org)` | `LEt()` IPC handlers region | `ivo()` switchToOrganization closure | Req: `{organizationId}`; Resp: `{success, error?}` |
| 5 | `dialog:selectDirectory` | R→M→R | `selectDirectory()` | `LEt()` IPC handlers region | Hypothesized: settings page | Req: none; Resp: `{canceled, path?}` |
| 6 | `app:getVersion` | R→M→R | `getAppVersion()` | `LEt()` IPC handlers region | `MBs()` lazy cached | Req: none; Resp: `string` (semver) |
| 7 | `computer:getLocalId` | R→M→R | `getLocalComputerId()` | `LEt()` IPC handlers region | Hypothesized: computer settings | Req: none; Resp: `string` |
| 8 | `computer:getRemoteAccessEnabled` | R→M→R | `getRemoteAccessEnabled()` | `LEt()` IPC handlers region | Hypothesized: remote access settings | Req: none; Resp: `boolean` |
| 9 | `computer:setRemoteAccessEnabled` | R→M→R | `setRemoteAccessEnabled(enabled)` | `LEt()` IPC handlers region | Hypothesized: remote access settings | Req: `{enabled: boolean}`; Resp: void |
| 10 | `notification:show` | R→M→R | `notification.show(notif)` | `LEt()` IPC handlers region | Hypothesized: notification service | Req: `{title, body, sessionId?}`; Resp: void |
| 11 | `notification:setBadgeCount` | R→M→R | `notification.setBadgeCount(count)` | `LEt()` IPC handlers region | Hypothesized: dock badge handler | Req: `{count: number}`; Resp: void |
| 12 | `session:readFile` | R→M→R | `readSessionFile(fileRef)` | `LEt()` IPC handlers region | Hypothesized: session viewer | Req: `{sessionId}`; Resp: `{messages, metadata, settings}` or `{error}` |
| 13 | `session:searchSessions` | R→M→R | `searchSessions(criteria)` | `LEt()` IPC handlers region | Hypothesized: session search | Req: `{query: string}`; Resp: `{query, sessions[]}` or `{error}` |
| 14 | `window:isFullscreen` | R→M→R | `window.isFullscreen()` | `LEt()` IPC handlers region | Hypothesized: window controls | Req: none; Resp: `boolean` |
| 15 | `bugReport:getLogs` | R→M→R | `getBugReportLogs()` | `LEt()` IPC handlers region | Hypothesized: bug report modal | Req: none; Resp: `{[filename: string]: string}` |

#### B. Push Channels (`send` from Main → `on` in Renderer): 6 channels

| # | Channel Name | Direction | Preload Listener | Main Push Location | Payload Shape | Trigger |
|---|-------------|-----------|-----------------|-------------------|---------------|---------|
| P1 | `navigate` | M→R | `onNavigate(cb)` | Deep link router, notification click | `{path: string}` | Deep link or notification clicked |
| P2 | `app:error` | M→R | `onAppError(cb)` | `XT()` function (daemon errors, spawn failures) | `{error: string}` | Daemon crash, health timeout, spawn failure |
| P3 | `auth:success` | M→R | `auth.onAuthSuccess(cb)` | OAuth callback completion (`nEt()`) | auth data object | OAuth deep link returns success |
| P4 | `mcp-oauth-callback` | M→R | `onMcpOAuthCallback(cb)` | MCP OAuth callback handler | `{code, state}` | MCP server OAuth callback |
| P5 | `app:reportBug` | M→R | `onReportBug(cb)` | Menu trigger | none (signal only) | User selects "Report Bug" from menu |
| P6 | `window:fullscreenChange` | M→R | `window.onFullscreenChange(cb)` | macOS fullscreen enter/exit handler | `boolean` | macOS fullscreen state change |

#### C. ErrTracker Channels (`send` from Renderer → `on` in Main): 6 channels

| # | Channel Name | Direction | Preload Method | Main Handler | Payload |
|---|-------------|-----------|---------------|-------------|---------|
| S1 | `errtracker-ipc.start` | R→M | `sendRendererStart()` | `Ont()` ErrTracker handler region | none |
| S2 | `errtracker-ipc.scope` | R→M | `sendScope(scope)` | `Ont()` ErrTracker handler region | scope object |
| S3 | `errtracker-ipc.envelope` | R→M | `sendEnvelope(envelope)` | `Ont()` ErrTracker handler region | envelope data |
| S4 | `errtracker-ipc.structured-log` | R→M | `sendStructuredLog(log)` | `Ont()` ErrTracker handler region | log data |
| S5 | `errtracker-ipc.metric` | R→M | `sendMetric(metric)` | `Ont()` ErrTracker handler region | metric data |
| S6 | `errtracker-ipc.status` | R→M | `sendStatus(status)` | `Ont()` ErrTracker handler region (conditional) | status data |

### Cross-Reference Consistency Matrix

| Check | Result | Notes |
|-------|--------|-------|
| All 15 `invoke` methods have `handle` counterpart | ✅ PASS | 1:1 mapping confirmed |
| All 6 push channels have `on` listener in preload | ✅ PASS | 1:1 mapping confirmed |
| All 6 ErrTracker `send` methods have `on` counterpart | ✅ PASS | 1:1 mapping (S1 status conditional) |
| All renderer calls map to preload methods | ✅ PASS | 8/8 direct calls confirmed; 0 orphans |
| All preload methods have renderer callers | ⚠️ PARTIAL | 8/21 direct confirmed; 13 hypothesized in chunks |
| No renderer calls to undefined methods | ✅ PASS | Zero broken references |

## Integration Points

### Cross-System Dependencies

1. **Daemon Binary (Infrastructure section: Infrastructure)**: The `opendroid.exe`/`droid` binary is spawned by the main process. Its internal HTTP/WebSocket server architecture, JSON-RPC handler, and 50+ method implementations are Infrastructure section scope. 04-desktop-gui documents only the IPC boundary and lifecycle management.

2. **Renderer WebSocket Client (Desktop GUI section: Renderer Core)**: The renderer connects directly to the daemon via WebSocket using ports 31415 (production) and 41832 (development). The WebSocket client, JSON-RPC framing, and 50+ `daemon.*` method calls are documented in `gui-renderer-core.md` and `gui-renderer-mission-panel.md`. The main process is NOT in this data path.

3. **Auth System (Terminal UI section / Tool & Agent section)**: The IdProvider OAuth flow, token management, and user profile storage are cross-mission concerns. 04-desktop-gui documents the IPC boundary (`auth:signIn`, `auth:success`, etc.) but not the auth provider implementation.

4. **ErrTracker Error Reporting (Third-Party)**: The ErrTracker Electron SDK provides its own IPC bridge (`__ERRTRACKER_IPC__`) independent of the application IPC. It can be cleanly removed or replaced without affecting application functionality.

5. **Session Search Module**: The `session-search.js` (small helper chunk) is dynamically loaded by the `session:searchSessions` handler. This is the only lazy-loaded module in the IPC surface and represents a cross-file dependency within 04-desktop-gui.

### Cross-Mission Boundary Points

| Boundary | 04-desktop-gui Documentation | Other Mission Scope |
|----------|----------------------|---------------------|
| IPC channel names | Full catalog (this report) | TUI/CLI may share auth channels (Terminal UI section) |
| WebSocket daemon protocol | URL pattern, ports, CSP allowance | Daemon RPC implementation (Infrastructure section) |
| Daemon lifecycle | Spawn config, health poll, restart | Daemon binary internals (Infrastructure section) |
| Auth token flow | IPC channel shapes | Auth provider implementation (Terminal UI section/3) |
| Computer ID / Remote Access | IPC channel shapes | Infrastructure config (Infrastructure section) |

## Prior Report Verification

### Section Completeness Check

All 14 prior feature reports were verified for required sections:

| Report | Overview | File Map | Architecture | Key Findings | Code Examples | IPC Channels | Integration Points | Implementation Notes | Module Reference |
|--------|----------|----------|-------------|-------------|---------------|-------------|-------------------|---------------------|-----------------|
| `gui-html-entry.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-preload-bridge.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-main-process-core.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-main-process-ipc.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-main-process-daemon.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-main-process-helpers.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-renderer-core.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-renderer-ipc-client.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-renderer-onboarding.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-renderer-mission-panel.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-renderer-diff-settings.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-renderer-secondary-chunks.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-theme-css.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `gui-pdfjs-xlsx-note.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

**Result: 14/14 reports contain all required sections. Zero missing sections.**

### Implementation Notes Content Check

All 14 reports contain **non-empty** Implementation Notes sections with actionable, specific guidance:
- `gui-html-entry.md`: CSP replication, theme script, loader SVG
- `gui-preload-bridge.md`: Namespace mapping, channel convention, listener cleanup, ErrTracker removal
- `gui-main-process-core.md`: BrowserWindow security config, preload path adaptation
- `gui-main-process-ipc.md`: Handler registration pattern, payload standardization, dynamic module loading
- `gui-main-process-daemon.md`: DaemonManager class port, HTTP health check, liveness FD, log rotation
- `gui-main-process-helpers.md`: main.js stub, search module lazy-loading
- `gui-renderer-core.md`: TanStack Router port, Allotment split pane, auth gateway
- `gui-renderer-ipc-client.md`: Centralized auth gateway, platform detection, token propagation
- `gui-renderer-onboarding.md`: State machine replication, video playback, theme-aware assets
- `gui-renderer-mission-panel.md`: MissionStore port, feature list, sidebar virtual scrolling
- `gui-renderer-diff-settings.md`: Diff viewer, settings panels, storage mechanism
- `gui-renderer-secondary-chunks.md`: Mammoth/Mermaid lazy-loading, ELK worker handling
- `gui-theme-css.md`: CSS token extraction, dark mode toggle, Tailwind audit
- `gui-pdfjs-xlsx-note.md`: Lazy-loading preservation, singleton cache

**Result: 14/14 reports have specific, actionable Implementation Notes content.**

## Implementation Notes

### Porting the Complete IPC Topology

1. **Preserve the three-layer architecture**: OpenDroid should replicate the preload ↔ main ↔ renderer boundary exactly. Do not let the renderer directly access `require('electron')`: always route through `contextBridge.exposeInMainWorld`.

2. **Adopt the `domain:action` channel naming convention**: The colon separator (`auth:signIn`, `session:readFile`, `notification:show`) is clean and extensible. Use it consistently for all application IPC channels.

3. **Standardize payload wrapping**: Decide on ONE convention: either always wrap single parameters in objects (`{enabled}`, `{count}`) or always pass directly. The current OpenDroid codebase has inconsistency here that should be fixed in OpenDroid.

4. **Separate ErrTracker IPC from application IPC**: The `__ERRTRACKER_IPC__` namespace is entirely independent. If OpenDroid uses a different error reporting solution (or none), remove the ErrTracker bridge without touching `electronAPI`.

5. **Use the same listener cleanup pattern**: Every `.on()` listener in preload returns an unsubscribe function that calls `removeListener`. The renderer correctly cleans these up in `useEffect` returns. Preserve this pattern to prevent memory leaks.

6. **Centralize auth IPC in a single gateway component**: The `ivo()` pattern: a desktop root component that checks `window.electronAPI` existence, sets up all auth-related IPC, and returns cleanup: is excellent. Port this as `DesktopAuthProvider`.

7. **Main process as daemon lifecycle manager only**: Do NOT route daemon data through the main process. The renderer should connect directly to the daemon via WebSocket, while the main process handles spawn, health polling, and restart. This avoids bottlenecking high-throughput data (terminal streams, session messages) through Electron IPC.

8. **Implement exponential backoff restart**: The DaemonManager's restart pattern (2s, 4s, 8s, max 3 attempts) with metrics tracking is production-grade. Port the `eEt` class with its state machine and health polling.

9. **Handle renderer-not-ready retries**: The `XT()` function retries `webContents.send()` up to 5 times with 500ms delays if the renderer is still loading. This is critical for early daemon crashes during app startup.

10. **Lazy-load heavy IPC handlers**: The `session:searchSessions` handler dynamically loads `session-search.js` on first use. Apply this pattern to any expensive operations in OpenDroid (e.g., git diff computation, large file parsing).

11. **Notification click routing**: The pattern of "notification click → window restore → navigate to session" is valuable. Port the 100ms delay (`setTimeout(..., 100)`) that ensures the renderer is ready before pushing the navigate event.

12. **CSP WebSocket allowance**: If OpenDroid's renderer connects to a local daemon via WebSocket, explicitly allow `ws://localhost:{port}` in the CSP `connect-src` directive and add a bypass for localhost connections.

## Module Reference

| File | Offset Range | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| `build/preload.js` | Line 1–26 | `contextBridge.exposeInMainWorld("electronAPI", {...})` | Full IPC bridge: 21 methods |
| `build/preload.js` | Line 1–25 | `s()` / `__ERRTRACKER_IPC__` | ErrTracker bridge: 6 methods |
| `build/main-process.js` | ErrTracker handler region | `Ont(e, t, n)` | ErrTracker `ipcMain.on` registration |
| `build/main-process.js` | IPC registration region–242 | `LEt()` | Main IPC handler registration: 15 `ipcMain.handle` channels |
| `build/main-process.js` | DaemonManager region | `eEt` class / `eEt.start()` / `eEt.stop()` | DaemonManager lifecycle |
| `build/main-process.js` | DaemonManager region | `XT(e, t, n)` | Push error to renderer with retry |
| `build/main-process.js` | DaemonManager region | `Jgt()` | Daemon spawn command builder |
| `build/main-process.js` | IPC handlers region | `rEt()` | Auth OAuth flow initiator |
| `build/main-process.js` | IPC handlers region | `nEt()` | Post-login: start daemon + push auth:success |
| `build/main-process.js` | IPC handlers region | `kEt()` | Stop daemon (called on logout) |
| `build/session-search.js` | Full file | `runDroidSearch(query)` | Lazy-loaded session search |
| `renderer/assets/renderer-main.js` | ~9405 | `MBs()` | Lazy-cached `getAppVersion()` |
| `renderer/assets/renderer-main.js` | ~10565 | `ivo()` | Desktop root: ALL direct electronAPI calls |
| `renderer/assets/renderer-main.js` | ~10565 | `ivo() → useEffect` | Auth init, listener setup, cleanup |
| `renderer/assets/renderer-main.js` | ~10565 | `rl()` | API client factory: uses `getAccessToken()` from IPC |
| `renderer/assets/renderer-main.js` | ~10565 | `vu()` | Platform detection helper |
