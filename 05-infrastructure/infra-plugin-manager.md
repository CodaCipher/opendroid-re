# infra-plugin-manager: Architecture Notes

## Overview

OpenDroid's plugin system is implemented as an MCP (Model Context Protocol) Service (1178.js). MCP is an extensible plugin architecture that provides the ability to dynamically discover, register, start, and stop external tool servers. Plugin (MCP server) discovery is realized through file-based settings management (`mcpSettingsManager`), which reads MCP server definitions from user configuration files. Lifecycle is managed through the `start() → registerAllMcpTools() → [tool registration] → cleanup()` flow. Instead of a hook system, an event emitter (`WD1` base class) is used to publish `servers_reloading`, `servers_reloaded`, `server_started`, `server_stopped`, `auth_required`, `auth_completed`, and `error` events. Each MCP server dynamically registers and unregisters its tool set, which is the foundation of OpenDroid's runtime-extensible tool architecture.

## Module Map
| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 1178.js | 691 lines | MCP Service (Plugin Manager): lifecycle, tool registration, OAuth, config sync | App |
| 0992.js | 483 lines | MCP Protocol Schema: JSON-RPC message types, notifications, capabilities | App |
| 2945.js | 774 lines | AWS STS Credential Provider: SDK plugin pattern (getSerdePlugin, getEndpointPlugin, clientPlugins) | Vendor (AWS SDK) |
| 1525.js | 984 lines | OpenTelemetry Tracer: BasicTracerProvider, SpanProcessor exports | Vendor (@opentelemetry/sdk-trace-base) |
| 3126.js | 845 lines | Gemini Response Parser: JSON parsing, thought signature handling | App (Google AI) |
| 3347.js | 58 lines | Intel x86 Assembly Syntax Highlighter (highlight.js grammar) | Vendor |
| 3288.js | 195 lines | PostgreSQL Syntax Highlighter (highlight.js grammar) | Vendor |

**Note:** 4 out of 6 seed modules were analyzed (1178, 2945, 1525, 3126). 3347 and 3288 are syntax highlighter data and are not related to the plugin system.

## Architecture

### Plugin Discovery (MCP Server Discovery)

Plugin discovery is realized through file-based configuration management:

1. **McpSettingsManager** (`Q9.getInstance()`) reads MCP server definitions from configuration files via `mcpSettingsManager.getEnabledMcpServers()`
2. Each server definition is classified by its `type` field: `"http"` (remote) or `"stdio"` (local process)
3. **SettingsManager** (`MD.getInstance()`) monitors file changes (`settings-changed` event)
4. When configuration changes, existing servers are compared against the new configuration (`hasMcpConfigChanged`)
5. New or changed servers are automatically started

```
Config File → McpSettingsManager → McpService.start() → McpHub.reloadServers()
```

### Lifecycle: register → init → hooks → dispose

**McpService Lifecycle Phases:**

1. **Construction**: `settingsManager`, `mcpSettingsManager`, `oauthStorage`, `callbackServer` initialize
2. **Start (Initialization)**:
   - `getEnabledMcpServers()` → enabled server configs are retrieved
   - `callbackServer.start()` → OAuth callback server is started
   - OAuth providers are created (for HTTP-type servers)
   - `McpHub` is created (clientInfo: `{ name: "opendroid-cli", version: VT.version }`)
   - `mcpHub.reloadServers()` → servers are started
   - `registerAllMcpTools()` → all server tools are registered
   - Marked as `initialized = true`
3. **Runtime (Tool Registration)**:
   - `registerAllMcpTools()` → registers each server's tools via `k0()` (global tool registry)
   - `unregisterServerTools(serverName)` → unregisters a single server's tools
   - `unregisterAllMcpTools()` → unregisters all server tools
4. **Config Change (Hot Reload)**:
   - When the `settings-changed` event is triggered
   - Existing tools are removed (`unregisterAllMcpTools`)
   - OAuth providers are recreated
   - Servers are reloaded
   - Tools are registered again
5. **Cleanup/Dispose**:
   - `unregisterAllMcpTools()` → all tools are removed
   - `mcpHub.closeAllServers()` → all server connections are closed
   - `callbackServer.close()` → OAuth callback is closed
   - Settings listener is removed

### Hook System (Event-Based)

Instead of a classic hook system (`onSessionStart`, `onToolCall`, etc.), OpenDroid uses an **event emitter pattern**. McpService extends the `WD1` base class and publishes the following events:

| Event | Trigger Condition | Payload |
|-------|-------------------|---------|
| `servers_reloading` | Before servers are reloaded | (none) |
| `servers_reloaded` | When server reload completes | `{ startedServers, erroredServers, serverErrors, ... }` |
| `server_started` | When a single server is started | `{ serverName, success, error? }` |
| `server_stopped` | When a single server is stopped | `{ serverName, success }` |
| `auth_required` | When OAuth authentication is required | Auth metadata |
| `auth_completed` | When OAuth authentication completes | `{ serverName, outcome, message }` |
| `error` | On general error conditions | Error object |

