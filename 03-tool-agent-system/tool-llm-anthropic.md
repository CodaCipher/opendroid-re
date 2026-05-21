# Anthropic Model Client Adapter: Tool System Architecture Notes

## Overview

OpenDroid integrates with the Anthropic API (Claude models) through a layered architecture: a vendor-provided SDK client (class `rB` in 2375.js), streaming event parsers (2362.js, 2371.js), content format converters (0943.js), telemetry/observability wrappers (1466.js), and a comprehensive model catalog with parameter mapping (0257.js). The system supports multiple API providers (direct Anthropic, AWS Bedrock, Google Vertex) for each model, with feature-flag-driven provider routing. Tool-calling follows the Anthropic `tool_use`/`tool_result` content block format, with streaming incremental JSON parsing for tool inputs. The adapter includes automatic retry logic with exponential backoff, caching headers, and extended thinking support.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 2375.js | 412 lines | Anthropic SDK base client (`rB` class): HTTP client, auth, retry, headers | Vendor (Anthropic SDK) |
| 2373.js | ~80 lines | Messages resource (`Xs` class): `/v1/messages` endpoint, create/stream/countTokens | Vendor (Anthropic SDK) |
| 2366.js | ~84 lines | Beta messages resource (`Vs` class): `/v1/messages?beta=true`, structured outputs | Vendor (Anthropic SDK) |
| 2362.js | 484 lines | Streaming message accumulator (`P2H` class): SSE event parsing, tool_use assembly | Vendor (Anthropic SDK) |
| 2347.js | ~180 lines | SSE stream parser (`x2` class): fromSSEResponse, JSON event extraction | Vendor (Anthropic SDK) |
| 0943.js | 318 lines | Content format converter: `t4$`/`a4$`/`JEH`/`XvH` message format mapping | App (OpenDroid) |
| 0257.js | 710 lines | Model catalog (`_X` object): all Claude model definitions, params, routing | App (OpenDroid) |
| 0251.js | ~80 lines | Model defaults/presets: default model IDs, reasoning effort presets | App (OpenDroid) |
| 1466.js | 163 lines | Telemetry streaming wrapper (`Su$`/`yu$`): token counting, tool call tracking | App (OpenDroid) |
| 1468.js | ~30 lines | Method allowlist: `bu$` array of monitored Anthropic API method names | App (OpenDroid) |
| 2345.js | ~80 lines | Logging utilities: log level filtering, header redaction (x-api-key, authorization) | Vendor (Anthropic SDK) |
| 3134.js | 1078 lines | LLM call orchestrator: high-level message send, streaming/non-streaming dispatch | App (OpenDroid) |
| 2536.js | 117 lines | Droid config generator: Anthropic proxy usage example via `nZ` auth | App (OpenDroid) |

## Architecture

### 1. Client Initialization & Authentication (2375.js)

The `rB` class (mapped to `kW` / Anthropic in the export) is the base HTTP client:

- **Constructor** accepts `{ baseURL, apiKey, authToken, ...options }`
- **Environment variables**: `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`
- **Default baseURL**: `https://api.anthropic.com`
- **Default timeout**: via `NiA.DEFAULT_TIMEOUT`
- **Default maxRetries**: `2`
- **Browser detection**: throws error if running in browser unless `dangerouslyAllowBrowser: true`
- **Auth**: dual-mode: `X-Api-Key` header (apiKey) or `Authorization: Bearer` (authToken)
- **API version header**: `anthropic-version: 2023-06-01` (hardcoded in `buildHeaders`)

### 2. Messages Endpoints (2373.js, 2366.js)

Two message resource classes handle the `/v1/messages` endpoint:

**`Xs` (2373.js)**: Standard messages API:
- `create(H, A)` → POST `/v1/messages` (non-streaming)
- `stream(H, A)` → Returns `K2H.createMessage(this, H, A)` streaming handle
- `countTokens(H, A)` → POST `/v1/messages/count_tokens`

**`Vs` (2366.js)**: Beta messages API:
- `create(H, A)` → POST `/v1/messages?beta=true` with `anthropic-beta` header
- `parse(H, A)` → Structured outputs with `structured-outputs-2025-11-13` beta
- `stream(H, A)` → Returns `P2H.createMessage(this, H, A)` streaming handle
- `countTokens(H, A)` → with `token-counting-2024-11-01` beta
- `toolRunner(H, A)` → Returns tool execution helper `T2H`

