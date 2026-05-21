# Tool & Agent System: Integrated Architecture Report

## Overview

This document synthesizes findings from 18 individual reverse-engineering feature reports into a unified architecture description of OpenDroid's Tool & Agent System. The system comprises a centralized tool registry (`tool-core-registry`), 14+ built-in tools (`tool-edit`, `tool-execute`, `tool-create-read`, `tool-grep-glob`, `tool-websearch-fetchurl`, `tool-todo-askuser`, `tool-task-subagent`, `tool-apply-patch`), a Model Context Protocol integration layer (`tool-mcp-client`, `tool-mcp-server-mgmt`, `tool-figma-mcp`), a permissions and sandboxing framework (`tool-permissions`, `tool-sandbox`), LLM adapters (`tool-llm-anthropic`, `tool-llm-openai`), a streaming response pipeline (`tool-streaming`), and an agent state machine (`tool-agent-state`). Together these subsystems form a complete agentic tool-use cycle: LLM inference → tool-call detection → permission gating → sandbox validation → tool execution → result serialization → re-entry into the LLM context.

## Feature Reference Index

| # | Feature ID | Report File | Subsystem | Key Modules |
|---|-----------|-------------|-----------|-------------|
| 1 | `tool-core-registry` | tool-core-registry.md | Registry Core | 1176.js, 1177.js, 1201.js, 2548.js, 2518.js, 2519.js, 0966.js, 1029.js, 1175.js, 1178.js, 2547.js |
| 2 | `tool-edit` | tool-edit.md | File Editing | 1217.js, 1250.js, 2321.js, 2322.js, 2484.js, 2188.js, 3139.js, 3370.js, 2528.js, 2548.js, 1220.js, 0943.js |
| 3 | `tool-execute` | tool-execute.md | Shell Execution | 1201.js, 2335.js, 2339.js, 1183.js, 1123.js, 1256.js, 0966.js, 0944.js, 3867.js, 3428.js |
| 4 | `tool-create-read` | tool-create-read.md | File I/O | 1217.js, 1218.js, 1224.js, 1213.js, 2510.js, 2320.js, 2509.js, 1148.js, 1174.js, 2477.js, 2197.js, 2199.js, 2189.js |
| 5 | `tool-grep-glob` | tool-grep-glob.md | Code Search | 1217.js, 1222.js, 1223.js, 0966.js, 2501.js, 2502.js, 2503.js, 2504.js, 2505.js, 2506.js, 3374.js, 3375.js, 3744.js, 1251.js |
| 6 | `tool-websearch-fetchurl` | tool-websearch-fetchurl.md | Web Integration | 0966.js, 1206.js, 1207.js, 1232.js, 2519.js, 2855.js, 2856.js, 3373.js, 3386.js, 3387.js, 3922.js |
| 7 | `tool-todo-askuser` | tool-todo-askuser.md | Planning & Interaction | 1246.js, 1225.js, 1217.js, 2317.js, 2318.js, 2316.js, 3361.js, 2572.js, 3411.js, 3399.js, 2519.js, 0966.js, 0060.js, 0724.js, 3417.js, 3420.js, 0084.js, 3428.js, 3384.js, 0226.js |
| 8 | `tool-task-subagent` | tool-task-subagent.md | Subagent Delegation | 2522.js, 2521.js, 1147.js, 1228.js, 1227.js, 2519.js, 2518.js, 2545.js, 3384.js, 3710.js, 2334.js, 0724.js |
| 9 | `tool-mcp-client` | tool-mcp-client.md | MCP Protocol Client | 1107.js, 1126.js, 1178.js, 1175.js, 1124.js, 1110.js, 1125.js, 1148.js, 0948.js, 0087.js |
| 10 | `tool-mcp-server-mgmt` | tool-mcp-server-mgmt.md | MCP Server Lifecycle | 1178.js, 1126.js, 0267.js, 0076.js, 0261.js, 0259.js, 0264.js, 0087.js, 0910.js |
| 11 | `tool-permissions` | tool-permissions.md | Permission Engine | 3139.js, 3141.js, 2324.js, 0885.js, 1185.js, 0724.js, 3417.js, 3418.js, 3413.js, 3416.js, 0212.js, 3137.js, 1256.js, 3140.js |
| 12 | `tool-sandbox` | tool-sandbox.md | Sandbox / Isolation | 0212.js, 0213.js, 2324.js, 0939.js, 3748.js, 3749.js, 0278.js, 0894.js, 1217.js, 3137.js, 3139.js, 3144.js, 3371.js, 3872.js, 0076.js, 0074.js, 2553.js, 2568.js |
| 13 | `tool-apply-patch` | tool-apply-patch.md | Patch Application | 1211.js, 2479.js, 1191.js, 1192.js, 1189.js, 0905.js, 0896.js, 0903.js, 0084.js, 0060.js |
| 14 | `tool-llm-anthropic` | tool-llm-anthropic.md | Anthropic LLM Adapter | 2375.js, 2373.js, 2366.js, 2362.js, 2347.js, 0943.js, 0257.js, 0251.js, 1466.js, 1468.js, 2345.js, 3134.js, 2536.js |
| 15 | `tool-llm-openai` | tool-llm-openai.md | OpenAI LLM Adapter | 3134.js, 3127.js, 3126.js, 2572.js, 2580.js, 2404.js, 2402.js, 2398.js, 1463.js, 1028.js |
| 16 | `tool-agent-state` | tool-agent-state.md | Agent State Machine | 0059.js, 0060.js, 0061.js, 0919.js, 2575.js, 3399.js, 3419.js, 3428.js, 3708.js, 3761.js, 4040.js, 4065.js |
| 17 | `tool-streaming` | tool-streaming.md | Streaming Pipeline | 3126.js, 3134.js, 2572.js, 3399.js, 1253.js, 2578.js, 3417.js, 3419.js, 3428.js, 0061.js, 3866.js |
| 18 | `tool-figma-mcp` | tool-figma-mcp.md | Figma MCP Bridge | 3430.js, 1146.js, 1109.js, 1178.js, 1175.js, 1148.js, 1127.js, 0991.js |

