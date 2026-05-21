# Main Process Core: BrowserWindow & App Lifecycle: GUI Architecture Notes

## Overview

This report analyzes the Electron main process bundle `main-process.js` (main process bundle, minified into ~242 lines), focusing on BrowserWindow creation, app lifecycle hooks, auto-updater setup, tray/menu systems, and the daemon manager. The main process serves as the application orchestrator: it creates a single BrowserWindow with strict security settings (contextIsolation=true, nodeIntegration=false), manages a spawned `opendroid.exe` daemon child process with health polling and automatic restart, handles OAuth authentication via deep links (`opendroid://` protocol), and uses ErrTracker for crash/error reporting. The app uses `electron-updater` with static storage for auto-updates on a 10-minute interval.

## File Map

| File | Size | Role |
|------|------|------|
| `build/main-process.js` | main process bundle (~1.4M bytes) | Electron main process: all application logic |
| `build/preload.js` | small preload bridge | IPC bridge (analyzed in gui-preload-bridge.md) |
| `build/session-search.js` | small helper chunk | Session search module (lazy-loaded) |
| `build/main.js` | 45 bytes | Stub/entry point |
| `assets/tray-icon.png` | Referenced | Notification icon (tray-icon.png in assets/) |

## Architecture

### Main Process Architecture

The main process follows a sequential bootstrap pattern implemented in the `BEt()` function (the main lifecycle entry point):

```
Bootstrap Order:
1. Auto-updater initialization (electron-updater, static storage)
2. Single instance lock (requestSingleInstanceLock)
3. Deep link / protocol handler setup (open-url, second-instance)
4. app.whenReady()
5. Shell environment loading (yrt(): macOS shell env)
6. IPC handler registration (LEt(): 15+ handlers)
7. Protocol registration (setAsDefaultProtocolClient for "opendroid://")
8. CLI setup (OR(): binary integrity check)
9. Version compatibility check
10. BrowserWindow creation (Iv())
11. CLI install prompt (Pot(): droid CLI PATH installer)
12. Daemon start (qu.start(): spawns opendroid.exe)
13. ErrTracker user context ($ne())
14. CLI install flow (yEt())
15. App lifecycle hooks (activate, window-all-closed, before-quit)
16. Windows deep link handling (GEt())
```

### Singleton Window Pattern

The app uses a module-level `St` variable (aliased as the main window reference). Key functions:
- **`Iv()`**: Creates the BrowserWindow (called "createWindow" equivalent)
- **`pa()`**: Gets or creates the window (show/focus/restore)
- **`$c()`**: Returns the current window reference (may be null)

### Daemon Manager (eEt class)

A class-based daemon manager with state machine:
- States: `Stopped`, `Starting`, `Running`, `Stopping`
- Spawns `opendroid.exe` (or `opendroid-dev` in dev) as child process
- Health polling via HTTP GET to `http://localhost:{port}/health`
- Auto-restart with exponential backoff (max 3 attempts, 2s base)
- Process generation tracking to prevent stale health checks

## Key Findings

### 1. BrowserWindow Configuration

The BrowserWindow is created with platform-aware settings:

```js
// DaemonManager region: Function Iv() (createWindow)
St = new re.BrowserWindow({
  width: 1200,
  height: 800,
  titleBarStyle: t ? "default" : "hidden",  // t = process.platform === "win32"
  trafficLightPosition: t ? void 0 : {x: 12, y: 10},  // macOS traffic lights
  webPreferences: {
    devTools: !re.app.isPackaged,   // DevTools only in development
    contextIsolation: true,          // Security: context isolation ON
    nodeIntegration: false,          // Security: Node.js OFF in renderer
    preload: Ze.join(__dirname, "preload.js")  // Preload script path
  }
})
```

**Key security observations:**
- `contextIsolation: true`: Properly isolated
- `nodeIntegration: false`: No Node.js in renderer
- `devTools` disabled in production (`!re.app.isPackaged`)
- No `sandbox` option explicitly set (Electron default applies)
- No `frame: false`: uses `titleBarStyle: "hidden"` on macOS for frameless look with native controls

### 2. App Lifecycle Hooks

All lifecycle hooks are registered in the `BEt()` bootstrap function:

| Hook | Location | Behavior |
|------|----------|----------|
| `app.whenReady()` | IPC handlers region (`BEt()`) | Gate for all subsequent setup: window creation, IPC, daemon |
| `app.on("activate")` | IPC handlers region | macOS: re-creates window when dock icon clicked and no windows exist (`Iv()`) |
| `app.on("window-all-closed")` | IPC handlers region | Quits app on non-macOS platforms (`process.platform !== "darwin" && re.app.quit()`) |
| `app.on("before-quit")` | IPC handlers region | Stops daemon gracefully with 10s timeout, then force-quits. Increments `DESKTOP_APP_QUIT_COUNT` metric. |
| `app.on("open-url")` | IPC handlers region (via `oEt()`) | macOS: handles deep link URLs (`opendroid://` protocol) |
| `app.on("second-instance")` | IPC handlers region (via `oEt()`) | Routes deep link from second instance to existing window |
| `app.on("browser-window-created")` | Line 178 (ErrTracker BrowserWindowSession) | ErrTracker session tracking: focus/blur/show/hide events |
| `app.on("before-quit")` (ErrTracker) | Line 178 (iee()) | ErrTracker session cleanup before quit |

### 3. before-quit Handler (Graceful Shutdown)

The quit handler prevents default quit, stops the daemon, then re-triggers quit:

```js
// IPC handlers region
re.app.on("before-quit", async t => {
  if (Fr.addToCounter(Vs.DESKTOP_APP_QUIT_COUNT, 1), !qu.isStopped()) {
    t.preventDefault();  // Prevent immediate quit
    const n = setTimeout(() => {
      Pe("[lifecycle] Daemon stop timed out after 10s, forcing quit");
      re.app.exit(0);  // Force quit after 10s
    }, 1e4);
    try {
      await qu.stop();  // Gracefully stop daemon
      clearTimeout(n);
      re.app.quit();    // Re-quit after daemon stopped
    } catch (r) {
      clearTimeout(n);
      En("[lifecycle] Error stopping daemon:", {cause: r});
      re.app.exit(1);   // Exit with error code
    }
  }
})
```

### 4. Single Instance Lock

```js
// IPC handlers region: FEt()
re.app.requestSingleInstanceLock()
```

If lock fails (another instance running), marks outcome as "single_instance_conflict" and quits.

### 5. Auto-Updater

Uses `electron-updater` (aliased as `art` for `autoUpdater`, `Wee` for updater types):

```js
// IPC handlers region
art({
  updateSource: {
    type: Wee.StaticStorage,
    baseUrl: new URL(`${DEt}opendroid-desktop/updates/${process.platform}/${process.arch}`, qnt)
  },
  updateInterval: "10 minutes",
  logger: { /* log/info/warn/error mapped to app logging */ }
})
```

- **Source**: Static storage at `opendroid-desktop/updates/{platform}/{arch}`
- **Interval**: 10 minutes
- **Base URL**: Constructed from empty string `DEt=""` and `qnt` (likely the update server base URL)

### 6. Menu System

The app does set up an application menu via `re.Menu.buildFromTemplate`:

```js
// IPC registration region
re.Menu.setApplicationMenu(re.Menu.buildFromTemplate(n))
```

The menu template `n` contains standard items (from the visible code): About, version info, Node.js version display. This appears to be a minimal application menu.

### 7. Tray / Notifications

**No system tray is created.** The app does NOT use `new re.Tray()`.

Notifications are created on-demand via IPC (`notification:show` handler):

```js
// IPC handlers region: Inside LEt()
const o = re.nativeImage.createFromPath(Ze.join(__dirname, "..", "..", "assets", "tray-icon.png"));
const a = new re.Notification({title: r, body: s, icon: o.isEmpty() ? void 0 : o});
a.on("click", () => {
  // Restore window, navigate to session if sessionId provided
  const c = $c();
  c && !c.isDestroyed() && (
    c.isMinimized() && c.restore(),
    c.show(), c.focus(),
    i && setTimeout(() => {
      c.isDestroyed() || c.webContents.send("navigate", {path: `/sessions/${i}`})
    }, 100)
  )
});
a.show();
```

Badge count support: `re.app.setBadgeCount(r)` via `notification:setBadgeCount` IPC.