Both include deprecated model warnings and automatic timeout calculation for non-streaming requests.

### 3. Model Catalog & Parameter Mapping (0257.js)

The `_X` object is the complete model registry. Anthropic models include:

| Model ID | Provider | API Providers | Tier | Token Multiplier |
|----------|----------|---------------|------|-----------------|
| claude-3-5-sonnet-20241022 | anthropic | anthropic, bedrock_anthropic | standard | 1.2 |
| claude-3-7-sonnet-20250219 | anthropic | anthropic, bedrock, vertex | standard | 1.2 |
| claude-sonnet-4-20250514 | anthropic | anthropic, bedrock, vertex | standard | 1.2 |
| claude-opus-4-1-20250805 | anthropic | anthropic, bedrock, vertex | premium | 6.0 |
| claude-sonnet-4-5-20250929 | anthropic | anthropic, bedrock, vertex | standard | 1.2 |
| claude-opus-4-5-20251101 | anthropic | anthropic, bedrock, vertex | premium | 2.0 |
| claude-sonnet-4-6 | anthropic | anthropic, bedrock, vertex | standard | 1.2 |
| claude-opus-4-6 | anthropic | anthropic, bedrock, vertex | premium | 2.0 |
| claude-opus-4-6-fast | anthropic | anthropic only | premium | 12.0 |
| claude-haiku-4-5-20251001 | anthropic | anthropic, bedrock, vertex | standard | 0.4 |
| alto-02-04 | anthropic | anthropic only | standard | 1.2 |

Each model entry contains:
- **reasoningEffort**: `{ supported: [...], default: "..." }`: e.g., `["off", "low", "medium", "high", "max"]`
- **providerRouting**: Maps feature flags to API providers (e.g., `UseAnthropicBedrock → "bedrock_anthropic"`)
- **thinking**: Extended thinking configuration (`UIH`, `VBA`, `lTH` references)
- **contextLimits**: Input/output token limits with thinking context consumption flags
- **cost**: `{ tokenMultiplier: N }` for billing calculation
- **matchPatterns**: Regex patterns for fuzzy model name matching
- **anthropicFastMode**: Boolean flag for fast mode models (e.g., `claude-opus-4-6-fast`)
- **tier**: "standard" or "premium"
- **availableInCLI**: Whether the model is exposed in CLI mode

**Model aliasing** via `shL`: `"claude-opus-4-20250514" → "claude-opus-4-1-20250805"`

**Default presets** (0251.js):
- Default model: `claude-sonnet-4-5-20250929`
- Worker model: `claude-opus-4-6` with reasoning effort `"high"`
- Orchestrator model: `claude-opus-4-6`

### 4. Tool-Calling Format Conversion (0943.js)

OpenDroid uses an internal canonical format for messages that gets converted to/from Anthropic wire format via `t4$` (to Anthropic) and `JEH` (from Anthropic):

**To Anthropic format** (`t4$` function):
```js
// Internal → Anthropic wire format
case "tool_use":
  return {
    type: "tool_use",
    id: L.id,
    name: L.name,
    input: L.input,
    ...(L.cache_control && A)    // cache_control: { type: "ephemeral" }
  };
case "tool_result": {
  let $ = { type: "tool_result", tool_use_id: L.toolUseId };
  if (L.isError !== void 0) $.is_error = L.isError;
  if (L.content !== void 0)
    $.content = typeof L.content === "string" ? L.content : L.content.map((I) => n4$(I));
  return $;
}
```

**Key conversions**:
- `toolUseId` → `tool_use_id` (camelCase to snake_case)
- `isError` → `is_error`
- `thinking` blocks pass through with `signature` and `signatureProvider`
- `redacted_thinking` blocks pass through with `data`
- `document` blocks are converted to `<attached-file>` text or PDF base64
- `image` blocks: `mediaType` → `media_type`, `source.type: "base64"` preserved

**From Anthropic format** (`JEH` function): Reverse mapping with `tool_use_id` → `toolUseId`, `media_type` → `mediaType`.