---

## End-to-End Tool Execution Flow

The following diagram documents the complete lifecycle of a tool invocation, from LLM inference through result serialization:

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. LLM INFERENCE                                                    │
│  ┌───────────────────────────────────────────────┐                  │
│  │ tool-llm-anthropic / tool-llm-openai          │                  │
│  │ Client sends messages + tool manifest to LLM  │                  │
│  │ API (Anthropic/OpenAI/Gemini).                │                  │
│  │                                                │                  │
│  │ Tool manifest: built from registry k0()        │                  │
│  │ via eI() → JSON Schema conversion (1029.js)    │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │ SSE streaming response                        │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 2. STREAMING RESPONSE (tool-streaming)         │                  │
│  │ 3126.js: StreamingState accumulator (IYH)      │                  │
│  │ Parser per provider:                           │                  │
│  │  • rBD(): Anthropic SSE                       │                  │
│  │  • oBD(): OpenAI Responses API                │                  │
│  │  • tBD(): OpenAI Chat Completions             │                  │
│  │  • aBD(): Google Gemini                       │                  │
│  │                                                │                  │
│  │ Accumulates: streamingContent, toolUses[],     │                  │
│  │ toolInputBuffers, usage tokens, contentBlocks  │                  │
│  │                                                │                  │
│  │ State: streaming_assistant_message              │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │ tool_use blocks detected                      │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 3. STATE TRANSITION (tool-agent-state)         │                  │
│  │ 3399.js: Agent runner loop                     │                  │
│  │ State: streaming → executing_tool               │                  │
│  │ setWorkingState(uJ.ExecutingTool)              │                  │
│  │ SessionController (2575.js) emits               │                  │
│  │ "working-state-changed" event                   │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │                                                │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 4. VALIDATE (tool-core-registry)               │                  │
│  │ Registry lookup: k0().getTool(llmId)            │                  │
│  │ Schema validation against inputZodSchema       │                  │
│  │ Resolve executor: registry.getExecutor(id)      │                  │
│  │ Check isToolEnabled({modelProvider})            │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │ tool is valid                                  │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 5. PERMsample itemION GATE (tool-permissions)          │                  │
│  │ 3139.js: hWD(toolName, input)                  │                  │
│  │ Confirmation type resolution:                  │                  │
│  │  • Read/Grep/LS → auto-approve (false)         │                  │
│  │  • Create/Edit → check autoApproveFileEdits    │                  │
│  │  • Execute → risk level vs autonomy level      │                  │
│  │  • MCP tools → readOnlyHint + autonomy check   │                  │
│  │                                                │                  │
│  │ If confirmation needed:                        │                  │
│  │  State → waiting_for_tool_confirmation          │                  │
│  │  User options: proceed_once, proceed_always,   │                  │
│  │  cancel (7 outcome types)                       │                  │
│  │                                                │                  │
│  │ Autonomy model: 3 modes × 4 levels → 5 modes   │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │ approved                                       │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 6. SANDBOX CHECK (tool-sandbox)                │                  │
│  │ 2324.js: lCI command policy engine             │                  │
│  │  • Command denylist/allowlist (regex-based)    │                  │
│  │  • FS path sandbox (3748.js, 3749.js)          │                  │
│  │  • Git security gate (0278.js)                 │                  │
│  │  • Environment scrubbing (3144.js)             │                  │
│  │  • CSP network gating (0894.js)               │                  │
│  │                                                │                  │
│  │ If violation: SandboxViolation event →         │                  │
│  │  permission escalation or rejection            │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │ sandbox passed                                 │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 7. TOOL HANDLER EXECUTION                      │                  │
│  │ Dispatched by tool type:                       │                  │
│  │                                                │                  │
│  │ File Tools:                                    │                  │
│  │  • tool-edit: 1250.js → oGH/tGH (text match)  │                  │
│  │  • tool-create-read: 2510.js/2320.js (file IO) │                  │
│  │  • tool-apply-patch: 2479.js → 1191.js (RX)    │                  │
│  │                                                │                  │
│  │ Search Tools:                                  │                  │
│  │  • tool-grep-glob: 2502.js → ripgrep (rg)      │                  │
│  │                                                │                  │
│  │ Execution Tools:                               │                  │
│  │  • tool-execute: 2335.js → cross-spawn          │                  │
│  │    (PowerShell on Windows, bash on Unix)        │                  │
│  │  • tool-task-subagent: 2521.js → child proc     │                  │
│  │                                                │                  │
│  │ Web Tools (server-side):                       │                  │
│  │  • tool-websearch-fetchurl: HTTP to OpenDroid     │                  │
│  │    server → search/scrape → markdown result     │                  │
│  │                                                │                  │
│  │ Planning Tools:                                │                  │
│  │  • tool-todo-askuser: 2572.js state manager     │                  │
│  │    + 2316.js/2318.js (ask-user flow)            │                  │
│  │                                                │                  │
│  │ MCP Tools:                                     │                  │
│  │  • tool-mcp-client: 1107.js → callTool()        │                  │
│  │  • tool-figma-mcp: via HTTP+SSE to mcp.figma   │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │ tool result                                    │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 8. SERIALIZE RESULT                            │                  │
│  │ • Text: string result (may be truncated)       │                  │
│  │ • Images: base64-encoded (1148.js, 1174.js)    │                  │
│  │ • Diffs: formatted output (2188.js)            │                  │
│  │ • MCP: xOA() serializer (1148.js)              │                  │
│  │ • Large output: artifact persistence (2855.js) │                  │
│  │                                                │                  │
│  │ Metrics: MR tracker (execution time, counts)   │                  │
│  │ File tracking: wmA/$rH (2189.js)               │                  │
│  └──────────────────┬────────────────────────────┘                  │
│                     │                                                │
│                     ▼                                                │
│  ┌───────────────────────────────────────────────┐                  │
│  │ 9. RE-ENTER LLM CONTEXT                       │                  │
│  │ Tool result appended to conversation history   │                  │
│  │ as tool_result content block.                  │                  │
│  │ Agent loop (3399.js) checks for more tool_use  │                  │
│  │ blocks. If present → dispatch next tool.       │                  │
│  │ If none → return to step 1 for next LLM call. │                  │
│  │                                                │                  │
│  │ State: executing_tool → streaming_assistant    │                  │
│  └───────────────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Cross-System Dependency Map

