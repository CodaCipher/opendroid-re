# MCP Client Integration: Tool System Architecture Notes

## Overview

OpenDroid implements a full **Model Context Protocol (MCP) client** that connects to external MCP servers over stdio and HTTP transports, discovers tools via `listTools`, invokes them via `callTool`, and dynamically merges discovered tools into the internal tool registry under a namespaced `mcp__{server}__{tool}` convention. The client-side implementation spans three layers: the low-level **MCP Client** class (1107.js) handling protocol handshake and JSON-RPC messaging, the **McpHub** orchestrator (1126.js) managing multiple server connections and tool aggregation, and the **McpService** (1178.js) singleton bridging settings, OAuth, lifecycle events, and registry integration. Tools discovered from MCP servers are wrapped as `HZA` executor instances (1175.js) that forward invocations through the hub to the remote server with 180-second timeout, result validation, and error mapping.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 1107.js | ~531 lines | MCP Client class (Zk): protocol connect, initialize handshake, listTools, callTool, capability negotiation | App (MCP protocol client) |
| 1126.js | ~560 lines | McpHub (VOA): multi-server orchestrator, addServer/removeServer, listAllTools, callTool delegation, resource subscriptions | App (MCP orchestration) |
| 1178.js | ~400+ lines | McpService (NO): singleton service, lifecycle, settings sync, OAuth, tool registration into registry, namespace creation | App (MCP service layer) |
| 1175.js | ~100 lines | HZA executor: MCP tool execution handler, async generator, error mapping, telemetry | App (MCP tool executor) |
| 1124.js | ~100 lines | StdioClientTransport (POA): stdio transport for MCP servers, child process management | App (transport) |
| 1110.js | ~100 lines | HTTP transport setup (HOA function): HTTP/SSE transport for MCP servers with OAuth | App (transport) |
| 1125.js | ~50 lines | stdio transport factory (lq$): creates StdioClientTransport + Zk client, connects | App (transport factory) |
| 1148.js | ~80 lines | xOA result formatter: maps CallToolResult to string, handles errors/content/images | App (result formatting) |
| 0948.js | small | Constants: EG$=180000ms (callTool timeout), wvH=3000ms (SIGTERM→SIGKILL delay) | App (config) |
| 0087.js | ~170 lines | Zod schemas for MCP tool listings, toggle, JSON-RPC method definitions | App (schemas) |

## Architecture

### Connection Lifecycle

```
McpService.start()
  ├─ Create OAuth providers for HTTP servers
  ├─ new McpHub({ userMcpConfigs, clientInfo, getAuthProvider, onAuthFlowCompleted })
  ├─ mcpHub.reloadServers()
  │   ├─ Stop stale servers: mcpHub.removeServer(name)
  │   ├─ Start HTTP servers first (sequential): mcpHub.addServer(name, config)
  │   ├─ Start stdio servers (parallel): mcpHub.addServer(name, config)
  │   └─ Broadcast toolsChange to all clients
  └─ registerAllMcpTools() → k0().register(toolImpl)
```

### Transport Selection (addServer in 1126.js)

For **stdio** servers:
1. Create `StdioClientTransport` with command, args, env (1124.js)
2. Create Zk client (1107.js) with `clientInfo: { name: "opendroid-cli", version }`
3. `lq$()` factory (1125.js) wires transport → client → connect

For **HTTP** servers:
1. Create `Yt` (StreamableHTTPClientTransport) with URL, headers, authProvider
2. Create Zk client with same clientInfo
3. `HOA()` factory (1110.js) handles OAuth callback, connects

### Protocol Handshake (1107.js, Zk.connect)

```
Client.connect(transport)
  → super.connect(transport)
  → request({ method: "initialize", params: { protocolVersion, capabilities, clientInfo } })
  → Validate server protocolVersion against supported versions (yJ$)
  → Store serverCapabilities, serverVersion, instructions
  → Send notification: { method: "notifications/initialized" }
  → If listChanged configured, setup debounced refresh handlers
```

### Tool Discovery (listTools)