**Conversation reconstruction** (`XvH` function) rebuilds message content from stored events:
- Handles orphaned thinking blocks (missing signatures → converted to `<thinking>` text)
- Reconstructs `tool_use` blocks by matching `toolUseId` to original tool call data
- Preserves `redacted_thinking` data blobs

### 5. Streaming Response Handling (2362.js, 2347.js, 1466.js)

**SSE Parser** (2347.js: `x2` class):
- `fromSSEResponse()`: parses SSE lines, filters for Anthropic events:
  - `message_start`, `message_delta`, `message_stop`
  - `content_block_start`, `content_block_delta`, `content_block_stop`
  - `ping` (ignored), `error` (throws)
- Each event's `data` field is JSON-parsed and yielded

**Stream Accumulator** (2362.js: `P2H` class):
- Event-driven architecture with `on('text')`, `on('thinking')`, `on('inputJson')`, etc.
- **State machine**: `message_start` → `content_block_start` → `content_block_delta` (N times) → `content_block_stop` → ... → `message_delta` → `message_stop`
- **Tool input assembly**: `input_json_delta` events accumulate partial JSON strings into `WYI` (hidden property), with real-time `JSON.parse` attempts via `ioH`
- **Delta types handled**: `text_delta`, `citations_delta`, `input_json_delta`, `thinking_delta`, `signature_delta`
- **Error handling**: Order validation (`"message_start" before "message_stop"`), JSON parse errors emit error event

**Tool call extraction from streaming** (2362.js, `BYI` handler):
```js
case "content_block_start":
  return (L.content.push(A.content_block), L);
case "content_block_delta": {
  // For input_json_delta:
  if ($ && TYI($)) {
    let I = $[WYI] || "";
    I += A.delta.partial_json;
    let D = { ...$ };
    Object.defineProperty(D, WYI, { value: I, enumerable: false, writable: true });
    if (I) try { D.input = ioH(I); } catch (E) { /* emit error */ }
    L.content[A.index] = D;
  }
}
```

**Telemetry wrapper** (1466.js: `Su$` generator):
- Wraps the stream for observability (OpenTelemetry-compatible)
- Tracks: `responseId`, `responseModel`, `promptTokens`, `completionTokens`, `cacheCreationInputTokens`, `cacheReadInputTokens`, `toolCalls`, `finishReasons`
- Tool call assembly: `content_block_start` with `tool_use` type → accumulate `partial_json` → `content_block_stop` → parse final JSON → push to `toolCalls[]`

### 6. Error Handling & Retry Logic (2375.js)

**Retry strategy** in `shouldRetry`:
- HTTP 408 (Request Timeout) → retry
- HTTP 409 (Conflict) → retry
- HTTP 429 (Rate Limit) → retry
- HTTP 5xx (Server Error) → retry
- `x-should-retry` header honored (explicit `"true"` / `"false"`)

**Backoff calculation** (`calculateDefaultRetryTimeoutMillis`):
```js
let I = A - H;                           // retry number
let D = Math.min(0.5 * Math.pow(2, I), 8); // exponential, capped at 8s
let E = 1 - Math.random() * 0.25;        // jitter (75-100%)
return D * E * 1000;                      // ms
```

**Retry-after header parsing**: Supports both `retry-after-ms` (milliseconds) and `retry-after` (seconds or HTTP date).

**Timeout**: Default via `NiA.DEFAULT_TIMEOUT`; non-streaming requests auto-calculated based on `max_tokens` with 10-minute maximum.

**Error types**: `lD` (AnthropicError), `rG` (AbortError), `c7H` (TimeoutError), `_s` (ConnectionError).

### 7. LLM Call Orchestrator Integration (3134.js)

The high-level LLM orchestrator (3134.js) uses the Anthropic client for:
- **Provider detection**: `KH.apiModelProvider === "anthropic"` routes to Anthropic client
- **Caching preparation**: `prepareMessagesWithCaching` adds cache breakpoints and validates tool_use/tool_result pairs
- **Streaming/non-streaming dispatch**: Based on `SH` flag, uses `messages.create` (non-stream) or `messages.stream` (stream)
- **Response collection**: Aggregates `streamingContent`, `toolUses`, `usage`, `thinkingContent`, `thinkingSignature`, `contentBlocks`
- **Empty response detection**: `DYH(z, signal.aborted)` checks for empty bodies (possible rate limit)