### Internal Tool System Dependencies

```
                    ┌─────────────────────────┐
                    │   tool-core-registry     │
                    │  1176.js (qgH class)     │
                    │  1177.js (k0 singleton)   │
                    │  1201.js (eI factory)     │
                    └─────────┬───────────────┘
                              │
              ┌───────────────┼──────────────────────┐
              │               │                      │
              ▼               ▼                      ▼
    ┌─────────────┐  ┌──────────────┐     ┌──────────────────┐
    │ Built-in     │  │  MCP Tools   │     │ Task Subagent    │
    │ Tools (14+)  │  │ (tool-mcp-   │     │ (tool-task-      │
    │              │  │  client)     │     │  subagent)       │
    │ 1175.js wrap │  │              │     │                  │
    └──────┬──────┘  │ 1178.js svc  │     │ 2547.js lazy-reg │
           │         └──────┬───────┘     └──────┬───────────┘
           │                │                     │
           └────────────────┼─────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │    tool-permissions     │
              │ 3139.js (hWD resolver)  │
              │ 1185.js (OGH service)   │
              │ 3141.js (options gen)   │
              └─────────┬───────────────┘
                        │
                        ▼
              ┌─────────────────────────┐
              │      tool-sandbox       │
              │ 2324.js (lCI policy)    │
              │ 3748.js (FS sandbox)    │
              │ 0278.js (git gate)      │
              └─────────┬───────────────┘
                        │
                        ▼
         ┌──────────────────────────────┐
         │    Tool Handlers             │
         │  tool-edit, tool-execute,    │
         │  tool-create-read,           │
         │  tool-grep-glob,             │
         │  tool-apply-patch,           │
         │  tool-todo-askuser,          │
         │  tool-websearch-fetchurl     │
         └──────────┬───────────────────┘
                    │ results
                    ▼
         ┌──────────────────────────────┐
         │    tool-streaming            │
         │ 3126.js (StreamingState)     │
         │ 2578.js (callback bridge)    │
         └──────────┬───────────────────┘
                    │
                    ▼
         ┌──────────────────────────────┐
         │    tool-agent-state          │
         │ 2575.js (SessionController)  │
         │ 3399.js (agent loop)         │
         │ 0060.js (state enum uJ)      │
         └──────────┬───────────────────┘
                    │
                    ▼
         ┌──────────────────────────────┐
         │    LLM Adapters              │
         │ tool-llm-anthropic           │
         │ tool-llm-openai              │
         │ (back to LLM for next turn)  │
         └──────────────────────────────┘
```

