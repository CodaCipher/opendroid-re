# mission-task-tool: Mission System RE

## Overview

The Task tool is OpenDroid's subagent invocation mechanism. It allows any agent (orchestrator or worker) to spawn a specialized subagent session by specifying a `subagent_type` (droid name) and a `prompt`. The system resolves the droid configuration, assembles a rich prompt template, spawns a child `droid` process, streams JSON events from its stdout, and returns a structured result. This report documents the Task tool API, prompt construction, droid selection and validation, subagent spawning, result handling, timeout/abort management, and how the orchestrator uses Task to delegate investigation.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 2522.js | ~5.5 KB | TaskCliExecutor: core Task tool implementation | `execute()`, `buildCommandArgs()`, `listAvailableSubagents()` |
| 2521.js | ~11 KB | SubagentStreamProcessor (yoA): process runner & result handler | Spawns `droid` child process, parses NDJSON stdout, handles abort/kill |
| 2519.js | ~7.3 KB | OpenDroidManager: droid CRUD and resolution | Loads droids from `.opendroid/droids/` and `~/.opendroid/droids/`, name normalization |
| 2518.js | ~8.5 KB | DroidValidator (k5): metadata & prompt validation | `validateMetadata()`, `validateSystemPrompt()`, `validateTools()`, `expandTools()` |
| 2176.js | ~6.5 KB | Built-in droid definitions & bootstrap | `ensureBuiltInDroids()`, defines mission-worker-base, scrutiny-feature-reviewer, user-testing-flow-validator |
| 3983.js | ~12 KB | TUI model selector for mission models | Orchestrator/worker/validation model config UI |
| 2175.js | ~136 KB | Orchestrator core: contains Task usage guidance | Defines how orchestrator delegates to subagents via Task tool |

## Architecture / Flow

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Caller Agent  │────▶│  TaskCliExecutor │────▶│  OpenDroidManager   │
│ (orchestrator/  │     │    (2522.js)     │     │   (2519.js)     │
│     worker)     │◄────│                  │◄────│                 │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │ SubagentStream   │
                        │  Processor (2521)│
                        └──────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  droid exec      │
                        │  child process   │
                        └──────────────────┘
```

### Execution Flow

1. **Invocation**: Caller invokes `Task({ subagent_type, description?, prompt })`
2. **Parameter Validation** (2522.js:18-41): `subagent_type` and `prompt` are validated as required strings
3. **Droid Resolution** (2522.js:70-71): `OpenDroidManager.loadAllDroids()` loads all droid configs; finds match by `metadata.name`
4. **Prompt Assembly** (2522.js:75-113): Constructs rich prompt template with subagent identity, mission context, non-negotiable rules, and the caller's task
5. **CLI Args Construction** (2522.js:113-139): Builds `droid exec` argument array with model, reasoning effort, auto mode, output format, enabled tools, depth, session IDs
6. **Process Spawn** (2521.js:91-105): Spawns `droid` (or `opendroid-dev` in development) child process via `child_process.spawn`
7. **Event Streaming** (2521.js:180-220): Parses stdout as newline-delimited JSON, yielding typed events
8. **Result Assembly** (2521.js:354-391): `buildFinalResult()` constructs final result from exit code, captured messages, errors, and stderr
9. **Cleanup** (2521.js:141-149): Unregisters process, removes abort listeners

## Key Findings

### 1. Task Tool API and Parameters

The Task tool is implemented as a CLI executor class (`TaskCliExecutor` in 2522.js) that exposes an `execute()` method taking:

- `subagent_type` (string, required): The droid name to invoke
- `prompt` (string, required): The task description for the subagent
- `description` (string, optional): Shown in the prompt template header

Parameter validation at 2522.js:18-41 rejects missing/invalid params with structured error objects:
```js
// Module 2522.js, line 21-26
if (!$ || typeof $ !== "string") {
  yield {
    type: "result",
    isError: true,
    errorType: "invalidParameterLLMError",
    llmError: "subagent_type is required and must be a string",
    userError: "Invalid subagent type provided",
  };
  return;
}
```

### 2. Prompt Construction and Template Assembly

The prompt template is assembled in `buildCommandArgs()` at 2522.js:75-113. It creates a multi-section markdown document:

```js
// Module 2522.js, line 75-113
let P = [
  "exec",
  [
    "# Task Tool Invocation",
    "",
    `Subagent type: ${L}`,
    ...($ ? [`Task description: ${$}`, ""] : [""]),
    "## Context",
    "You are a specialized subagent invoked by another agent within an ongoing OpenDroid session.",
    ...
    "## Your Subagent Identity",
    "Your core identity and specialized capabilities are defined by the following system prompt:",
    "",
    "---BEGIN SUBAGENT SYSTEM PROMPT---",
    f.systemPrompt,                    // <-- droid's system prompt injected here
    "---END SUBAGENT SYSTEM PROMPT---",
    "",
    "## Mission",
    "Follow the instructions from your subagent system prompt and the task below.",
    "Complete only what is explicitly requested. Stop immediately once the task is done.",
    ...
    "## Task",
    "Execute the following assignment precisely and efficiently. Do not perform any other work.",
    "",
    "---BEGIN TASK FROM PARENT AGENT---",
    I,                                 // <-- caller's prompt injected here
    "---END TASK FROM PARENT AGENT---",
    ...
  ].join(`\n`),
];
```

Key template sections:
- **Context**: Establishes subagent identity within OpenDroid session
- **Subagent Identity**: Injects the droid's `systemPrompt` between `---BEGIN/END SUBAGENT SYSTEM PROMPT---` markers
- **Non-negotiable rules**: Stay in scope, no tangents, report blockers
- **Task**: Injects caller's prompt between `---BEGIN/END TASK FROM PARENT AGENT---` markers
- **Reporting requirements**: Summarize actions, list output files, note blockers

### 3. Droid Selection and Skill Resolution

Droid resolution happens in two layers:

**Layer 1: OpenDroidManager (2519.js)**: loads and resolves droid configurations:
- Loads from `.opendroid/droids/` (project-level) and `~/.opendroid/droids/` (personal-level)
- Droids stored as `.md` files with YAML frontmatter (name, description, model, tools, reasoningEffort) + markdown body (system prompt)
- Name normalization via `bm()` (2519.js:29-35): strips `.md`, lowercases, replaces non-alphanumeric with `-`
- `readDroid()` prefers project over personal location (2519.js:86-89)

**Layer 2: DroidValidator (2518.js)**: validates droid configuration:
- `validateMetadata()` checks name format, description length, model validity, reasoning effort compatibility, tools selection
- `validateSystemPrompt()` ensures prompt is non-empty
- `expandTools()` resolves tool aliases (e.g., "read-only", "edit", "execute", "web") to actual tool IDs

If droid not found, error includes list of available subagents (2522.js:46-51):
```js
let E = await cs.listAvailableSubagents(),
  f = E.length ? E.join(", ") : "none";