### Plugin Context API

The "plugin context" for each MCP server includes the following information:

- **Tool Registry**: `k0()` global tool registry with `register()` and `unregisterTool()` methods
- **Tool Creation**: `McpService.createMcpTool(serverToolDef, serverName)`, for each server tool:
  - `id`: `mcp_${serverName}_${toolName}`
  - `llmId`: `${serverName}___${toolName}`
  - `displayName`: `[MCP] ${serverName}:${toolName}`
  - `toolkit`: `MCP:${serverName}`
  - `executionLocation`: `"client"`
  - `isMcpTool: true`, `isVisibleToUser: true`, `isTopLevelTool: true`
  - `requiresConfirmation: true`
- **OAuth Context**: OAuth provider, token storage, and callback server for HTTP servers
- **Error Tracking**: Server-based error tracking via the `lastServerErrors` Map
- **Telemetry**: Counter metrics via `logMcpServerMetrics()` (`mcp_servers_to_start_count`, `mcp_reload_count`, etc.)

### Plugin Isolation

- **Server-based isolation**: Each MCP server maintains its own tool set in the `registeredToolsByServer` Map
- **Tool removal**: Only the tools of the relevant server are removed via `unregisterServerTools(serverName)`
- **OAuth isolation**: Separate OAuth provider and token storage for each server
- **Error isolation**: A server failure does not affect others; each server is handled independently via `try/catch`
- **Config isolation**: Atomic configuration changes via `withTargetedOperation()`

## Key Findings

1. **MCP = Plugin System**: OpenDroid's plugin system is based on MCP (Model Context Protocol). There is no classic "PluginManager" class; instead, `McpService` (1178.js) assumes this role.

2. **Hot Reload**: Configuration file changes are automatically detected and servers are reloaded. `SettingsManager` publishes a `settings-changed` event via file watching.

3. **AWS SDK Plugin Pattern**: 2945.js demonstrates AWS SDK's own plugin architecture (`getSerdePlugin`, `getEndpointPlugin`, `clientPlugins`). This is an SDK-level pattern distinct from OpenDroid's MCP plugin system.

4. **Tool Registration via Global Registry**: MCP tools are registered through the `k0()` global tool registry. This is the direct integration point with the Tool & Agent section (Tool System).

5. **OAuth Support**: A full OAuth flow (discovery → authentication → token storage → callback) is supported for HTTP-type MCP servers.

6. **No Traditional Hook System**: Classic hooks such as `onSessionStart`, `onToolCall`, `onModelResponse`, and `onExit` are not present. Instead, an event emitter pattern is used.

## Code Examples (minimal)

### MCP Service Construction + Start (1178.js, lines 106-185)
```js
constructor() {
  super();
  this.settingsManager = MD.getInstance();
  this.mcpSettingsManager = Q9.getInstance();
  this.oauthStorage = new wOA(oauthPath, storageMode);
  this.callbackServer = new XOA();
}

async start() {
  if (this.initialized) return;
  if (this.initializing) return;
  this.initializing = true;
  // ... setup OAuth, create McpHub, reload servers
  await this.registerAllMcpTools();
  this.initialized = true;
}
```

### Tool Registration (1178.js, lines 583-612)
```js
async registerAllMcpTools() {
  let H = await this.mcpHub.listAllTools();
  let A = k0(); // global tool registry
  for (let [L, $] of Object.entries(H)) {
    let I = NO.createMcpToolImplementations(L, $);
    for (let E of I) {
      A.register(E); // register in global tool registry
    }
    this.registeredToolsByServer.set(L, toolIds);
  }
}

unregisterServerTools(H) {
  let A = this.registeredToolsByServer.get(H);
  for (let L of A) k0().unregisterTool(L);
  this.registeredToolsByServer.delete(H);
}
```

### MCP Tool Definition (1178.js, lines 613-640)
```js
static createMcpTool(H, A) {
  return {
    id: `mcp_${A}_${H.name}`,
    llmId: `${A}___${H.name}`,
    displayName: `[MCP] ${A}:${H.name}`,
    description: H.description || `MCP tool from ${A}`,
    toolkit: `MCP:${A}`,
    executionLocation: "client",
    isMcpTool: true,
    isTopLevelTool: true,
    requiresConfirmation: true,
    isToolEnabled: true,
  };
}
```

### Config Change Handler (1178.js, lines 192-250)
```js
let H = async (A) => {
  if (this.authenticatingServer) return; // skip during auth
  if (this.targetedServerOperation) return; // skip during targeted ops
  let L = await this.mcpSettingsManager.getEnabledMcpServers();
  let $ = this.mcpHub.getUserMcpConfigs();
  if (!NO.hasMcpConfigChanged($, L)) return; // skip if unchanged
  this.unregisterAllMcpTools();
  // ... recreate OAuth providers, reload servers
  await this.registerAllMcpTools();
};
this.settingsManager.on("settings-changed", H);
```