## Key Findings

### Finding 1: Anthropic SDK is vendored with minimal modification

Module 2375.js is the Anthropic TypeScript SDK's `Anthropic` base class, vendored directly into the bundle. It includes the full HTTP client with retry logic, auth header management, and request building. The SDK exports are registered in 2376.js:
```js
// Module 2376.js, line 29
rB.AnthropicError = lD;
```

### Finding 2: Dual content format conversion layer

The `t4$` (to-Anthropic) and `JEH` (from-Anthropic) functions in 0943.js implement a complete bidirectional content block mapping. This allows OpenDroid to maintain its own internal message format while correctly serializing to/from the Anthropic wire format. Key mapping:
```js
// Internal "tool_result" → Anthropic wire format
let $ = { type: "tool_result", tool_use_id: L.toolUseId };
if (L.isError !== void 0) $.is_error = L.isError;
if (L.content !== void 0)
  $.content = typeof L.content === "string"
    ? L.content
    : L.content.map((I) => n4$(I));
```

### Finding 3: Incremental tool input JSON parsing during streaming

Module 2362.js implements real-time tool input parsing during streaming. As `input_json_delta` events arrive, partial JSON strings are accumulated and parsed incrementally:
```js
// Module 2362.js, ~line 370
case "input_json_delta": {
  if (TYI($) && $.input) this._emit("inputJson", A.delta.partial_json, $.input);
  break;
}
```
The `BYI` handler stores partial JSON in a non-enumerable property (`WYI`) and attempts `JSON.parse` after each delta, enabling live preview of tool inputs.

### Finding 4: Multi-provider routing with feature flags

The model catalog (0257.js) supports three Anthropic API providers: direct Anthropic, AWS Bedrock (`bedrock_anthropic`), and Google Vertex (`vertex_anthropic`). Routing is controlled by feature flags:
```js
// Module 0257.js, claude-sonnet-4-6 entry
providerRouting: {
  UseSonnet46Bedrock: "bedrock_anthropic",
  UseSonnet46Vertex: { provider: "vertex_anthropic" },
}
```
The `JBA` function in 0255.js resolves the provider by checking feature flags in order.

### Finding 5: Extended thinking with signature validation

The `XvH` function in 0943.js handles extended thinking blocks with special care:
- Blocks with valid `signature` pass through as `type: "thinking"`
- Blocks missing signatures are converted to `<thinking>` text blocks (logged as warning)
- `redacted_thinking` blocks are preserved as opaque data blobs
- Non-Anthropic signature providers (`signatureProvider !== "anthropic"`) have their thinking text extracted and sent as plain text instead

### Finding 6: Telemetry integration for Anthropic streams

Module 1466.js wraps every Anthropic streaming response with OpenTelemetry-compatible span tracking:
```js
// Module 1466.js, Su$ generator
function ju$(H, A, L, $) {
  if (xV1(H, $)) return;   // error detection
  vV1(H, A);                // usage/token extraction
  uV1(H, A);                // tool_use block start
  gV1(H, A, L);             // content_block_delta (text + json)
  cV1(H, A);                // content_block_stop → finalize tool call
}
```
This tracks cache token metrics (`cacheCreationInputTokens`, `cacheReadInputTokens`) unique to Anthropic.

### Finding 7: Header security via redaction

Module 2345.js ensures sensitive headers (`x-api-key`, `authorization`, `cookie`, `set-cookie`) are redacted in logs:
```js
// Module 2345.js, line 49
A.toLowerCase() === "x-api-key" ||
A.toLowerCase() === "authorization" ||
A.toLowerCase() === "cookie" ||
A.toLowerCase() === "set-cookie"
  ? "***"
  : L
```

## Integration Points (Cross-Mission)

