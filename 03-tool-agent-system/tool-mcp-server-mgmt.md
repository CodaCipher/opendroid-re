# MCP Server Configuration & Lifecycle Management: Tool System Architecture Notes

## Overview

OpenDroid implements a comprehensive MCP (Model Context Protocol) server management system that handles the full lifecycle of external tool servers: from configuration parsing via `mcp.json` files, through server initialization over stdio and HTTP transports, to runtime lifecycle management (start/stop/restart/toggle). The system uses a layered settings hierarchy (org → user → project → folder) with Zod-validated schemas, automatic config-change detection and hot-reload, OAuth 2.0 authentication for HTTP servers, and a centralized `McpService` singleton (1178.js) that coordinates the `McpHub` (1126.js) transport layer, `McpSettingsManager` (0267.js), and dynamic tool registration into the global tool registry.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 1178.js | 691 lines | **McpService**: Central lifecycle orchestrator. Singleton managing server start/stop, config sync, OAuth, tool registration, metrics | app-tools |
| 1126.js | 560 lines | **McpHub (VOA)**: Transport hub. Manages stdio/HTTP connections, server add/remove, reload, resource subscriptions | app-tools |
| 0267.js | 593 lines | **McpSettingsManager (Q9)** + **SettingsProvider (MD)**: Settings CRUD, level resolution, mcp server CRUD | app-tools |
| 0076.js | 171 lines | **MCP Config Schema**: Zod schemas for stdio, HTTP transports, server config with disabled/disabledTools | app-tools |
| 0261.js | 297 lines | **Plugin MCP Loader**: `loadMcpWithWarnings()`, validates plugin mcp.json against schema | app-tools |
| 0259.js | 657 lines | **Folder MCP Loader**: `loadMcp()` / `persistMcp()` for per-folder mcp.json files | app-tools |
| 0264.js | 198 lines | **Config Merger**: `QjL()` merges mcpServers objects, `A0E()` merges individual server configs | app-tools |
| 0087.js | 278 lines | **Session/Protocol Schema**: mcpServers transport definitions for session init, daemon messages | app-tools |
| 0910.js | 70 lines | **Daemon MCP Constants**: `daemon.toggle_mcp_server`, `daemon.add_mcp_server`, etc. | app-tools |

## Architecture

### Configuration Schema (0076.js)

MCP server configurations are defined using Zod discriminated unions. The schema supports two transport types:

```js
// Module 0076.js, lines 131-145
// stdio transport
Tf0 = p.object({
  type: p.literal("stdio").optional().default("stdio"),
  command: p.string(),
  args: p.array(p.string()).optional().default([]),
  env: p.record(p.string(), p.string()).optional(),
})

// HTTP transport
Vf0 = p.object({
  type: p.literal("http"),
  url: p.string(),
  headers: p.record(p.string(), p.string()).optional(),
})

// Discriminated union + server config overlay
Xf0 = p.discriminatedUnion("type", [Tf0, Vf0])
Gf0 = Xf0.and(p.object({
  disabled: p.boolean().optional().default(false),
  disabledTools: p.array(p.string()).optional(),
}))

// Root schema: mcp.json shape
IFL = p.object({ mcpServers: p.record(p.string(), Gf0) })
```

The session-level schema (0087.js) adds a third transport variant: SSE:
```js
// Module 0087.js, lines 41-46
oM0 = AH.object({
  type: AH.literal("sse"),
  name: AH.string(),
  url: AH.string().url(),
  headers: VFL.array().default([]),
})
// Union: [stdio (nM0), http (rM0), sse (oM0)]
aNH = AH.union([nM0, rM0, oM0]).array()
```

### Configuration Sources & Merge Hierarchy

MCP configs come from multiple sources and are merged in priority order:

1. **System/Org level**: managed via server, read-only locally
2. **User level**: `~/.opendroid/settings.json` → mcp section
3. **Project level**: `.opendroid/settings.json`
4. **Folder level**: per-folder settings
5. **Plugin mcp.json**: `plugins/<plugin>/mcp.json`

The merge function `QjL()` (0264.js) implements right-to-left priority:
```js
// Module 0264.js, lines 113-116
function QjL(H, A) {
  let L = {};
  for (let [$, I] of Object.entries(A.mcpServers)) if (!($ in H.mcpServers)) L[$] = I;
  for (let [$, I] of Object.entries(H.mcpServers)) L[$] = I;
  return { mcpServers: L };
}
```

