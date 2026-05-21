# tui-panel-mcp-manager: TUI Architecture Notes

## Overview

The MCP Manager is a two-panel TUI subsystem for managing Model Context Protocol (MCP) servers and their tools within OpenDroid's interactive CLI. It consists of a server list view (3959.js) that displays connected/disabled/connecting states and supports add/remove flows, and a tool management view (3963.js) that lists all tools grouped by server with per-tool and per-server enable/disable toggling. The management logic is layered: RPC method constants (0910.js) define the daemon ↔ TUI contract, Zod schemas (0916.js) validate RPC payloads, a JsonRpc handler layer (3708.js) bridges the TUI to the service, McpService (1178.js) orchestrates server lifecycle (add/remove/enable/disable/retry), and McpSettingsManager (0267.js) persists configuration to user-level settings files with `mcp.mcpServers` structure.

## Module Map
| Module | Size | Role | Vendor/App |
|-------|-------|-----|------------|
| 3963.js | ~283 lines | MCP Tools panel: tool list rendering, per-tool/per-server toggle UI | App |
| 3959.js | ~66 lines | MCP Server list panel: server status display, add/registry navigation | App |
| 0910.js | ~91 lines | RPC method constants: daemon MCP methods (GET_MCP_CONFIG, ADD_MCP_SERVER, etc.) | App |
| 0916.js | ~85 lines | RPC Zod schemas: MCP config/tool/server request/response validation | App |
| 3708.js | ~929 lines | JsonRpc handler: MCP status/auth event listeners, server CRUD handlers | App |
| 3754.js | ~2197 lines | JsonRpc request dispatcher: toggleMcpTool delegation to daemon | App |
| 1178.js | ~691 lines | McpService: server lifecycle orchestrator (add/remove/enable/disable/retry) | App |
| 0267.js | ~593 lines | McpSettingsManager: settings persistence, enable/disable tools CRUD | App |

## Architecture

The McpManager panel follows a layered architecture:

```
┌─────────────────────────────────────────────────┐
│  TUI Layer (React Ink)                          │
│  ┌──────────────┐    ┌─────────────────────────┐ │
│  │ 3959.js      │    │ 3963.js                 │ │
│  │ Server List  │───▶│ Tool List + Toggles     │ │
│  │ Panel        │    │ (scrollable, 12 items)   │ │
│  └──────────────┘    └─────────────────────────┘ │
│         │                      │                  │
│    useInput (jL)          useInput (jL)          │
└─────────┼──────────────────────┼──────────────────┘
          │                      │
          ▼                      ▼
┌─────────────────────────────────────────────────┐
│  RPC Layer                                      │
│  0910.js (constants) ─── 0916.js (schemas)      │
│          │                      │                │
│          ▼                      ▼                │
│  3708.js (JsonRpc handlers)                     │
│  3754.js (toggleMcpTool delegation)             │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│  Service Layer                                  │
│  1178.js (McpService) ── server lifecycle       │
│  0267.js (McpSettingsManager) ── persistence     │
└─────────────────────────────────────────────────┘
```

**Server List View** (3959.js `ptD` component): Renders a selectable list of configured MCP servers. Each entry shows name + status suffix (`connected`, `disabled`, `connecting...`, `disconnected · Enter to login`). Extra entries: "Manage All Tools", "+ Add MCP server from registry", "+ Add MCP server manually". Uses `f3` panel wrapper.

**Tool List View** (3963.js `rtD` component): Groups tools by server name (sorted alphabetically). Each server header shows `serverName (X/Y enabled)`. Tools show checkbox `[✓/ ] toolName [read-only]`. Virtual scrolling with 12-item window. Supports toggling individual tools, all tools (A key), or per-server (S key).

## Key Components

### 3963.js: MCP Tools Panel (`rtD` function, lines 5-282)

Tool management rendering with virtual scrolling:

```js
// Module 3963.js, lines 34-55: Tool grouping and status computation
let M = IMA.useMemo(
    () =>
      Object.entries(H)
        .filter(([w, C]) => C.length > 0)
        .map(([w, C]) => {
          let Y = A[w] ?? new Set(),
            q = C.map((N) => ({
              serverName: w,
              toolName: N.name,
              description: N.description,
              isEnabled: !Y.has(N.name),
              isReadOnly: N.annotations?.readOnlyHint === true,
            }));
          return {
            serverName: w,
            tools: q,
            enabledCount: q.filter((N) => N.isEnabled).length,
            totalCount: C.length,
          };
        })
        .sort((w, C) => w.serverName.localeCompare(C.serverName)),
    [H, A],
  ),
```

Empty state rendering:

```js
// Module 3963.js, lines 109-123: Empty state
if (P === 0)
    return kT.jsxDEV(a, { flexDirection: "column", marginTop: 1, children: [
        kT.jsxDEV(u, { bold: true, children: "MCP Tools (0 tools)" }),
        kT.jsxDEV(a, { marginTop: 1, children: kT.jsxDEV(u, {
            color: o.text.muted,
            children: "No MCP tools available. Add MCP servers to see their tools.",
        })}),
        kT.jsxDEV(a, { marginTop: 1, children: kT.jsxDEV(u, {
            color: o.text.muted, children: "Press ESC to go back"
        })}),
    ]});
```

### 3959.js: MCP Server List Panel (`ptD` function, lines 8-65)

Server list with status indicators and action entries:

```js
// Module 3959.js, lines 25-41: Server status rendering
$.forEach((E, f) => {
    let M = "", U = L.has(E.name);
    if (E.isDisabled) M = " (disabled)";
    else if (E.isConnected) M = " (connected)";
    else if (U) M = " (connecting...)";
    else if (E.authStatus === "needs_auth") M = " (disconnected · Enter to login)";
    else M = " (disconnected)";
    let P = E.source === "org" ? " [Org]" : E.isManaged ? " [Project]" : "";
    D.push({ label: `${f + 1}. ${E.name}${P}`, value: E.name, suffix: M });
});
D.push({ label: "Manage All Tools", value: "__manage_all_tools__" });
D.push({ label: "+ Add MCP server from registry", value: "__add_server_from_registry__" });
D.push({ label: "+ Add MCP server manually", value: "__add_server_manually__" });
```

### 0910.js: RPC Method Constants (lines 20-33)

