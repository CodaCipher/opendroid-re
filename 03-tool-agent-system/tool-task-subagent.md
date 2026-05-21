# Task Tool (Subagent Spawning): Tool System Architecture Notes

## Overview

The Task tool is OpenDroid's subagent delegation mechanism, allowing a parent agent to spawn autonomous child agents (called "droids") to handle complex, multi-step tasks. The system operates as a child-process-based sandbox: each subagent is launched as a separate `droid exec` CLI invocation with its own context window, tool set, and system prompt. The subagent runs statelessly: it receives a single prompt, executes autonomously, and returns a single final report to the parent. Droid types are resolved from configuration files in `.opendroid/droids/` (project) and `~/.opendroid/droids/` (personal), with project-level droids taking precedence. The system supports parallel subagent invocation, depth-limited recursion to prevent infinite nesting, and full abort/cleanup lifecycle management.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 2522.js | ~4 KB | Task tool executor (TaskCliExecutor class): validates inputs, resolves droid config, builds CLI command args | App |
| 2521.js | ~12 KB | SubagentStreamProcessor: spawns child process, parses streaming JSON events, manages process lifecycle | App |
| 1147.js | ~2 KB | Task tool description/prompt (KgH constant): LLM-facing documentation | App |
| 1228.js | ~0.5 KB | Task tool input schema (P11): zod schema for subagent_type, description, prompt | App |
| 1227.js | ~2 KB | Task tool registration (RZA function): eI() call registering "task-cli" tool | App |
| 2519.js | ~6 KB | OpenDroidManager class (d2): CRUD for droid configs, loadAllDroids(), settings-backed storage | App |
| 2518.js | ~8 KB | DroidConfigValidator (k5): validateMetadata, validateSystemPrompt, expandTools, tool normalization | App |
| 2545.js | ~3 KB | Custom droid system prompt injection: YNI() builds droid availability section for agent prompt | App |
| 3384.js | ~10 KB | Task tool TUI renderer (nJD): Ink-based rendering of progress updates, tool calls, results | App |
| 3710.js | ~22 KB | CLI entry point: `droid exec` command with --depth, --calling-session-id, --calling-tool-use-id, --enabled-tools flags | App |
| 2334.js | ~3 KB | ProcessManager (x6): registerProcess, unregisterProcess, killToolProcesses for child process tracking | App |
| 0724.js | ~1 KB | Depth tracking: getDepth()/setDepth() for recursion depth management | App |

## Architecture

### Task Tool Schema

```js
// 1228.js: Input schema (P11)
p.object({
  subagent_type: p.string().describe("The type of specialized agent to use for this task"),
  description: p.string().describe("A short (3-5 word) description of the task"),
  prompt: p.string().describe("The task for the agent to perform"),
})

// 1227.js: Output schema
outputSchemas: { result: p.string().describe("The output from the task subagent execution") }
```

The tool is registered as `task-cli` with `llmId: "Task"`, `executionLocation: "client"`, `isTopLevelTool: true`, and `requiresConfirmation: false`.

### Execution Flow

```
LLM → Task tool call → TaskCliExecutor.execute()
  1. Validate subagent_type (required string) and prompt (required string)
  2. Resolve droid config: iZ().loadAllDroids().find(name === subagent_type)
  3. If not found → error with available droids list
  4. Build CLI args via buildCommandArgs():
     - Compose system prompt wrapping droid config + user prompt
     - Resolve model (inherit or custom)
     - Expand tools from droid metadata
     - Set depth = current + 1
     - Link session/tool-call IDs
  5. Spawn child process: `droid exec <composed-prompt>`
  6. SubagentStreamProcessor.process() streams JSON events from stdout
  7. Parse events: tool_call, tool_result, message, completion, error, status, system
  8. Build final result from last assistant message or error
```

### Droid Type Resolution