yield {
  type: "result",
  isError: true,
  errorType: "toolInternalError",
  llmError: `Droid configuration not found for subagent: ${$}. Available subagents: ${f}`,
  userError: `Droid "${$}" not found. Available subagents: ${f}.`,
};
```

### 4. Subagent Spawning and CLI Args

The `SubagentStreamProcessor` (class `yoA` in 2521.js) spawns the subagent process:

```js
// Module 2521.js, line 91-105
let P = OD().env === "development" ? "opendroid-dev" : "droid",
  B = h9f(P, H, {
    shell: false,
    env: { ...process.env },
    cwd: process.cwd(),
    stdio: ["pipe", "pipe", "pipe"],
  });
B.stdin?.end();
```

CLI arguments built by `buildCommandArgs()` (2522.js:113-139):
- `--model <model>`: Droid's model, or inherited from parent session
- `--reasoning-effort <level>`: If specified in droid config
- `--auto high`: Autonomy mode
- `--output-format debug`: Output format
- `--enabled-tools <tools>`: Comma-separated tool list (if droid has tools)
- `--depth <N>`: Subagent depth (parent depth + 1)
- `--calling-session-id <id>`: Parent session ID
- `--calling-tool-use-id <id>`: Parent tool call ID
- `--session-title <title>`: Session title from `subagent_type: description`

Process tracking (2521.js:98-105):
```js
if ($ && B.pid) {
  x6.registerProcess($, B.pid, {
    command: "droid exec (task tool)",
    cwd: process.cwd(),
    startTime: Date.now(),
  });
}
```

### 5. Result Handling and Error States

The stream processor parses stdout as **newline-delimited JSON (NDJSON)**. Each line is a typed event:

| Event Type | Handling |
|-----------|----------|
| `tool_call` | Added to `toolsExecuted` set; yielded as update with status "executing" |
| `tool_result` | Yielded as update with status "completed" or "error" |
| `message` (assistant) | Appended to `messageTexts` array |
| `completion` | Appended to `messageTexts`, sets `lastAssistantMessage` |
| `error` | Sets `hasErrors = true`; yielded as error update |
| `status` | Yielded as status update |
| `system` (subtype "init") | Captures `session_id` as `subagentSessionId` |

Final result assembly in `buildFinalResult()` (2521.js:354-391):
```js
// Module 2521.js, line 354-362
buildFinalResult(H, A) {
  let L = this.lastAssistantMessage || this.messageTexts.join(`\n`);
  if (H === 0 && !this.hasErrors && L.trim())
    return { type: "result", isError: false, value: L.trim() };
  // ... error assembly with exit code, stderr, tools executed
}
```

Success condition: exit code === 0 AND no errors AND non-empty assistant message.
Error states include:
- **Process spawn failure**: `Failed to spawn subagent process: ...`
- **Non-zero exit code**: Warning emoji + exit code in result
- **Empty output**: "No output received from task subagent" + debug info
- **Stderr captured**: Included in error result

### 6. Timeout Management and Abort Signals

The Task tool supports cooperative cancellation via `abortSignal`:

```js
// Module 2521.js, line 79-80
let { abortSignal: L, toolCallId: $ } = A ?? {};
```

Abort handling (2521.js:140-149):
```js
if (L) {
  let l = () => {
    if (I) return;
    ((I = true), (D = new II()), X("SIGTERM"));
  };
  if (((E = l), L.addEventListener("abort", l, { once: true }), L.aborted)) l();
}
```

Kill sequence (2521.js:114-129):
1. On abort: send **SIGTERM**
2. If process still alive after 1000ms: escalate to **SIGKILL**
3. Process tree kill via `pzI.default(LH, l, ...)` (uses `tree-kill` or similar)

Cleanup is guaranteed in `finally` block (2521.js:141-149):
```js
let W = () => {
  if (E && L) (L.removeEventListener("abort", E), (E = null));
  if (M && $ && f !== null) (x6.unregisterProcess($, f), (M = false));
  if (U) (U(), (U = null));
};
```

### 7. Orchestrator Usage of Task Tool

The orchestrator (documented in 2175.js) uses Task tool extensively for delegation:

**Research delegation** (2175.js:595):
> "How to research: Delegate to subagents. For each technology that needs research, spawn a subagent to look up current documentation (using WebSearch and FetchUrl)."

**Review delegation** (2175.js:708):
> "After drafting the contract, run at least 2 sequential review passes. Each review pass can spawn parallel subagents by section for efficiency: one reviewer per area plus one for cross-area."

**Scrutiny validator** (2175.js:1824-1839) spawns `scrutiny-feature-reviewer` subagents:
```js
Task({
  subagent_type: "scrutiny-feature-reviewer",
  description: "Review feature <feature-id>",
  prompt: `You are reviewing feature "<feature-id>" for milestone "<milestone>"...`
})
```

**User-testing validator** (2175.js:2066-2067) spawns `user-testing-flow-validator` subagents:
```js
Task({
  subagent_type: "user-testing-flow-validator",
  ...
})
```

Key orchestrator patterns:
- Spawn subagents **in parallel** when reviewing multiple features
- Wait for all subagents to complete before synthesizing
- Re-run only for fix features (not all features)

### 8. Built-in Droids for Mission System

Module 2176.js defines the built-in droids that are auto-created on first run:

| Droid Name | Purpose |
|-----------|---------|
| `mission-planning` | Guides orchestrator through planning phase |
| `define-mission-skills` | Guides orchestrator through designing worker types |
| `refactoring-playbook` | Code modernization and architecture migrations |
| `tui-application-playbook` | Terminal UI application missions |
| `agent-browser` | Browser automation for web testing |
| `tuistory` | Terminal UI testing automation |
| `mission-worker-base` | Base procedures for all mission workers |
| `scrutiny-feature-reviewer` | Code review subagent for milestone validation |
| `user-testing-flow-validator` | Flow validation subagent for user testing |

Built-in droids are written to `{configDir}/droids/` as `.md` files by `ensureBuiltInDroids()` (2176.js:117-137).

## Code Examples

### Task Tool Parameter Validation
```js
// Module 2522.js, line 17-41
class cs {
  async *execute(H, A) {
    let { prompt: L, subagent_type: $ } = A;
    if (!$ || typeof $ !== "string") {
      yield {
        type: "result",
        isError: true,
        errorType: "invalidParameterLLMError",
        llmError: "subagent_type is required and must be a string",
        userError: "Invalid subagent type provided",
      };
      return;
    }
    if (!L || typeof L !== "string") {
      yield {
        type: "result",
        isError: true,
        errorType: "invalidParameterLLMError",
        llmError: "prompt is required and must be a string",
        userError: "Invalid prompt provided",
      };
      return;
    }
    // ...
  }
}
```

### Droid Name Normalization
```js
// Module 2519.js, line 29-35
function bm(H) {
  return H.replace(/\.md$/, "")
    .toLowerCase()
    .replace(/[^a-z0-9-_]/g, "-")
    .replace(/--+/g, "-")
    .replace(/^-|-$/g, "");
}
```

### Process Spawn and Kill Sequence
```js
// Module 2521.js, line 91-129
let P = OD().env === "development" ? "opendroid-dev" : "droid",
  B = h9f(P, H, {
    shell: false,
    env: { ...process.env },
    cwd: process.cwd(),
    stdio: ["pipe", "pipe", "pipe"],
  });

