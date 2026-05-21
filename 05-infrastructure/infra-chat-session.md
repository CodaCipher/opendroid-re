# infra-chat-session: Architecture Notes

## Overview

OpenDroid's chat/conversation model is built around a central **ConversationStateManager** (module 2572.js, class `as`, ~987 lines) that manages the full conversation lifecycle: user messages, assistant streaming responses, tool calls/results, system notifications, hook executions, and todo state. The conversation is stored as an in-memory `conversationHistory` array of structured message objects with roles (user/assistant/system/tool), visibility flags (both/llm_only/user_only), and rich content blocks (text, thinking, tool_use, tool_result). Streaming from LLM providers is handled by OpenAI-compatible streaming wrappers (modules 2362.js, 2398.js, 2399.js) that parse SSE events into messages. Message role mapping for LangChain compatibility is provided by module 1475.js. The system supports multi-turn conversations with tool use, streaming text/thinking blocks, pending tool result tracking, and UI rendering optimizations.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 2572.js | ~35KB | Core ConversationStateManager (`as`): conversation history, message state, tool tracking, streaming blocks, UI rendering | App |
| 2362.js | ~20KB | OpenAI streaming message wrapper (`P2H`): SSE stream consumption, message/event emission, abort handling | App |
| 2398.js | ~10KB | Chat completion stream handler (`a2H`): extends base with `_addMessage`, `_addChatCompletion`, tool call extraction | App |
| 2399.js | ~8KB | Tool-running stream handler (`s2H`): extends `a2H` with `runTools`, content emission on message | App |
| 1475.js | ~6KB | LangChain message role mapping utilities: role normalization, message formatting for LLM APIs | App |
| 0945.js | ~3KB | JSONL session file parser: parses session_start + message events (covered in infra-session-manager) | App |

> **Note:** Seed modules 1543.js (ErrTracker ANR), 0111.js (OTel metric constants), 3322.js (highlight.js syntax), 4071.js (app init/logging), 2686.js (gRPC Server), 3266.js (Maxima CAS syntax) are **not** chat-session related. Actual chat modules were discovered through grep-based discovery targeting `conversationHistory`, `_addMessage`, role patterns, and streaming patterns.

## Architecture

### Conversation Turn Model

The conversation is a sequence of message objects stored in `conversationHistory[]`. Each message has:

```
{
  id: string,           // UUID
  role: "user" | "assistant" | "system" | "tool",
  content: string | ContentBlock[],
  visibility: "both" | "llm_only" | "user_only",
  createdAt: number,    // Date.now()
  updatedAt: number,
  // Optional fields:
  openaiMessageId?: string,
  openaiPhase?: number,
  hookEventName?: string,
  hookStatus?: string,
  transient?: boolean
}
```

**Content block types** (in `content[]` arrays):
- `{ type: "text", text: string }`: Text content
- `{ type: "thinking", thinking: string, signature: string }`: Extended thinking (Claude)
- `{ type: "redacted_thinking", data: string }`: Redacted thinking
- `{ type: "tool_use", id: string, name: string, input: object }`: Tool call
- `{ type: "tool_result", toolUseId: string, content: string, isError?: boolean }`: Tool result
- `{ type: "image" }`: Image attachment

### Message Flow (User Input → Model → Response)

```
1. User types message
   → updateAction({ type: "ADD_USER_MESSAGE", content })
   → conversationHistory.push({ role: "user", content, visibility: "both" })

2. API call initiated
   → startAssistantMessage(id)
   → conversationHistory.push({ role: "assistant", content: [] })

3. Streaming response chunks arrive
   → appendAssistantText(index, delta)        // text blocks
   → appendThinkingDelta(index, delta)         // thinking blocks
   → addToolCall(id, name, input, sig)         // tool calls
   → streamingContentBlocks accumulates
   → syncContentBlocksToMessage() merges into assistant content[]

4. Stream completes
   → finalizeContentBlocks(thinkingBlocks, toolUses)
   → finalizeAssistantMessage(openaiMessageId, phase)
   → If tool uses exist: auto-create tool result message with __TOOL_RESULT_PENDING__

5. Tool executes
   → updateToolResult(toolUseId, result)
   → Pending tool result replaced with actual content
   → If tool is TodoWrite → todo state updated

6. Repeat from step 2 if tool results need model processing
```

