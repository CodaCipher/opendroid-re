# infra-ipc-daemon: Architecture Notes

## Overview

OpenDroid uses a **daemon-based architecture** where `opendroidd` is a long-running background process that manages droid sessions, terminals, automations, code-server instances, and tunnel connections. The daemon exposes a **WebSocket-based JSON-RPC 2.0 protocol** on localhost for bidirectional communication with GUI clients (Electron app, Desktop GUI section) and CLI clients. The IPC system follows a strict lifecycle: **daemon start → WebSocket listen → client connect → authenticate → JSON-RPC request/response → session notifications → ping/pong keepalive → graceful close**. The daemon is built on **Bun.serve()** with native WebSocket support, handling three connection types: JSON-RPC control channels, code-server proxy, and tunnel/SSH relay.

## Module Map

| Module | Size | Rol | Category |
|--------|------|-----|----------|
| 0965.js | ~188 lines | LocalDaemonClient: WebSocket client for CLI-to-daemon IPC | app/core |
| 3757.js | ~361 lines | ConnectionManager (bl): WebSocket server handler, auth, ping/pong, lifecycle | app/core |
| 3758.js | ~163 lines | DaemonServer (FWL): Bun.serve() HTTP/WS server, request routing | app/core |
| 3779.js | ~80 lines | DaemonOrchestrator (NWL): Top-level daemon startup/shutdown coordinator | app/core |
| 3754.js | ~2197 lines | DroidRequestHandler (tD): JSON-RPC method dispatcher for droid operations | app/core |
| 3755.js | ~202 lines | RpcRouter (kl) + TerminalHandler: JSON-RPC message routing and terminal ops | app/core |
| 3756.js | ~100 lines | Broadcaster (XWL) + CLI detector: Session notification fan-out | app/core |
| 0910.js | ~100 lines | Method constants: All daemon.* RPC method name enum definitions | app/core |
| 0919.js | ~280 lines | Zod schemas: Request/response validation for all daemon RPC methods | app/core |
| 0928.js | ~104 lines | Daemon starter: Auto-spawn opendroidd, port config, readiness probe | app/core |
| 0920.js | ~10 lines | Daemon config constants: DEFAULT_HOST, DEFAULT_PROD_PORT, DEFAULT_DEV_PORT | app/core |
| 3735.js | ~178 lines | AuthenticatedConnection wrapper: WebSocket + user identity + send() | app/core |
| 0001.js | ~50 lines | Constants: OpenDroidBinaryName (Droid, Opendroidd), ModelOption, etc | app/core |

> **Note:** The 6 seed modules from the feature description (0267, 3317, 2901, 3256, 1151, 1126) were misattributed: they map to SettingsManager, CSS/SCSS highlighter, AWS SSO OIDC, Less highlighter, JPEG decoder, and MCP server manager respectively. The actual IPC/Daemon modules were discovered via targeted grep patterns.

## Architecture

### Daemon Startup Sequence

```
1. CLI entry → `droid daemon` subcommand (or auto-spawned by GUI)
2. DaemonOrchestrator (NWL) created with config
   ├── TerminalManager (i1A): manages PTY terminals
   ├── DroidRegistry (hZH): tracks active droid sessions
   ├── CodeServerManager (I3): VS Code server lifecycle
   ├── DaemonServer (FWL): Bun.serve() HTTP + WebSocket
   │   ├── ConnectionManager (bl): WS auth + lifecycle
   │   │   ├── RpcRouter (kl): JSON-RPC routing
   │   │   │   ├── DroidRequestHandler (tD): session/automation ops
   │   │   │   └── TerminalHandler (VWL): terminal CRUD
   │   │   └── Broadcaster (XWL): notification fan-out
   │   ├── TunnelHandler (hBL): SSH tunnel relay
   │   └── CodeServerProxy (Yl): /code-server HTTP + WS proxy
   ├── HeartbeatService (CWL): server-side heartbeat
   └── DueRunPoller (c1A): scheduled automation polling
3. Bun.serve() binds to {host, port} or Unix socket
4. HeartbeatService starts
5. DueRunPoller starts
6. Optional: relay connection for remote computers
```

### Network Configuration

