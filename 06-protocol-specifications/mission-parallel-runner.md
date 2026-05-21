# mission-parallel-runner: Mission System RE

## Overview

OpenDroid's parallel execution model operates at two distinct levels: **feature-level sequential execution** (one mission feature at a time via `start_mission_run`) and **tool-level parallel execution** (multiple Task/subagent invocations within a single assistant message run concurrently via `Promise.all`). This report documents the parallel worker spawning mechanism, concurrency control at the LLM API layer, result aggregation from parallel tool calls, resource isolation between subagents, and how the system decides between sequential vs parallel execution paths.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 1147.js | 42 lines | Task tool prompt template | Explicitly instructs: "issue multiple Task tool calls in the same assistant message" for parallel subagents |
| 0279.js | 1894 lines | Git scheduler + task queue | `Scheduler` class with `concurrency` limit, `pending`/`running` queues (default concurrency=2) |
| 2362.js | 484 lines | LLM client / tool runner | Core parallel execution: `Promise.all(L.map(async (I) => D.run(E)))` over all `tool_use` blocks |
| 2521.js | 394 lines | SubagentStreamProcessor | Spawns child `droid`/`opendroid-dev` processes, streams JSON events, handles tool call lifecycle |
| 2522.js | 150 lines | TaskCliExecutor | Validates `subagent_type`, builds command args with `--depth`, `--calling-session-id`, `--calling-tool-use-id` |
| 0965.js | 188 lines | LocalDaemonClient | `spawnWorkerSession()` via WebSocket to `opendroidd` daemon for worker session initialization |
| 2175.js | 2602 lines | Orchestrator skill prompt | Guidance for when to spawn parallel subagents (reviews, validation, investigation) |
| 1236.js | 23 lines | `start_mission_run` tool def | Explicitly states mission runner "executes workers sequentially": feature level is serial |
| 0257.js | 710 lines | Model configuration registry | `parallelToolCalls` flag per model (e.g., `false` for GPT-5-Codex, `true` for GPT-5.2-Codex) |
| 3988.js | 75 lines | Model selection UI | `orchestratorConfig`/`workerConfig`/`validatorConfig`: model assignment for parallel workers |

**Note on target modules:** The feature spec listed modules 0932, 1232, 3398, and 0885 as targets. Grep analysis revealed these modules contain custom model provider logic (0932), web search tool definitions (1232), tagged path reminders (3398), and reasoning effort utilities (0885): none contain parallel execution machinery. The actual parallel runner code was found in adjacent modules (0279, 2362, 2521, 2522, 0965, 0257).

## Architecture / Flow

### 1. Feature Level: Sequential Execution

The mission runner (`start_mission_run` tool, Module 1236.js:12) explicitly executes **one feature at a time** in array order:

> "The tool call remains open while the mission runner executes workers **sequentially**."

This means:
- Features from `features.json` run strictly one-after-another
- A worker must complete and hand off before the next pending feature starts
- There is no feature-level parallelism in the core mission runner

### 2. Tool Level: Parallel Execution via `Promise.all`

When an assistant message contains multiple `tool_use` blocks (e.g., multiple `Task` tool calls), the system executes them concurrently. This is the primary parallel execution primitive.

**Flow:**

```
LLM generates assistant message with N tool_use blocks
    ↓
Tool runner (2362.js) extracts all tool_use blocks: L = A.content.filter(I => I.type === "tool_use")
    ↓
Promise.all(L.map(async (I) => {
    let D = H.tools.find(E => E.name === I.name);
    let f = await D.run(E);   // ← each tool runs concurrently
    return { type: "tool_result", tool_use_id: I.id, content: f };
}))
    ↓
Aggregates all tool_result blocks into a single user message
    ↓
Returns to LLM with parallel results
```

Each `Task` tool invocation (Module 2522.js) spawns a separate `droid` CLI child process (Module 2521.js) via `SubagentStreamProcessor.process()`. The child process gets:
- Its own `--session-title` (`${subagent_type}: ${description}`)
- `--depth` incremented from parent
- `--calling-session-id` and `--calling-tool-use-id` for traceability
- Isolated `cwd` and `env` (inherited from parent process)

### 3. LLM API Concurrency Control

Not all models support parallel tool calls. Module 0257.js configures this per model:

| Model | `parallelToolCalls` |
|-------|---------------------|
| gpt-5-codex | `false` |
| gpt-5.1-codex | `false` |
| gpt-5.2-codex | `true` |
| gpt-5.3-codex | `true` |
| orbit-02-24 | `true` |

This flag is passed to the LLM API via `apiRequest: PK({ parallelToolCalls: true/false })` (0257.js:336, 381). When `false`, the model will emit tool calls sequentially; when `true`, it may emit multiple tool calls in a single response.

### 4. Subagent Process Isolation

Each parallel subagent runs as an independent OS process:

```
Parent agent (orchestrator or worker)
    ├─ spawns droid process #1 (Task tool call 1)
    │   ├─ opendroidd WebSocket session (0965.js:129)
    │   ├─ separate Node.js process
    │   └─ isolated context window
    ├─ spawns droid process #2 (Task tool call 2)
    │   └─ ...
    └─ spawns droid process #N (Task tool call N)
        └─ ...
```

Process registration (2521.js:96):
- `x6.registerProcess($, B.pid, { command: "droid exec (task tool)", ... })`
- Each subagent gets a unique PID tracked under the parent `toolCallId`
- Abort signal propagation kills all registered child processes

### 5. Result Aggregation

Results from parallel tool calls are aggregated into a single user message containing an array of `tool_result` blocks (2362.js:448-484):

```js
return {
    role: "user",
    content: await Promise.all(L.map(async (I) => {
        // ... execute tool ...
        return { type: "tool_result", tool_use_id: I.id, content: f };
    })),
};
```

Each `tool_result` is matched to its corresponding `tool_use` via `tool_use_id`. The LLM then sees all results simultaneously and can synthesize them.

### 6. Orchestrator-Level Parallel Decision Logic

The orchestrator skill (2175.js) instructs the LLM on **when** to use parallel subagents:

| Scenario | Parallel Strategy |
|----------|-------------------|
| Contract review passes | "spawn parallel subagents by section for efficiency: one reviewer per area plus one for cross-area" (2175.js:708) |
| Scrutiny validation | "Spawn subagents in parallel when reviewing multiple features" (2175.js:1824) |
| User-testing validation | "Spawn subagents in parallel when testing multiple assertion groups" (2175.js:2096) |
| Significant scope changes | "use multiple subagents (e.g., one per affected area) followed by a synthesis pass" (2175.js:1067) |
| Investigation | "Spawn a subagent (Task tool) to systematically extract all assertion IDs" (2175.js:790) |

The decision is **prompt-driven**, not hardcoded: the orchestrator prompt explicitly tells the model to use parallel subagents for efficiency, and the model generates multiple `tool_use` blocks accordingly.

### 7. Internal Scheduler Concurrency (Git Operations)

Module 0279.js contains a `Scheduler` class (`src/lib/runners/scheduler.ts`, lines 1315-1339) used for git operation queueing:

```js
constructor(H = 2) {
    this.concurrency = H;
    this.pending = [];
    this.running = [];
}
schedule() {
    if (!this.pending.length || this.running.length >= this.concurrency) {
        // Skip: at concurrency limit
        return;
    }
    let H = cM(this.running, this.pending.shift());
    H.done(() => {
        phH(this.running, H);
        this.schedule();  // Try to schedule next
    });
}
```

This demonstrates the system's general concurrency primitive: a queue with configurable max concurrency. While this specific scheduler is for git operations, the pattern (pending queue + running set + concurrency limit) is the same concurrency control model used elsewhere.

## Key Findings

1. **Parallel execution is tool-level, not feature-level.** The mission runner executes features sequentially. Parallelism only occurs when the LLM emits multiple `tool_use` blocks in a single message.

2. **`Promise.all` is the core primitive.** Module 2362.js wraps all tool invocations in `Promise.all()`, making them truly concurrent at the Node.js event loop level.

3. **Model capability gates parallelism.** The `parallelToolCalls` flag (0257.js) determines whether the LLM API will even emit multiple tool calls. Some models are explicitly configured to disable this.

4. **Each subagent is an isolated OS process.** Process isolation via `child_process.spawn` (2521.js) ensures memory and context separation. The parent tracks child PIDs and can abort all children via signal propagation.

5. **Result aggregation is synchronous after `Promise.all`.** All parallel subagents must complete (or fail) before the aggregated `tool_result` array is returned to the LLM. There is no streaming partial results back.

6. **Depth tracking prevents runaway recursion.** Each subagent gets `--depth` incremented (2522.js). The system can enforce max depth limits to prevent infinite subagent spawning.

7. **The orchestrator prompt is the concurrency trigger.** The system does not automatically decide to go parallel: it relies on the orchestrator skill prompt (2175.js) instructing the LLM to "spawn subagents in parallel" for specific scenarios (reviews, validation, investigation).

## Code Examples

### Parallel tool execution core (2362.js:448-484)

```js
async function cEf(H, A = H.messages.at(-1)) {
  if (!A || A.role !== "assistant" || !A.content || typeof A.content === "string") return null;
  let L = A.content.filter((I) => I.type === "tool_use");
  if (L.length === 0) return null;
  return {
    role: "user",
    content: await Promise.all(
      L.map(async (I) => {
        let D = H.tools.find((E) => E.name === I.name);
        if (!D || !("run" in D))
          return {
            type: "tool_result",
            tool_use_id: I.id,
            content: `Error: Tool '${I.name}' not found`,
            is_error: true,
          };
        try {
          let E = I.input;
          if ("parse" in D && D.parse) E = D.parse(E);
          let f = await D.run(E);
          return { type: "tool_result", tool_use_id: I.id, content: f };
        } catch (E) {
          return {
            type: "tool_result",
            tool_use_id: I.id,
            content: `Error: ${E instanceof Error ? E.message : String(E)}`,
            is_error: true,
          };
        }
      }),
    ),
  };
}
```