Droid configurations are loaded via `OpenDroidManager.loadAllDroids()` (2519.js), which reads from the resolved settings store. Droids exist in two locations:
- **Project**: `.opendroid/droids/` (resolved from `process.cwd()`)
- **Personal**: `~/.opendroid/droids/` (resolved from `homedir()`)

Project-level droids take precedence during lookup (project is searched first). The resolution chain:
```
TaskCliExecutor.buildCommandArgs()
  → iZ().loadAllDroids()                    // OpenDroidManager singleton
  → .find(droid => metadata.name === subagent_type)
  → k5.expandTools(droid.metadata.tools, model)  // tool expansion/validation
```

### System Prompt Composition

When a subagent is spawned, the parent builds a composed system prompt (2522.js, `buildCommandArgs`):

```
# Task Tool Invocation
Subagent type: {name}
Task description: {description}

## Context
You are a specialized subagent invoked by another agent within an ongoing OpenDroid session.
You operate in your own context window but your work directly supports the parent workflow.

## Your Subagent Identity
---BEGIN SUBAGENT SYSTEM PROMPT---
{droid.systemPrompt}
---END SUBAGENT SYSTEM PROMPT---

## Mission
Follow the instructions from your subagent system prompt and the task below.
Complete only what is explicitly requested. Stop immediately once the task is done.

## Non-negotiable rules
- Stay strictly within scope
- Do not pursue tangents
- If something is unclear, report it instead of guessing

## Task
---BEGIN TASK FROM PARENT AGENT---
{prompt}
---END TASK FROM PARENT AGENT---

## Reporting requirements
- Summarize concrete actions and outcomes
- List all file paths written
- Note blockers, uncertainties, or follow-ups
```

### Stateless Execution Model

Each subagent invocation is **stateless**:
- Launched as a fresh `droid exec` child process (not a thread or in-process call)
- Receives the composed prompt as a CLI argument
- Has its own context window, system prompt, and tool set
- Returns a **single final message**: no follow-up questions possible
- The `--output-format debug` flag ensures JSON-streamed output for parsing

### Session Management and Process Lifecycle

```js
// 2521.js: Process spawning
let P = OD().env === "development" ? "opendroid-dev" : "droid";
let B = h9f(P, H, {     // spawn("droid", args)
  shell: false,
  env: { ...process.env },
  cwd: process.cwd(),
  stdio: ["pipe", "pipe", "pipe"],
});
B.stdin?.end();          // No stdin: subagent cannot receive input

// Process tracking
x6.registerProcess(toolCallId, childPid, {
  command: "droid exec (task tool)",
  cwd: process.cwd(),
  startTime: Date.now(),
});
```

Key lifecycle features:
- **Abort signal**: Parent's `abortSignal` cascades SIGTERM → SIGKILL (1s timeout) to child
- **Process cleanup**: `x6.unregisterProcess()` on exit, `x6.killToolProcesses()` on abort
- **SubagentStop hook**: After completion, `SubagentStop` event hooks are executed with session_id, task_name, task_result, task_error
- **Session linking**: `--calling-session-id` and `--calling-tool-use-id` CLI flags link the child session to the parent

### Depth Management (Recursion Prevention)

```js
// 2522.js: Depth tracking
let V = nf().getDepth();
P.push("--depth", String(V + 1));   // Child gets parent depth + 1
```

The depth is managed by `nf()` singleton (0724.js) with `getDepth()`/`setDepth()`. The CLI passes `--depth` to the child process. A check exists at `3399.js:1084` for non-interactive CLI mode with depth > 0.

### Streaming Event Parsing

The `SubagentStreamProcessor.parseEventToUpdate()` method handles 7 event types from child stdout:

| Event Type | Purpose | Update Generated |
|------------|---------|-----------------|
| `system` (subtype `init`) | Session initialization | Captures `subagentSessionId`, emits status update |
| `tool_call` | Tool invocation start | Records tool name, status (pending/executing/completed/error) |
| `tool_result` | Tool invocation result | Status + details + value snippet |
| `message` | Assistant text | Accumulates message text |
| `completion` | Final output | Captures `finalText` as last assistant message |
| `error` | Error event | Marks `hasErrors = true`, emits error update |
| `status` | Status update | Emits text status update |

