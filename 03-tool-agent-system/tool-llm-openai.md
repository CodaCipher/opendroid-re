# OpenAI / Azure OpenAI Client Adapter: Tool System Architecture Notes

## Overview

OpenDroid integrates with OpenAI through a dual-path adapter supporting both the **Responses API** (`responses.create`) and the **Chat Completions API** (`chat.completions.create`). The adapter handles format conversion between the internal unified message format (Anthropic-style `tool_use`/`tool_result` content blocks) and OpenAI's native formats: the Responses API's `function_call`/`function_call_output` items and the Chat Completions API's `tool_calls`/`tool` role messages. Azure OpenAI is supported as an API provider variant (`azure_openai`) with provider-specific endpoint routing. The system also handles OpenAI-specific features including encrypted reasoning/thinking blocks, reasoning summaries, multi-phase responses, and streaming SSE event parsing.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 3134.js | ~1078 lines | **Primary OpenAI adapter**: client initialization, Responses API streaming (`sendOpenAIResponseMessage`), Chat Completions streaming (`sendOpenAIChatMessage`), provider routing (`openai`/`azure_openai`/`xai`/`google`), retry logic | App |
| 3127.js | ~845 lines | **Message format converters**: `LWD()` (Anthropic→OpenAI Responses), `$WD()` (Anthropic→Chat Completions), `AWD()` (tool→function format), `oEL()` (tool→Responses tool format), `jx()` (JSON Schema sanitizer), `eBD()` (assistant message builder) | App |
| 3126.js | ~845 lines | **Stream event processors**: `oBD()` (Responses API SSE events), `tBD()` (Chat Completions SSE events), `aBD()` (Gemini events), usage/token tracking, tool argument incremental parsing (`$YH`) | App |
| 2572.js | ~987 lines | **Conversation state manager**: `openaiMessageId`, `openaiPhase`, `openaiEncryptedContent`, `openaiReasoningId`, `openaiReasoningSummary` fields; `streamingContentBlocks` for OpenAI content block tracking | App |
| 2580.js | ~large | **Summarizer**: uses `hD` (OpenAI client) for summary generation via `responses.create` or `chat.completions.create` depending on provider; demonstrates custom model config usage | App |
| 2404.js | ~small | **OpenAI SDK ChatCompletions resource**: `create()`, `parse()`, `stream()`, `runTools()` methods; thin wrapper over `/chat/completions` endpoint | Vendor (OpenAI SDK) |
| 2402.js | ~306 lines | **OpenAI SDK streaming wrapper**: `ChatCompletionStream` class; handles SSE chunk accumulation, tool call delta assembly, content/refusal/logprobs events | Vendor (OpenAI SDK) |
| 2398.js | ~227 lines | **OpenAI SDK non-streaming runner**: `_runTools()` loop for Chat Completions with automatic tool execution | Vendor (OpenAI SDK) |
| 1463.js | ~small | **API operation identifiers**: `chat.completions.create` and `responses.create` operation names, SSE event type lists, response type detection helpers (`Cu$`, `qu$`, `Ou$`) | Vendor (OpenAI SDK) |
| 1028.js | ~small | **Zod→JSON Schema** with `openAi` target mode: schema conversion warning for union root types | Vendor (zod-to-json-schema) |

## Architecture

### Dual API Path: Responses API vs Chat Completions

The adapter supports **two distinct OpenAI API surfaces**:

1. **Responses API** (preferred/newer): `$.current.responses.create()` in 3134.js
   - Input format: flat item array with `type: "message"`, `type: "function_call"`, `type: "function_call_output"`, `type: "reasoning"` items
   - Tool format: `{ type: "function", name, description, parameters, strict: false }` via `oEL()`
   - SSE events: `response.output_item.added`, `response.output_text.delta`, `response.function_call_arguments.delta/done`, `response.completed`
   - Supports encrypted reasoning with `encrypted_content`, `reasoning_id`, `summary`

2. **Chat Completions API** (legacy/compatibility): `$.current.chat.completions.create()` in 3134.js
   - Input format: standard `messages` array with `role: "user"/"assistant"/"tool"/"system"` entries
   - Tool format: `{ type: "function", function: { name, description, parameters } }` via `AWD()`
   - SSE events: standard `choices[].delta.content`, `choices[].delta.tool_calls`, `choices[].finish_reason`
   - Supports `reasoning` and `reasoning_content` fields for thinking