### Streaming Architecture

The streaming layer uses an event-driven architecture:

```
P2H (2362.js): Base streaming wrapper
  ├── Connects to OpenAI API via SSE
  ├── _createMessage() → creates streaming connection
  ├── _addMessageParam() → stores request messages
  ├── _addMessage() → stores received messages, emits events
  ├── _emitFinal() → emits finalChatCompletion, finalMessage, finalContent
  └── Events: "message", "content", "end", "abort", "error", "connect"

a2H (2398.js): Chat completion stream
  ├── Extends P2H with _addChatCompletion()
  ├── _runChatCompletion() → non-streaming API call
  ├── _addMessage() → extracts tool calls, emits functionToolCall events
  └── finalContent(), finalMessage(), finalFunctionToolCall()

s2H (2399.js): Tool-running stream
  ├── Extends a2H with runTools()
  ├── _addMessage() → emits "content" on assistant messages
  └── Automatic tool execution during streaming
```

### Visibility System

Messages have a visibility flag controlling who sees them:

- **"both"**: Visible to both UI and LLM (normal user/assistant messages)
- **"llm_only"**: Sent to LLM but hidden from UI (system notifications injected for model context)
- **"user_only"**: Shown in UI but not sent to LLM (hook execution status, UI indicators)

### UI Rendering Optimization

- **Debounced rendering**: `scheduleReactUpdate()` batches UI updates with 20ms debounce
- **UI updates can be suspended**: `setUiUpdatesSuspended(true)` during heavy operations
- **Initial render limit**: Only last 30 messages shown initially; older messages loaded on demand
- **UI render cutoff**: After compaction, only messages after cutoff point are rendered; older ones hidden
- **Streaming content blocks**: `streamingContentBlocks` Map tracks in-progress text/thinking blocks

### Tool Execution State Machine

```
Tool lifecycle in ConversationStateManager:

1. addToolCall(id, name, input)
   → toolExecutions.set(id, { status: "pending" })

2. updateToolCallInput(id, partialInput)
   → Merges streaming tool input

3. updateToolStatus(id, "executing")
   → status = "executing"

4. updateToolResult(id, result)
   → status = "completed" | "error"
   → Content truncated at 4000 chars for memory efficiency
   → Pending result placeholder replaced

5. setToolError(id, errorMessage)
   → status = "error"

6. pruneOldToolExecutions()
   → Keeps max 50 tool executions, prunes oldest completed
```

### Pending Tool Results Pattern

When an assistant message contains tool uses, the system creates a placeholder tool result message:

```js
// finalizeAssistantMessage creates:
{
  role: "tool",
  content: [{ type: "tool_result", toolUseId, content: "__TOOL_RESULT_PENDING__" }]
}
```

This placeholder is later replaced with the actual result via `updateToolResult()`.

### Validation: Tool Use Pairing

`validateToolUsePairing()` ensures every `tool_use` block has a matching `tool_result`. If a tool result is missing or has `__TOOL_RESULT_PENDING__`, a synthetic error result is injected:

```js
{ type: "tool_result", toolUseId, content: "Error: Tool execution was cancelled" }
```

### Message Role Mapping (LangChain)

Module 1475.js provides role normalization for LangChain compatibility:

```
human → "user"
ai → "assistant"
assistant → "assistant"
system → "system"
function → "function"
tool → "tool"
```

The `LX1()` function converts LangChain message objects to standard `{ role, content }` format by inspecting `_getType()`, `constructor.name`, `type`, `role`, or LangChain `lc`/`kwargs` properties.

## Key Findings

1. **ConversationStateManager is the central state machine**: Module 2572.js class `as` manages ALL conversation state in a single in-memory array with ~987 lines. It handles user messages, assistant streaming, tool calls, tool results, hook executions, and todo state through an action-based `updateInMemoryState()` switch.

2. **Visibility-aware message model**: Every message has a `visibility` flag ("both"/"llm_only"/"user_only") that controls whether it appears in the UI, is sent to the LLM, or both. System notifications are injected as `role: "user"` with `visibility: "llm_only"` so the model sees context the user doesn't.