| Config | Value |
|--------|-------|
| DEFAULT_HOST | `localhost` |
| DEFAULT_PROD_PORT | `31415` |
| DEFAULT_DEV_PORT | `41832` |
| COMPUTER_PORT | `8080` |
| Ping interval | `30000ms` (30s) |
| WebSocket idle timeout | `lcD` (defined in module) |

### WebSocket Connection Types

The DaemonServer routes incoming connections based on URL path and headers:

1. **JSON-RPC** (default WebSocket): `/` → `authAdapter` → ConnectionManager
2. **Code-Server Proxy**: `/code-server/*` → CodeServerProxy (HTTP + WS upgrade)
3. **Tunnel**: `?tunnel=<port>` or `/tunnel` → TunnelHandler (SSH relay)

### JSON-RPC Message Protocol

**Wire format:** JSON over WebSocket, one message per frame.

```typescript
// Request (client → daemon)
{
  type: "request",
  jsonrpc: "2.0",            // D0 constant
  opendroidApiVersion: string, // I0 constant
  id: string,                // UUID v4
  method: "daemon.*",        // e.g. "daemon.initialize_session"
  params: { ... },
  _meta?: {
    traceparent?: string,    // OpenTelemetry trace propagation
    tracestate?: string
  }
}

// Response (daemon → client)
{
  type: "response",
  jsonrpc: "2.0",
  opendroidApiVersion: string,
  id: string,                // matches request ID
  result?: { ... },          // on success
  error?: {                  // on failure
    code: number,            // -32600 to -32603 (JSON-RPC) or -32001 (auth)
    message: string,
    data?: any
  }
}

// Notification (daemon → client, unsolicited)
{
  jsonrpc: "2.0",
  opendroidApiVersion: string,
  type: "notification",
  method: "daemon.session_notification",
  params: {
    sessionId: string,
    notification: { ... }    // session events, terminal data, permission requests
  }
}

// Or: connection status notification
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
2. AUTHENTICATE: Client sends daemon.authenticate with JWT token
   - Token verified via w1A() (JWT/ApiKey validation)
   - On success: ConnectionManager caches authenticated connection
   - On failure: Error -32001 returned
3. REQUEST/RESPONSE: Client sends JSON-RPC requests, daemon responds
   - Each request has unique UUID for correlation
   - Pending requests tracked in Map<id, {resolve, reject}>
4. NOTIFICATIONS: Daemon pushes session events to subscribed clients
   - Session notifications (stream chunks, tool results)
   - Terminal data/exit events
   - Permission/ask_user prompts requiring GUI response
5. PING/PONG: Server sends ping every 30s, client must respond with pong
   - Unauthenticated clients also get logged on pong
6. CLOSE: WebSocket closed (code 1000 = normal, 1001 = going away)
   - Terminals scheduled for cleanup with grace period
   - Droid sessions scheduled for cleanup
```

### RPC Method Routing

The `RpcRouter` (kl) routes methods to handlers:

**TerminalHandler** (VWL) handles:
- `daemon.create_terminal`
- `daemon.write_terminal_data`
- `daemon.resize_terminal`
- `daemon.close_terminal`
- `daemon.list_terminals`

**DroidRequestHandler** (tD) handles 35+ methods:
- **Session:** initialize_session, load_session, add_user_message, interrupt_session, kill_worker_session, list_opened_sessions, list_available_sessions, update_session_settings, search_sessions, archive_session, unarchive_session
- **MCP:** toggle_mcp_server, authenticate_mcp_server, cancel_mcp_auth, clear_mcp_auth, add_mcp_server, remove_mcp_server, list_mcp_registry, list_mcp_tools, toggle_mcp_tool
- **Files:** list_files, search_files
- **Automation:** list/run/pause/resume/get_history/get_visual/create/update_model/rename/delete_automation
- **Git:** get_git_diff, git_push, create_pr
- **Other:** validate_working_directory, get_default_settings, start_code_server, submit_bug_report, get/save/generate_semantic_diff, get/save_semantic_diff_cache

### Daemon Auto-Spawn (Client Side)

