# Figma MCP Bridge: Tool System Architecture Notes

## Overview

The Figma MCP bridge is implemented as an HTTP-based MCP server integration within OpenDroid's generic MCP client infrastructure. Rather than having dedicated Figma-specific code modules, the system treats Figma as one of 30+ pre-configured MCP server presets accessible through a unified HTTP+SSE transport layer. The Figma server connects to `https://mcp.figma.com/mcp` using the Streamable HTTP transport, authenticates via OAuth 2.0 with PKCE (using a special "Gemini CLI MCP Client" user-agent for Figma compatibility), and dynamically discovers and registers Figma's tools (design file access, component extraction, screenshot capture, metadata retrieval) into the local tool registry with `mcp_figma_<toolname>` namespace identifiers. All Figma tool invocations flow through the standard MCP tool execution pipeline: callTool → result serialization → image compression → LLM-formatted output.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 3430.js | ~381 lines | MCP server preset catalog (`I0M` array) containing Figma entry and server status polling | App |
| 1146.js | ~132 lines | OAuth client class (`bOA`) with Figma-specific client_name handling | App |
| 1109.js | ~314 lines | `StreamableHTTPClientTransport` (`Yt`): HTTP+SSE transport for remote MCP servers like Figma | App |
| 1178.js | ~691 lines | `McpService` singleton: server lifecycle, OAuth provider management, tool registration | App |
| 1175.js | ~105 lines | `HZA` class: MCP tool executor wrapping callTool with metrics/telemetry | App |
| 1148.js | ~74 lines | `xOA` function: MCP result serializer handling text/image/resource content types | App |
| 1127.js | ~120 lines | `XOA` class: OAuth callback HTTP server for authorization code flow | App |
| 0991.js | ~175 lines | MCP protocol type definitions and constants (protocol version "2025-11-25") | App |

## Architecture

### Figma MCP Server Preset Registration

Figma is defined as a preset in the `I0M` catalog array (3430.js, line 3) alongside 30+ other MCP server integrations:

```js
{
  name: "figma",
  description: "Design and collaboration platform for teams.",
  type: "http",
  url: "https://mcp.figma.com/mcp",
}
```

This preset is displayed in the MCP server catalog UI and can be enabled by users. When enabled, the `McpService` (1178.js) creates the corresponding server configuration.

### Connection & Transport Layer

The `StreamableHTTPClientTransport` class (1109.js) handles all HTTP-based MCP server connections including Figma:

1. **SSE Stream Opening**: GET request to `https://mcp.figma.com/mcp` with `Accept: text/event-stream`
2. **Message Sending**: POST requests with JSON-RPC payloads to the same endpoint
3. **Session Management**: Tracks `mcp-session-id` header for session continuity
4. **Auto-Reconnection**: Exponential backoff reconnection with configurable retries

### OAuth 2.0 Authentication Flow

Figma uses OAuth 2.0 with PKCE for authentication. The `bOA` class (1146.js) manages the OAuth flow:

1. **OAuth Discovery**: `XgH.discoverOAuthSupport()` probes the server for OAuth metadata
2. **Client Registration**: Dynamic client registration with Figma-specific workaround:
   - When `serverUrl.includes("figma.com")`, the client identifies as `"Gemini CLI MCP Client"` instead of `"OpenDroid CLI - figma"`: likely a Figma API compatibility requirement
3. **Authorization Code Flow**: PKCE-based flow with local callback server
4. **Token Persistence**: Tokens stored in `~/.opendroid/mcp-oauth.json` via `wOA` storage class
5. **Auto-Refresh**: Token refresh handled automatically on 401 responses

### Tool Discovery & Registration

When the Figma MCP server connects:

1. `McpService.registerAllMcpTools()` (1178.js, line 583) calls `mcpHub.listAllTools()` 
2. Each discovered tool is wrapped via `NO.createMcpTool()` (1178.js, line 616):
   - Tool ID: `mcp_figma_{toolName}` (e.g., `mcp_figma_get_file`)
   - LLM ID: `figma___{toolName}` (e.g., `figma___get_file`)
   - Display name: `[MCP] figma:{toolName}`
   - Toolkit: `MCP:figma`
   - `requiresConfirmation: true`: every MCP tool invocation requires user approval
3. Tools are registered in the global tool registry via `k0().register()`

### Tool Execution Pipeline