### mcp.json File I/O (0259.js, 0261.js)

Per-folder mcp.json is loaded and persisted by the folder settings source:
```js
// Module 0259.js, lines 276-309
async loadMcp() {
  let H = q0.join(this.folderPath, eTH); // eTH = "mcp.json"
  try {
    if (!(await b7(H))) return;
    let L = await NU.promises.readFile(H, "utf-8"),
      $ = JSON.parse(L);
    if (!$.mcpServers || typeof $.mcpServers !== "object") return { mcpServers: {} };
    return { mcpServers: $.mcpServers };
  } catch (A) {
    gH("Failed to load mcp.json", { path: H, error: A });
    return;
  }
}

async persistMcp(H) {
  let A = q0.join(this.folderPath, eTH),
    L = JSON.stringify(H, null, 2);
  try { await ky(A, L); }
  catch ($) { uH($, "Failed to write mcp.json", { path: A }); }
}
```

Plugin mcp.json loading (0261.js) validates against the `IFL` schema:
```js
// Module 0261.js, lines 240-262
async loadMcpWithWarnings(H, A) {
  let L = [], $ = lJ.join(A, _jL); // _jL = "mcp.json"
  try {
    if (!(await b7($))) return { data: void 0, warnings: L };
    let I = await fr.promises.readFile($, "utf-8"),
      D = JSON.parse(I),
      E = IFL.safeParse(D);
    if (!E.success)
      return (
        gH("Invalid mcp.json schema", { path: $, error: E.error.message }),
        L.push({ pluginId: H, component: "mcp", file: _jL, error: "Invalid MCP configuration." }),
        { data: void 0, warnings: L }
      );
    if (!E.data.mcpServers) return { data: void 0, warnings: L };
    return { data: { mcpServers: E.data.mcpServers }, warnings: L };
  } catch { return { data: void 0, warnings: L }; }
}
```

### McpSettingsManager (0267.js): Server CRUD

The `Q9` singleton wraps the settings provider to expose MCP-specific CRUD operations:

```js
// Module 0267.js, lines 424-490
class Q9 {
  async getMcpServers() {
    return (await this.settingsProvider.getResolvedSettings()).mcp?.mcpServers ?? {};
  }
  async getEnabledMcpServers() {
    let H = await this.getMcpServers();
    return Object.fromEntries(Object.entries(H).filter(([A, L]) => !L.disabled));
  }
  async addMcpServer(H, A, L) {
    if (L !== "user") throw new vH("Cannot add MCP servers at non-user level");
    let $ = W_(H),
      D = (await this.settingsProvider.getLevelSettings(L)).mcp?.mcpServers ?? {};
    await this.settingsProvider.updateLevelSettings(L, {
      mcp: { mcpServers: { ...D, [$]: { ...A, disabled: A.disabled ?? false } } },
    });
  }
  async removeMcpServer(H, A) {
    if (A !== "user") throw new vH("Cannot remove MCP servers at non-user level");
    let L = W_(H),
      I = (await this.settingsProvider.getLevelSettings(A)).mcp?.mcpServers ?? {};
    if (!(L in I)) return false;
    await this.settingsProvider.updateLevelSettings(A,
      { mcp: { mcpServers: { [L]: void 0 } } });
    return true;
  }
  async enableMcpServer(H, A) { return this.updateMcpServer(H, false, A); }
  async disableMcpServer(H, A) { return this.updateMcpServer(H, true, A); }
}
```

Key constraints: Add/remove are **user-level only**. Enable/disable can override project/folder servers but persist to user level via `ensureMcpServerInUserLevel()`.

### McpHub (1126.js): Transport Layer

The `VOA` class manages actual MCP server connections. Server startup differs by transport:

**stdio transport:**
```js
// Module 1126.js, lines 183-196
if (A.type !== "http") {
  let I = { ...process.env, ...(A.env || {}) },
    D = I ? Object.fromEntries(
      Object.entries(I).filter(([, E]) => E !== void 0).map(([E, f]) => [E, String(f)])
    ) : void 0;
  ({ client: L, transport: $ } = await lq$({
    serverArgs: { name: H, command: A.command, args: A.args ?? [], env: D },
    logger: this.logger?.child({ name: H }),
    clientInfo: this.clientInfo,
  }));
}
```