```
Client.listTools(params, schema)
  → request({ method: "tools/list", params }, Gt schema)
  → cacheToolMetadata(tools): validate outputSchema, classify taskSupport
  → Returns { tools: [...] }

McpHub.listAllTools()
  → Promise.allSettled(servers.map(listToolsForServer))
  → Filters disabled tools per user config (disabledTools set)
  → Returns { [serverName]: Tool[] }
```

### Tool Invocation (callTool)

```
McpHub.callTool(serverName, toolName, args, sessionId, caller)
  → servers[serverName].client.callTool({ name, arguments, _meta: { assemblySessionId, caller } }, E2, { timeout: 180000 })
  → E2.safeParse(result): validate CallToolResult schema
  → Return validated data

Client.callTool(params, resultSchema=E2, options)
  → If tool requires task execution → throw McpError (use callToolStream instead)
  → request({ method: "tools/call", params }, resultSchema, options)
  → Validate structuredContent against tool output schema if present
  → Return CallToolResult
```

### Dynamic Tool Namespace Merging (1178.js)

The namespace convention is critical:

```
Tool ID:    mcp_{serverName}_{toolName}     (e.g., mcp_github_create_issue)
LLM ID:     {serverName}___{toolName}       (e.g., github___create_issue)
Display:    [MCP] {serverName}:{toolName}
```

Registration flow:
1. `McpService.registerAllMcpTools()` calls `mcpHub.listAllTools()`
2. For each `(serverName, tools[])` pair → `createMcpToolImplementations(serverName, tools)`
3. Each tool → `createMcpTool(toolDef, serverName)` builds tool schema with:
   - `id: mcp_{serverName}_{toolName}`
   - `llmId: serverName___toolName`
   - `isMcpTool: true`
   - `requiresConfirmation: true`
   - `executionLocation: "client"`
4. `createMcpToolImplementation(toolDef, serverName)` wraps with executorFactory → `new HZA(serverName, toolName)`
5. Each implementation → `k0().register(impl)` into the central tool registry

### Timeout & Error Handling

- **callTool timeout**: 180 seconds (`EG$` in 0948.js)
- **Server kill timeout**: 3 seconds between SIGTERM and SIGKILL (`wvH`)
- **Error mapping in HZA.execute** (1175.js):
  - Service not initialized → `isError: true, errorType: "toolInternalError"`
  - MCP call exception → `llmError: "Error executing MCP tool {server}___{tool}: {msg}"`
  - Result validated via `xOA()` (1148.js): maps content to text, errors to formatted messages
  - Image content: compressed, base64 handled

### Reconnection & Config Watching

- `McpService` listens to `settings-changed` events from `settingsManager`
- On change: `hasMcpConfigChanged()` diff check → skip if unchanged
- If changed: unregister all MCP tools → recreate OAuth providers → `mcpHub.reloadServers({ force: true })` → re-register tools
- `mcpHub.reloadServers()` diff-checks configs, stops stale servers, starts new ones

## Key Findings

### 1. Namespace Function `nq$`: Server/Tool Name Joiner (1126.js, line 13)

```js
function nq$(H, A) {
  return `${H}___${A}`;
}
```

Used during `toolsChange` broadcast to create namespaced tool names:
```js
// 1126.js, line ~122
let W = Object.entries(B).flatMap(([V, X]) =>
  X.map((Q) => ({ ...Q, name: nq$(V, Q.name) }))
);
```

### 2. MCP Tool Creation with Namespace Convention (1178.js, createMcpTool)

```js
static createMcpTool(H, A) {
  let L = `mcp_${A}_${H.name}`,     // id: mcp_serverName_toolName
      $ = `${A}___${H.name}`,         // llmId: serverName___toolName
      I = p.record(p.unknown()),
      D = H.inputSchema
        ? { ...H.inputSchema, properties: H.inputSchema.properties ?? {} }
        : { type: "object", properties: {} };
  return {
    id: L,
    llmId: $,
    displayName: `[MCP] ${A}:${H.name}`,
    description: H.description || `MCP tool from ${A}`,
    inputSchema: D,
    inputZodSchema: I,
    toolkit: `MCP:${A}`,
    executionLocation: "client",
    isMcpTool: true,
    isVisibleToUser: true,
    isTopLevelTool: true,
    requiresConfirmation: true,
    outputSchemas: { result: p.string() },
    isToolEnabled: true,
  };
}
```