### MCP Integration Dependency Chain

```
tool-mcp-server-mgmt          tool-figma-mcp
(0267.js settings mgr)        (3430.js presets)
(0076.js config schema)       (1146.js OAuth)
         │                           │
         ▼                           ▼
    tool-mcp-client ◄────────────────┘
    (1107.js Zk client)
    (1126.js McpHub VOA)
    (1124.js stdio transport)
    (1110.js HTTP transport)
         │
         ▼
    tool-core-registry (1178.js registers MCP tools)
         │
         ▼
    tool-permissions (MCP tools: requiresConfirmation: true)
         │
         ▼
    tool-sandbox (default high-impact for MCP)
```

### Permission ↔ State Machine Coupling

```
Agent Loop (3399.js)
    │
    ├─ LLM response has tool_use
    │   └─ setWorkingState(executing_tool)
    │       │
    │       ├─ hWD() returns confirmation type
    │       │   └─ setWorkingState(waiting_for_tool_confirmation)
    │       │       │
    │       │       ├─ User approves
    │       │       │   └─ setWorkingState(executing_tool)
    │       │       │       └─ execute tool → serialize result
    │       │       │
    │       │       └─ User rejects
    │       │           └─ setWorkingState(streaming_assistant)
    │       │               └─ re-enter LLM with rejection message
    │       │
    │       └─ hWD() returns false (auto-approved)
    │           └─ execute tool → serialize result
    │
    ├─ All tool_use blocks processed
    │   └─ setWorkingState(streaming_assistant_message)
    │       └─ re-enter LLM with tool results
    │
    └─ LLM response has no tool_use
        └─ setWorkingState(idle)
```