### Client Initialization

```js
// Module 3134.js, line 444-454: OpenAI client creation
if (!$.current) {
  let y = QH  // customModel config
    ? { apiKey: QH.apiKey, baseURL: QH.baseUrl, organization: null, project: null }
    : {
        apiKey: A8H,  // proxy API key
        baseURL: `${OD().apiBaseUrl}/api/llm/o/v1`,  // proxy endpoint
        organization: null,
        project: null,
      };
  $.current = new hD(y);  // hD = OpenAI SDK client constructor
}
```

### Provider Routing

```js
// Module 3134.js, line 456-461: API provider lock and correction
let JH = H.getLockedApiProvider() ?? void 0,
  MH = JH;
if (HH === "xai") MH = "xai";
else if (HH === "openai") {
  if (MH !== "openai" && MH !== "azure_openai") MH = "openai";
}
```

The system detects `modelProvider` via `F$(modelId).modelProvider` and locks the API provider accordingly. Azure OpenAI is recognized as a valid provider alongside standard OpenAI.

### Message Format Conversion (Anthropic → OpenAI)

#### Responses API Conversion (`LWD()` in 3127.js)

Converts internal Anthropic-style messages to OpenAI Responses API format:

```js
// Module 3127.js: LWD function (key excerpt)
// Tool results → function_call_output
A.push({
  type: "function_call_output",
  call_id: M.toolUseId,
  output: U || "Image content read successfully",
});

// Tool uses → function_call items
A.push({
  type: "function_call",
  call_id: E.id,
  name: E.name,
  arguments: JSON.stringify(E.input),
});

// Thinking/reasoning → reasoning items
let D = {
  type: "reasoning",
  encrypted_content: $.openaiEncryptedContent,
  summary: $.openaiReasoningSummary
    ? [{ type: "summary_text", text: $.openaiReasoningSummary }]
    : [],
};
```

#### Chat Completions Conversion (`$WD()` in 3127.js)

Converts to standard Chat Completions format:

```js
// Module 3127.js: $WD function (key excerpt)
// Tool uses → function tool_calls
let P = {
  id: U.id,
  type: "function",
  function: { name: U.name, arguments: JSON.stringify(U.input) },
};

// Tool results → role: "tool" messages
$.push({ role: "tool", content: M, tool_call_id: f.toolUseId });
```

### Tool Schema Conversion

Two converters for the two API paths:

```js
// Module 3127.js: AWD (Chat Completions format)
function AWD(H) {
  return H.map((A) => ({
    type: "function",
    function: { name: A.name, description: A.description || "", parameters: A.input_schema },
  }));
}

// Module 3127.js: oEL (Responses API format)
function oEL(H) {
  return H.map((A) => ({
    type: "function",
    name: A.name,
    description: A.description || "",
    parameters: A.input_schema,
    strict: false,
  }));
}
```

### Streaming Event Processing

#### Responses API Events (`oBD()` in 3126.js)

```js
// Module 3126.js, line 423-612: key event handlers
case "response.created":           // Capture message ID
case "response.output_item.added": // New text/reasoning/function_call item
case "response.output_item.done":  // Item completion + signature capture
case "response.output_text.delta": // Text streaming
case "response.reasoning_summary_text.delta": // Thinking content streaming
case "response.function_call_arguments.delta": // Tool arg streaming
case "response.function_call_arguments.done":  // Tool arg completion
case "response.completed":         // Final response + usage stats
case "response.failed":            // Error handling
```

#### Chat Completions Events (`tBD()` in 3126.js)

```js
// Module 3126.js, line 613-749: processes choices[].delta
// Content streaming: delta.content
// Thinking: delta.reasoning / delta.reasoning_content
// Tool calls: delta.tool_calls[].function.name/arguments
// Completion: choices[].finish_reason → stopReason mapping
```

### Azure OpenAI Handling