3. **Pending tool result pattern**: When a tool call is made, a placeholder `__TOOL_RESULT_PENDING__` is immediately inserted into the conversation. This allows the conversation history to remain valid (every tool_use has a matching tool_result) even before execution completes.

4. **Streaming content blocks**: Text and thinking blocks are accumulated in a separate `streamingContentBlocks` Map during streaming, then periodically synced into the assistant message's content array. This allows real-time UI updates without corrupting the message structure.

5. **Tool execution pruning**: Only the last 50 tool executions are kept in memory. Results are truncated at 4000 characters for memory efficiency.

6. **OpenAI-compatible streaming wrappers**: The streaming layer (2362.js/2398.js/2399.js) follows the OpenAI SDK pattern with event emitters for "message", "content", "chatCompletion", "functionToolCall", and "end" events.

## Code Examples

### Adding a user message (2572.js, line ~68)
```js
case "ADD_USER_MESSAGE": {
  let A = H.id || U1();
  this.conversationHistory.push({
    id: A,
    role: "user",
    content: H.content,
    visibility: "both",
    createdAt: Date.now(),
    updatedAt: Date.now(),
  });
  break;
}
```

### Finalizing an assistant message with tool use (2572.js, line ~808)
```js
finalizeAssistantMessage(H, A) {
  // ...
  if (this.currentAssistantMessage.toolUses.length > 0) {
    let L = this.currentAssistantMessage.toolUses.map((I) => ({
      type: "tool_result",
      toolUseId: I.id,
      content: aZ,  // "__TOOL_RESULT_PENDING__"
    }));
    this.conversationHistory.push({
      id: U1(),
      role: "tool",
      content: L,
      visibility: "both",
      createdAt: Date.now(),
      updatedAt: Date.now(),
    });
  }
}
```

### Getting conversation history for LLM (2572.js, line ~462)
```js
getConversationHistory() {
  let H = this.conversationHistory.filter((L) => {
    if (!gxH(L)) return false;
    if (L.role === "assistant") {
      let $ = L.content.some((D) => D.type === "text" && D.text?.trim() !== ""),
        I = L.content.some((D) => D.type === "tool_use");
      if (L.content.length === 0 || (!$ && !I)) return false;
    }
    if (L.role === "user" && Array.isArray(L.content)) {
      if (L.content.every((I) => I.type === "tool_result" && I.content === aZ))
        return false;
    }
    return true;
  });
  return as.validateToolUsePairing(H).map(({ visibility: L, ...$ }) => $);
}
```

### Streaming message creation (2362.js, line ~80)
```js
static createMessage(H, A, L) {
  let $ = new P2H(A);
  for (let I of A.messages) $._addMessageParam(I);
  return (
    OI($, pMH, { ...A, stream: true }, "f"),
    $._run(() => $._createMessage(H, { ...A, stream: true }, L)),
    $
  );
}
```

### LangChain role mapping (1475.js, line ~1)
```js
Hg$ = {
  human: "user",
  ai: "assistant",
  assistant: "assistant",
  system: "system",
  function: "function",
  tool: "tool",
};
```

## Integration Points (Cross-System)

- **Session Manager (infra-session-manager)**: `ConversationStateManager` loads conversation history from session files via `loadConversationHistory()` and `loadUiRenderCutoffFromSession()` which calls `BA().loadLatestCompactionSummary()`. Session files (`.jsonl`) persist the full conversation.

- **Session Compaction**: After compaction, `uiRenderCutoffMessageId` is set to hide older messages from the UI while keeping the compacted summary in the LLM context.

- **TUI (Terminal UI section)**: `uiMessageCallback` is the bridge to the TUI rendering layer. `scheduleReactUpdate()` batches UI updates and calls this callback with derived UI messages.

- **GUI (Desktop GUI section)**: Module 4065.js (GUI session management) likely consumes the same `ConversationStateManager` for the Electron GUI's conversation display.

- **Tool System (Tool & Agent section)**: Tool calls are tracked in `toolExecutions` Map with lifecycle (pending → executing → completed/error). Tool progress updates flow through `UPDATE_TOOL_PROGRESS` action. Tool results are validated and paired via `validateToolUsePairing()`.

