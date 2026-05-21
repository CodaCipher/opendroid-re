# Main Process Daemon Connection: GUI Architecture Notes

## Overview

This report analyzes the daemon connection architecture in the Electron main process bundle `main-process.js` (main process bundle). The critical finding is that **the main process does NOT use WebSocket to communicate with the daemon**. Instead, it uses three complementary mechanisms: (1) `child_process.spawn` to launch `opendroid.exe` as a child process, (2) HTTP health polling via `fetch()` to `http://localhost:{port}/health`, and (3) a liveness file descriptor (FD 3 pipe) passed to the daemon for process health signaling. The WebSocket connections visible in the CSP (`ws://localhost:31415`, `ws://localhost:41832`) are between the **renderer process** and the daemon: the main process never opens a WebSocket. The main process acts as a daemon lifecycle manager (spawn, health-check, restart, stop) rather than a communication intermediary. Daemon RPC method definitions (50+ methods across session, terminal, MCP, git, automation domains) are embedded in the main process bundle as Zod schema definitions, but the actual transport is renderer↔daemon, not main↔daemon.

## File Map

| File | Size | Role |
|------|------|------|
| `build/main-process.js` | main process bundle (~1.4M bytes) | Electron main process: daemon lifecycle manager |
| `build/preload.js` | small preload bridge | IPC bridge (cross-reference) |
| `build/session-search.js` | small helper chunk | Session search module (lazy-loaded by daemon RPC) |
| `<resources>/bin/opendroid.exe` | Referenced | Daemon binary (spawned by main process) |
| `~/.opendroid/logs/daemon-stderr.log` | Runtime | Daemon stderr output (rotated at 1MB) |

## Architecture

### Daemon Communication Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Electron Main Process                        │
│                      (main-process.js, main process bundle)                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  DaemonManager (eEt class)                                  │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │    │
│  │  │ spawn()      │  │ Health Poll  │  │ Signal Handling  │  │    │
│  │  │ child_process│  │ HTTP fetch   │  │ SIGTERM/SIGKILL  │  │    │
│  │  │ .spawn()     │  │ /health      │  │                  │  │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────────────┘  │    │
│  │         │                 │                                  │    │
│  │  State: Stopped → Starting → Running → Stopping → Stopped  │    │
│  │  Restart: exponential backoff (2s, 4s, 8s), max 3 attempts │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │ (stdio pipe, FD3 liveness)                                │
│         │ (stderr log file)                                         │
│         ▼                                                           │
│  ┌─────────────────────┐                                            │
│  │   opendroid.exe / opendroid  │  ← Child process of main process         │
│  │   (Daemon Process)   │                                            │
│  │   Port: DEFAULT_PORT │                                            │
│  │   HTTP: /health      │  ← Health check endpoint                  │
│  └──────┬───────────────┘                                            │
│         │                                                            │
└─────────┼────────────────────────────────────────────────────────────┘
          │ WebSocket (renderer↔daemon direct)
          │ ws://localhost:{port}
          │
┌─────────┼────────────────────────────────────────────────────────────┐
│         ▼  Renderer Process                                          │
│  ┌──────────────────────┐                                            │
│  │  React App            │  ← Connects to daemon via WebSocket       │
│  │  (renderer-main.js) │     CSP allows ws://localhost:31415       │
│  │                       │     and ws://localhost:41832              │
│  └──────────────────────┘                                            │
└──────────────────────────────────────────────────────────────────────┘

IPC Push (Main → Renderer):
  Main process pushes daemon lifecycle events to renderer via:
  webContents.send("app:error", {error})     : daemon crash/failure
  webContents.send("navigate", {path})        : routing commands
  webContents.send("auth:success", {user})    : OAuth completion
  webContents.send("mcp-oauth-callback", data): MCP OAuth
  webContents.send("app:reportBug")           : bug report trigger