**HTTP transport (with OAuth):**
```js
// Module 1126.js, lines 197-216
} else {
  let I = this.getAuthProvider?.(H),
    D = I?.enableInteractiveAuth ?? false;
  let E = I && D ? async () => await I.waitForAuthorizationCode() : void 0;
  ({ client: L, transport: $ } = await HOA({
    serverArgs: { name: H, url: A.url, headers: A.headers },
    logger: this.logger?.child({ name: H }),
    authProvider: I,
    onOAuthCallback: E,
    clientInfo: this.clientInfo,
    onAuthFlowCompleted: (f) => {
      this.onAuthFlowCompleted?.({ serverName: H, ...f });
    },
  }));
}
```

**Server removal and process cleanup:**
```js
// Module 1126.js, lines 240-275
async removeServer(H) {
  let L = this.servers[H].transport,
    $ = $$1(L); // gets PID from stdio transport
  try {
    if (L && typeof L.close === "function") {
      await L.close();
      if ($ !== null && rq$($)) // checks if process still running
        await this.killServerProcessTree($, H);
    }
  } finally {
    delete this.availableResources[H];
    delete this.servers[H];
  }
}
```

### Server Reload Strategy (1126.js)

`reloadServers()` implements intelligent diff-based reload:
```js
// Module 1126.js, lines 59-97
async reloadServers(H) {
  // 1. Identify servers to stop (config changed or force flag)
  let E = Object.entries(this.servers).filter(([B, { config: W }]) => {
    if (H?.force) return true;
    if (B in this.systemMcpConfigs && TOA.default.isEqual(W, this.systemMcpConfigs[B]))
      return (I.add(B), false); // unchanged
    if (B in this.userMcpConfigs && TOA.default.isEqual(W, this.userMcpConfigs[B]))
      return (I.add(B), false); // unchanged
    return true; // needs restart
  });
  // 2. Stop stale servers in parallel
  await Promise.all(E.map(async ([B]) => { await this.removeServer(B); }));
  // 3. Start new servers (HTTP first sequentially, then stdio in parallel)
  let M = f.filter(([, B]) => B.type === "http"),
    U = f.filter(([, B]) => B.type === "stdio");
  for (let [B, W] of M) await this.addServer(B, W); // sequential for OAuth
  await Promise.all(U.map(([B, W]) => this.addServer(B, W))); // parallel
  // 4. Notify clients of tool changes
  await this.notifyAll({ method: "toolsChange", params: { tools: W } });
}
```

### McpService (1178.js): Lifecycle Orchestrator

The `NO` class is the top-level singleton that ties everything together:

**Initialization flow:**
```js
// Module 1178.js, lines 130-180
async start() {
  let A = await this.mcpSettingsManager.getEnabledMcpServers();
  // Start OAuth callback server
  await this.callbackServer.start();
  // Create OAuth providers for HTTP servers
  for (let [D, E] of Object.entries(A))
    if (E.type === "http") this.oauthProviders.set(D, this.createOAuthProvider(...));
  // Create McpHub
  this.mcpHub = new VOA({
    userMcpConfigs: A,
    clientInfo: { name: "opendroid-cli", version: VT.version },
    getAuthProvider: (D) => this.oauthProviders.get(D),
    onAuthFlowCompleted: ({ serverName: D, outcome: E }) => { ... },
  });
  // Initial reload
  let I = await this.mcpHub.reloadServers();
  // Register all MCP tools into global registry
  await this.registerAllMcpTools();
  // Watch for settings changes
  this.settingsManager.on("settings-changed", H);
  this.settingsManager.enableWatching();
}
```