### 3. MCP Tool Executor (1175.js, HZA class)

```js
class HZA {
  serverName;
  toolName;
  constructor(H, A) {
    ((this.serverName = H), (this.toolName = A));
  }
  async *execute(H, A) {
    if (H.abortSignal?.aborted) throw new II();
    try {
      let L = JE();  // McpService singleton
      if (!L.isInitialized()) { /* yield error */ return; }
      let $ = await L.callTool({
        serverName: this.serverName,
        toolName: this.toolName,
        args: A,
        sessionId: H.sessionId,
      });
      // Telemetry + result formatting
      let I = await BD1($),
          D = xOA(I, { serverName: this.serverName, toolName: this.toolName });
      yield { type: "result", isError: false, value: D };
    } catch (L) {
      // Error mapping with telemetry
      yield {
        type: "result",
        isError: true,
        errorType: "toolInternalError",
        llmError: `Error executing MCP tool ${this.serverName}___${this.toolName}: ${$}`,
      };
    }
  }
}
```

### 4. CallTool with Timeout and Validation (1126.js)

```js
async callTool(H, A, L, $ = "unknown-session", I = "AGENT") {
  if (!this.servers[H]) throw new vH("Server does not exist", { name: H });
  let D = await this.servers[H].client.callTool(
      { name: A, arguments: L, _meta: { assemblySessionId: $, caller: I } },
      E2,
      { timeout: EG$ },  // 180 seconds
    ),
    E = E2.safeParse(D);
  if (!E.success)
    throw new vH("Invalid CallToolResult", {
      name: H,
      errorMessage: JSON.stringify(E.error.issues),
    });
  return E.data;
}
```

### 5. Protocol Initialize Handshake (1107.js)

```js
async connect(H, A) {
  if ((await super.connect(H), H.sessionId !== void 0)) return;
  try {
    let L = await this.request(
      {
        method: "initialize",
        params: {
          protocolVersion: hEH,
          capabilities: this._capabilities,
          clientInfo: this._clientInfo,
        },
      },
      bCA,
      A,
    );
    if (!yJ$.includes(L.protocolVersion))
      throw Error(`Server's protocol version is not supported: ${L.protocolVersion}`);
    this._serverCapabilities = L.capabilities;
    this._serverVersion = L.serverInfo;
    await this.notification({ method: "notifications/initialized" });
  } catch (L) {
    throw (this.close(), L);
  }
}
```

### 6. Dual Transport Support (1126.js, addServer)

```js
async addServer(H, A) {
  if (A.type !== "http") {
    // stdio transport: merge env, spawn child process
    let D = { ...process.env, ...(A.env || {}) };
    ({ client: L, transport: $ } = await lq$({
      serverArgs: { name: H, command: A.command, args: A.args ?? [], env: D },
      logger: this.logger?.child({ name: H }),
      clientInfo: this.clientInfo,
    }));
  } else {
    // HTTP transport: OAuth support, callback handling
    let I = this.getAuthProvider?.(H);
    ({ client: L, transport: $ } = await HOA({
      serverArgs: { name: H, url: A.url, headers: A.headers },
      logger: this.logger?.child({ name: H }),
      authProvider: I,
      onOAuthCallback: ...,
      clientInfo: this.clientInfo,
    }));
  }
  this.servers[H] = { name: H, config: A, client: L, transport: $ };
}
```

## Integration Points

- **Tool Registry (1176.js)**: MCP tools registered via `k0().register()` into the central `qgH` registry alongside built-in tools. All tools (built-in + MCP) unified via `getAllTools()`.
- **Permissions (3139.js)**: MCP tool invocations checked against autonomy levels. Tools with `___` in name are identified as MCP tools. `readOnlyHint` annotation from server checked for risk level classification.
- **Agent Runner (3419.js)**: `tool-call-start/complete/progress` events emitted for MCP tool invocations through the standard event bus (`v$`).
- **Session Management (3407.js)**: JSON-RPC methods `droid.list_mcp_tools`, `droid.toggle_mcp_tool` exposed to CLI/GUI.
- **Daemon JSON-RPC (3708.js, 3754.js)**: `daemon.list_mcp_tools`, `daemon.toggle_mcp_tool`, `daemon.add_mcp_server`, `daemon.remove_mcp_server` methods for inter-process MCP control.
- **MCP Server Config (0267.js)**: `McpSettingsManager` provides `getEnabledMcpServers()`, `addMcpServer()`, `disableMcpServer()`, tool-level disable via `disabledTools` arrays.
- **GUI (3964.js)**: React components for MCP server management UI, tool listing, enable/disable toggling.
- **Settings Schema (0087.js)**: Zod schemas for MCP tool configurations and JSON-RPC method validation.
- **OAuth/Auth (1110.js, 1178.js)**: OAuth 2.0 flow for HTTP-based MCP servers with callback server, token storage.

## Implementation Notes

### MCP Client Core API

```typescript
// Transport creation
class StdioClientTransport {
  constructor({ command, args, env, stderr, cwd })
  async start(): Promise<void>  // spawns child process
  async close(): Promise<void>
  async send(message, timeout): Promise<Response>
}