```

### Key Architectural Insight

The main process is NOT in the daemon communication data path. It:
1. **Spawns** the daemon as a child process
2. **Monitors** daemon health via HTTP polling
3. **Restarts** the daemon on crash with exponential backoff
4. **Stops** the daemon on logout/quit
5. **Pushes** lifecycle error events to the renderer via IPC

The renderer connects to the daemon **directly** via WebSocket (allowed by CSP). This is why the CSP has `ws://localhost:31415` and `ws://localhost:41832` in `connect-src`: these are renderer↔daemon connections, not main↔daemon.

## Key Findings

### Finding 1: No WebSocket in Main Process

Comprehensive grep of `main-process.js` for all WebSocket patterns (`new WebSocket`, `ws://`, `wss://`, `ws+unix://`, `.on("message")`, `.on("open")`, `.on("close")`, `.on("error")`) found **zero matches** in application code. The only `ws://` references are in the CSP configuration object (`Gi`), which is a declaration for Content Security Policy headers, not actual connection code.

**Conclusion**: The main process does not communicate with the daemon via WebSocket. The daemon connection architecture is:
- **Main ↔ Daemon**: child_process.spawn (lifecycle), HTTP fetch (health), FD3 pipe (liveness)
- **Renderer ↔ Daemon**: WebSocket (JSON-RPC, actual data transport)

### Finding 2: DaemonManager (eEt) Class: Full State Machine

The `eEt` class implements a complete daemon lifecycle state machine:

```
States: Stopped → Starting → Running → Stopping → Stopped

Constants:
  MAX_RESTART_ATTEMPTS = 3
  RESTART_BACKOFF_MS = 2000 (2 seconds base)
  HEALTH_CHECK_TIMEOUT_MS = 2000 (2 seconds)
  HEALTH_POLL_INTERVAL_MS = 2000 (2 seconds)
  MAX_HEALTH_POLL_ATTEMPTS = 15
```

**Start flow**:
1. State check: must be `Stopped`
2. Set state to `Starting`
3. Build spawn command via `Jgt()`
4. Open stderr log file via `Qgt()` (with rotation at 1MB)
5. Spawn via `Qh.spawn(command, args, options)`
6. Attach `exit` and `error` handlers
7. Set state to `Running`, increment `processGeneration`
8. Start health polling

**Stop flow**:
1. Set `isShuttingDown = true`, state to `Stopping`
2. Stop health polling
3. Send `SIGTERM`
4. Set 10-second force-kill timeout (SIGKILL)
5. Wait for `exit` event
6. Set state to `Stopped`

### Finding 3: Exponential Backoff Reconnection

```js
scheduleRestart() {
  if (this.restartAttempts >= this.MAX_RESTART_ATTEMPTS) {
    // Give up after 3 attempts
    return;
  }
  this.restartAttempts++;
  const delay = this.RESTART_BACKOFF_MS * 2 ** (this.restartAttempts - 1);
  // Attempt 1: 2000ms, Attempt 2: 4000ms, Attempt 3: 8000ms
  setTimeout(() => { this.start(); }, delay);
}
```

**Reconnection behavior**:
- **Trigger**: Daemon process exits unexpectedly (not during shutdown)
- **Backoff**: Exponential: 2s, 4s, 8s
- **Max attempts**: 3 (then gives up permanently)
- **Reset on health**: Restart counter resets to 0 when health check passes
- **Tracking**: Metrics counters for `DESKTOP_DAEMON_CRASH_COUNT`, `DESKTOP_DAEMON_RESTART_COUNT`, `DESKTOP_DAEMON_RESTART_EXHAUSTED_COUNT`

### Finding 4: Health Check Protocol

