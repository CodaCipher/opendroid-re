# Protocol: Session Format
## Status: DEFINITIVE (cross-validated)

Cross-referenced observed protocol samples from `protocol-samples/runtime-findings.md` with architecture notes reports `infra-session-manager.md` and `infra-chat-session.md`. Observed protocol samples are the highest authority; all discrepancies are marked with ⚠️.

---

## 1. Directory Structure

### 1.1 Session Storage Layout

```
~/.opendroid/
├── sessions/                              # All session data
│   ├── {cwd-slug}/                        # One subdirectory per working directory
│   │   ├── {session-uuid}.jsonl           # Session transcript (JSONL)
│   │   ├── {session-uuid}.settings.json   # Session settings sidecar
│   │   └── ...
│   └── ...
└── sessions-index.json                    # Global session index
```

### 1.2 Directory Naming Convention

The `{cwd-slug}` is derived from the working directory path by replacing path separators and colons with hyphens:

| Working Directory | Session Directory |
|---|---|
| `<workspace>\OpenDroid` | `-workspace-OpenDroid` |
| `C:\Users\<user>` | `-C-Users-user` |
| `<workspace>\SampleProject` | `-workspace-SampleProject` |

**Algorithm**: Replace `\` and `:` with `-`. The resulting slug is used directly as the subdirectory name.

⚠️ **Discrepancy**: architecture notes (infra-session-manager.md) describes `Fk(cwd)` as a hash function that creates a "project-hash". Observed protocol samples shows the slug is a simple character-replacement encoding, NOT a hash. The RE code likely uses the encoding internally but the observable output is the path-based slug.

### 1.3 File Naming

- **Transcript**: `{sessionId}.jsonl` where `sessionId` is a UUID (e.g., `<session-id-1>.jsonl`)
- **Settings**: `{sessionId}.settings.json` (sidecar pattern: settings stored separately from transcript)

---

## 2. Session JSONL Format

Each line in the `.jsonl` file is a JSON object. The format is append-only: new messages are appended without modifying previous lines.

### 2.1 Event Types

#### 2.1.1 `session_start`: Session Initialization

Always the first line. Creates the session metadata.

```json
{
  "type": "session_start",
  "id": "<session-id-1>",
  "title": "<session-title>",
  "sessionTitle": "New Session",
  "owner": "<user>",
  "version": 2,
  "cwd": "C:\\Users\\<user>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"session_start"` |
| `id` | string (UUID) | Yes | Session identifier |
| `title` | string | Yes | Session title (derived from first user message or auto-generated) |
| `sessionTitle` | string | Yes | Initial title placeholder (observed: `"New Session"`) |
| `owner` | string | Yes | Username of the session owner |
| `version` | integer | Yes | Schema version. Currently `2` |
| `cwd` | string | Yes | Working directory for this session |

#### 2.1.2 `message`: Conversation Messages

All subsequent lines are message events.

**User message (text):**
```json
{
  "type": "message",
  "id": "<session-id-2>",
  "timestamp": "2026-04-20T03:25:46.750Z",
  "message": {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "<system-reminder>...</system-reminder>"
      },
      {
        "type": "text",
        "text": "actual user message"
      }
    ]
  },
  "parentId": "<session-id-2>"
}
```

**Assistant message with Anthropic thinking + tool use:**
```json
{
  "type": "message",
  "id": "...",
  "timestamp": "...",
  "message": {
    "role": "assistant",
    "content": [
      {
        "type": "thinking",
        "signature": "<thinking-signature>",
        "signatureProvider": "anthropic",
        "thinking": "the thinking content..."
      },
      {
        "type": "tool_use",
        "id": "call_function_oplxzp4kozqm_1",
        "name": "FetchUrl",
        "input": {"url": "https://docs.opendroid.dev/llms.txt"}
      }
    ]
  },
  "parentId": "..."
}
```

**Tool result message:**
```json
{
  "type": "message",
  "message": {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "call_function_oplxzp4kozqm_1",
        "is_error": false,
        "content": "URL Content from: ..."
      }
    ]
  },
  "parentId": "..."
}
```

**⚠️ Key observation**: Tool results use `role: "user"`, NOT `role: "tool"`. This differs from the internal ConversationStateManager which uses `role: "tool"` in memory. The JSONL persistence layer normalizes tool results under the user role.

**Message with visibility (system-generated):**
```json
{
  "type": "message",
  "message": {
    "role": "user",
    "content": [{"type": "text", "text": "Error: Connection error."}],
    "visibility": "user_only"
  }
}
```

#### 2.1.3 `compaction_state`: Compaction Summary (RE Only)

⚠️ **Not observed in observed protocol samples**. Documented from architecture notes only.

```json
{
  "type": "compaction_state",
  "id": "comp-uuid",
  "timestamp": "ISO",
  "summaryText": "...",
  "summaryTokens": 500,
  "summaryKind": "llm_summary",
  "removedCount": 42,
  "systemInfo": {...}
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"compaction_state"` |
| `id` | string | Yes | Unique ID for this compaction |
| `timestamp` | string (ISO 8601) | Yes | When compaction occurred |
| `summaryText` | string | Yes | The LLM-generated summary |
| `summaryTokens` | integer | Yes | Estimated token count of summary |
| `summaryKind` | string | Yes | `"llm_summary"` observed |
| `removedCount` | integer | Yes | Number of messages removed |
| `systemInfo` | object | No | System context preserved across compaction |

#### 2.1.4 `todo_state`: Todo Tracking (RE Only)

⚠️ **Not observed in observed protocol samples**. Documented from architecture notes only.

```json
{
  "type": "todo_state",
  "id": "todo-uuid",
  "timestamp": "ISO",
  "todos": [...],
  "messageIndex": 15
}
```

### 2.2 Message Envelope Schema

All `message` events share this envelope:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"message"` |
| `id` | string (UUID) | No | Message identifier. Sometimes absent on tool results |
| `timestamp` | string (ISO 8601) | No | When the message was created |
| `message` | object | Yes | The message payload |
| `message.role` | string | Yes | `"user"` or `"assistant"` |
| `message.content` | array | Yes | Array of content blocks |
| `message.visibility` | string | No | `"user_only"` observed. Default: not present (visible to both) |
| `parentId` | string | No | Reference to previous message ID (for threading) |

### 2.3 Content Block Types

#### 2.3.1 `text`: Text Content

```json
{"type": "text", "text": "Hello, how can I help?"}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"text"` |
| `text` | string | Yes | The text content |

Used in: All roles. User messages, assistant responses, system notifications.

#### 2.3.2 `thinking`: Extended Thinking (Anthropic Provider)

```json
{
  "type": "thinking",
  "signature": "<thinking-signature>",
  "signatureProvider": "anthropic",
  "thinking": "Let me analyze this step by step..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"thinking"` |
| `thinking` | string | Yes | The thinking/reasoning content |
| `signature` | string | Yes | Hex-encoded signature for thinking verification |
| `signatureProvider` | string | Yes | Always `"anthropic"` |

Used in: Assistant messages only. Only when using Anthropic provider with extended thinking enabled.

#### 2.3.3 `tool_use`: Tool Invocation

```json
{
  "type": "tool_use",
  "id": "call_function_oplxzp4kozqm_1",
  "name": "FetchUrl",
  "input": {"url": "https://docs.opendroid.dev/llms.txt"}
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"tool_use"` |
| `id` | string | Yes | Tool call identifier. Format varies by provider |
| `name` | string | Yes | Tool name (e.g., `"FetchUrl"`, `"Read"`, `"Execute"`) |
| `input` | object | Yes | Tool input parameters as key-value pairs |

Used in: Assistant messages only.

**Tool ID format**: `call_function_{randomString}_{counter}` observed for OpenAI-compatible providers.

#### 2.3.4 `tool_result`: Tool Execution Result

```json
{
  "type": "tool_result",
  "tool_use_id": "call_function_oplxzp4kozqm_1",
  "is_error": false,
  "content": "URL Content from: https://docs.opendroid.dev/llms.txt"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"tool_result"` |
| `tool_use_id` | string | Yes | References the `id` of the matching `tool_use` block |
| `content` | string | Yes | Tool execution output |
| `is_error` | boolean | Yes | Whether the tool execution resulted in an error |

⚠️ **Discrepancy**: architecture notes (infra-chat-session.md) uses `toolUseId` (camelCase) in the internal ConversationStateManager. Runtime JSONL uses `tool_use_id` (snake_case). The JSONL serialization layer converts to snake_case.

⚠️ **Discrepancy**: architecture notes uses `isError` (camelCase). Runtime JSONL uses `is_error` (snake_case).

#### 2.3.5 `redacted_thinking`: Redacted Thinking (RE Only)

⚠️ **Not observed in observed protocol samples**. Documented from architecture notes only.

```json
{"type": "redacted_thinking", "data": "..."}
```

Used when Anthropic returns redacted thinking blocks that should not be displayed.

#### 2.3.6 `image`: Image Attachment (RE Only)

⚠️ **Not observed in observed protocol samples**. Documented from architecture notes only.

```json
{"type": "image"}
```

Used for image attachments in messages. Full schema not captured.

### 2.4 Provider-Specific Fields

#### 2.4.1 OpenAI-Compatible Provider

For models using the OpenAI-compatible API (generic-chat-completion-api), assistant messages include additional top-level fields:

```json
{
  "openaiMessageId": "<message-id>",
  "chatCompletionReasoningField": "reasoning_content",
  "chatCompletionReasoningContent": "Let me think about this approach..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `openaiMessageId` | string | No | OpenAI API message identifier |
| `chatCompletionReasoningField` | string | No | Name of the reasoning field (`"reasoning_content"` observed) |
| `chatCompletionReasoningContent` | string | No | The reasoning/thinking text |

**Key difference**: OpenAI reasoning is stored as **top-level fields on the message envelope**, not as a content block. Anthropic thinking is a **content block** within `content[]`.

#### 2.4.2 Anthropic Provider

Anthropic uses the `thinking` content block type (see §2.3.2) embedded within the `content[]` array. No additional top-level fields needed.

**Comparison table:**

| Aspect | Anthropic | OpenAI-Compatible |
|--------|-----------|-------------------|
| Thinking location | Content block in `content[]` | Top-level field on message |
| Thinking type | `"thinking"` block | `chatCompletionReasoningContent` |
| Signature | `signature` + `signatureProvider` fields | Not present |
| Message ID | Standard `id` field | Additional `openaiMessageId` |
| Format verification | Hex signature hash | No verification |

### 2.5 Visibility System

Messages can have an optional `visibility` field controlling who sees them:

| Value | UI | LLM | Description |
|-------|-----|-----|-------------|
| (absent) | ✅ | ✅ | Default: visible to both. Normal messages. |
| `"user_only"` | ✅ | ❌ | Shown in UI but NOT sent to LLM. Used for system-generated errors, hook status. |
| `"llm_only"` | ❌ | ✅ | [INFERRED from RE] Sent to LLM but hidden from UI. System context injections. |

⚠️ **Discrepancy**: Only `"user_only"` observed in observed protocol samples. architecture notes documents all three values (`"both"`, `"llm_only"`, `"user_only"`). The `"both"` value is equivalent to absent (default).

---

## 3. Session Settings Schema

Each session has a `{sessionId}.settings.json` sidecar file.

### 3.1 Complete Schema

```json
{
  "assistantActiveTimeMs": 242854,
  "model": "custom:built-in model-[Z.AI]-0",
  "reasoningEffort": "none",
  "interactionMode": "agi",
  "autonomyLevel": "off",
  "autonomyMode": "normal",
  "tags": [
    {
      "name": "mission-session",
      "metadata": {
        "role": "orchestrator",
        "missionId": "<mission-id>"
      }
    }
  ],
  "providerLock": "generic-chat-completion-api",
  "providerLockTimestamp": "2026-04-20T23:31:29.449Z",
  "tokenUsage": {
    "inputTokens": 86496,
    "outputTokens": 7416,
    "cacheCreationTokens": 0,
    "cacheReadTokens": 1595840,
    "thinkingTokens": 0
  }
}
```

### 3.2 Field Documentation

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `assistantActiveTimeMs` | integer | Yes | `0` | Total time the assistant was active in milliseconds |
| `model` | string | Yes | Config default | Model ID. Format: `"custom:{displayName}-{index}"` for BYOK, or built-in name like `"claude-opus-4-7"` |
| `reasoningEffort` | string enum | Yes | `"none"` | Reasoning depth. Values: `"none"`, `"low"`, `"medium"`, `"high"` |
| `interactionMode` | string enum | Yes | `"auto"` | Interaction mode. `"auto"` = user approves actions. `"agi"` = autonomous |
| `autonomyLevel` | string | Yes | `"off"` | Autonomy level. `"off"` observed |
| `autonomyMode` | string | Yes | `"normal"` | Autonomy mode. `"normal"` observed |
| `tags` | array | No | `[]` | Session tags for categorization |
| `providerLock` | string | No |: | Provider type locked for this session. Observed: `"generic-chat-completion-api"`, `"anthropic"` |
| `providerLockTimestamp` | string (ISO 8601) | No |: | When the provider was locked |
| `tokenUsage` | object | No |: | Token usage statistics |

### 3.3 Tags Schema

```json
{
  "name": "tag-name",
  "metadata": { ... }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Tag identifier. Observed values: `"mission-session"`, `"exec"`, `"subagent"` |
| `metadata` | object | No | Additional tag metadata |

**Tag `metadata` for mission sessions:**

| Field | Type | Description |
|-------|------|-------------|
| `role` | string | `"orchestrator"` or `"worker"` |
| `missionId` | string (UUID) | The mission this session belongs to |

**Observed tag names and their meanings:**

| Tag | Purpose |
|-----|---------|
| `"mission-session"` | Session is part of a mission. Has `metadata` with role and missionId |
| `"exec"` | Session was created via exec/subagent dispatch |
| `"subagent"` | Session is a subagent spawned by another session |

### 3.4 Token Usage Schema

```json
{
  "inputTokens": 86496,
  "outputTokens": 7416,
  "cacheCreationTokens": 0,
  "cacheReadTokens": 1595840,
  "thinkingTokens": 0
}
```

| Field | Type | Description |
|-------|------|-------------|
| `inputTokens` | integer | Total input tokens consumed |
| `outputTokens` | integer | Total output tokens generated |
| `cacheCreationTokens` | integer | Tokens written to prompt cache |
| `cacheReadTokens` | integer | Tokens read from prompt cache |
| `thinkingTokens` | integer | Tokens used for thinking/reasoning |

---

## 4. Session Index (sessions-index.json)

A global index file at `~/.opendroid/sessions-index.json` tracking all sessions across all working directories.

### 4.1 Schema

```json
{
  "version": 1,
  "entries": [
    {
      "sessionId": "<session-id-3>",
      "mtime": <timestamp-ms-1>,
      "settingsMtime": <timestamp-ms-2>,
      "title": "LLM Competency Assessment for Concept Change Analysis",
      "cwd": "<workspace>\\SampleWorkspace\\SampleApp",
      "messagesCount": 730,
      "callingSessionId": "...",
      "callingToolUseId": "...",
      "tags": [
        {"name": "exec"},
        {"name": "subagent"}
      ]
    }
  ]
}
```

### 4.2 Field Documentation

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | integer | Yes | Schema version. Currently `1` |
| `entries` | array | Yes | All session metadata entries |

### 4.3 Entry Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sessionId` | string (UUID) | Yes | Unique session identifier |
| `mtime` | integer | Yes | Last modified time (Unix epoch milliseconds) |
| `settingsMtime` | integer | Yes | Settings file modification time (Unix epoch ms) |
| `title` | string | Yes | Session title (first user message or auto-generated) |
| `cwd` | string | Yes | Working directory for the session |
| `messagesCount` | integer | Yes | Total number of messages in the session |
| `callingSessionId` | string | No | Parent session that spawned this (for workers/subagents) |
| `callingToolUseId` | string | No | Tool call ID that spawned this session |
| `tags` | array | No | Tags array (same format as settings.json tags) |

---

## 5. Worker Session Protocol

### 5.1 Worker Session Creation

When the mission orchestrator spawns a worker, it creates a new session:

1. **Orchestrator session** (parent) calls the `Task` tool with `subagent_type` and task details
2. **New session** is created with `callingSessionId` set to the orchestrator's session ID
3. **`callingToolUseId`** is set to the tool call ID of the `Task` invocation
4. **Tags** are set: `["exec", "subagent"]` for subagent sessions, `["mission-session"]` with metadata for worker sessions

### 5.2 callingSessionId and callingToolUseId

These fields in `sessions-index.json` entries form a **parent-child relationship tree**:

```
Orchestrator Session (no callingSessionId)
├── Worker Session A (callingSessionId = orchestrator.id, callingToolUseId = tool_call_1)
│   └── Sub-agent Session (callingSessionId = workerA.id, callingToolUseId = tool_call_2)
├── Worker Session B (callingSessionId = orchestrator.id, callingToolUseId = tool_call_3)
└── Validator Session (callingSessionId = orchestrator.id, callingToolUseId = tool_call_4)
```

**Use cases:**
- **Mission workers**: `callingSessionId` → orchestrator session, `callingToolUseId` → the Task tool call
- **Subagents**: `callingSessionId` → parent session, `callingToolUseId` → the Task tool call
- **User sessions**: Both fields absent

### 5.3 Worker Session Transcripts

Worker sessions are stored in the same directory structure but have a specialized transcript reader (RE: module 3975.js):
- Tail-reading for efficiency (max 2MB read, 10K lines)
- Used by the orchestrator to monitor worker progress

---

## 6. Session Compaction Protocol

⚠️ **Primarily from architecture notes**. Compaction was not triggered during the observed runtime session data.

### 6.1 Compaction Types

| Type | Trigger | Mechanism | Result |
|------|---------|-----------|--------|
| Auto | Token budget exceeded | `GRI()` algorithm | Injects summary, keeps suffix |
| Manual | `/compact` command | `FRI()` algorithm | Creates new session with summary |

### 6.2 Auto-Compaction Thresholds

| Parameter | Value | Description |
|-----------|-------|-------------|
| `postAbsolute` | 40,000 tokens | Max tokens after compaction |
| `summarySoftCap` | 2,000 tokens | Target summary length |
| `summaryReserve` | 4,000 tokens | Reserved for summary overhead |
| Conversation limit | 250,000 tokens | Flag to trigger compaction |

### 6.3 Summary Injection

When a compacted session is loaded, the summary is injected as a synthetic user message:

```
"A previous instance of Droid has summarized the conversation thus far as follows:
<summary>
{summaryText}
</summary>

IMPORTANT: This summary was created by a previous instance of Droid. Files referenced 
in the summary may not be available until you explicitly view them again."
```

### 6.4 Delta Compaction

When a previous LLM summary exists:
1. Only messages since the last summary anchor are summarized
2. The new summary incorporates both the previous summary and new messages
3. This makes repeated compactions efficient

---

## 7. Examples

### Example 1: Complete Session Start + User Message + Assistant Response

```jsonl
{"type":"session_start","id":"<session-id-1>","title":"<session-title>","sessionTitle":"New Session","owner":"<user>","version":2,"cwd":"C:\\Users\\<user>"}
{"type":"message","id":"<session-id-2>","timestamp":"2026-04-20T03:25:46.750Z","message":{"role":"user","content":[{"type":"text","text":"<system-reminder>...</system-reminder>"},{"type":"text","text":"actual user message"}]},"parentId":"<session-id-2>"}
{"type":"message","id":"msg-002","timestamp":"2026-04-20T03:25:50.100Z","message":{"role":"assistant","content":[{"type":"thinking","signature":"<thinking-signature>","signatureProvider":"anthropic","thinking":"Let me think about this..."},{"type":"text","text":"Here's what I found..."}]},"parentId":"<session-id-2>"}
```

### Example 2: Assistant with Tool Use + Tool Result

```jsonl
{"type":"message","id":"msg-003","timestamp":"2026-04-20T03:26:00.000Z","message":{"role":"assistant","content":[{"type":"thinking","signature":"hex...","signatureProvider":"anthropic","thinking":"I need to fetch that URL"},{"type":"tool_use","id":"call_function_oplxzp4kozqm_1","name":"FetchUrl","input":{"url":"https://docs.opendroid.dev/llms.txt"}}]},"parentId":"msg-002"}
{"type":"message","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"call_function_oplxzp4kozqm_1","is_error":false,"content":"URL Content from: https://docs.opendroid.dev/llms.txt\n\n..."}]},"parentId":"msg-003"}
```

### Example 3: OpenAI-Compatible Model with Reasoning

```jsonl
{"type":"message","id":"msg-010","timestamp":"2026-04-21T07:31:31.000Z","message":{"role":"assistant","content":[{"type":"text","text":"Let me analyze the codebase..."}]},"parentId":"msg-009","openaiMessageId":"<message-id>","chatCompletionReasoningField":"reasoning_content","chatCompletionReasoningContent":"The user wants me to implement session tracking. I should first check the existing code..."}
```

### Example 4: System Error Message with User-Only Visibility

```jsonl
{"type":"message","id":"msg-050","timestamp":"2026-04-20T05:30:00.000Z","message":{"role":"user","content":[{"type":"text","text":"Error: Connection error."}],"visibility":"user_only"},"parentId":"msg-049"}
```

### Example 5: Mission Worker Session in sessions-index.json

```json
{
  "sessionId": "<run-id>",
  "mtime": <timestamp-ms-1>,
  "settingsMtime": <timestamp-ms-2>,
  "title": "Worker: project-scaffold",
  "cwd": "<workspace>\\SampleProject",
  "messagesCount": 45,
  "callingSessionId": "orchestrator-session-uuid",
  "callingToolUseId": "call_function_worker_spawn_1",
  "tags": [
    {"name": "mission-session", "metadata": {"role": "worker", "missionId": "<mission-id>"}}
  ]
}
```

---

## 8. Discrepancies (Inferred vs Observed)

| # | Field/Behavior | Architecture Notes Says | Observed Samples Show | Resolution |
|---|---------------|------------------|--------------------|------------|
| 1 | Directory naming | `Fk(cwd)` hash function | Simple path encoding (`\` and `:` → `-`) | Runtime is correct. RE describes internal function; output is the path slug |
| 2 | Tool result `role` | `role: "tool"` (ConversationStateManager) | `role: "user"` (JSONL persistence) | JSONL normalizes tool results under user role. Internal model uses "tool" |
| 3 | `tool_use_id` field | `toolUseId` (camelCase) | `tool_use_id` (snake_case) | JSONL uses snake_case. Internal model uses camelCase |
| 4 | `is_error` field | `isError` (camelCase) | `is_error` (snake_case) | JSONL uses snake_case. Internal model uses camelCase |
| 5 | `compaction_state` event | Documented in session manager | Not observed in observed protocol samples | Event type exists but compaction wasn't triggered during observation |
| 6 | `todo_state` event | Documented in session manager | Not observed in observed protocol samples | Event type exists but not triggered during observation |
| 7 | `redacted_thinking` block | Documented in chat session | Not observed in observed protocol samples | Block type exists but not triggered (requires specific Anthropic config) |
| 8 | `image` block | Documented in chat session | Not observed in observed protocol samples | Block type exists but no image attachments in observed sessions |
| 9 | `visibility: "both"` | Explicit "both" value used | Absent = both (no explicit value) | Absence of field implies "both". RE documents explicit value; runtime omits it |
| 10 | `tokens` on messages | Mentioned in session manager code | Not present on messages in JSONL | Token tracking is in settings sidecar, not per-message |
| 11 | `parentId` field | Not documented | Confirmed present on messages | Thread chain for message ordering |
| 12 | OpenAI reasoning format | Not documented in session manager | `chatCompletionReasoningContent` as top-level field | RE missed the OpenAI-compatible reasoning serialization |
| 13 | `signatureProvider` field | Not documented | `"anthropic"` confirmed | Provider tag for thinking signature verification |
| 14 | `providerLock` in settings | Not documented | `"generic-chat-completion-api"` or `"anthropic"` | Runtime reveals provider locking mechanism |
| 15 | `tokenUsage` in settings | Schema documented | Confirmed with all 5 sub-fields | RE and runtime agree |

---

## 9. State Machine: Session Lifecycle

```
                    ┌──────────────┐
                    │   CREATED    │  session_start event written
                    │  (new .jsonl)│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
              ┌─────│    ACTIVE    │─────┐
              │     │  (messaging) │     │
              │     └──────┬───────┘     │
              │            │             │
     ┌────────▼───────┐   │    ┌────────▼────────┐
     │   COMPACTING   │   │    │     PAUSED      │
     │ (auto/manual)  │   │    │  (user paused)  │
     └────────┬───────┘   │    └────────┬────────┘
              │           │             │
     ┌────────▼───────┐   │             │
     │  COMPACTED     │   │    ┌────────▼────────┐
     │ (summary +     │   │    │    RESUMED      │
     │  suffix msgs)  │───┤    │  (re-activated) │──────► ACTIVE
     └────────────────┘   │    └─────────────────┘
                          │
                   ┌──────▼───────┐
                   │   ARCHIVED   │
                   │ (preserved)  │
                   └──────────────┘
```

**Transitions:**
| From | To | Trigger |
|------|----|---------|
| CREATED | ACTIVE | First user message appended |
| ACTIVE | COMPACTING | Token budget exceeded (auto) or `/compact` command |
| COMPACTING | COMPACTED | Summary generated, suffix selected |
| COMPACTED | ACTIVE | New messages continue in compacted session |
| ACTIVE | PAUSED | User pauses session (e.g., mission paused) |
| PAUSED | RESUMED | User resumes session |
| RESUMED | ACTIVE | Continue messaging |
| ACTIVE | ARCHIVED | User deletes/archives session |
| COMPACTING | ACTIVE | Compaction blocked by plugin hook (exit code 2/3) |

---

## 10. Open Questions

1. **Full `visibility` enum values**: Only `"user_only"` confirmed in runtime. `"llm_only"` and `"both"` are from architecture notes. [INFERRED]
2. **`autonomyLevel` values beyond `"off"`**: Only `"off"` observed. RE suggests more levels exist. [INFERRED]
3. **`autonomyMode` values beyond `"normal"`**: Only `"normal"` observed. [INFERRED]
4. **Compaction trigger timing**: Auto-compaction thresholds are from RE only. Not verified against runtime.
5. **Cloud sync behavior**: RE documents `/api/sessions/create` and `/api/v0/sessions/{id}` endpoints. Not observable from local data alone.
6. **`messagesCount` accuracy**: The index tracks message count but it's unclear if it counts all event types or just `message` events.
7. **Worker transcript format**: The worker-transcripts.jsonl stores full skeletons separately. Relationship between `.jsonl` session files and worker transcripts needs further investigation.