```js
// Module 0910.js, lines 20-33: MCP daemon RPC methods
RH.GET_MCP_CONFIG = "daemon.get_mcp_config";
RH.UPDATE_MCP_CONFIG = "daemon.update_mcp_config";
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

### 0916.js: MCP RPC Zod Schemas (lines 1-85)

Two server config types are supported:

```js
// Module 0916.js, lines 7-19: Server config schemas
acE = AH.object({
    type: AH.literal("stdio").optional().default("stdio"),
    command: AH.string(),
    args: AH.array(AH.string()),
    env: AH.record(AH.string(), AH.string()).optional(),
    disabled: AH.boolean().optional().default(false),
});
scE = AH.object({
    type: AH.literal("http"),
    url: AH.string(),
    headers: AH.record(AH.string(), AH.string()).optional(),
    disabled: AH.boolean().optional().default(false),
});
y3$ = AH.union([acE, scE]); // Server config = stdio | http
```

### 3708.js: JsonRpc MCP Handlers (lines 89-105, 655-795)

Event listener setup for MCP status and auth:

```js
// Module 3708.js, lines 89-100: MCP event listeners
setupMcpStatusListeners() {
    let H = JE();
    this.mcpEventCleanup = F7D((A) => {
        v$.emit("mcp-status-changed", { notification: A });
    }, H);
}
setupMcpAuthListeners() {
    let H = JE();
    // emits "mcp-auth-required" events for OAuth flows
```

Add server handler:

```js
// Module 3708.js, lines 660-681: Add MCP server
async handleAddMcpServer(H) {
    let { name: A, type: L, url: $, command: I, args: D, env: E } = H.params;
    let U;
    if (L === "http") {
        U = { type: "http", url: $, headers: f, disabled: false };
    } else {
        if (!D) { /* error: command required for stdio */ }
        U = { type: "stdio", command: D, args: E || [], env: f, disabled: false };
    }
    await M.addServer(A, U);
}
```

### 1178.js: McpService Lifecycle (lines 310-395)

Server lifecycle operations:

```js
// Module 1178.js, lines 327-340: Enable server flow
async enableServer(H, A) {
    let L = W_(H);
    await this.withTargetedOperation(async () => {
        if (!(await this.mcpSettingsManager.enableMcpServer(L, A)))
            throw new vH("MCP server not found in configuration");
        await this.syncConfigAndEnsureOAuthProvider(L);
        await this.startServerAndEmitResult(L);
        await this.registerAllMcpTools();
    });
}

// Module 1178.js, lines 366-380: Remove server flow
async removeServer(H, A) {
    let L = W_(H);
    await this.withTargetedOperation(async () => {
        if (!(await this.mcpSettingsManager.removeMcpServer(L, A)))
            throw new vH("MCP server not found in configuration");
        await this.syncConfigWithoutReload();
        await this.stopServerAndEmitResult(L);
        this.unregisterServerTools(L);
        this.oauthProviders.delete(L);
    });
}
```

Tool toggle:

```js
// Module 1178.js, lines 387-393: Toggle tool
async toggleTool(H, A, L) {
    await this.withTargetedOperation(async () => {
        if (L) await this.mcpSettingsManager.enableMcpTools(H, [A]);
        else await this.mcpSettingsManager.disableMcpTools(H, [A]);
        await this.syncConfigWithoutReload();
    });
}
```

### 0267.js: McpSettingsManager (lines 425-550)

Settings persistence for MCP config:

```js
// Module 0267.js, lines 475-485: Add MCP server to user settings
async addMcpServer(H, A, L) {
    if (L !== "user") throw new vH("Cannot add MCP servers at non-user level");
    let D = (await this.settingsProvider.getLevelSettings(L)).mcp?.mcpServers ?? {};
    await this.settingsProvider.updateLevelSettings(L, {
        mcp: { mcpServers: { ...D, [$]: { ...A, disabled: A.disabled ?? false } } },
    });
}
```

Tool enable/disable tracking via `disabledTools` array:

```js
// Module 0267.js, lines 511-527: Enable MCP tools
let f = new Set(E.disabledTools ?? []), M = false;
for (let U of A) {
    if (f.has(U)) { f.delete(U); M = true; }
    $[U] = true;
}
if (M) {
    await this.settingsProvider.updateLevelSettings("user", {
        mcp: { mcpServers: { ...U, [L]: { ...E, disabledTools: f.size > 0 ? Array.from(f) : void 0 } } },
    });
}
```

## Theme / Style Tokens

The MCP panels use the shared theme object `o`:

| Token | Usage | Context |
|-------|-------|---------|
| `o.text.muted` | Muted/help text, status messages, empty state text | Gray/dim color |
| `o.text.primary` | Normal tool item text | Default foreground |
| `o.primary` | Selected/highlighted item (server headers, cursor) | Accent color |
| `bold: true` | Panel title "Manage MCP servers", "MCP Tools (X/Y enabled)" | Bold text |
| `dimColor: !isEnabled` | Disabled tools rendered dimmed | Opacity/dimming for disabled state |

Panel component `f3` provides standard frame with `helpText` and `fullWidth: true`.

## Keyboard / Input Handling

### Server List Panel (3959.js)

| Key | Action |
|-----|--------|
| ↑/↓ | Navigate server list |
| Enter | View server details / authenticate (for `needs_auth` servers) |
| ESC | Exit panel |

Navigation bounded by `Math.max(0, M - 1)` and `Math.min(I - 1, M + 1)` where `I = servers.length + 3`.

### Tool List Panel (3963.js)

| Key | Action |
|-----|--------|
| ↑/↓ | Navigate tool list |
| Space | Toggle individual tool (on tool item) or toggle all tools for server (on server header) |
| A / Shift+A | Toggle all tools across all servers |
| S / Shift+S | Toggle all tools for the current server |
| Enter | View tool detail |
| ESC | Go back |

Virtual scrolling: 12-item viewport (`V = 12`), computed scroll offset keeps selected item centered when possible. Uses `jL` (useInput) for key binding.

## Integration Points

- **tui-theme-system**: MCP panels consume theme tokens (`o.text.muted`, `o.text.primary`, `o.primary`) from the shared theme provider (likely 3773.js / 3910.js).
- **tui-renderer-main-loop**: Panel coordination: the MCP manager panels are likely instantiated by the main TUI app entry point, sharing the Ink rendering context.
- **Tool System (02-orchestration+ boundary)**: `registerAllMcpTools()` in 1178.js registers MCP tools via `A.register(E)` where `A = k0()`: this is the tool registry that belongs to the mission/tool subsystem (NOT analyzed here). MCP tools become callable through the tool execution pipeline.
- **JsonRpc Protocol**: The MCP management flows are primarily driven via JsonRpc when running in `interactive-cli` / `json-rpc` mode (3708.js). The TUI panels emit JsonRpc requests that flow through the protocol adapter.
- **Settings System**: `McpSettingsManager` (0267.js) uses a `settingsProvider` with hierarchical levels (user, project, org, folder). MCP servers can only be added/removed at `user` level.
- **OAuth Provider System**: HTTP MCP servers use OAuth authentication (`oauthProviders` Map, `callbackServer`, `oauthStorage` in 1178.js). Auth state flows through `mcp-auth-required` events.

## Implementation Notes

To implement an MCP Manager panel in OpenDroid:

1. **Server list component**: Create a scrollable list showing server name, connection status (`connected`/`disabled`/`connecting`), and source attribution (`[Org]`/`[Project]`). Add action entries for adding servers from registry or manually.

2. **Tool list component**: Group tools by server with server headers showing enabled/total counts. Each tool has a checkbox toggle. Support virtual scrolling (12-item window).

3. **Key bindings**: ↑↓ navigation, Space to toggle, A for toggle-all, S for toggle-server, Enter for detail, ESC to go back.

4. **RPC contract**: Use the `daemon.*` methods: `get_mcp_config`, `update_mcp_config`, `toggle_mcp_server`, `add_mcp_server`, `remove_mcp_server`, `list_mcp_registry`, `list_mcp_tools`, `toggle_mcp_tool`.

5. **Server config types**: Support `stdio` (command + args + env) and `http` (url + headers) server types, both with `disabled` boolean.

6. **Tool disable tracking**: Per-server `disabledTools` array stored in settings. Use Set operations for efficient add/remove of tool names.

7. **Event-driven status**: Subscribe to MCP service events (`mcp-status-changed`, `mcp-auth-required`) for real-time UI updates.

## Module Reference

| Module | Lines | Content |
|--------|-------|---------|
| 3963.js | 5-60 | `rtD` function: tool grouping, status computation, virtual scroll |
| 3963.js | 60-80 | useInput keyboard handler |
| 3963.js | 80-130 | Empty state rendering |
| 3963.js | 130-220 | Server headers + tool items rendering |
| 3963.js | 220-283 | Help text, module exports |
| 3959.js | 8-65 | `ptD` function: complete server list panel |
| 0910.js | 20-33 | MCP RPC method constants |
| 0916.js | 7-85 | MCP Zod schemas (server config, RPC messages) |
| 3708.js | 89-105 | MCP event listener setup |
| 3708.js | 655-735 | Add/remove/list MCP server handlers |
| 3708.js | 735-795 | Toggle MCP tool handler, auth handlers |
| 3754.js | 1378-1390 | handleToggleMcpTool delegation |
| 1178.js | 70-150 | McpService constructor, start, metrics |
| 1178.js | 310-400 | enable/disable/add/remove/toggleTool flows |
| 1178.js | 550-600 | registerAllMcpTools, clearAuth |
| 0267.js | 420-500 | getMcpServers, addMcpServer, removeMcpServer, enableMcpServer |
| 0267.js | 500-550 | enableMcpTools, disableMcpTools |
