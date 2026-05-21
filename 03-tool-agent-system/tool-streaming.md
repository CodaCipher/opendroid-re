# LLM Streaming Response Handling: Tool System Architecture Notes

## Overview

The streaming response pipeline is the core data-path that connects LLM API responses (Anthropic, OpenAI, Google Gemini) to OpenDroid's tool execution loop. Three provider-specific streaming parsers (`rBD` for Anthropic SSE, `oBD` for OpenAI Responses API, `tBD` for OpenAI Chat Completions, `aBD` for Google Gemini) accumulate text deltas, thinking blocks, and tool-call arguments incrementally into a unified `StreamingState` object. The accumulated `toolUses` array feeds directly into the agent loop (`3399.js`) which dispatches tool execution, collects results, and re-enters the LLM call: forming the agentic tool-use cycle. The pipeline uses incremental JSON parsing for tool input buffers (`$YH`) enabling mid-stream tool-input preview. Event emitters (`v$`) broadcast streaming progress to ACP adapters (3417.js), JSON-RPC adapters (3428.js), and UI consumers (2572.js).

## Module Map
| Module | Size (KB) | Role | Category |
|--------|-----------|------|----------|
| 3126.js | ~28.7 | Streaming state factory (`IYH`), Anthropic SSE parser (`rBD`), OpenAI Responses parser (`oBD`), OpenAI Chat parser (`tBD`), Gemini parser (`aBD`), incremental JSON parser (`$YH`) | app-tools |
| 3134.js | ~28.7 | LLM streaming client (`wWD`): orchestrates provider selection, API calls, stream iteration, retry, and result aggregation | app-tools |
| 2572.js | ~22.1 | UI conversation store (`class as`): streamingContentBlocks Map, deriveUIMessages, scheduleReactUpdate, tool execution tracking | app-tools |
| 3399.js | ~34.1 | Agent loop: `while(true)` streaming→tool-execution cycle, working state transitions, context compaction trigger | app-tools |
| 1253.js | ~18.7 | DataStream protocol parts: `tool_call_streaming_start`, `tool_call_delta`, `finish_message`, `finish_step` wire format | app-tools |
| 2578.js | ~22.1 | Callback bridge (`ytA`): connects streaming callbacks (onTextDelta, onToolUseDetected, onStreamingComplete) to UI state mutations | app-tools |
| 3417.js | ~18.2 | ACP adapter: subscribes to streaming events, relays `assistant-text-delta`, `thinking-text-delta`, `tool-call-start/progress/complete` to connected clients | app-tools |
| 3419.js | ~22.1 | Shared agent runner event emitter (`cIA`): emits all streaming lifecycle events via `v$` EventEmitter | app-tools |
| 3428.js | ~22.1 | JSON-RPC adapter: subscribes to streaming events, sends `assistant_text_delta`, `thinking_text_delta`, `tool_progress_update` notifications | app-tools |
| 0061.js | ~43.8 | Working state labels: defines `streaming_assistant_message`, `executing_tool`, `compacting_conversation` states | app-tools |
| 3866.js | ~22.3 | Agent working states Set: `thinking`, `streaming`, `compressing`, `executing_tool` | app-tools |

## Architecture

### Streaming Response Pipeline

The streaming pipeline follows a layered architecture:

```
LLM API (Anthropic/OpenAI/Gemini)
    ↓ (SSE / streaming response)
Streaming Parser (3126.js: rBD / oBD / tBD / aBD)
    ↓ (increments StreamingState object)
LLM Client (3134.js: wWD)
    ↓ (callbacks: onTextDelta, onToolUseDetected, etc.)
Callback Bridge (2578.js: ytA)
    ↓ (UI state updates + event emission)
┌──────────────────────┬──────────────────────┐
│  UI Store (2572.js)  │  Event Bus (v$)      │
│  - streamingContent   │  - assistant-text-delta
│  - toolUses tracking  │  - tool-call-start   │
│  - React debounced    │  - tool-streaming-update
│    re-render (20ms)   │                      │
└──────────────────────┴──────────────────────┘
                              ↓
                    ACP (3417.js) / JSON-RPC (3428.js)
```

### StreamingState Object

The central data structure (`IYH()` in 3126.js) that accumulates all streaming data:

```js
function IYH() {
  return {
    streamingContent: "",       // Accumulated text content
    toolUses: [],               // Array of {id, name, input} tool calls
    toolInputBuffers: {},       // Raw JSON argument buffers per tool index
    usage: {                    // Token usage tracking
      inputTokens: 0,
      outputTokens: 0,
      cacheCreationInputTokens: 0,
      cacheReadInputTokens: 0,
      thinkingTokens: 0,
    },
    openaiMessageId: void 0,
    openaiPhase: void 0,
    openaiReasoningId: void 0,
    openaiEncryptedContent: void 0,
    finalStreamingContent: void 0,
    thinkingContent: void 0,
    thinkingSignature: void 0,
    contentBlocks: [],          // Ordered content block array
  };
}
```