### 8. Window Zoom Management

Zoom level is persisted in an Electron Store (`window-zoom` config):

```js
// DaemonManager region
const Pne = 0;  // default zoom level
function wne() { return SV ?? (SV = new fM({name: "window-zoom", defaults: {zoomLevel: Pne}})), SV; }
```

- Zoom is restored on window show/focus
- Zoom changes via `zoom-changed` event (0.5 step increments)

### 9. CSP (Content Security Policy)

Comprehensive CSP applied via `onHeadersReceived`:

```
default-src 'self'
script-src 'self' 'unsafe-inline' 'unsafe-eval' blob: [analyticssvc, clouddb, google, linkedin]
style-src 'self' 'unsafe-inline'
img-src 'self' blob: data: [google, linkedin, idpcdn]
font-src 'self'
connect-src 'self' ws://localhost:31415 ws://localhost:41832 [dev servers, errtracker, idprovider, analyticssvc, clouddb, sandboxsvc, opendroid API, relay, analytics]
frame-src 'self' http://localhost:31415 http://localhost:41832 [google, clouddb]
object-src 'none'
base-uri 'self'
form-action 'self'
frame-ancestors 'self'
```

CSP is bypassed for localhost:41832 and localhost:31415 (daemon/WebSocket ports).

### 10. Window Open Handler

External URLs are opened in the system browser, not in Electron:

```js
St.webContents.setWindowOpenHandler(({url: r}) => {
  const s = new URL(r);
  return ["http:", "https:", "vscode:", "vscode-insiders:", "cursor:", "zed:", "jetbrains:", "file:", "ghostty:", "iterm2:", "warp:", "xcode:"].includes(s.protocol)
    ? (re.shell.openExternal(s.href), {action: "deny"})
    : {action: "deny"};
})
```

Supported external protocols: VS Code, VS Code Insiders, Cursor, Zed, JetBrains, Ghostty, iTerm2, Warp, Xcode.

### 11. Fullscreen Events (macOS)

```js
// DaemonManager region
zgt && (  // zgt = process.platform === "darwin"
  St.on("enter-full-screen", () => { St?.webContents.send("window:fullscreenChange", true) }),
  St.on("leave-full-screen", () => { St?.webContents.send("window:fullscreenChange", false) })
)
```

### 12. Daemon Manager Constants

```js
// eEt class: DaemonManager
MAX_RESTART_ATTEMPTS = 3
RESTART_BACKOFF_MS = 2000       // 2 seconds base
HEALTH_CHECK_TIMEOUT_MS = 2000  // 2 seconds
HEALTH_POLL_INTERVAL_MS = 2000  // 2 seconds
MAX_HEALTH_POLL_ATTEMPTS = 15
```

## Code Examples

### Renderer Entry Point Loading

```js
// DaemonManager region: Conditional loadFile vs loadURL
re.app.isPackaged
  ? St.loadFile(Ze.join(__dirname, "..", "renderer", "main_window", "index.html"))
  : (St.loadURL("http://localhost:5173"), St.webContents.openDevTools())
```

Production loads from `../renderer/main_window/index.html`, development from Vite dev server at `localhost:5173`.

### Daemon Spawn Configuration

```js
// DaemonManager region: Jgt() function
const e = re.app.isPackaged
  ? Ze.join(process.resourcesPath, "bin", process.platform === "win32" ? "opendroid.exe" : "opendroid")
  : "opendroid-dev";
const t = ["daemon", "--droid-path", e];
re.app.isPackaged || t.push("--debug");
// ... port override if non-default
t.push("--enable-code-server");
t.push("--liveness-fd", "3");
```

Daemon environment variables:
```js
env: {
  ...process.env,
  OPENDROID_AUTO_UPDATE_ENABLED: "false",
  OPENDROID_API_BASE_URL: "https://api.opendroid.dev",
  OPENDROID_DEPLOYMENT_ENV: "production",
  OPENDROID_OTEL_ENABLED: "true",
  OPENDROID_MACHINE_TYPE: "local",
  TERM: "xterm-256color"
}
```

### Deep Link Protocol Registration