### Scheduler concurrency control (0279.js:1315-1339)

```js
constructor(H = 2) {
  ((this.concurrency = H),
    (this.logger = TWA("", "scheduler")),
    (this.pending = []),
    (this.running = []),
    this.logger("Constructed, concurrency=%s", H));
}
schedule() {
  if (!this.pending.length || this.running.length >= this.concurrency) {
    this.logger(
      "Schedule attempt ignored, pending=%s running=%s concurrency=%s",
      this.pending.length,
      this.running.length,
      this.concurrency,
    );
    return;
  }
  let H = cM(this.running, this.pending.shift());
  (this.logger("Attempting id=%s", H.id),
    H.done(() => {
      (this.logger("Completing id=", H.id), phH(this.running, H), this.schedule());
    }));
}
```

### Subagent process spawn (2521.js:89-104)

```js
let P = OD().env === "development" ? "opendroid-dev" : "droid",
  B = h9f(P, H, {
    shell: false,
    env: { ...process.env },
    cwd: process.cwd(),
    stdio: ["pipe", "pipe", "pipe"],
  });
if ((B.stdin?.end(), (f = B.pid ?? null), $ && B.pid))
  ((f = B.pid),
    x6.registerProcess($, B.pid, {
      command: "droid exec (task tool)",
      cwd: process.cwd(),
      startTime: Date.now(),
    }),
    (M = true));
```

### Task tool parallel instruction (1147.js:38)

```
1. If you need parallel subagents, issue multiple Task tool calls in the same assistant message.
2. When the subagent is done, it returns a single message to you. The result is not shown to the user unless you summarize it.
3. Clearly tell the subagent whether you expect it to write code or only do research, and specify exactly what it should return.
```

### Model parallel tool call config (0257.js:336)

```js
"gpt-5.2-codex": {
  // ...
  apiRequest: PK({ parallelToolCalls: true, extendedCache: true, safetyId: true }),
  // ...
}
```

## Integration Points

- **01-terminal-ui (TUI):** The TUI displays parallel subagent progress via `tool_call`/`tool_result` status updates streamed from `SubagentStreamProcessor` (2521.js). Each parallel task shows as a separate executing tool.
- **03-tool-agent-system (Tool):** The Task tool (1147.js) is the primary surface for parallel execution. Tool definitions in 2522.js validate `subagent_type` against available droids.
- **04-desktop-gui (GUI):** Model selection UI (3988.js) assigns different models to orchestrator/worker/validation roles, affecting which `parallelToolCalls` capability is used for parallel subagents.
- **05-infrastructure (Infra):** `LocalDaemonClient` (0965.js) communicates with `opendroidd` via WebSocket to spawn worker sessions. Parallel subagents each get independent opendroidd sessions.

## Implementation Notes

1. **Preserve `Promise.all` tool execution.** The core primitive (2362.js) is simple and essential: extract all `tool_use` blocks from an assistant message, map to `tool.run()`, wrap in `Promise.all`, aggregate results.

2. **Implement `parallelToolCalls` model capability tracking.** OpenDroid should track which LLM providers support parallel tool calls and gate accordingly (like 0257.js).

3. **Subagent process isolation.** Each parallel subagent should be a separate process with:
   - Unique session ID
   - Depth counter (prevent infinite recursion)
   - Parent session/tool tracking
   - Abort signal propagation to children

4. **Feature-level stays sequential.** The mission runner should not attempt feature-level parallelism. Parallelism belongs at the tool/subagent level within a single worker/orchestrator turn.

5. **Prompt engineering drives parallelism.** The orchestrator skill prompt (2175.js) is what convinces the LLM to emit multiple tool calls. OpenDroid should include similar explicit instructions for review, validation, and investigation phases.

6. **Result aggregation pattern.** After `Promise.all`, construct a user message containing an array of `tool_result` blocks, each matched by `tool_use_id`. The LLM then synthesizes all parallel outputs.

## Known Gaps

- **Resource limits for parallel subagents:** Module 2175.js mentions "Each tool session consumes memory, and multiple subagents creating many sessions can exhaust system resources and crash the host" (2175.js:2293), but no hard resource limit or backpressure mechanism was found in the code. The system relies on the LLM's judgment and orchestrator guidance to limit parallelism.
- **Git scheduler integration:** The `Scheduler` class in 0279.js is currently only used for git operations. It is unclear if this same scheduler is reused for subagent task queueing or if subagent concurrency is unbounded (limited only by OS process limits).
- **Error handling partial failures:** When one parallel subagent fails and others succeed, `Promise.all` semantics mean all results are still returned, with the failing tool marked `is_error: true`. More sophisticated partial failure handling (e.g., retry only failed subagents) was not found.
- **Target module staleness:** Modules 0932, 1232, 3398, and 0885 from the feature spec do not contain parallel execution logic. They may have been relevant in an earlier version of the codebase.