```js
async checkDaemonHealth() {
  const port = Cv();  // DEFAULT_PROD_PORT
  try {
    const response = await fetch(`http://localhost:${port}/health`, {
      signal: AbortSignal.timeout(this.HEALTH_CHECK_TIMEOUT_MS)  // 2s timeout
    });
    return { healthy: response.ok, error: response.ok ? undefined : `HTTP ${response.status}` };
  } catch (error) {
    return { healthy: false, error: error.code ?? error.name ?? "unknown" };
  }
}
```

**Health polling lifecycle**:
1. Started immediately after spawn (`startHealthPoll()`)
2. Polls every 2000ms via `setInterval`
3. On success: stops polling, resets restart counter, marks daemon as healthy
4. On failure: increments attempt counter, logs warning
5. After 15 consecutive failures: stops polling, marks outcome as `daemon_health_timeout`
6. Process generation tracking prevents stale health checks from a previous daemon instance

### Finding 5: Daemon Spawn Configuration

```js
function Jgt() {
  const command = re.app.isPackaged
    ? Ze.join(process.resourcesPath, "bin", process.platform === "win32" ? "opendroid.exe" : "opendroid")
    : "opendroid-dev";
  const args = ["daemon", "--droid-path", command];
  // Development: add "--debug"
  // Non-default port: add "--port", String(port)
  args.push("--enable-code-server");
  args.push("--liveness-fd", "3");
  return { command, args, cwd: os.homedir() };
}
```

**Environment variables passed to daemon**:
```
OPENDROID_AUTO_UPDATE_ENABLED = "false"
OPENDROID_API_BASE_URL = "https://api.opendroid.dev"
OPENDROID_DEPLOYMENT_ENV = "production"
OPENDROID_OTEL_ENABLED = "true"
OPENDROID_MACHINE_TYPE = "local"
TERM = "xterm-256color"
```

**stdio configuration**: `["ignore", "inherit", stderrFd, "pipe"]`
- FD 0 (stdin): ignored
- FD 1 (stdout): inherited from parent
- FD 2 (stderr): written to log file (rotated at 1MB)
- FD 3: pipe for liveness signaling (`--liveness-fd 3`)

### Finding 6: Daemon RPC Method Catalog (50+ Methods)

The main process bundle contains Zod schema definitions for the daemon's JSON-RPC protocol. These define the method names, parameter shapes, and response shapes used between the renderer and daemon via WebSocket. Key method groups:

**Authentication**:
| Method | Description |
|--------|-------------|
| `daemon.authenticate` | Authenticate with token/apiKey/connectionId |
| `daemon.connection_status` | Get connection status, CLI version info |

**Session Management**:
| Method | Description |
|--------|-------------|
| `daemon.initialize_session` | Create new session with token, cwd, model |
| `daemon.load_session` | Load existing session |
| `daemon.add_user_message` | Send user message to session |
| `daemon.interrupt_session` | Interrupt running session |
| `daemon.close_session` | Close session |
| `daemon.kill_worker_session` | Kill worker session |
| `daemon.list_opened_sessions` | List open sessions |
| `daemon.list_available_sessions` | List available sessions |
| `daemon.update_session_settings` | Update session settings |
| `daemon.validate_working_directory` | Validate working directory |
| `daemon.compact_session` | Compact session context |
| `daemon.fork_session` | Fork a session |
| `daemon.warmup_cache` | Warm up cache for session |

**File Operations**:
| Method | Description |
|--------|-------------|
| `daemon.list_files` | List files in session directory |
| `daemon.search_files` | Search files in session |
| `daemon.search_sessions` | Full-text search across sessions |

**Terminal Management**:
| Method | Description |
|--------|-------------|
| `daemon.create_terminal` | Create terminal with cols/rows/cwd |
| `daemon.write_terminal_data` | Write data to terminal |
| `daemon.resize_terminal` | Resize terminal |
| `daemon.close_terminal` | Close terminal |
| `daemon.list_terminals` | List terminals |

**MCP (Model Context Protocol)**:
| Method | Description |
|--------|-------------|
| `daemon.get_mcp_config` | Get MCP server configuration |
| `daemon.update_mcp_config` | Update MCP configuration |
| `daemon.toggle_mcp_server` | Enable/disable MCP server |
| `daemon.authenticate_mcp_server` | Start MCP OAuth flow |
| `daemon.cancel_mcp_auth` | Cancel MCP auth |
| `daemon.clear_mcp_auth` | Clear MCP auth |
| `daemon.submit_mcp_auth_code` | Submit MCP OAuth code |
| `daemon.add_mcp_server` | Add new MCP server |
| `daemon.remove_mcp_server` | Remove MCP server |
| `daemon.list_mcp_registry` | List MCP registry |
| `daemon.list_mcp_tools` | List MCP tools |
| `daemon.list_mcp_servers` | List MCP servers |
| `daemon.toggle_mcp_tool` | Toggle MCP tool |

**Git Operations**:
| Method | Description |
|--------|-------------|
| `daemon.get_git_diff` | Get git diff for session |
| `daemon.git_push` | Push changes |
| `daemon.git_commit` | Commit changes |
| `daemon.create_pr` | Create pull request |
| `daemon.get_semantic_diff_cache` | Get cached semantic diff |
| `daemon.save_semantic_diff_cache` | Save semantic diff cache |
| `daemon.generate_semantic_diff` | Generate semantic diff |

**Automation**:
| Method | Description |
|--------|-------------|
| `daemon.list_automations` | List automations |
| `daemon.run_automation` | Run automation |
| `daemon.pause_automation` | Pause running automation |
| `daemon.resume_automation` | Resume paused automation |
| `daemon.get_automation_history` | Get automation run history |
| `daemon.get_automation_visual` | Get automation visual |
| `daemon.create_automation` | Create new automation |
| `daemon.update_automation_model` | Update automation model |
| `daemon.update_automation_privacy` | Update automation privacy |
| `daemon.rename_automation` | Rename automation |
| `daemon.delete_automation` | Delete automation |
| `daemon.fork_automation` | Fork automation |

**Other**:
| Method | Description |
|--------|-------------|
| `daemon.start_code_server` | Start code-server for session |
| `daemon.submit_bug_report` | Submit bug report |
| `daemon.get_rewind_info` | Get rewind info for message |
| `daemon.execute_rewind` | Execute rewind operation |
| `daemon.list_skills` | List available skills |
| `daemon.get_default_settings` | Get default session settings |
| `daemon.archive_session` | Archive session |
| `daemon.unarchive_session` | Unarchive session |
| `daemon.rename_session` | Rename session |
| `daemon.trigger_update` | Trigger daemon self-update |
| `daemon.install_ssh_key` | Install SSH public key |

**Daemon Push Events (daemon → renderer)**:
| Event Type | Description |
|------------|-------------|
| `daemon.session_notification` | Session state changes, messages, terminal data |
| `daemon.request_permission` | Permission request for tool execution |
| `daemon.ask_user` | Ask user for input during session |
| `daemon.relay.status_changed` | Relay connection status change |

**Session notification subtypes**:
- Session state updates (working state, message count, cwd)
- Session inactivity timeout
- Session process exit
- Session unsubscribe
- Terminal data stream
- Terminal exit

### Finding 7: Daemon Connection Lifecycle

```
1. App Start (BEt bootstrap):
   ├── app.whenReady()
   ├── ... IPC setup, window creation ...
   ├── Daemon start: qu.start()  (qu = eEt instance)
   └── Daemon monitors health