```js
// IPC handlers region: iEt()
process.defaultApp
  ? re.app.setAsDefaultProtocolClient(Bf, process.execPath, [Ze.resolve(process.argv[1] || "")])
  : re.app.setAsDefaultProtocolClient(Bf)
```

Where `Bf` is the custom protocol scheme (likely `"opendroid"`).

### Renderer Error Forwarding

```js
// DaemonManager region: XT() function
function XT(e, t = 5, n = 500) {
  let r = 0;
  const s = () => {
    const i = $c();
    i && !i.isDestroyed()
      ? i.webContents.isLoading()
        ? r < t && (r++, setTimeout(s, n))
        : i.webContents.send("app:error", {error: e})
      : r < t && (r++, setTimeout(s, n));
  };
  s();
}
```

Sends error messages to renderer via `app:error` IPC channel with retry logic (5 attempts, 500ms interval).

## Theme / Style Tokens

Not applicable for this feature: this is the main process, not the renderer. Theme tokens are in `theme.css` (covered in gui-theme-css.md).

## IPC Channels

### Main→Renderer Push Channels

| Channel | Trigger | Payload |
|---------|---------|---------|
| `window:fullscreenChange` | macOS fullscreen enter/exit | `boolean` |
| `app:error` | Daemon crash, spawn failure, etc. | `{error: string}` |
| `navigate` | Deep link routing | `{path: string}` |
| `auth:success` | OAuth callback complete | `{user: object}` |
| `mcp-oauth-callback` | MCP OAuth deep link | `{code: string, state: string}` |

### Renderer→Main IPC Handle Channels (registered in LEt())

| Channel | Purpose |
|---------|---------|
| `app:getVersion` | Returns `re.app.getVersion()` |
| `auth:signIn` | Initiates OAuth flow via IdProvider |
| `auth:getAuthData` | Returns current user + access token |
| `auth:switchToOrganization` | Switches org context |
| `auth:signOut` | Stops daemon, clears auth, signs out |
| `dialog:selectDirectory` | Opens native directory picker |
| `notification:show` | Shows native notification |
| `notification:setBadgeCount` | Sets dock badge count |
| `session:readFile` | Reads session JSONL file |
| `session:searchSessions` | Searches sessions (delegates to session-search.js) |
| `computer:getLocalId` | Gets local computer ID |
| `computer:getRemoteAccessEnabled` | Gets remote access pref |
| `computer:setRemoteAccessEnabled` | Sets remote access pref |
| `window:isFullscreen` | Returns fullscreen state |
| `bugReport:getLogs` | Collects log files for bug report |

## Integration Points

- **Daemon process (`opendroid.exe`)**: Spawned as child process with `child_process.spawn()`. Communicates via HTTP health endpoint and file descriptor 3 (liveness FD). Detailed in gui-main-process-daemon.md.
- **WebSocket ports**: CSP allows `ws://localhost:31415` and `ws://localhost:41832`: these connect to the daemon. Cross-ref with 05-infrastructure (Infrastructure).
- **Preload bridge**: `preload.js` at `__dirname/preload.js`: see gui-preload-bridge.md for the complete IPC surface.
- **Session search**: Lazy-loads `session-search.js` via dynamic `require()` for session search functionality.
- **ErrTracker integration**: Extensive error tracking with custom integrations (BrowserWindowSession, AdditionalContext, electron-store session persistence).
- **IdProvider OAuth**: Uses `@idprovider-inc/node` for authentication flow, custom protocol handler `opendroid://`.
- **electron-store**: Used for config persistence (`window-zoom`, `computer-remote-access`, and likely others).
- **electron-updater**: Auto-updates from static storage, 10-minute check interval.

## Implementation Notes

1. **BrowserWindow config**: The security settings (`contextIsolation: true`, `nodeIntegration: false`, `devTools` disabled in prod) should be replicated exactly. The preload path resolution (`Ze.join(__dirname, "preload.js")`) is Vite-specific: adapt to OpenDroid's build system.

2. **Daemon manager**: The `eEt` class is a well-designed daemon manager with exponential backoff restart, health polling, and process generation tracking. Port this pattern directly: the state machine (Stopped→Starting→Running→Stopping) is clean and handles edge cases.

3. **Single instance lock**: Use `app.requestSingleInstanceLock()` to prevent multiple instances. The deep link routing from second instance to first is essential for OAuth flows.