---

## System Architecture by Subsystem

### 1. Tool Registry Core (tool-core-registry)

The registry is the central hub of the tool system. The `qgH` class (1176.js) maintains a `Map<toolId, ToolEntry>` keyed by tool ID. The `k0()` function (1177.js) provides a lazily-initialized singleton. Tools are defined via `eI()` (1201.js) which wraps a Zod schema, converts to JSON Schema for LLM consumption (1029.js), and attaches metadata. 19 built-in tools are registered at startup in 2548.js.

**Key design**: The `d1` enum (2519.js) maps human-readable tool names to LLM-facing IDs. Tool categories (`O9f`, `Z9f`) in 2518.js classify tools as read-only, edit, execute, web, or mcp: used by the permission engine.

### 2. Built-in Tools

#### File Editing Layer
- **tool-edit**: Two variants: single Edit (`R7H` in 2322.js) and MultiEdit (`xrA` in 2484.js). Core matching in 1250.js uses exact `String.includes()` with CRLF normalization and fuzzy-match fallback (bit-parallel algorithm). Edit is conditionally disabled for OpenAI models (`isToolEnabled: ({modelProvider}) => modelProvider !== "openai"`).
- **tool-apply-patch**: Custom diff format (`*** Begin Patch` / `*** Update File:`) parsed by `RX` class (1191.js). Multi-tier context matching: exact → trim-end → trim-both → ordered subsequence. Enabled for OpenAI models as an alternative to Edit.
- **tool-create-read**: Create writes files with directory creation and newline normalization (2320.js). Read supports text with offset/limit pagination (2509.js), images with JPEG compression pipeline (1174.js), and PDF reading. Both enforce absolute-path validation and write-protection for artifacts/mission files.

#### Search Layer
- **tool-grep-glob**: Both tools delegate to a bundled ripgrep binary extracted at runtime from embedded base64 (2501.js). Shared execution wrapper `N9H()` (2502.js) handles timeout, exit codes, and platform differences. Grep supports regex, file type filtering, context lines, and dual output modes. Glob supports inclusion/exclusion patterns.

#### Execution Layer
- **tool-execute**: Cross-platform shell execution via `child_process.spawn`. Windows routes through PowerShell (`-ExecutionPolicy Bypass`), Unix through bash. Three-tier risk level system (low/medium/high) gates user approval. Background process support via `nohup` (Unix) / `Start-Process` (Windows). Dangerous command detection via regex blocklist.
- **tool-task-subagent**: Spawns child `droid exec` processes with composed system prompts. Droid configurations resolved from `.opendroid/droids/` (project) and `~/.opendroid/droids/` (personal). Depth-limited recursion. Stateless execution: single prompt → autonomous run → single final report.

#### Interaction Layer
- **tool-todo-askuser**: TodoWrite parses `[completed]/[in_progress]/[pending]` status markers into structured objects stored in `ConversationStateManager` (2572.js). Staleness detection injects reminders after threshold tool calls. AskUser presents structured questionnaires with `[question]`/`[option]` markers, collecting answers via JSON-RPC or TUI modes.

#### Web Layer
- **tool-websearch-fetchurl**: Both server-side tools (`executionLocation: "server"`). WebSearch provides keyword/neural/auto search with category and domain filtering. FetchUrl scrapes URLs with extensive validation (blocks localhost, private networks, corporate infrastructure) and HTML-to-markdown conversion. Large outputs are truncated with artifact persistence.