### Final Result Construction

```js
// 2521.js: buildFinalResult()
// Success: exit code 0, no errors, has output text
if (exitCode === 0 && !hasErrors && lastMessage.trim())
  return { type: "result", isError: false, value: lastMessage.trim() };

// Failure: exit code != 0 or has errors
// Includes: error code, output text, stderr, tools executed list
return { type: "result", isError: true, errorType: "toolInternalError", ... };
```

## Key Findings

### 1. Droid Config Resolution (2522.js, 2519.js)

```js
// 2522.js: Droid lookup
let f = (await iZ().loadAllDroids()).find((X) => X.metadata.name === L);
if (!f) return null;  // Triggers "Droid not found" error

// 2519.js: Storage is settings-based, not filesystem-based
async getAllDroids() {
  return (await MD.getInstance().getResolvedSettings()).droids?.customDroids ?? [];
}
```

Droid configs are stored in the resolved settings system (not raw filesystem reads). The settings manager resolves both project and personal levels.

### 2. Tool Expansion per Droid (2518.js)

```js
// 2518.js: expandTools handles model-specific tool mapping
static expandTools(H, A) {
  // H = tools config (string, array, or undefined)
  // A = model identifier
  // Returns: expanded tool list with model-specific adjustments
  // e.g., Edit → ApplyPatch for OpenAI models
}

// 2545.js: Full tool registry
var d1 = {
  Read, Grep, Glob, Ls, Create, Edit, Execute, AskUser,
  WebSearch, FetchUrl, ApplyPatch, TodoWrite, Skill,
  ExitSpecMode, GenerateDroid, slackPostMessage
};
```

### 3. Child Process Spawning with Full Lifecycle (2521.js)

```js
// Spawning with stdin closed (stateless!)
B = h9f(P, H, { shell: false, env: {...process.env}, cwd: process.cwd(),
                stdio: ["pipe", "pipe", "pipe"] });
B.stdin?.end();  // No interactive input possible

// Graceful shutdown cascade
async killProcessTree(signal = "SIGTERM") {
  // SIGTERM first, then SIGKILL after 1s timeout
  if (signal === "SIGTERM")
    setTimeout(() => {
      if (B.exitCode === null && !B.killed) V("SIGKILL");
    }, 1000).unref?.();
}
```

### 4. Available Droids Injection into Agent Prompt (2545.js)

```js
// 2545.js: Custom droid availability injected into system prompt
function zUf(H) {
  let { project, personal } = QaH();  // Resolve droid directories
  // Builds markdown section listing available droids with:
  // - name, description, model, location
  // - Guidance: "If one is relevant, launch it immediately"
  // - Warning: "Only invoke subagents that are currently available"
}

async function YNI() {
  let H = await NUf();  // Load valid droids
  if (H.length === 0) return { count: 0, description: null };
  return { count: H.length, description: KgH + zUf(H) };
}
```

### 5. Task Tool Description Template (1147.js)

```js
var KgH = `Launch a new subagent (custom droid) to handle a complex, multi-step task autonomously.

  Required inputs:
  - subagent_type: the droid name/identifier (example: "worker")
  - description: a short 3–5 word label for the UI
  - prompt: the full task to execute

  Where to find available droids:
  - ~/.opendroid/droids (personal)
  - .opendroid/droids (project)

  Usage notes:
  1. If you need parallel subagents, issue multiple Task tool calls in the same assistant message.
  2. When the subagent is done, it returns a single message to you.
  3. Clearly tell the subagent whether you expect it to write code or only do research.`;
```

### 6. Process Tracking System (2334.js)