2. Login Complete (nEt):
   └── If daemon is stopped: start daemon again

3. Daemon Crash:
   ├── handleProcessExit() called
   ├── State → Stopped
   ├── Error pushed to renderer: webContents.send("app:error")
   ├── scheduleRestart() with exponential backoff
   └── After 3 failures: gives up, logs metric

4. Sign Out (auth:signOut handler):
   ├── Stop daemon: kEt() = qu.stop()
   ├── Clear ErrTracker user
   └── Clear auth tokens

5. App Quit (before-quit):
   ├── preventDefault() to delay quit
   ├── Stop daemon with 10s timeout
   ├── After 10s: force SIGKILL
   └── app.quit() or app.exit()
```

### Finding 8: CSP WebSocket Allowance (Renderer ↔ Daemon)

The Content Security Policy explicitly allows the renderer to connect to the daemon via WebSocket:

```js
connectSrc: ["'self'", "ws://localhost:31415", "ws://localhost:41832", ...]
frameSrc: ["'self'", "http://localhost:31415", "http://localhost:41832", ...]
```

Port 31415 and port 41832 are the two daemon ports. The CSP also exempts localhost connections from CSP enforcement:

```js
// CSP bypass for daemon ports:
if (url.includes("localhost:41832") || url.includes("localhost:31415")) {
  callback({ responseHeaders: details.responseHeaders });
  return;  // Skip CSP application
}
```

## Code Examples

### DaemonManager Constructor (DaemonManager region)

```js
class eEt {
  constructor() {
    this.state = pi.Stopped;
    this.process = null;
    this.isShuttingDown = false;
    this.restartAttempts = 0;
    this.healthPollInterval = null;
    this.healthPollAttempts = 0;
    this.processGeneration = 0;
    this.MAX_RESTART_ATTEMPTS = 3;
    this.RESTART_BACKOFF_MS = 2e3;
    this.HEALTH_CHECK_TIMEOUT_MS = 2e3;
    this.HEALTH_POLL_INTERVAL_MS = 2e3;
    this.MAX_HEALTH_POLL_ATTEMPTS = 15;
  }
}
```

### Health Check (DaemonManager region)

```js
async checkDaemonHealth() {
  const port = Cv();  // Cte.DEFAULT_PROD_PORT
  try {
    const response = await fetch(`http://localhost:${port}/health`, {
      signal: AbortSignal.timeout(this.HEALTH_CHECK_TIMEOUT_MS)
    });
    return { healthy: response.ok, error: response.ok ? void 0 : `HTTP ${response.status}` };
  } catch (error) {
    return { healthy: false, error: error.code ?? error.name ?? "unknown" };
  }
}
```

### Schedule Restart with Exponential Backoff (DaemonManager region)

```js
scheduleRestart() {
  if (this.restartAttempts >= this.MAX_RESTART_ATTEMPTS) {
    Pe("[daemon] Crashed too many times, giving up", { attempt: this.restartAttempts });
    Fr.addToCounter(Vs.DESKTOP_DAEMON_RESTART_EXHAUSTED_COUNT, 1);
    return;
  }
  this.restartAttempts++;
  Fr.addToCounter(Vs.DESKTOP_DAEMON_RESTART_COUNT, 1);
  const delay = this.RESTART_BACKOFF_MS * 2 ** (this.restartAttempts - 1);
  // Attempt 1: 2000ms, Attempt 2: 4000ms, Attempt 3: 8000ms
  setTimeout(() => {
    (async () => {
      try { await this.start(); }
      catch (error) { En("[daemon] Restart failed", { error }); }
    })();
  }, delay);
}
```

### Error Push to Renderer (DaemonManager region)

```js
function XT(e, t = 5, n = 500) {
  let r = 0;
  const s = () => {
    const i = $c();
    if (i && !i.isDestroyed()) {
      if (i.webContents.isLoading()) {
        r < t && (r++, setTimeout(s, n));
      } else {
        i.webContents.send("app:error", { error: e });
      }
    } else {
      r < t && (r++, setTimeout(s, n));
    }
  };
  s();
}
```

### Daemon Spawn Configuration (DaemonManager region)

```js
function Jgt() {
  const command = re.app.isPackaged
    ? Ze.join(process.resourcesPath, "bin", process.platform === "win32" ? "opendroid.exe" : "opendroid")
    : "opendroid-dev";
  const args = ["daemon", "--droid-path", command];
  re.app.isPackaged || args.push("--debug");
  const port = Cv();
  String(port) !== String(Cte.DEFAULT_PROD_PORT) && args.push("--port", String(port));
  args.push("--enable-code-server");
  args.push("--liveness-fd", "3");
  return { command, args, cwd: os.homedir() };
}
```

## Theme / Style Tokens

Not applicable for this feature: daemon connection is a main process concern with no CSS/theme involvement.

## IPC Channels

### Daemon Lifecycle → Renderer Push Channels

| Channel | Trigger | Payload | Direction | Preload Listener |
|---------|---------|---------|-----------|-----------------|
| `app:error` | Daemon crash, spawn failure, health timeout | `{error: string}` | M→R | `electronAPI.onAppError(cb)` |
| `navigate` | Notification click, deep link | `{path: string}` | M→R | `electronAPI.onNavigate(cb)` |
| `auth:success` | OAuth login complete | `{user: object}` | M→R | `electronAPI.auth.onAuthSuccess(cb)` |
| `app:reportBug` | Menu trigger | none (signal) | M→R | `electronAPI.onReportBug(cb)` |

### Daemon-Related IPC Handle Channels

| Channel | Daemon Interaction | Purpose |
|---------|-------------------|---------|
| `auth:signOut` | Stops daemon (`kEt()`) before clearing auth | Graceful logout |
| `session:readFile` | Reads daemon session JSONL files | Session data access |
| `session:searchSessions` | Delegates to `session-search.js` | Session search |
| `bugReport:getLogs` | Reads daemon log files | Bug report collection |

### Daemon RPC Channels (Renderer ↔ Daemon via WebSocket, NOT through main)

These are the 50+ JSON-RPC methods defined in the main process bundle but used by the renderer to communicate directly with the daemon via WebSocket. The main process is NOT in this data path: it only manages the daemon's lifecycle.

## Integration Points

### Cross-System Dependencies

1. **Renderer ↔ Daemon WebSocket (gui-renderer-ipc-client)**: The renderer connects to the daemon directly via WebSocket using the ports allowed by CSP (31415, 41832). The main process only spawns and monitors the daemon: it does NOT proxy daemon messages to the renderer.

2. **IPC Push Channels (gui-main-process-ipc)**: The main process pushes daemon lifecycle events (crash, error) to the renderer via `webContents.send()`. These are `app:error` for daemon failures and `navigate` for notification-triggered routing.

3. **Auth Flow (gui-main-process-core)**: The daemon is started after login (`nEt()` → `UEt()`) and stopped on logout (`auth:signOut` → `kEt()`). The daemon requires an auth token for `daemon.authenticate`.

4. **CSP Configuration (gui-main-process-core)**: The CSP `connect-src` allows `ws://localhost:31415` and `ws://localhost:41832` for renderer↔daemon WebSocket. The CSP bypass for localhost daemon ports ensures the renderer can communicate without CSP restrictions.