When a client (CLI or GUI) needs the daemon but it's not running:
1. `EmE()` checks if daemon port is reachable via TCP probe
2. If not, spawns `droid daemon --droid-path <path> [--debug] --port <port>` as detached child
3. Polls port readiness with exponential backoff (up to `AmE` timeout)
4. Returns `true` if daemon becomes ready

### Pending Request Pattern

The daemon uses a **pending request map** for interactive flows:
- `daemon.request_permission` → sent to GUI client, response awaited by droid
- `daemon.ask_user` → sent to GUI client, response awaited by droid
- Response correlated by JSON-RPC `id` field

## Key Findings

### KF-1: Bun-Native WebSocket Server (Not ws/http)
The daemon uses **`Bun.serve()`** with built-in WebSocket support, NOT the `ws` npm package. The `ws` module (module 3787) is only used for the **client-side** SSH tunnel connections. Bun's WebSocket API provides native `open/message/close/pong` handlers directly in the serve config.

### KF-2: JSON-RPC 2.0 with OpenDroid Extensions
The protocol extends JSON-RPC 2.0 with two custom fields: `opendroidApiVersion` (versioning) and `type` (request/response/notification discrimination). OpenTelemetry trace context is propagated via `_meta.traceparent/tracestate`.

### KF-3: Single-Port Multiplexing
All traffic flows through **one port**: JSON-RPC, code-server proxy, and SSH tunnels are demultiplexed by URL path (`/code-server/*`, `/tunnel`, `?tunnel=`) and HTTP `Upgrade: websocket` header inspection.

### KF-4: JWT + API Key Dual Authentication
Auth accepts either a JWT token (for browser/GUI clients) or an API key (for CLI clients), verified via `w1A()` which calls the OpenDroid API. Connection state is tracked in `WeakMap<WebSocket, AuthConnection>`.

### KF-5: Session Notification Pub/Sub
Session events (stream chunks, tool calls, permission prompts) are pushed as `daemon.session_notification` to all subscribed WebSocket connections. The `LocalDaemonClient` on the CLI side maintains `sessionNotificationHandlers: Map<sessionId, Set<handler>>` for subscription management.

### KF-6: Terminal Association Tracking
Terminals are associated with authenticated WebSocket connections. On disconnect, terminals are scheduled for cleanup with a grace period, allowing reconnection.

## Code Examples

### Client sends request (0965.js, line ~73)
```js
async sendRequest(H, A) {
  let L = await this.ensureConnected(),
    $ = rf(), // UUID v4
    I = { type: "request", jsonrpc: D0, opendroidApiVersion: I0, id: $, method: H, params: A },
    D = new Promise((E, f) => {
      this.pending.set($, { resolve: E, reject: f });
    });
  return (L.send(JSON.stringify(I)), D);
}
```

### RPC Router dispatches to handler (3755.js, line ~148)
```js
getHandlerForMethod(H) {
  if (kl.isDaemonTerminalMethod(H)) return this.terminalHandler;
  if (kl.isDaemonDroidMethod(H) || kl.isDaemonSettingsMethod(H)) return this.droidHandler;
  return null;
}
```

### Connection authentication (3757.js, line ~176)
```js
async handleAuthenticate(H, A) {
  let { token: L, apiKey: $, connectionId: I, caller: D } = A.params;
  let E = await w1A({ apiKey: $, token: L, apiBaseUrl: this.apiBaseUrl, ... });
  let f = new jBL(H, E, I); // AuthenticatedConnection wrapper
  this.cacheAuthenticatedConnection(H, f);
  this.authenticatedWebSockets.add(f);
  this.terminalManager.registerClient(f);
  return { result: { userId: E.userId, orgId: E.orgId }, authenticatedWs: f };
}
```

### Daemon auto-spawn (0928.js, line ~55)
```js
async function EmE() {
  let { host: H, port: A } = M4H();
  if (await S4$(H, A)) return true; // TCP probe
  let $ = OD().env === "development" ? "opendroid-dev" : "droid";
  let I = ["daemon", "--droid-path", $, "--port", String(A)];
  let D = tpE($, I, { detached: false, stdio: "ignore", cwd: spE.homedir(), env: { ...process.env } });
  return await DmE(H, A); // poll until ready
}
```