**Lifecycle operations:**
```js
// Module 1178.js, lines 327-385
async enableServer(H, A) {
  await this.withTargetedOperation(async () => {
    await this.mcpSettingsManager.enableMcpServer(L, A);
    await this.syncConfigAndEnsureOAuthProvider(L);
    await this.startServerAndEmitResult(L);
    await this.registerAllMcpTools();
  });
}
async disableServer(H, A) {
  await this.withTargetedOperation(async () => {
    await this.mcpSettingsManager.disableMcpServer(L, A);
    await this.syncConfigWithoutReload();
    if (I?.type === "http") await this.oauthStorage.clearAll(L, I.url);
    await this.stopServerAndEmitResult(L);
    this.unregisterServerTools(L);
  });
}
async addServer(H, A) {
  await this.withTargetedOperation(async () => {
    await this.mcpSettingsManager.addMcpServer(H, A, "user");
    await this.syncConfigWithoutReload();
    if (A.type === "http") this.oauthProviders.set(H, ...);
    await this.startServerAndEmitResult(H);
    await this.registerAllMcpTools();
  });
}
async removeServer(H, A) {
  await this.withTargetedOperation(async () => {
    await this.mcpSettingsManager.removeMcpServer(L, A);
    await this.stopServerAndEmitResult(L);
    this.unregisterServerTools(L);
  });
}
```

**Config-change auto-reload:**
When settings files change, the McpService detects diffs and hot-reloads:
```js
// Module 1178.js, lines 209-260
let H = async (A) => {
  if (this.authenticatingServer) return; // skip during auth
  if (this.targetedServerOperation) return; // skip during targeted ops
  let L = await this.mcpSettingsManager.getEnabledMcpServers(),
    $ = this.mcpHub.getUserMcpConfigs();
  if (!NO.hasMcpConfigChanged($, L)) return; // skip if unchanged
  this.unregisterAllMcpTools();
  // Recreate OAuth providers
  this.oauthProviders.clear();
  for (let [E, f] of Object.entries(L))
    if (f.type === "http") this.oauthProviders.set(E, ...);
  // Force reload
  let D = await this.mcpHub.reloadServers({ force: true });
  this.emit("servers_reloaded", D);
  await this.registerAllMcpTools();
};
```

### Dynamic Tool Registration

MCP server tools are registered into the global tool registry with namespaced IDs:

```js
// Module 1178.js, lines 616-640
static createMcpTool(H, A) {
  let L = `mcp_${A}_${H.name}`,       // tool id: mcp_{server}_{tool}
    $ = `${A}___${H.name}`,            // llm id: {server}___{tool}
  return {
    id: L,
    llmId: $,
    displayName: `[MCP] ${A}:${H.name}`,
    description: H.description || `MCP tool from ${A}`,
    inputSchema: D,
    toolkit: `MCP:${A}`,
    executionLocation: "client",
    isMcpTool: true,
    requiresConfirmation: true,
  };
}
```

### Daemon Protocol Commands (0910.js)

The daemon exposes MCP management via IPC commands:
```js
// Module 0910.js, lines 22-29
RH.TOGGLE_MCP_SERVER = "daemon.toggle_mcp_server";
RH.AUTHENTICATE_MCP_SERVER = "daemon.authenticate_mcp_server";
RH.CANCEL_MCP_AUTH = "daemon.cancel_mcp_auth";
RH.CLEAR_MCP_AUTH = "daemon.clear_mcp_auth";
RH.ADD_MCP_SERVER = "daemon.add_mcp_server";
RH.REMOVE_MCP_SERVER = "daemon.remove_mcp_server";
RH.LIST_MCP_REGISTRY = "daemon.list_mcp_registry";
RH.LIST_MCP_TOOLS = "daemon.list_mcp_tools";
RH.TOGGLE_MCP_TOOL = "daemon.toggle_mcp_tool";
```

## Key Findings

### Finding 1: Two Transport Types with Distinct Initialization

**stdio transport** spawns a child process with configurable command, args, and environment variables. The child process PID is tracked for cleanup via `killServerProcessTree()` using SIGTERM/SIGKILL cascade.

**HTTP transport** connects to a remote URL with optional headers and full OAuth 2.0 PKCE flow support. OAuth providers are created per-server and managed through a callback server.

### Finding 2: `withTargetedOperation()` Guards Config Reload

The `withTargetedOperation()` method temporarily disables file watching during add/remove/enable/disable operations to prevent the settings-change listener from triggering a duplicate reload. It uses a counter (`pendingSettingsChangeIgnoreCount`) to skip exactly the right number of change events.

### Finding 3: HTTP Servers Start Sequentially, stdio in Parallel