When a Figma tool is invoked (e.g., to fetch a design file or capture a screenshot):

1. `HZA.execute()` (1175.js) is called with the tool's `serverName` ("figma") and `toolName`
2. `McpService.callTool()` delegates to `mcpHub.callTool()` which sends a JSON-RPC `tools/call` request via `StreamableHTTPClientTransport`
3. The Figma MCP server returns results containing text, images, or resources
4. `BD1()` compresses any image results exceeding the size limit (supported formats: JPEG, PNG)
5. `xOA()` (1148.js) serializes the result:
   - **Text content**: Pretty-printed JSON
   - **Image content**: Base64-encoded with media type for LLM consumption
   - **Resource content**: Embedded text extracted or type indicated
   - **Audio content**: Type label only

### Server Status Monitoring

`xfL()` (3430.js) provides real-time MCP server status including Figma:
- Tracks states: `connected`, `connecting`, `failed`, `disabled`
- Monitors OAuth token validity (`hasValidOAuthTokens`)
- Emits `mcp_status_changed` events for UI updates

## Key Findings

### 1. Figma-Specific OAuth Identity (1146.js, line 42)

```js
get clientMetadata() {
  let H = this.discoveredMetadata?.scopes?.join(" ") || "mcp:tools",
    A = this.getTokenEndpointAuthMethod();
  return {
    client_name: this.serverUrl.includes("figma.com")
      ? "Gemini CLI MCP Client"
      : `OpenDroid CLI - ${this.serverName}`,
    redirect_uris: [this.redirectUrl],
    grant_types: ["authorization_code", "refresh_token"],
    response_types: ["code"],
    token_endpoint_auth_method: A,
    scope: H,
  };
}
```

**Finding**: Figma's MCP endpoint requires a specific client_name ("Gemini CLI MCP Client") for compatibility. This is a hardcoded workaround: all other servers use `"OpenDroid CLI - {serverName}"`.

### 2. MCP Tool Namespace Convention (1178.js, line 617)