4. **Graceful shutdown**: The `before-quit` handler with daemon stop + 10s timeout + force exit is a good pattern. Port this with the same timeout logic.

5. **CSP configuration**: The CSP is comprehensive but OpenDroid-specific. Remove OpenDroid domains (errtracker, analyticssvc, google analytics, sandboxsvc, relay) and replace with OpenDroid's infrastructure. Keep the strict `object-src 'none'` and `frame-ancestors 'self'`.

6. **Zoom management**: The Electron Store-based zoom persistence is a good UX pattern: port the `window-zoom` store approach.

7. **Notification click→navigate**: The notification click handler that restores the window and navigates to a specific session is a useful pattern for session-aware notifications.

8. **Window open handler**: The protocol whitelist (`vscode:`, `cursor:`, `zed:`, etc.) for external URL handling is a good reference. OpenDroid should support the same editor protocols.

9. **Auto-updater**: Replace `electron-updater` with OpenDroid's update mechanism. The static storage approach is simple and effective.

## Module Reference

| File | Line(s) | Symbol / Function | Notes |
|------|---------|-------------------|-------|
| main-process.js | 218 | `Iv()` | BrowserWindow creation: singleton window |
| main-process.js | 218 | `pa()` | Get or create window (show/focus/restore) |
| main-process.js | 218 | `$c()` | Return current window reference |
| main-process.js | 218 | `XT()` | Send error to renderer with retry |
| main-process.js | 218 | `KO()` | Get zoom level from store |
| main-process.js | 218 | `CS()` | Set zoom level in store |
| main-process.js | 218 | `Kgt()` | Build CSP configuration |
| main-process.js | 218 | `qgt()` | Serialize CSP to string |
| main-process.js | 218 | `eEt` class | Daemon manager: spawn, health poll, restart |
| main-process.js | 218 | `Jgt()` | Build daemon spawn command/args |
| main-process.js | 218 | `Cv()` | Get daemon port (DEFAULT_PROD_PORT) |
| main-process.js | 218 | `Zgt()` | Get daemon stderr log path |
| main-process.js | 218 | `Qgt()` | Open daemon stderr log file (with rotation) |
| main-process.js | 241 | Menu setup | `re.Menu.buildFromTemplate(n)`: minimal app menu |
| main-process.js | 242 | `BEt()` | Main lifecycle bootstrap: entry point |
| main-process.js | 242 | `LEt()` | IPC handler registration: 15+ channels |
| main-process.js | 242 | `FEt()` | Single instance lock check |
| main-process.js | 242 | `GEt()` | Windows deep link handling from argv |
| main-process.js | 242 | `iEt()` | Protocol handler registration (`opendroid://`) |
| main-process.js | 242 | `oEt()` | open-url + second-instance event handlers |
| main-process.js | 242 | `vS()` | Deep link URL router (callback, mcp-callback, navigate) |
| main-process.js | 242 | `rEt()` | Initiate OAuth flow (IdProvider) |
| main-process.js | 242 | `sEt()` | Complete OAuth flow (exchange code for token) |
| main-process.js | 242 | `nEt()` | Post-login: start daemon, set ErrTracker user |
| main-process.js | 242 | `$ne()` | Set ErrTracker user context |
| main-process.js | 242 | `tEt()` | Clear ErrTracker user context |
| main-process.js | 178 | `Z7` (AdditionalContext) | ErrTracker integration: screen info, device model |
| main-process.js | 178 | `Att` (BrowserWindowSession) | ErrTracker integration: window focus/blur session tracking |
| main-process.js | 178 | `oee` | ErrTracker electron main process integration |
| main-process.js | 242 | `UEt()` | Start daemon (alias for qu.start()) |
| main-process.js | 242 | `kEt()` | Stop daemon (alias for qu.stop()) |
| main-process.js | 242 | `xEt()` | Check if daemon is stopped |
| main-process.js | 242 | `yEt()` | CLI install flow on startup |
| main-process.js | 242 | `Pot()` | CLI install prompt |
| main-process.js | 242 | `wne()` | Window zoom store singleton |
| main-process.js | 218 | `fM` class | Electron Store wrapper (extends conf) |