## Integration Points (cross-system)

- **Tool & Agent section (Tool System)**: MCP tools are registered in the `k0()` global tool registry. The `register()` and `unregisterTool()` methods integrate directly with the Tool & Agent section's tool framework.
- **Orchestration section (Mission/Orchestration)**: McpService can be used for dynamic tool discovery during missions. The `getAllTools()` method lists all MCP tools.
- **Desktop GUI section (GUI)**: `enableServer`, `disableServer`, `addServer`, and `removeServer` methods are used for MCP server management from the GUI. `listServers()` and `getServerErrors()` are used to display server status in the GUI.
- **infra-config-loader**: McpSettingsManager is integrated with the config loader system (`MD.getInstance()`, `Q9.getInstance()`).
- **infra-telemetry-otel**: MCP metrics (`mcp_servers_to_start_count`, `mcp_reload_count`, etc.) are published as OpenTelemetry counters (`iL.addToCounter()`).
- **infra-ipc-daemon**: MCP tool calls can be forwarded to the daemon via IPC (for HTTP-type servers).
- **infra-auth**: The OAuth flow (discovery, token storage, callback) provides the authentication infrastructure for MCP HTTP servers.

## Implementation Notes

1. **MCP Protocol Implementation**: MCP is an open protocol (JSON-RPC based). You can support external tool servers by implementing the same protocol in OpenDroid.
2. **Tool Registry Pattern**: Instead of the `k0()` global registry, design a more explicit tool registration API in OpenDroid: `ToolRegistry.register(tool)` and `ToolRegistry.unregister(toolId)`.
3. **Config-driven Discovery**: Instead of file-based config reading for plugin discovery, use a programmatic `PluginRegistry.discover()` API in OpenDroid.
4. **Event System**: The `EventEmitter`-based hook system is suitable for MCP server lifecycle events. Provide type safety with `TypedEventEmitter` in OpenDroid.
5. **OAuth Integration**: The OAuth 2.0 flow (with PKCE) for HTTP-based plugins can be adapted from OpenDroid's existing implementation.

## Module Reference

### 1178.js: MCP Service (Plugin Manager)
| Line | Function/Field | Description |
|------|----------------|-------------|
| 26-30 | `NO class fields` | `mcpHub`, `registeredToolsByServer`, `initialized`, `initializing` fields |
| 106-118 | `constructor()` | Settings manager, OAuth storage, callback server initialization |
| 133-185 | `start()` | MCP hub creation, server reload, tool registration, config watcher |
| 192-250 | Config change handler | `settings-changed` event listener, hot reload logic |
| 272-278 | `createOAuthProvider()` | OAuth provider factory method |
| 289-300 | `cleanup()` | Closing all servers, listener cleanup |
| 310-313 | `getAllTools()` | Listing all MCP tools |
| 318-320 | `callTool()` | MCP tool invocation |
| 322-323 | `isInitialized()` | Initialization state query |
| 333-343 | `enableServer()` | Server activation + tool registration |
| 345-358 | `disableServer()` | Server deactivation + tool unregistration |
| 360-372 | `addServer()` | Adding a new server |
| 374-384 | `removeServer()` | Removing a server |
| 583-603 | `registerAllMcpTools()` | Registering all server tools in the global registry |
| 604-609 | `unregisterServerTools()` | Unregistering a single server's tools |
| 610-612 | `unregisterAllMcpTools()` | Unregistering all tools |
| 613-640 | `createMcpTool()` / `createMcpToolImplementation()` | MCP tool definition creation |

### 0992.js: MCP Protocol Schema
| Line | Function/Field | Description |
|------|----------------|-------------|
| 5-20 | Zod schemas | JSON-RPC message schemas, error codes |
| 30-60 | Capability schemas | Server/client capability definitions |

### 2945.js: AWS STS Credential Provider (SDK Plugin Pattern)
| Line | Function/Field | Description |
|------|----------------|-------------|
| 528-529 | SDK plugin calls | `getSerdePlugin`, `getEndpointPlugin`: AWS SDK plugin pattern |
| 732 | `clientPlugins` | AWS SDK client plugin configuration |

### 1525.js: OpenTelemetry Tracer (Vendor)
| Line | Function/Field | Description |
|------|----------------|-------------|
| 20-70 | Re-exports | `BasicTracerProvider`, `BatchSpanProcessor`, `ConsoleSpanExporter`, etc. |

### 3126.js: Gemini Response Parser
| Line | Function/Field | Description |
|------|----------------|-------------|
| 5-160 | JSON parsers | String/value/number parsing functions |
| 677 | `geminiThoughtSignature` | Google Gemini thought signature parsing |