5. **Daemon Binary (Infrastructure section: Infrastructure)**: The daemon binary (`opendroid.exe` / `droid`) is in `<resources>/bin/`. Its internal architecture (HTTP server, WebSocket server, JSON-RPC handler) is Infrastructure section scope.

6. **Daemon Protocol (Tool & Agent section: Tool & Agent)**: The 50+ RPC methods (session management, MCP, git, automation, terminal) define the daemon's API surface. Analysis of the daemon's implementation is Tool & Agent section/Infrastructure section scope.

### Cross-Mission References

- **Daemon binary internals** → Infrastructure section (Infrastructure)
- **Daemon RPC method implementations** → Tool & Agent section (Tool & Agent)
- **Renderer WebSocket client** → Desktop GUI section (gui-renderer-ipc-client feature, pending)
- **Daemon CLI tool** → Terminal UI section (TUI): `droid` command installed to PATH

## Implementation Notes

### Porting the Daemon Manager

1. **DaemonManager class is self-contained and portable**: The `eEt` class with its state machine (Stopped→Starting→Running→Stopping), exponential backoff restart, and health polling can be directly adapted for OpenDroid. The key constants (MAX_RESTART_ATTEMPTS=3, RESTART_BACKOFF_MS=2000, HEALTH_POLL_INTERVAL=2000) are reasonable defaults.