### 3. MCP Integration Layer

- **tool-mcp-client**: Full MCP client implementation over stdio and HTTP transports. `Zk` class (1107.js) handles protocol handshake, `listTools`/`callTool` JSON-RPC methods. `McpHub` (1126.js) manages multiple server connections. Discovered tools are registered in the global registry with `mcp__{server}__{tool}` namespace.
- **tool-mcp-server-mgmt**: Configuration via `mcp.json` files with layered settings hierarchy (org → user → project → folder → plugin). `McpService` singleton (1178.js) coordinates lifecycle (start/stop/restart), OAuth for HTTP servers, and hot-reload on config changes.
- **tool-figma-mcp**: Figma is an MCP preset connecting to `https://mcp.figma.com/mcp` via HTTP+SSE transport with OAuth 2.0 PKCE authentication. Uses `"Gemini CLI MCP Client"` user-agent for Figma compatibility. Tools registered as `mcp_figma_{toolName}` with `requiresConfirmation: true`.

### 4. Permission & Sandbox Framework

- **tool-permissions**: Layered autonomy model combining 3 interaction modes (auto/spec/agi) with 4 autonomy levels (off/low/medium/high) producing 5 composite modes. Each tool call classified into 7 confirmation types. User presented with up to 7 outcome options. The `hWD()` function (3139.js) is the central decision point; `OGH` session service (1185.js) manages autonomy state.
- **tool-sandbox**: Defense-in-depth without OS-level containers. Command allowlist/denylist (`lCI` in 2324.js), FS path sandboxing (3748.js), git argument exploit filtering (0278.js), CSP network gating (0894.js), and environment variable scrubbing (3144.js). Validation happens at tool-handler layer with UI confirmation prompts.

### 5. LLM Adapters

- **tool-llm-anthropic**: Vendor SDK client (`rB` in 2375.js) with dual auth (`X-Api-Key` / `Authorization: Bearer`). Streaming via `P2H` accumulator (2362.js) parsing Anthropic SSE events. Tool-calling uses `tool_use`/`tool_result` content block format. Supports Bedrock and Vertex provider routing. Extended thinking support with reasoning effort configuration.
- **tool-llm-openai**: Dual-path adapter supporting both Responses API (`responses.create`) and Chat Completions API (`chat.completions.create`). Format conversion between internal Anthropic-style format and OpenAI native formats. Azure OpenAI as provider variant with endpoint routing. Encrypted reasoning blocks, reasoning summaries, and multi-phase responses.

### 6. Streaming Pipeline (tool-streaming)

`StreamingState` accumulator (`IYH()` in 3126.js) captures all streaming data: text content, tool calls, token usage, thinking blocks. Provider-specific parsers (`rBD`, `oBD`, `tBD`, `aBD`) incrementally parse SSE events. Callback bridge (2578.js) connects streaming events to UI state mutations and event bus emission. The pipeline feeds directly into the agent loop (3399.js) which dispatches tools based on accumulated `toolUses[]`.

### 7. Agent State Machine (tool-agent-state)

Five-state enum `uJ`: idle → streaming_assistant_message → executing_tool → waiting_for_tool_confirmation → compacting_conversation. Transitions are imperative (not formal state machine), scattered across 3399.js (agent loop), 3419.js (shared runner), 3428.js (JSON-RPC), and 3708.js (CLI runner). `SessionController` (2575.js) is the single source of truth via `setWorkingState()`. Event bus propagates changes to all consumers via `"working-state-changed"` event.

---

## Module Concentration Map

The following modules appear across multiple features and represent the highest-value integration points:

| Module | Appears In | Role | Cross-Cut Importance |
|--------|-----------|------|---------------------|
| 2519.js | tool-core-registry, tool-websearch-fetchurl, tool-todo-askuser, tool-task-subagent | Tool name enum + droid manager | **Critical**: central tool ID mapping |
| 1217.js | tool-edit, tool-create-read, tool-grep-glob, tool-todo-askuser, tool-execute, tool-sandbox | Shared Zod schemas | **Critical**: all tool input schemas |
| 1201.js | tool-core-registry, tool-edit, tool-execute | Tool factory `eI()` | **Critical**: tool creation |
| 1178.js | tool-core-registry, tool-mcp-client, tool-mcp-server-mgmt, tool-figma-mcp | McpService singleton | **Critical**: MCP lifecycle |
| 3399.js | tool-todo-askuser, tool-agent-state, tool-streaming | Agent runner loop | **Critical**: main execution driver |
| 3139.js | tool-edit, tool-permissions, tool-sandbox, tool-agent-state | Confirmation gate | **Critical**: permission decision point |
| 2572.js | tool-todo-askuser, tool-llm-openai, tool-streaming, tool-agent-state | Conversation state | **Critical**: state persistence |
| 3134.js | tool-llm-anthropic, tool-llm-openai, tool-streaming | LLM call orchestrator | **Critical**: provider routing |
| 0966.js | tool-core-registry, tool-execute, tool-grep-glob, tool-websearch-fetchurl, tool-todo-askuser | Tool name constants | **High**: ID string constants |
| 3428.js | tool-execute, tool-todo-askuser, tool-agent-state, tool-streaming | JSON-RPC adapter | **High**: IPC bridge |
| 3417.js | tool-todo-askuser, tool-permissions, tool-mcp-client, tool-streaming | ACP adapter | **High**: IDE bridge |
| 1175.js | tool-core-registry, tool-mcp-client, tool-figma-mcp | MCP tool wrapper | **High**: MCP tool execution |
| 2324.js | tool-permissions, tool-sandbox | Command policy engine | **High**: sandbox enforcement |
| 0060.js | tool-todo-askuser, tool-agent-state | State enums | **High**: state definitions |
| 1148.js | tool-create-read, tool-mcp-client, tool-figma-mcp | Content serialization | **High**: result formatting |

---

## Cross-Mission Integration Points

### Terminal UI section (TUI / Ink Components)
- Tool rendering: 3370.js (Edit), 3374.js (Glob), 3375.js (Grep), 3373.js (FetchUrl), 3386.js (WebSearch), 3384.js (TodoWrite/Task), 3371.js (Sandbox labels)
- Agent state: 4040.js hook provides working state to Ink components; 4065.js gates message queue on idle state

### Orchestration section (Mission Orchestration / Validation)
- Permission flow: `start_mission_run` confirmation type gates mission execution
- Task tool: orchestrator spawns worker subagents via Task tool with composed system prompts
- File change tracking: `captureToolFileChange()` integrates with validation system

### Desktop GUI section (GUI / Electron)
- ACP protocol: 3417.js bridges permission requests, streaming events, and tool results to IDE
- MCP management UI: add/remove/toggle/auth for MCP servers
- Tool descriptions: 3922.js renders tool list in GUI with descriptions

### Infrastructure section (Session / Infrastructure)
- Settings persistence: command allowlist/denylist, autonomy levels, MCP configs stored in settings files
- Session state: working state persisted per session via OpenDroidSessionManager (3761.js)
- Daemon IPC: all tool operations exposed as `daemon.*` JSON-RPC methods

---

## Key Architectural Patterns

### 1. Singleton Registry with Lazy Initialization
`k0()` (1177.js) uses `og()` lazy initializer pattern: registry is created on first access, not at import time. This avoids circular dependency issues between tool definitions and the registry.

### 2. Zod Schema → JSON Schema Pipeline
All tool inputs defined as Zod schemas (1217.js), converted to JSON Schema via `f7$()` (1029.js) for LLM function-calling manifests. OpenAI uses a different Zod-to-JSON converter (1028.js) with `openAi` target mode.

### 3. Dual API Compatibility (OpenAI Models)
Edit tool disabled for OpenAI (`isToolEnabled: ({modelProvider}) => modelProvider !== "openai"`), replaced by apply_patch. This is because OpenAI's function calling doesn't handle old_str/new_str replacement patterns well.