Azure OpenAI is treated as an API provider variant. The system:
1. Recognizes `"azure_openai"` as a valid provider lock value (3134.js, line 460)
2. Routes through the same `hD` (OpenAI SDK) client: the SDK handles Azure endpoint differences via `baseURL` and `apiKey` from the custom model config
3. No separate Azure-specific code path exists in the adapter layer; differentiation happens at the configuration/proxy level

### Reasoning/Thinking Support

OpenAI's encrypted reasoning is fully supported:

```js
// Module 3126.js: Encrypted content capture
case "response.output_item.done":
  if (I.type === "reasoning") {
    A.openaiReasoningId = I.id;
    A.openaiEncryptedContent = I.encrypted_content;
    // Summary extraction
    let E = I.summary.filter((f) => f.type === "summary_text").map((f) => f.text);
    if (E.length > 0) A.openaiReasoningSummary = E.join("\n");
  }
```

For Chat Completions, thinking is captured via `reasoning` or `reasoning_content` delta fields (3126.js, line 655-676).

### Error Handling & Retry

```js
// Module 3134.js: Retry wrapper
return UEH(
  async () => { /* main request */ },
  {
    ...MYH({ isExecMode: U }),
    signal: XH.signal,
    onAllError: async (y) => {
      if (sG(y)) return BqH;  // retryable error → returns retry signal
      throw y;                  // non-retryable → propagate
    },
    onRetry: (y, t) => { /* logging */ },
  }
);
```

## Key Findings

### 1. Dual API Surface with Unified Internal Format

The most significant architectural finding is that OpenDroid maintains an **Anthropic-style internal message format** (with `tool_use`/`tool_result` content blocks, `thinking` blocks) and converts to/from OpenAI formats at the adapter boundary. This allows the system to swap between Anthropic, OpenAI, and Gemini providers without changing internal state management.

### 2. Responses API is the Primary Path

```js
// Module 3134.js, line 504: Responses API call
hH = await $.current.responses.create({
  model: i,
  input: bH,
  store: false,
  tools: oEL(F),
  instructions: e.map((NH) => NH.text).join("\n\n"),
  stream: true,
  ...RH.requestParams,
  ...(QH?.extraArgs ?? {}),
}, { signal: XH.signal, headers: SH });
```

The Responses API path is preferred for OpenAI models (GPT-4o, o1, o3, etc.) with Chat Completions as a fallback. The `store: false` flag prevents conversation storage on OpenAI's servers.

### 3. Tool Argument Incremental Parsing

```js
// Module 3126.js: Incremental JSON parsing for streaming tool args
case "response.function_call_arguments.delta": {
  if (!A.toolInputBuffers[H.output_index]) A.toolInputBuffers[H.output_index] = "";
  A.toolInputBuffers[H.output_index] += H.delta;
  if ($.onToolInputDelta && H.delta) {
    let D = $YH(A.toolInputBuffers[H.output_index]);  // partial JSON parser
    if (Object.keys(D.data).length > 0) $.onToolInputDelta(I.id, D.data);
  }
}
```

The `$YH()` function (in 3126.js, `Ntf`) is a custom **incremental JSON key-value parser** that can extract partial data from incomplete JSON strings: enabling real-time tool parameter streaming to the UI.

### 4. OpenAI-Specific Message Metadata

```js
// Module 2572.js: Conversation state tracks OpenAI-specific fields
openaiMessageId: void 0,        // Response message ID (msg_*)
openaiPhase: void 0,            // Response phase
openaiEncryptedContent: void 0,  // Encrypted reasoning content
openaiReasoningId: void 0,      // Reasoning block ID
openaiReasoningSummary: void 0,  // Reasoning summary text
```

These fields are persisted in conversation history and round-tripped through the Responses API to maintain reasoning context across turns.

### 5. JSON Schema Sanitization for OpenAI Compatibility

```js
// Module 3127.js: jx() function sanitizes schemas for OpenAI
// Handles: nullable via anyOf, const→enum, type arrays with null
// Module 1028.js: Warning for unsupported union root types
if (L.target === "openAi" && ("anyOf" in f || "oneOf" in f || "allOf" in f || ...))
  console.warn("Warning: OpenAI may not support schemas with unions as roots!");
```

### 6. Image Content Round-Tripping