## Integration Points

### Cross-System Dependencies

| System | Integration | Direction |
|--------|------------|-----------|
| **GUI (M4)** | Primary consumer of daemon WebSocket. Sends requests, receives session notifications, permission prompts, terminal data | Bidirectional |
| **CLI (M1/TUI)** | LocalDaemonClient (0965.js) for headless/worker mode. Spawn worker sessions, add messages, subscribe to notifications | Client → Daemon |
| **Session Manager (infra-session-manager)** | Sessions created/loaded/restored via `daemon.initialize_session` and `daemon.load_session` | Daemon → Session |
| **Config Loader (infra-config-loader)** | Settings retrieved via `daemon.get_default_settings` and updated via `daemon.update_session_settings` | Daemon → Config |
| **Plugin Manager (infra-plugin-manager)** | MCP server lifecycle managed through `daemon.toggle_mcp_server`, `daemon.add_mcp_server`, `daemon.remove_mcp_server` | Daemon → Plugins |
| **Telemetry (infra-telemetry-otel)** | OpenTelemetry trace context propagated via `_meta.traceparent`. Spans created for each RPC method via `ED.trace()` | Daemon → Telemetry |
| **Auth (infra-auth)** | JWT verification in daemon authentication via `w1A()`. Token refresh handled by `LEH()` | Daemon → Auth |
| **Tool System (M3)** | Tools invoked within sessions managed by daemon. Permission prompts routed through daemon IPC | Daemon ↔ Tools |
| **Mission System (M2)** | Worker sessions spawned by `daemon.initialize_session` with `decompSessionType: "worker"` for mission workers | Daemon → Mission |

## Implementation Notes

1. **Replace Bun.serve()** with Node.js `http` + `ws` or `fastify-websocket`. Bun's native WebSocket API must be swapped for a portable alternative.
2. **Preserve JSON-RPC 2.0 protocol**: the message format is transport-agnostic. Keep `type`, `jsonrpc`, `opendroidApiVersion`, `id`, `method`, `params` structure.
3. **Implement daemon process manager**: the auto-spawn logic (port probe → child process → readiness check) is environment-specific. Adapt for the target runtime's process API.
4. **Authentication adapter**: JWT/API key verification calls `w1A()` which depends on the OpenDroid API. Replace with the target platform's auth system.
5. **Terminal multiplexer**: PTY management uses Bun-specific APIs. Replace with `node-pty` or equivalent for the target platform.

## Module Reference

| Module | Lines | Key Exports |
|--------|-------|-------------|
| 0965.js:1-188 | 188 | `HF$` (LocalDaemonClient), `zX()` (singleton getter) |
| 3757.js:1-361 | 361 | `bl` (ConnectionManager), `hhM` (ping interval = 30000) |
| 3758.js:1-163 | 163 | `FWL` (DaemonServer), `GWL()`, `g1A()` (connection type classifiers) |
| 3779.js:1-80 | 80 | `NWL` (DaemonOrchestrator) |
| 3754.js:1-2197 | 2197 | `tD` (DroidRequestHandler): all daemon.* RPC method implementations |
| 3755.js:1-202 | 202 | `kl` (RpcRouter), `VWL` (TerminalHandler) |
| 3756.js:1-100 | 100 | `XWL` (Broadcaster), `pcD()` (CLI detector) |
| 0910.js:1-100 | 100 | `TEH` (terminal methods), `UO` (droid methods), `VEH` (settings methods) |
| 0919.js:1-280 | 280 | Zod schemas for all daemon RPC request/response types |
| 0928.js:1-104 | 104 | `M4H()` (get daemon host:port), `EmE()` (auto-spawn daemon), `S4$()` (TCP probe) |
| 0920.js:1-10 | 10 | `_O` constants: DEFAULT_HOST, DEFAULT_PROD_PORT=31415, DEFAULT_DEV_PORT=41832 |
| 3735.js:1-40 | 40 | `jBL` (AuthenticatedConnection) |
| 0001.js:1-50 | 50 | `M9A` (OpenDroidBinaryName: Droid, Opendroidd), `$NH` (ModelOption), `I$H` (OpenDroidBinaryS3Prefix) |