```js
// 2334.js: ProcessManager tracks all child processes by toolCallId
class ProcessManager {
  registerProcess(toolCallId, pid, metadata) { ... }
  unregisterProcess(toolCallId, pid) { ... }
  async killToolProcesses(toolCallId, signal = "SIGTERM") { ... }
  async killAllProcesses() { ... }
}
```

## Integration Points (cross-mission: TUI/Mission/GUI/Infra)

- **TUI (01-terminal-ui)**: 3384.js provides Ink-based TUI rendering for Task tool progress updates, tool call summaries, and results: the `nJD` renderer object renders `renderResult()` and `renderDetailedView()` components.
- **Mission orchestration (02-orchestration)**: The Task tool is heavily used by orchestrator system prompt (2175.js) to delegate feature analysis, validation, and research to worker subagents. The orchestrator prompt contains detailed guidance on when/how to spawn subagents.
- **Agent state (tool-agent-state)**: Subagent spawning interacts with agent state via depth tracking (`nf().getDepth()`) and session linking (`--calling-session-id`). The `subagentSessionId` field (0084.js) appears in session state schemas.
- **Permissions (tool-permissions)**: Task tool itself is `requiresConfirmation: false`, but spawned subagents may have different permission modes via `--auto high` flag.
- **Sandbox (tool-sandbox)**: Subagent child processes inherit `process.env` but operate in their own process space; the sandbox system's process tracking (x6) is used to manage child cleanup.
- **Streaming (tool-streaming)**: The SubagentStreamProcessor parses JSON-streamed events from child stdout, which follow the same streaming protocol as the main agent streaming system.

## Implementation Notes

### Task Tool API

```typescript
// Input schema
interface TaskInput {
  subagent_type: string;    // Droid name/identifier
  description: string;      // Short 3-5 word label
  prompt: string;           // Full task description
}

// Output
type TaskOutput = string;   // Final report text from subagent

// Error types
interface TaskError {
  type: "result";
  isError: true;
  errorType: "invalidParameterLLMError" | "toolInternalError";
  llmError: string;        // Technical error message
  userError: string;       // User-friendly error message
}
```

### Droid Configuration Format

```typescript
interface DroidConfig {
  metadata: {
    name: string;           // Alphanumeric + hyphens/underscores
    description?: string;   // Help text
    model?: string;         // "inherit" (default) or specific model ID
    tools?: string | string[];  // Tool names or categories
    reasoningEffort?: number;
    createdAt: string;
    updatedAt: string;
  };
  systemPrompt: string;     // System prompt for the droid
  location: "project" | "personal";
  filePath: string;
  validationResult: {
    valid: boolean;
    errors: string[];
    warnings: string[];
  };
}
```

### Key Implementation Points for OpenDroid

1. **Child process spawning**: Use `child_process.spawn()` with `shell: false`, stdin piped then immediately closed
2. **Streaming protocol**: JSON objects on stdout, one per line: event types: `system`, `tool_call`, `tool_result`, `message`, `completion`, `error`, `status`
3. **Depth management**: Track recursion depth per session, increment for each child, pass via CLI flag
4. **Tool expansion**: Map droid tool config to actual tool IDs, handle model-specific substitutions (Edit→ApplyPatch for OpenAI)
5. **Graceful shutdown**: SIGTERM → 1s timeout → SIGKILL cascade, with process tracking cleanup

## Open Questions / Left Undone

- **Max depth limit**: The exact max depth value (`JPA.depth` in 0221.js) was not traced: this prevents infinite subagent nesting but the specific threshold is unknown
- **Claude Code subagent import**: 3924.js/3925.js contain Claude Code subagent detection and import logic: this cross-system integration was not fully analyzed
- **Tool category presets**: `Z9f()` in 2518.js provides named tool category groups (referenced but not fully traced)
- **SubagentStop hooks**: The hook execution mechanism (`BO().executeHooks`) was identified but not deeply analyzed: belongs to hook/plugin system (05-infrastructure)
- **Session transcript linking**: How `subagentSessionId` is stored and linked back to parent session transcripts for display in the UI was not fully traced