### 4. Event-Driven State Propagation
Agent state changes emitted via `v$` event bus (`"working-state-changed"`). Consumers (TUI hooks, ACP adapter, JSON-RPC adapter) subscribe independently. No direct coupling between state producer and consumers.

### 5. MCP Tool Namespace Convention
MCP tools use `mcp__{server}__{tool}` internal ID and `{server}___{tool}` LLM ID format. The triple-underscore convention is used by the permission engine (3139.js) to identify MCP tools: `toolName.includes("___")`.

### 6. Provider-Specific Streaming Parsers
Each LLM provider has a dedicated SSE parser in 3126.js but they all write to the same `StreamingState` object. This allows the agent loop (3399.js) to be provider-agnostic.

### 7. Defense-in-Depth Security
No single security gate: command policy, FS sandbox, git gate, CSP, and environment scrubbing all operate independently. Permission engine operates above sandbox; sandbox can escalate back to permissions.

---

## Implementation Notes: System-Level Recommendations

### Priority 1: Core Framework
1. **Tool Registry**: Port `qgH` class (1176.js) + `k0()` singleton (1177.js) + `eI()` factory (1201.js). These three modules form the foundation.
2. **Zod Schema Pipeline**: Port 1217.js (all tool schemas) + 1029.js (Zod→JSON Schema converter). Required for LLM function-calling manifest generation.
3. **Agent Loop**: Port 3399.js (core loop) + 2575.js (SessionController). The tool-use cycle is driven here.

### Priority 2: Essential Tools
4. **Edit + Create + Read**: File manipulation trio. Port 1250.js (matching engine), 2320.js (Create), 2510.js (Read).
5. **Execute**: Port 2335.js (ShellExecutor) + 2339.js (handler). Critical for code operations.
6. **Grep + Glob**: Port 2502.js (ripgrep wrapper). High-value for code understanding.

### Priority 3: Safety & Integration
7. **Permissions**: Port 3139.js (hWD resolver) + 3141.js (option generator) + 1185.js (session service).
8. **Sandbox**: Port 2324.js (command policy) + 3748.js (FS sandbox) + 0278.js (git gate).
9. **Streaming**: Port 3126.js (StreamingState + parsers). Required for real-time tool execution.

### Priority 4: Advanced Features
10. **MCP**: Port 1107.js (client) + 1126.js (hub) + 1178.js (service). Enables tool extensibility.
11. **Task Subagent**: Port 2522.js (executor) + 2521.js (stream processor). Enables delegation.
12. **LLM Adapters**: Port 3134.js (orchestrator) + provider-specific modules.

---

## Open Questions / Left Undone

1. **Tool output caching**: Several reports reference output caching mechanisms (2855.js, 2856.js) but the full caching strategy was not deeply analyzed.
2. **ACP protocol details**: The ACP adapter (3417.js) bridges tools to IDE but the full ACP protocol specification is outside 03-tool-agent-system scope.
3. **Droid configuration schema**: The OpenDroidManager (2519.js) and DroidConfigValidator (2518.js) schemas for custom droid configurations were not fully traced.
4. **OAuth token refresh**: The OAuth flow for HTTP-based MCP servers (Figma, etc.) was documented at a high level but token lifecycle details are incomplete.
5. **Context compaction**: The `compacting_conversation` state in the agent state machine triggers context compression, but the compaction algorithm itself was not analyzed in detail.
6. **Retry logic specifics**: Both LLM adapters include retry logic with exponential backoff but the exact retry strategies (max attempts, backoff intervals) vary by provider and were not exhaustively documented.
7. **Skill system**: The Skill tool (referenced in registry but not a separate feature) and GenerateDroid tool were not analyzed as individual features.
8. **Metrics/telemetry**: The `MR` metric recorder and `iL.addToCounter` telemetry calls appear across many modules but the full metrics pipeline was not traced.

---

*This report synthesizes 18 individual feature reports totaling approximately 270KB of analysis. Each feature report should be consulted for detailed code snippets, module-level architecture diagrams, and specific implementation details.*