class StreamableHTTPClientTransport {
  constructor(url, { requestInit, authProvider })
  // Same interface as StdioClientTransport
}

// Client class
class McpClient {
  async connect(transport, options?): Promise<void>
  async listTools(params?, schema?): Promise<{ tools: Tool[] }>
  async callTool(params, resultSchema?, options?): Promise<CallToolResult>
  async ping(options?): Promise<void>
  getServerCapabilities(): ServerCapabilities
  registerCapabilities(capabilities): void
}

// Hub orchestrator
class McpHub {
  async addServer(name, config): Promise<void>
  async removeServer(name): Promise<void>
  async reloadServers(options?): Promise<ReloadResult>
  async listAllTools(options?): Promise<Record<string, Tool[]>>
  async callTool(serverName, toolName, args, sessionId, caller): Promise<CallToolResult>
  async closeAllServers(): Promise<void>
}

// Service singleton
class McpService {
  async start(): Promise<void>
  async cleanup(): Promise<void>
  async getAllTools(options?): Promise<Record<string, Tool[]>>
  async callTool({ serverName, toolName, args, sessionId }): Promise<CallToolResult>
  isInitialized(): boolean
  async registerAllMcpTools(): Promise<void>
  unregisterAllMcpTools(): void
}
```

### Key Constants
- `EG$ = 180000`: callTool timeout (3 minutes)
- `wvH = 3000`: SIGTERM→SIGKILL grace period for server shutdown
- Tool ID pattern: `mcp_{serverName}_{toolName}`
- LLM ID pattern: `{serverName}___{toolName}`

### Namespace Merging for Porting
When implementing OpenDroid's MCP client, the namespace convention is:
1. **Internal ID**: `mcp_{serverName}_{toolName}`: used for registry lookup
2. **LLM-facing ID**: `{serverName}___{toolName}`: sent to LLM as function name
3. **Display**: `[MCP] {serverName}:{toolName}`: shown to user
4. All MCP tools set `isMcpTool: true`, `requiresConfirmation: true`, `executionLocation: "client"`

## Open Questions / Left Undone

- **callToolStream (1105.js)**: Task-based streaming execution (`experimental.tasks.callToolStream`) traced but not deeply analyzed: deferred to streaming feature analysis.
- **E2 (CallToolResult schema)**: Defined in 0992.js with content/structuredContent/isError fields; only the structure location was identified, not the full schema definition.
- **Protocol version constants** (`hEH`, `yJ$`): Not traced to their actual values: need to grep for MCP protocol version strings.
- **Resource subscription flow**: `subscribeClientToResources` and `handleResourceUpdate` in McpHub analyzed at surface level only: resource lifecycle is secondary to tool invocation.
- **OAuth flow details** (`bOA` class, `XgH.discoverOAuthSupport`): OAuth provider creation and token management traced at service level but provider internals not deeply analyzed: belongs more to MCP server management feature.
- **MCP tool annotations impact on permissions**: `readOnlyHint` mentioned in 3139.js as affecting risk level, but full annotation-to-permission mapping not traced.