- **Plugin/Hook System (infra-plugin-manager)**: Hook executions appear as system messages with `hookEventName`, `hookMatcher`, `hookCommands`, `hookStatus` fields. They use `visibility: "user_only"`: shown in UI but not sent to LLM.

- **Config Loader (infra-config-loader)**: System prompts and model settings from config are injected as messages with `visibility: "llm_only"`.

## Implementation Notes

1. **Extract ConversationStateManager**: The core state machine in 2572.js is self-contained with minimal external dependencies. Port the class directly, replacing `U1()` (UUID) and `BH()`/`gH()` (logging) with your own implementations.

2. **Replace streaming wrappers**: Modules 2362.js/2398.js/2399.js are OpenAI SDK-specific. Replace with your LLM provider's streaming SDK or use the OpenAI-compatible API directly.

3. **Implement visibility-aware rendering**: The three-tier visibility system (both/llm_only/user_only) is crucial for separating model context from user display. Ensure your UI layer respects these flags.

4. **Port tool result pending pattern**: The `__TOOL_RESULT_PENDING__` placeholder pattern ensures conversation integrity during async tool execution. This is essential for multi-turn tool use flows.

5. **Simplify or replace LangChain mapping**: Module 1475.js's LangChain compatibility layer is only needed if using LangChain. For OpenDroid, use direct message objects with standard roles.

## Module Reference

| File | Lines | Function/Class | Description |
|------|-------|---------------|-------------|
| 2572.js | 1-987 | `class as` (ConversationStateManager) | Central conversation state management |
| 2572.js | 22-44 | `constructor(callback)` | Initializes conversationHistory, toolExecutions, streamingContentBlocks |
| 2572.js | 55-240 | `updateInMemoryState(action)` | Main action dispatcher: handles all message types |
| 2572.js | 241-270 | `scheduleReactUpdate(reason)` | Debounced UI update (20ms) |
| 2572.js | 271-310 | `syncContentBlocksToMessage()` | Merges streaming blocks into assistant message |
| 2572.js | 311-440 | `deriveUIMessages()` | Converts conversationHistory to UI-friendly message array |
| 2572.js | 462-519 | `getConversationHistory()` | Filters and validates history for LLM API call |
| 2572.js | 520-590 | `validateToolUsePairing(history)` | Ensures every tool_use has matching tool_result |
| 2572.js | 600-660 | `startAssistantMessage(id)` | Begins new assistant turn, force-completes stuck tools |
| 2572.js | 661-700 | `appendAssistantText(index, delta)` | Appends streaming text |
| 2572.js | 701-740 | `addToolCall(id, name, input)` | Registers tool call during streaming |
| 2572.js | 761-810 | `finalizeAssistantMessage(id, phase)` | Completes assistant turn, creates pending tool results |
| 2572.js | 841-890 | `updateToolResult(toolUseId, result)` | Replaces pending tool result with actual content |
| 2362.js | 1-484 | `class P2H` | OpenAI streaming message wrapper |
| 2362.js | 52-76 | `constructor(params)` | Initializes messages, receivedMessages, AbortController |
| 2362.js | 80-96 | `static createMessage()` | OpenDroid for streaming message creation |
| 2362.js | 97-155 | `_addMessageParam()`, `_addMessage()` | Message storage and emission |
| 2362.js | 156-200 | `_createMessage()` | SSE connection and stream consumption |
| 2398.js | 1-227 | `class a2H` | Chat completion stream handler |
| 2398.js | 15-40 | `_addChatCompletion()`, `_addMessage()` | Completion/message handling with tool extraction |
| 2398.js | 41-80 | `finalContent()`, `finalMessage()` | Async final result getters |
| 2399.js | 1-171 | `class s2H` | Extended stream with tool running |
| 2399.js | 5-10 | `static runTools()` | OpenDroid for tool-executing stream |
| 2399.js | 11 | `_addMessage()` override | Emits "content" on assistant messages |
| 1475.js | 1-156 | Role mapping utilities | LangChain message format conversion |
| 1475.js | 1-10 | `Hg$` | Role name mapping table |
| 1475.js | 20-35 | `CQH(role)`, `Ag$(className)` | Role normalization functions |
| 1475.js | 45-65 | `LX1(messages)` | LangChain → standard message array conversion |