let V = async (l = "SIGTERM") => {
  let LH = f ?? B.pid;
  if (!LH) return;
  if (B.exitCode !== null || B.signalCode) return;
  if (
    (await new Promise((DH) => {
      pzI.default(LH, l, (VH) => { /* ... */ });
    }),
    l === "SIGTERM")
  )
    setTimeout(() => {
      if (B.exitCode === null && !B.killed) V("SIGKILL");
    }, 1000).unref?.();
};
```

### Event Parsing and Final Result
```js
// Module 2521.js, IPC handlers region-256 (event parsing)
case "tool_result":
  if (H.toolName) {
    let A = H.isError ? "error" : "completed",
      L = czI(H.status, A);
    return {
      type: "update",
      value: {
        type: "tool_result",
        toolName: H.toolName,
        status: L,
        details: S9f(H),
        valueSnippet: dzI(H.value),
        timestamp: Date.now(),
      },
    };
  }
  break;
```

```js
// Module 2521.js, line 354-362 (final result)
buildFinalResult(H, A) {
  let L = this.lastAssistantMessage || this.messageTexts.join(`\n`);
  if (H === 0 && !this.hasErrors && L.trim())
    return { type: "result", isError: false, value: L.trim() };
  // ... error assembly
}
```

## Integration Points

- **01-terminal-ui (TUI)**: 3983.js shows the TUI model selector for orchestrator/worker/validation models: the Task tool inherits these model configs when `model: "inherit"` is set on a droid
- **03-tool-agent-system (Tool)**: The `droid exec` CLI is the actual tool execution layer; Task tool is the mission-system wrapper around it
- **04-desktop-gui (GUI)**: Not directly referenced; droid configs could be edited via GUI
- **05-infrastructure (Infra)**: Process spawn uses `child_process.spawn` with inherited env; no containerization observed

### Cross-system dependencies
- `.opendroid/droids/`: project-level droid storage
- `~/.opendroid/droids/`: personal-level droid storage
- `features.json`: read by orchestrator to determine what features need subagent review
- `validation-state.json`: updated by validator subagents
- `.opendroid/services.yaml`: validators run tests before spawning review subagents

## Implementation Notes

1. **Task tool API**: Port `TaskCliExecutor` as a standalone module exposing `execute(subagent_type, prompt, description?)`. Use async generators for streaming results.

2. **Droid registry**: Implement `OpenDroidManager` to load `.md` files from `.opendroid/droids/` and `~/.opendroid/droids/`. Parse YAML frontmatter + markdown body. Implement name normalization (`bm()` function).

3. **Droid validation**: Port `k5` validator class. Validate metadata (name, model, tools, reasoningEffort) and system prompt non-emptiness. Support tool alias expansion (`read-only`, `edit`, `execute`, `web`).

4. **Prompt template**: Hardcode the prompt assembly template from 2522.js:75-113. Inject droid's `systemPrompt` and caller's `prompt` between marked sections.

5. **Process spawning**: Use `child_process.spawn` to launch subagent process. Pass CLI args for model, depth, session IDs. Set `stdio: ['pipe', 'pipe', 'pipe']`, inherit env and cwd.

6. **Result streaming**: Parse stdout as NDJSON. Handle event types: `tool_call`, `tool_result`, `message`, `completion`, `error`, `status`, `system`. Yield typed updates.

7. **Abort/timeout**: Implement abort signal listener that sends SIGTERM, then SIGKILL after 1s. Use process tree kill (e.g., `tree-kill` package). Register/unregister processes for cleanup.

8. **Built-in droids**: On first run, create default droids (mission-worker-base, scrutiny-feature-reviewer, user-testing-flow-validator, etc.) in `.opendroid/droids/`.

9. **Orchestrator integration**: Expose `Task()` function to orchestrator and workers. Document parallel subagent spawning pattern for reviews and research.

## Known Gaps

- **Model inheritance chain**: How does `model: "inherit"` traverse multiple levels of subagent nesting? Only 1 level traced.
- **Tool execution filtering**: The exact `expandTools()` logic in 2518.js for resolving tool aliases was only partially read (line 120+ not fully explored).
- **Process registry (`x6`)**: The `x6.registerProcess()` / `unregisterProcess()` / `killToolProcesses()` interface was referenced but module not located.
- **SubagentStop hooks**: The hook execution at 2521.js:242-256 fires `SubagentStop` event: hook implementation module not traced.
- **Session depth limits**: No explicit max depth found in the code; possible enforcement exists elsewhere.
- **Parallel execution coordination**: While 2175.js mentions "spawn subagents in parallel", the actual Promise.all or concurrency control mechanism was not found in the analyzed modules.