2. **HTTP health check pattern**: The `fetch(`http://localhost:${port}/health`)` pattern with AbortSignal timeout is clean and portable. OpenDroid's daemon should expose a similar `/health` HTTP endpoint.

3. **Liveness FD pattern**: The `--liveness-fd 3` argument passes a file descriptor to the daemon for out-of-band health signaling. This is a POSIX-friendly pattern but needs consideration for Windows (named pipes alternative).

4. **Stderr log rotation**: The 1MB rotation pattern (`daemon-stderr.log` → `daemon-stderr.log.old`) is simple and effective. Port with configurable size threshold.

5. **Direct renderer↔daemon WebSocket**: The architecture where the renderer connects directly to the daemon (bypassing the main process) is efficient for high-throughput data (terminal streams, session messages). This avoids routing all daemon traffic through the main process IPC. **Recommend keeping this architecture for OpenDroid.**

6. **Daemon RPC method design**: The JSON-RPC method namespace (`daemon.*`) is clean and extensible. Adopt a similar pattern for OpenDroid with method namespacing by domain (`session.*`, `terminal.*`, `mcp.*`, `git.*`, `automation.*`).

7. **Environment variable convention**: The `FACTORY_*` environment variables passed to the daemon should be replaced with `OPENDROID_*` equivalents. Key variables: API base URL, deployment env, OTEL enabled, machine type.