```js
// Module 3127.js: Image handling in tool results for Responses API
let P = M.content.filter((B) => B.type === "image");
if (P.length > 0) {
  let B = P.map((X) => ({
    type: "input_image",
    image_url: `data:${X.source.mediaType};base64,${X.source.data}`,
    detail: "auto",
  }));
  A.push({ role: "user", content: [{ type: "input_text", text: V }, ...B] });
}
```

Tool results containing images are converted to separate user messages with `input_image` content items, as the Responses API doesn't support inline images in function outputs.

## Integration Points

- **Tool Registry (tool-core-registry)**: Tool definitions (`eI()`/`k0()`) provide `input_schema` which flows through `AWD()`/`oEL()` converters to OpenAI `function` format
- **Permissions Engine (tool-permissions)**: Tool execution authorization happens before OpenAI response processing triggers tool dispatch
- **Sandbox (tool-sandbox)**: Tool execution from OpenAI responses passes through sandbox validation
- **Agent State (tool-agent-state)**: `ConversationStateManager` (2572.js) stores OpenAI-specific metadata (`openaiMessageId`, encrypted reasoning) for multi-turn reasoning
- **Streaming (tool-streaming)**: `oBD()` and `tBD()` SSE processors feed into the UI streaming pipeline via callbacks (`onTextDelta`, `onToolInputDelta`, `onThinkingDelta`)
- **Anthropic Adapter (tool-llm-anthropic)**: Shares the same internal message format; provider switching happens at `modelProvider` detection level
- **TUI (01-terminal-ui)**: Streaming callbacks drive terminal UI rendering (text, thinking blocks, tool calls)
- **MCP Client (tool-mcp-client)**: MCP tools are merged into the tool list and converted to OpenAI function format alongside built-in tools

## Implementation Notes

### OpenAI Client Initialization
- Use `new OpenAI({ apiKey, baseURL })` for both OpenAI and Azure OpenAI
- For Azure: set `baseURL` to Azure endpoint, `apiKey` to Azure key
- Proxy path: `apiBaseUrl + "/api/llm/o/v1"` for standard, `apiBaseUrl + "/api/llm/o"` for direct

### Tool Registration for OpenAI
- Register tools using `eI()` with `input_schema` (Zod-converted JSON Schema)
- Tool format converter `AWD()` wraps as `{ type: "function", function: { name, description, parameters } }`
- Responses API uses `oEL()`: `{ type: "function", name, description, parameters, strict: false }`

### Message Conversion
- `LWD()` for Responses API: Anthropic-style → flat items (message, function_call, function_call_output, reasoning)
- `$WD()` for Chat Completions: Anthropic-style → standard messages array

### Streaming
- Responses API: iterate `for await (event)` and dispatch by `event.type`
- Chat Completions: iterate chunks and process `choices[].delta`
- Use `IYH()` for accumulator state, `oBD()`/`tBD()` for event processing

### Retry
- Wrap API calls with `UEH()` retry wrapper
- `sG(error)` checks if error is retryable
- `MYH({ isExecMode })` provides retry configuration

## Open Questions / Left Undone

- **Azure-specific configuration**: The exact Azure endpoint URL format and authentication headers are handled by the proxy/config layer, not directly visible in the adapter modules. The `azure_openai` provider string is recognized but no Azure-specific SDK initialization code was found: likely handled at a higher configuration level or by the OpenAI SDK's built-in Azure support.
- **Model parameter mapping (`ahL`/`GBA`)**: The reasoning effort and max output tokens parameter builder functions are imported from external modules and were not fully traced.
- **Proxy authentication (`nZ`)**: The proxy auth header generation function was not deeply analyzed: it generates headers like session ID and assistant message ID for the OpenDroid proxy.
- **Custom model config resolution (`v7`, `TT`)**: Model configuration lookup from custom model settings was not fully traced.
- **`$YH` incremental parser**: The full incremental JSON parsing logic in 3126.js could benefit from deeper analysis for understanding partial argument extraction.
- **SDK vendor modules (2402.js, 2398.js, 2404.js)**: Only top-level structure was analyzed; internal retry/timeout/parsing logic within the OpenAI SDK was not deeply traced.