### Provider-Specific Streaming Parsers

**1. Anthropic SSE Parser (`rBD`)**: processes Anthropic streaming events:
- `message_start` → captures usage (input tokens, cache tokens)
- `content_block_start` → detects `tool_use`, `thinking`, `redacted_thinking`, `text` block types
- `content_block_delta` → handles `text_delta`, `thinking_delta`, `signature_delta`, `input_json_delta`
- `content_block_stop` → marks block complete
- `message_delta` → captures output tokens and stop reason
- `message_stop` → finalizes, records token usage

**2. OpenAI Responses API Parser (`oBD`)**: processes OpenAI Responses streaming events:
- `response.created` → captures message ID
- `response.output_item.added` → detects `message`, `reasoning`, `function_call` items
- `response.output_text.delta` → accumulates text content
- `response.reasoning_summary_text.delta` → accumulates thinking content
- `response.function_call_arguments.delta` → incremental tool argument accumulation
- `response.function_call_arguments.done` → final JSON parse of tool arguments
- `response.completed` / `response.incomplete` → finalize with usage

**3. OpenAI Chat Completions Parser (`tBD`)**: processes Chat Completions streaming:
- `delta.content` → text accumulation
- `delta.reasoning` / `delta.reasoning_content` → thinking content
- `delta.tool_calls` → tool call detection with index tracking, argument accumulation
- `finish_reason` → finalizes all pending tool argument parsing

**4. Google Gemini Parser (`aBD`)**: processes Gemini SSE chunks:
- `candidates[0].content.parts` → iterates parts for text, thought, functionCall
- Handles `thoughtSignature` for Gemini-specific thinking signatures
- `usageMetadata` → token tracking

### Tool-Call Detection During Streaming

Tool calls are detected at different points depending on provider:

**Anthropic**: `content_block_start` with `type: "tool_use"` triggers `onToolUseDetected` callback immediately with `id`, `name`, and empty `input`. Arguments arrive incrementally via `input_json_delta` events, parsed through `$YH` incremental JSON parser and forwarded via `onToolInputDelta`.

**OpenAI Responses**: `response.output_item.added` with `type: "function_call"` triggers detection. Arguments accumulate via `response.function_call_arguments.delta` and finalize on `response.function_call_arguments.done`.

**OpenAI Chat**: `delta.tool_calls` array in streaming chunks. Each chunk may contain `function.name` (first detection) or `function.arguments` (incremental). Index tracking via `tool_calls[].index` or `id` mapping.

**Gemini**: `functionCall` parts detected in `candidates[0].content.parts`. Unlike others, Gemini sends complete tool calls (name + args) in a single chunk.

### Response Serialization

After streaming completes, the `StreamingState` is serialized into a standardized result object:

```js
{
  toolUses: z.toolUses,                              // Parsed tool calls
  finalStreamingContent: z.finalStreamingContent || z.streamingContent,
  usage: z.usage,                                     // Token counts
  toolInputBuffers: z.toolInputBuffers,               // Raw JSON buffers
  thinkingContent: z.thinkingContent,                 // Thinking block text
  contentBlocks: z.contentBlocks,                     // Ordered content blocks
  wasAborted: XH.signal.aborted,
  usedApiProvider: A,                                 // "anthropic"|"openai"|"fireworks"|"google"
  stopReason: z.stopReason,
}
```

The `contentBlocks` array is serialized into API-format content via `XvH()` (for structured blocks) or `CIM()` (legacy fallback) in 3399.js.

### Streaming → Tool Loop Integration

The agent loop in `3399.js` is a `while(true)` cycle:

1. **State**: `streaming_assistant_message`: LLM is streaming tokens
2. **Parse**: Streaming parsers accumulate `toolUses` array
3. **Serialize**: After stream ends, content blocks are serialized into assistant message
4. **Decision**: If `toolUses.length > 0` → enter tool execution phase
5. **State**: `executing_tool`: tools are dispatched via `wA(toolUses, messageId)`
6. **Results**: Tool results appended as `role: "tool"` message
7. **Loop**: Conversation re-enters step 1 with tool results added to history
8. **Exit**: When no tool calls present (stop_reason === "end_turn") or cancellation

### Event Emission System

The `v$` EventEmitter broadcasts streaming lifecycle events:

| Event | Payload | Consumers |
|-------|---------|-----------|
| `streaming-start` | `{sessionId}` | ACP adapter, JSON-RPC adapter |
| `assistant-text-delta` | `{messageId, blockIndex, textDelta, sessionId}` | ACP → `agent_message_chunk`, JSON-RPC → `assistant_text_delta` |
| `thinking-text-delta` | `{messageId, blockIndex, textDelta, sessionId}` | ACP → `agent_thought_chunk`, JSON-RPC → `thinking_text_delta` |
| `tool-call-start` | `{id, name, input, sessionId}` | ACP adapter (TodoWrite special handling) |
| `tool-call-progress` | `{id, partialInput, sessionId}` | ACP adapter (input merging) |
| `tool-streaming-update` | `{id, name, update, sessionId}` | JSON-RPC → `tool_progress_update` |
| `tool-call-complete` | `{id, result, isError, sessionId}` | ACP adapter, JSON-RPC adapter |
| `streaming-complete` | `{sessionId}` | ACP adapter, JSON-RPC adapter |
| `working-state-changed` | `{state, sessionId}` | All adapters |

### Incremental JSON Parsing

Tool input arguments are parsed incrementally using `$YH()` in 3126.js, which attempts partial JSON parsing on the accumulating buffer. This enables real-time UI display of tool parameters as they stream in (e.g., showing file paths in Edit tool before the full arguments are received).

## Key Findings

### 1. StreamingState Accumulator (3126.js, lines 199-218)

The `IYH()` factory creates a fresh accumulator per LLM call. The `toolUses` array grows as tool-call content blocks are detected, with `null` placeholders for non-tool block indices:

```js
function IYH() {
  return {
    streamingContent: "",
    toolUses: [],
    toolInputBuffers: {},
    usage: {
      inputTokens: 0, outputTokens: 0,
      cacheCreationInputTokens: 0,
      cacheReadInputTokens: 0,
      thinkingTokens: 0,
    },
    contentBlocks: [],
    // ... provider-specific fields
  };
}
```

### 2. Anthropic Tool-Call Detection Mid-Stream (3126.js, lines 305-330)

Tool calls are detected when `content_block_start` arrives with `type: "tool_use"`. The parser immediately notifies via `onToolUseDetected`:

```js
case "content_block_start": {
  let $ = H.index;
  if (H.content_block.type === "tool_use") {
    while (A.toolUses.length <= $) A.toolUses.push({ id: "", name: "", input: {} });
    A.toolUses[$] = {
      id: H.content_block.id,
      name: H.content_block.name,
      input: H.content_block.input,
    };
    A.contentBlocks[$] = {
      type: "tool_use", index: $, content: "",
      toolUseId: H.content_block.id,
      toolName: H.content_block.name, isComplete: false,
    };
    L.onToolUseDetected?.({
      id: H.content_block.id,
      name: H.content_block.name,
      input: H.content_block.input,
    });
  }
  // ... thinking, redacted_thinking, text cases
  break;
}
```

### 3. Incremental Tool Input Parsing (3126.js, lines 375-386)

Tool arguments arrive as partial JSON strings. The `$YH` function attempts to parse the incomplete buffer, enabling progressive UI updates:

```js
case "content_block_delta": {
  if (H.delta.type === "input_json_delta" && H.index !== void 0) {
    if (!A.toolInputBuffers[H.index]) A.toolInputBuffers[H.index] = "";
    let D = H.delta.partial_json || "";
    A.toolInputBuffers[H.index] += D;
    if (L.onToolInputDelta && D) {
      let E = A.toolUses[H.index];
      if (E && E.id) {
        let f = $YH(A.toolInputBuffers[H.index]);
        if (Object.keys(f.data).length > 0)
          L.onToolInputDelta(E.id, f.data);
      }
    }
  }
  break;
}
```

### 4. Agent Tool Loop (3399.js, lines 825-870)

After streaming completes, if `toolUses.length > 0`, the agent transitions to tool execution:

```js
let L1 = l$.toolUses.length > 0, cf = false;
if (L1) {
  EH((R0) => ({
    ...R0,
    status: { ...R0.status, state: "executing_tool",
      executionStartTime: Date.now(), invokingTools: false },
  }));
  HH?.onWorkingStateChange?.("executing_tool");
  v$.emit("working-state-changed", { state: "executing_tool", sessionId: sD });
  let { results: LD, wasCancelled: Q1 } = await wA(l$.toolUses, tI);
  cf = Q1;
  let VE = {
    id: LH() || U1(),
    role: "tool",
    content: LD,
    createdAt: Date.now(),
    updatedAt: Date.now(),
  };
  // Persist tool results and continue loop
}
```

### 5. DataStream Protocol Parts (1253.js, lines 102-128)

The wire protocol defines typed streaming parts including tool-call lifecycle events:

```js
$FH = {
  code: "5",
  name: "tool_call_streaming_start",
  parse: (H) => {
    if (!H || typeof H !== "object" ||
      !("toolCallId" in H) || typeof H.toolCallId !== "string" ||
      !("toolId" in H) || typeof H.toolId !== "string" ||
      !("toolDisplayName" in H) || typeof H.toolDisplayName !== "string")
      throw new vH("...");
    return { type: "tool_call_streaming_start", value: H };
  },
},
IFH = {
  code: "6",
  name: "tool_call_delta",
  parse: (H) => ({
    type: "tool_call_delta",
    value: { toolCallId: H.toolCallId, parametersTextDelta: H.parametersTextDelta }
  }),
},
```

### 6. UI Debounced Rendering (2572.js, lines 219-230)

The UI store debounces React re-renders to 20ms during streaming, preventing excessive DOM updates while maintaining responsive display:

```js
scheduleReactUpdate(H) {
  if (this.uiUpdatesSuspended) {
    this.hasPendingUiUpdate = true;
    return;
  }
  if (this.debounceTimer) clearTimeout(this.debounceTimer);
  this.debounceTimer = setTimeout(() => {
    let A = this.deriveUIMessages();
    this.uiMessageCallback(A);
  }, 20);
}
```

## Integration Points

- **01-terminal-ui (TUI/Ink)**: The UI store (2572.js) feeds React/Ink components via `deriveUIMessages()` and `uiMessageCallback`. Streaming content blocks drive real-time terminal rendering.
- **02-orchestration (Mission/Validation)**: The `streaming_assistant_message` and `executing_tool` working states (0061.js) are used by the mission system to track agent progress and enforce validation contracts.
- **04-desktop-gui (GUI/Electron)**: ACP adapter (3417.js) and JSON-RPC adapter (3428.js) relay streaming events to Electron renderer processes for GUI display.
- **05-infrastructure (Session/Infra)**: The streaming pipeline integrates with session management via `sessionId` passed through all event payloads. Context compaction (triggered when token count exceeds threshold) is part of the agent loop in 3399.js.
- **tool-agent-state**: Working state transitions (`streaming_assistant_message` → `executing_tool`) are defined in 0061.js and emitted via `working-state-changed` events.
- **tool-llm-anthropic / tool-llm-openai**: These features describe the LLM client adapters that feed into the streaming parsers in 3126.js.
- **tool-core-registry**: Tool definitions registered via `eI()` are fetched by the streaming client (`P()` function in 3134.js) and passed to LLM APIs as function-calling manifests.

## Implementation Notes

### Streaming Response Pipeline

1. **StreamingState accumulator**: Port the `IYH()` factory function as the core state object. It must hold `streamingContent`, `toolUses`, `toolInputBuffers`, `usage`, and `contentBlocks`.

2. **Provider parsers**: Each LLM provider needs its own streaming parser:
   - Anthropic: `rBD()`: handles SSE `content_block_start/delta/stop` events
   - OpenAI Responses: `oBD()`: handles `response.output_text.delta`, `response.function_call_arguments.delta`
   - OpenAI Chat: `tBD()`: handles `choices[].delta.content`, `choices[].delta.tool_calls`
   - Google: `aBD()`: handles `candidates[0].content.parts`

3. **Incremental JSON parser**: Port `$YH()` for progressive tool-input parsing during streaming.

4. **Agent loop**: The `while(true)` loop in 3399.js with state transitions between `streaming_assistant_message` and `executing_tool` phases.

5. **Event bus**: Port the `v$` EventEmitter pattern for decoupled streaming event propagation to UI, ACP, and JSON-RPC consumers.

6. **UI debouncing**: The 20ms debounce in `scheduleReactUpdate()` prevents rendering storms during high-throughput streaming.

## Open Questions / Left Undone

1. **Retry logic** (`UEH` in 3134.js): The retry wrapper around streaming calls was not fully traced: it appears to handle rate-limiting and provider rotation but the full mechanism needs deeper analysis.
2. **Context compaction trigger**: The compaction threshold (`daH` in 3399.js) and its interaction with streaming was identified but not fully traced.
3. **Tool streaming updates** (`tool-streaming-update` events from running tools like Execute): The mechanism for long-running tools to emit progress updates during execution was identified in 3417.js and 3428.js but the emission side (how tools push updates) was not fully traced.
4. **Streaming error recovery**: The `EmptyLLMResponseError_0947` detection and context-limit retry path in 3399.js was identified but the full recovery flow deserves deeper analysis.
5. **Multi-provider serialization**: The `XvH()` function that converts `contentBlocks` + `toolUses` into provider-specific message format was not fully traced.