During reload, HTTP servers are started **sequentially** (one at a time) because they may require OAuth flows that involve user interaction. stdio servers are started **in parallel** via `Promise.all()` since they're local processes.

### Finding 4: Plugin mcp.json Adds Server Definitions

Plugins can ship their own `mcp.json` file defining MCP servers. The `loadMcpWithWarnings()` method validates against the `IFL` Zod schema and reports validation errors as warnings without crashing.

### Finding 5: Tool-level Disable Permits Granular Control

Individual tools within an MCP server can be disabled via `disabledTools: string[]` on the server config. The `toggleTool()` method enables/disables individual tools and updates settings without full server restart.

### Finding 6: Metrics Tracking

Server lifecycle operations are tracked with counters:
```js
// Module 1178.js, lines 106-111
iL.addToCounter("mcp_servers_to_start_count", L, E),
iL.addToCounter("mcp_servers_start_errored_count", $, E),
iL.addToCounter("mcp_servers_to_stop_count", I, E),
iL.addToCounter("mcp_servers_stop_errored_count", D, E),
iL.addToCounter("mcp_servers_unchanged_count", H.unchangedServers.length, E),
iL.addToCounter("mcp_reload_count", 1, E),
```

## Integration Points

- **Tool Registry (3264.js)**: MCP tools registered via `k0().register()` with `mcp_{server}_{tool}` namespace. `k0().unregisterTool()` for cleanup.
- **MCP Client (tool-mcp-client feature)**: `McpHub.listAllTools()` and `McpHub.callTool()` are the client-side MCP protocol methods that the tool executor calls.
- **Permissions (tool-permissions feature)**: MCP tools set `requiresConfirmation: true`, routing through the permission system.
- **Sandbox (tool-sandbox feature)**: MCP tool executors (`HZA` class) run within the sandbox framework.
- **Settings/Infra (05-infrastructure)**: Settings provider, file watching, level resolution are infrastructure modules.
- **GUI (04-desktop-gui)**: MCP server management UI (add/remove/toggle/auth) sends daemon protocol messages.
- **Daemon IPC**: All MCP management operations exposed as `daemon.*` commands via the daemon protocol.

## Implementation Notes

### MCP Server Configuration API
```typescript
// Schema: { mcpServers: Record<string, ServerConfig> }
interface McpServerConfig {
  type: "stdio" | "http";
  // stdio fields
  command?: string;
  args?: string[];
  env?: Record<string, string>;
  // http fields
  url?: string;
  headers?: Record<string, string>;
  // common
  disabled?: boolean;
  disabledTools?: string[];
}
```

### Lifecycle Management
```typescript
// Add server (user-level only)
await mcpSettingsManager.addMcpServer(name, config, "user");
// Enable/disable
await mcpService.enableServer(name, level);
await mcpService.disableServer(name, level);
// Remove
await mcpService.removeServer(name, level);
// Toggle individual tools
await mcpService.toggleTool(serverName, toolName, enabled);
```

### Settings File Format (mcp.json)
```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["server.js"],
      "env": { "API_KEY": "..." }
    },
    "remote-server": {
      "type": "http",
      "url": "https://example.com/mcp",
      "headers": { "Authorization": "Bearer ..." }
    }
  }
}
```

### Key Classes to Port
1. `McpService` (1178.js) → lifecycle orchestrator singleton
2. `McpHub` (1126.js) → transport connection manager
3. `McpSettingsManager` (0267.js) → settings CRUD wrapper
4. Config schemas (0076.js) → Zod/JSON-Schema validation

## Open Questions / Left Undone

1. **OAuth provider internals** (`bOA`, `XOA`, `wOA` classes) not fully traced: these handle the OAuth 2.0 PKCE flow, token storage, and callback server.
2. **SSE transport** is defined in session schema (0087.js) but may not be fully implemented in `McpHub.addServer()`: only stdio and http are explicitly handled there.
3. **`lq$()` and `HOA()` transport factory functions** (in 1126.js deps) not traced: these wrap the actual MCP SDK client creation for stdio and HTTP respectively.
4. **Resource subscription system** (`clientResourceSubscriptions`) partially analyzed: the subscription/notification flow for MCP resources needs deeper investigation.
5. **ACP (Agent Configuration Protocol) merge**: `setMergedMcpConfigs()` allows external config injection; the source of these configs not yet traced.