- **Tool Registry (03-tool-agent-system - tool-core-registry)**: The `tool_use` content blocks reference tools registered in the tool registry (3264.js). Tool definitions are converted from OpenDroid's internal schema to Anthropic's `input_schema` format for API calls.
- **Agent State Machine (03-tool-agent-system - tool-agent-state)**: The LLM orchestrator (3134.js) feeds streaming responses into the agent state machine, which manages the tool execution loop.
- **Streaming Pipeline (03-tool-agent-system - tool-streaming)**: The `P2H` streaming accumulator and `x2` SSE parser form the core of the streaming response pipeline, shared with the OpenAI adapter.
- **Permissions Engine (03-tool-agent-system - tool-permissions)**: Tool calls received from the Anthropic API are validated against the permissions engine before execution.
- **Sandbox (03-tool-agent-system - tool-sandbox)**: Tool execution results pass through the sandbox before being formatted as `tool_result` blocks.
- **TUI (01-terminal-ui)**: Tool call events from streaming are rendered in the terminal UI (3391.js renders tool-specific UI components).
- **Session/Infra (05-infrastructure)**: API authentication tokens are managed through the session system (`nZ` function in 2536.js provides proxy auth headers).

## Implementation Notes

### Anthropic Client Initialization
```typescript
// Create Anthropic client
const client = new Anthropic({
  baseURL: process.env.ANTHROPIC_BASE_URL || "https://api.anthropic.com",
  apiKey: process.env.ANTHROPIC_API_KEY,
  maxRetries: 2,
});
```

### Sending Messages with Tools
```typescript
// Use the Messages resource
const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 8192,
  system: systemPrompt,
  messages: conversationHistory,
  tools: toolDefinitions,  // Must use Anthropic's input_schema format
});

// Handle tool_use blocks from response
for (const block of response.content) {
  if (block.type === "tool_use") {
    const result = await executeTool(block.name, block.input);
    // Return as tool_result in next message
    messages.push({
      role: "user",
      content: [{ type: "tool_result", tool_use_id: block.id, content: result }]
    });
  }
}
```

### Streaming with Tool Detection
```typescript
const stream = client.messages.stream({
  model: "claude-sonnet-4-6",
  messages: conversationHistory,
  tools: toolDefinitions,
});

stream.on("text", (text) => { /* handle text delta */ });
stream.on("inputJson", (partialJson, parsedInput) => { /* handle tool input streaming */ });
stream.on("contentBlock", (block) => {
  if (block.type === "tool_use") { /* tool call complete */ }
});

const finalMessage = await stream.finalMessage();
```

### Content Format Conversion
```typescript
// Convert internal format to Anthropic wire format
const anthropicContent = t4$(internalContent);
// Convert Anthropic response to internal format
const internalContent = JEH(anthropicResponse.content);
```

### Model Selection
```typescript
// Get model config from catalog
const modelConfig = _X["claude-sonnet-4-6"];
// modelConfig.provider === "anthropic"
// modelConfig.apiProviders === ["anthropic", "bedrock_anthropic", "vertex_anthropic"]
// modelConfig.reasoningEffort.supported === ["off", "low", "medium", "high", "max"]
```

## Open Questions / Left Undone

1. **Thinking configuration objects**: `UIH`, `VBA`, `lTH` thinking config objects are referenced but not fully traced: they likely contain `thinkingBudgetTokens`, `thinkingEnabled`, and signature provider details.
2. **Bedrock/Vertex client variants**: The `bedrock_anthropic` and `vertex_anthropic` provider paths involve separate client initialization (likely in unanalyzed modules) with different auth mechanisms (AWS SigV4, Google OAuth).
3. **`nZ` auth proxy function**: The auth header generation function used by the LLM orchestrator is defined in an unanalyzed module: it likely creates proxy tokens for OpenDroid's API relay.
4. **`T2H` tool runner class**: Referenced in 2366.js's `toolRunner()` method but not traced: likely implements the automatic tool execution loop.
5. **Beta feature flags**: The `anthropic-beta` header values (e.g., `structured-outputs-2025-11-13`, `token-counting-2024-11-01`) suggest active use of Anthropic beta features whose full implementation isn't traced.
6. **`anthropicFastMode`**: The `claude-opus-4-6-fast` model has `anthropicFastMode: true`: the exact API behavior difference isn't traced to the request building code.
7. **`cEf` function**: The tool auto-executor at end of 2362.js (`cEf`) finds `tool_use` blocks and executes matching tools: its integration into the main tool loop needs cross-referencing with the agent state machine.