```js
static createMcpTool(H, A) {
  let L = `mcp_${A}_${H.name}`,
    $ = `${A}___${H.name}`,
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

**Finding**: All MCP tools including Figma's use a dual-ID system: `mcp_{server}_{tool}` for internal ID and `{server}___{tool}` for LLM-facing ID. Input schemas are passed through from the MCP server's `listTools` response with property defaults applied.

### 3. HTTP Transport with SSE Reconnection (1109.js, line 130)

```js
_scheduleReconnection(H, A = 0) {
  let L = this._reconnectionOptions.maxRetries;
  if (A >= L) {
    this.onerror?.(Error(`Maximum reconnection attempts (${L}) exceeded.`));
    return;
  }
  let $ = this._getNextReconnectionDelay(A);
  this._reconnectionTimeout = setTimeout(() => {
    this._startOrAuthSse(H).catch((I) => {
      this.onerror?.(
        Error(`Failed to reconnect SSE stream: ${I instanceof Error ? I.message : String(I)}`),
      );
      this._scheduleReconnection(H, A + 1);
    });
  }, $);
}
```

**Finding**: The transport uses exponential backoff reconnection for SSE streams. The Figma server's SSE connection is automatically reconnected on disconnection with configurable delay growth.

### 4. MCP Result Image Compression (1175.js, line 10)

```js
async function BD1(H) {
  if (!H.content || H.content.length === 0) return H;
  let A = await Promise.all(
    H.content.map(async (L) => {
      if (L.type !== "image") return L;
      let $ = L.mimeType;
      if (!$ || !f2.includes($)) return L;
      try {
        let I = Buffer.from(L.data, "base64");
        if (I.length <= kc) return L;
        let D = await bc(I, $);
        return { ...L, mimeType: D.contentType || $, data: D.buffer.toString("base64") };
      } catch (I) {
        return {
          type: "text",
          text: "[Image result omitted because it could not be compressed under the size limit]",
        };
      }
    }),
  );
  return { ...H, content: A };
}
```

**Finding**: Figma screenshot/component image results are automatically compressed if they exceed a size threshold (`kc`). Supported formats are JPEG and PNG only (`f2 = ["image/jpeg", "image/png"]`). Failed compressions are replaced with a text placeholder.

### 5. MCP Tool Result Serialization (1148.js, line 14)

```js
function xOA(H, A) {
  if (H.isError) {
    let I = H.content?.[0];
    if (I?.type === "text") return `Error: ${m$1(I.text, A) ?? I.text}`;
    return "Error: Unknown error occurred";
  }
  if (!H.content || H.content.length === 0) return "[No content returned]";
  let L = [], $ = [];
  H.content.forEach((I) => {
    switch (I.type) {
      case "text":
        $.push(hY$(I.text));
        break;
      case "image": {
        L.push({ type: "image", source: { type: "base64", mediaType: D, data: I.data } });
        break;
      }
      case "resource":
        if ("text" in I.resource && I.resource.text) {
          let D = I.resource.text;
          if (typeof D === "string") $.push(hY$(D));
        }
        break;
      case "audio":
        $.push(`[Audio: ${I.mimeType}]`);
        break;
    }
  });
  // ...
}
```

**Finding**: The result serializer handles all MCP content types that Figma may return. Images are converted to Anthropic-compatible base64 format. Resources (design tokens, metadata) are extracted as text.

## Integration Points

### Cross-Mission Dependencies

- **Terminal UI section (TUI)**: MCP server status display, OAuth authentication prompts in terminal UI. Module 3387.js contains TUI rendering for tool plan/spec display including MCP tools.
- **Orchestration section (Mission)**: MCP tool confirmation flow: `requiresConfirmation: true` triggers the mission permission system before executing Figma tools.
- **Desktop GUI section (GUI)**: MCP server catalog UI, OAuth browser redirect handling, server enable/disable controls. Module 3964.js contains React hooks for MCP server state.
- **Infrastructure section (Infra/Session)**: MCP settings persistence via `mcp.json` config file, OAuth token storage in `mcp-oauth.json`.

### Internal Tool System Integration

- **Tool Registry** (tool-core-registry): MCP tools registered via `k0().register()` with standard tool interface
- **Permissions** (tool-permissions): MCP tools have `requiresConfirmation: true`: always go through permission check
- **Sandbox** (tool-sandbox): MCP tool execution is sandboxed; the HTTP transport runs outside sandbox but tool invocations are gated
- **Agent State** (tool-agent-state): MCP tool calls integrated into the agent working state machine

## Implementation Notes

### Figma MCP Server Configuration

To implement Figma MCP integration in OpenDroid:

1. **Server Definition**: Add Figma as an HTTP MCP server preset with URL `https://mcp.figma.com/mcp`
2. **OAuth Handler**: Implement OAuth 2.0 with PKCE flow. Note: Figma requires `"Gemini CLI MCP Client"` as the client_name during registration
3. **Transport**: Use Streamable HTTP transport with SSE for receiving server-initiated messages
4. **Tool Registration**: Dynamically register discovered Figma tools with `mcp_figma_*` namespace
5. **Result Handling**: Support image content (JPEG/PNG), text, and embedded resources from Figma API responses
6. **Image Pipeline**: Implement automatic compression for large screenshots/component images

### Key Configuration Files

- MCP server config: `~/.opendroid/mcp.json` (user settings level)
- OAuth token storage: `~/.opendroid/mcp-oauth.json`
- OAuth callback server: localhost port with `/callback` path

### Tool Namespace Convention

| Context | Format | Example |
|---------|--------|---------|
| Internal ID | `mcp_figma_{tool}` | `mcp_figma_get_file` |
| LLM-facing | `figma___{tool}` | `figma___get_file` |
| Display | `[MCP] figma:{tool}` | `[MCP] figma:get_file` |

## Open Questions / Left Undone

1. **Figma Tool Catalog**: The exact set of tools exposed by `https://mcp.figma.com/mcp` was not enumerated: this requires a live connection to the Figma MCP server. Expected tools include: `get_file`, `get_file_nodes`, `get_images` (screenshots), `get_component`, `get_style`, `search`.
2. **OAuth Scope Details**: The specific OAuth scopes requested by the Figma MCP server during discovery were not traced: likely includes file:read, components:read, images:read.
3. **Image Size Threshold**: The exact value of `kc` (image compression threshold) and `bc` (compression function) were not fully traced to their definitions.
4. **Figma-specific Error Handling**: Whether there are any Figma-specific error codes or retry behaviors beyond the standard MCP error mapping was not determined.
5. **ACP (Agent Configuration Protocol) Integration**: Module 1178.js references `setMergedMcpConfigs` for "ACP merge": the relationship between ACP and Figma server configuration was not explored.