8. **Process generation tracking**: The `processGeneration` counter prevents stale health checks from responding for a previous daemon instance. This is an important pattern to preserve.

## Module Reference

| File | Line(s) | Symbol / Function | Notes |
|------|---------|-------------------|-------|
| main-process.js | 207 | `cte.AUTHENTICATE` enum | Daemon authenticate method name |
| main-process.js | 207 | `ute.CONNECTION_STATUS` enum | Daemon connection status method |
| main-process.js | 207 | `ao.*` enum | Terminal RPC methods (create, write, resize, close, list) |
| main-process.js | 207 | `c$.*` enum | Terminal push events (data, exit, error) |
| main-process.js | 207 | `Qe.*` enum | 50+ daemon RPC method names (sessions, MCP, git, automation, etc.) |
| main-process.js | 207 | `fR.*` enum | Daemon push events (session_notification, request_permission, ask_user) |
| main-process.js | 207 | `u$.*` enum | Daemon admin methods (trigger_update, install_ssh_key) |
| main-process.js | 207 | `lR.*` enum | Daemon relay methods (start, stop, get_status) |
| main-process.js | 207 | `dR.*` enum | Session lifecycle events (inactivity, process_exited, unsubscribed) |
| main-process.js | 207 | `$ot` Zod schema | Daemon authenticate params (token, apiKey, connectionId) |
| main-process.js | 207 | Zod schemas | Full request/response schema definitions for all 50+ RPC methods |
| main-process.js | 218 | `Cv()` | Returns `Cte.DEFAULT_PROD_PORT`: daemon HTTP/WebSocket port |
| main-process.js | 218 | `Jgt()` | Build daemon spawn command/args/env |
| main-process.js | 218 | `Zgt()` | Get daemon stderr log file path |
| main-process.js | 218 | `Qgt()` | Open stderr log with 1MB rotation |
| main-process.js | 218 | `eEt` class | DaemonManager: full lifecycle state machine |
| main-process.js | 218 | `eEt.start()` | Spawn daemon, attach handlers, start health poll |
| main-process.js | 218 | `eEt.stop()` | SIGTERM daemon, 10s timeout, SIGKILL fallback |
| main-process.js | 218 | `eEt.handleProcessExit()` | Handle unexpected exit, push error, schedule restart |
| main-process.js | 218 | `eEt.scheduleRestart()` | Exponential backoff: 2s × 2^(n-1), max 3 attempts |
| main-process.js | 218 | `eEt.checkDaemonHealth()` | HTTP GET `http://localhost:{port}/health` with 2s timeout |
| main-process.js | 218 | `eEt.startHealthPoll()` | setInterval every 2s, track generation |
| main-process.js | 218 | `eEt.pollHealth()` | Check health, reset counter on success, exhaust after 15 fails |
| main-process.js | 218 | `XT()` | Push error to renderer via `webContents.send("app:error")` with retry |
| main-process.js | 242 | `UEt()` | Start daemon alias (qu.start()) |
| main-process.js | 242 | `kEt()` | Stop daemon alias (qu.stop()) |
| main-process.js | 242 | `xEt()` | Check if daemon is stopped |
| main-process.js | 242 | `nEt()` | Post-login: start daemon, set ErrTracker user |
| main-process.js | 218 | `Gi` (CSP config) | CSP connect-src allows `ws://localhost:31415`, `ws://localhost:41832` |
| main-process.js | 218 | `Kgt()` | CSP builder with localhost daemon port bypass |
