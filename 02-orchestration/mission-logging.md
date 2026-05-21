# mission-logging: Mission System RE

## Overview

The mission system employs a **three-layer logging architecture**: (1) a structured console logger (`BH`/`gH`/`uH` functions in module 0093.js) that writes categorized, sanitized messages to stdout with optional ErrTracker breadcrumb integration; (2) a persistent **progress log** stored as JSONL at `missionDir/progress_log.jsonl` (module 2201.js), where every mission lifecycle event (worker start, worker complete, worker fail, pause, milestone validation) is appended as a self-describing JSON object; and (3) an **event notification system** that emits typed events over a project-notification bus (`v$.emit("project-notification", ...)`), consumed by the TUI and IDE clients (module 0084.js). The assigned modules (2509, 1236, 1225, 1203, 1227, 1229) are tool definitions that produce structured output (truncated text, schema-validated results): they contribute to the *output formatting* layer rather than the logging pipeline itself. The real logging infrastructure lives in 0093.js, 2201.js, 2542.js, and 0084.js.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 2509.js | 95 lines | Text truncation / Read output formatter | `RzI()` builds `[Showing lines X-Y of Z]` output; `u6()` strips CWD prefix; `ewH()`/`jzI()` truncate strings; `HKH()` truncates multi-line text. Used by Read tool for structured output display. |
| 1236.js | 37 lines | `start-mission-run` tool definition | Declares the tool that starts the MissionRunner. Tool description documents blocking behavior, preconditions, and effects. Output schemas define structured result format. |
| 1225.js | ~30 lines | `ask-user` tool definition | Tool schema for user interaction during mission execution. Output is plain-text Q&A. |
| 1203.js | ~30 lines | `glob_tool` tool definition | File search tool with structured result output. |
| 1227.js | ~50 lines | `apply-patch-cli` + `task-cli` tool definitions | Patch application and subagent task tools. `outputTransform` on apply-patch formats success/error messages. |
| 1229.js | ~40 lines | `generate-droid-cli` tool definition | Droid configuration generator with structured output schema. |
| **0093.js** | ~100 lines | **Core logging functions** | `BH()` = info, `gH()` = warn, `uH()` = error with stack trace chaining, `_I()` = error message. All route through `getLogToConsole()`. ErrTracker breadcrumb integration. |
| **2201.js** | ~626 lines | **MissionFileService** | `appendProgressLog()` writes JSONL entries to `missionDir/progress_log.jsonl`. File paths: `state.json`, `features.json`, `progress_log.jsonl`. Incremental read with caching. |
| **2542.js** | ~811 lines | **MissionRunner** | Heartbeat emission every 180s. Calls `BH("[MissionRunner]", ...)` for every state change. `appendProgressLog()` for persistent events. |
| **0084.js** | ~242 lines | **Event notification schemas** | Zod schemas for all mission event types: `mission_state_changed`, `mission_features_changed`, `mission_progress_entry`, `mission_heartbeat`, `mission_worker_started`, `mission_worker_completed`. |

## Architecture / Flow

### Layer 1: Console Logger (0093.js)

The core logging functions are:

```
BH(message, context?, tags?)  →  INFO level
gH(message, context?, tags?)  →  WARN level
uH(error, message, context?, tags?)  →  ERROR level (with stack trace chain)
_I(message, context?, tags?)  →  ERROR level (message only)
```

All functions:
1. Sanitize the message via `Pf()` (strips sensitive data)
2. Sanitize context fields via `_RH()` (redacts `body`, `error`, `errorMessage`, `messages`, `headers`)
3. Route through `du.getLogToConsole()`: a singleton console transport
4. If ErrTracker is enabled (`ZU()`), add breadcrumbs (`addBreadcrumb`) or capture exceptions (`captureException`)

**Log format**: Each call produces `{level, message, context, tags}` and writes to console via the transport.

### Layer 2: Persistent Progress Log (2201.js)

The `MissionFileService` class manages file-based logging:

**Storage**: `~/.opendroid/missions/{sessionId}/progress_log.jsonl`

**Write path**: `appendProgressLog(entry)` → `JSON.stringify(entry)` + newline → `fs.appendFile()`

**Read path**: `readProgressLog()` with incremental caching (checks file size/mtime, only reads new bytes)

**Entry types** (discriminated by `type` field):

| Type | Fields | When |
|------|--------|------|
| `worker_started` | `workerSessionId`, `spawnId`, `featureId`, `timestamp` | Worker session spawned |
| `worker_selected_feature` | `workerSessionId`, `featureId`, `timestamp` | Feature assigned to worker |
| `worker_completed` | `workerSessionId`, `exitCode`, `timestamp`, `featureId`, `successState`, `returnToOrchestrator` | Worker finished (from handoff) |
| `worker_failed` | `workerSessionId`, `spawnId`, `exitCode?`, `reason`, `timestamp` | Worker error/crash |
| `worker_paused` | `workerSessionId`, `featureId`, `timestamp` | Worker paused mid-work |
| `mission_paused` | `timestamp` | Entire mission paused |
| `milestone_validation_triggered` | `milestone`, `featureId`, `timestamp` | Validation injected after milestone completion |

**Derived state**: `updateDerivedWorkerStatesFromProgressEntry()` computes per-worker `startedAt`/`completedAt`/`exitCode` from the log stream.

### Layer 3: Event Notification Bus (0084.js)

Mission events are emitted over a project-notification bus:

```
v$.emit("project-notification", { notification: { type: "...", ... } })
```

**Mission event types** (Zod-validated schemas):

| Event Type | Payload | Source |
|------------|---------|--------|
| `mission_state_changed` | `state: enum(pending|running|paused|completed|failed|orchestrator_turn)` | State file change detected in poll loop |
| `mission_features_changed` | `features: Feature[]` | Features file change detected in poll loop |
| `mission_progress_entry` | `progressLog: ProgressEntry[]` | After every `appendProgressLog()` call |
| `mission_heartbeat` | `timestamp: string` | Every 180 seconds (TUf = 180000ms) during worker polling |
| `mission_worker_started` | `workerSessionId: string` | Emitted by `appendProgressLog()` when entry type is `worker_started` |
| `mission_worker_completed` | `workerSessionId: string`, `exitCode: number` | Emitted by `appendProgressLog()` when entry type is `worker_completed` |

These events flow to the TUI layer (01-terminal-ui territory) via `droid.session_notification` JSON-RPC method.

### Tool Output Formatting (Assigned Modules)

The assigned modules contribute to the *output formatting* layer:

- **2509.js**: Text truncation for Read tool output. `RzI()` formats the `[Showing lines X-Y of Z total lines]` message. `ewH()` truncates to N chars with `...`. `jzI()` keeps head+tail with `...` middle. `HKH()` limits lines and chars.
- **1236.js**: `start-mission-run` tool: structured output includes `runningMissionCount` and `runningMissionSessionIds`
- **1203.js**: Glob tool: returns matched file paths with truncation at configurable limit
- **1227.js**: Task tool: `outputSchemas: { result: string }` captures subagent output
- **1225.js/1229.js**: User interaction / droid config tools with schema-validated output

## Key Findings

### Finding 1: Three-Tier Logging Architecture

Mission logging operates at three tiers:
1. **Console logging** (0093.js): `BH`/`gH`/`uH` write to stdout with ErrTracker integration: used by MissionRunner for real-time observability
2. **File-based JSONL progress log** (2201.js): `appendProgressLog()` appends typed entries to `progress_log.jsonl`: persistent, replayable audit trail
3. **Event bus** (0084.js): `v$.emit("project-notification", ...)` sends typed events to UI consumers: enables real-time TUI updates

All three fire simultaneously: MissionRunner calls `BH("[MissionRunner]", ...)` for console, `appendProgressLog({...})` for file, and `v$.emit(...)` for events.

### Finding 2: Heartbeat Mechanism

The heartbeat is emitted in the `waitForWorkerCompletion()` polling loop (2542.js, line ~787):

```js
if (Q - I >= TUf)  // TUf = 180000ms = 3 minutes
  ((I = Q),
    v$.emit("project-notification", {
      notification: { type: "mission_heartbeat", timestamp: new Date(Q).toISOString() },
    }));
```

- **Interval**: 180 seconds (3 minutes)
- **Transport**: Event bus only (NOT written to progress_log.jsonl)
- **Purpose**: Let TUI/IDE know the runner is alive during long-running worker execution
- **Separate from IDE heartbeat**: Module 2320.js has a separate `notifications/heartbeat` for IDE client connections with a 75-second timeout

### Finding 3: Progress Log is JSONL with Incremental Read

The progress log uses JSON Lines format (one JSON object per line). The `readProgressLogIncrementalOrThrow()` method tracks file offset and only reads new bytes on subsequent calls: this is an efficient tail-like pattern for long-running missions.

**Derived worker states** are computed from the log stream: `updateDerivedWorkerStatesFromProgressEntry()` tracks `startedAt`, `completedAt`, `exitCode` per worker session ID.

### Finding 4: Log Sanitization Pipeline

All console log messages go through:
1. `Pf()`: message-level sanitization (strips sensitive data)
2. `_RH()`: context-level sanitization that redacts sensitive fields: `body`, `error`, `errorMessage`, `messages`, `headers`
3. Error objects get stack trace chaining up to 3 levels deep

This ensures no API keys, tokens, or conversation content leak into logs.

### Finding 5: Structured Output from Tool Definitions

The assigned modules (1236, 1225, 1203, 1227, 1229) are tool definitions with Zod-validated input/output schemas. Each tool defines:
- `inputSchema`: Zod schema for parameters
- `outputSchemas`: Zod schema for results (e.g., `p.string().describe("The output from...")`)
- `outputTransform`: Post-processing function (e.g., 1227.js transforms patch results into human-readable messages)

This structured output pattern is the "logging format" for tool invocations: every tool result is schema-validated and consistently formatted.

## Code Examples

### Console Logger Functions (0093.js:72-100)
```js
// INFO level logging: used throughout MissionRunner
function BH(H, A, L) {
  let $ = Pf(H),           // sanitize message
    I = _RH(A);            // sanitize context
  if ((du.getLogToConsole()("info", $, I, L || {}), ZU()))
    zU()?.addBreadcrumb({ type: "debug", message: $, data: I });
}

// WARN level: used for non-fatal issues
function gH(H, A, L) {
  let $ = Pf(H),
    I = _RH(A);
  if ((du.getLogToConsole()("warn", $, I, L || {}), ZU()))
    zU()?.addBreadcrumb({ type: "warn", level: "warning", category: "warn", message: $, data: I });
}

// ERROR level: with stack trace chaining up to 3 levels
function uH(H, A, L, $) {
  // ... sanitizes, chains Error.cause up to 3 levels
  // ... captures to ErrTracker if enabled
  let P = du.getLogToConsole(),
    B = { ...U, error: fR(H) };
  P("error", E, B, $ || {});
}
```

### Progress Log Append (2201.js:395-420)
```js
async appendProgressLog(H) {
  let A = { ...H, timestamp: H.timestamp || new Date().toISOString() },
    L = `${JSON.stringify(A)}\n`;
  try {
    await sM.appendFile(this.progressLogPath, L);  // JSONL append
  } catch {
    await sM.writeFile(this.progressLogPath, L);    // create if not exists
  }
  let $ = await this.readProgressLog();
  // Emit events for UI consumers
  v$.emit("project-notification", {
    notification: { type: "mission_progress_entry", progressLog: $ },
  });
  if (A.type === "worker_started" && "workerSessionId" in A)
    v$.emit("project-notification", {
      notification: { type: "mission_worker_started", workerSessionId: A.workerSessionId },
    });
  else if (A.type === "worker_completed" && "workerSessionId" in A && "exitCode" in A)
    v$.emit("project-notification", {
      notification: {
        type: "mission_worker_completed",
        workerSessionId: A.workerSessionId,
        exitCode: A.exitCode,
      },
    });
}
```

### MissionRunner Heartbeat Emission (2542.js:784-789)
```js
if (Q - I >= TUf)  // TUf = 180000ms
  ((I = Q),
    v$.emit("project-notification", {
      notification: { type: "mission_heartbeat", timestamp: new Date(Q).toISOString() },
    }));
await _Uf(1000, this.abortSignal);  // poll every 1 second
```

### Mission Event Schemas (0084.js:112-120)
```js
BM0 = AH.object({ type: AH.literal("mission_state_changed"), state: AH.nativeEnum(oT) }),
WM0 = AH.object({ type: AH.literal("mission_features_changed"), features: AH.array(pNH) }),
TM0 = AH.object({ type: AH.literal("mission_progress_entry"), progressLog: AH.array(lNH) }),
VM0 = AH.object({ type: AH.literal("mission_heartbeat"), timestamp: AH.string() }),
XM0 = AH.object({ type: AH.literal("mission_worker_started"), workerSessionId: AH.string() }),
GM0 = AH.object({
  type: AH.literal("mission_worker_completed"),
  workerSessionId: AH.string(),
  exitCode: AH.number(),
}),
```

### Text Truncation for Read Output (2509.js:5-47)
```js
function RzI(H, A = 0, L = 2400, $ = false, I = 60000) {
  // H = content, A = offset, L = limit lines, $ = showLineNumbers, I = maxChars
  let D = H.split("\n"), E = D.length;
  // ... slice to range, optionally add line numbers
  if (f > 0 || M < E || W) {
    let V = f + 1, X = f + P.length,
      Q = `[Showing lines ${V}-${X} of ${E} total lines`;
    if (W) {
      let w = I >= 1000 ? `${Math.floor(I / 1000)}k` : `${I}`;
      Q += `, truncated to ${w} characters`;
    }
    Q += "]";
  }
  return B;
}
```

### MissionRunner State-Change Logging (2542.js:115-135)
```js
async pause() {
  if (!this.isRunning) return;
  BH("[MissionRunner] Pausing mission");
  // ... interrupt worker ...
  await this.missionFileService.appendProgressLog({
    timestamp: new Date().toISOString(),
    type: "worker_paused",
    workerSessionId: A,
    featureId: L ?? void 0,
  });
  await this.missionFileService.updateState({ state: "paused" });
  await this.missionFileService.appendProgressLog({
    timestamp: new Date().toISOString(),
    type: "mission_paused",
  });
}
```

## Integration Points

- **01-terminal-ui (TUI)**: Mission event notifications (`v$.emit("project-notification", ...)`) are consumed by the TUI layer for real-time status display. The event bus carries `mission_state_changed`, `mission_features_changed`, `mission_progress_entry`, `mission_heartbeat`, `mission_worker_started`, and `mission_worker_completed` events. TUI rendering is 01-terminal-ui territory.
- **03-tool-agent-system (Tool/Agent)**: Tool output schemas (modules 1236, 1225, 1203, 1227, 1229) define the structured output format that tools produce. The Task tool (1227.js `task-cli`) is the subagent invocation mechanism. Tool result logging ties into 03-tool-agent-system's agent tooling.
- **04-desktop-gui (GUI)**: IDE heartbeat (2320.js `notifications/heartbeat`) is a separate channel from mission heartbeat. The GUI likely consumes the same event bus for mission status display.
- **05-infrastructure (Infra)**: ErrTracker integration in 0093.js (`captureException`, `addBreadcrumb`) provides error tracking infrastructure. The `logLevel` configuration in 3774.js (terminal LogService) is a separate concern.

## Implementation Notes

### 1. Implement the Console Logger
Port 0093.js's `BH`/`gH`/`uH` functions:
```typescript
// Core logging functions
function logInfo(message: string, context?: Record<string, unknown>, tags?: Record<string, string>): void
function logWarn(message: string, context?: Record<string, unknown>, tags?: Record<string, string>): void
function logError(error: Error | unknown, message: string, context?: Record<string, unknown>, tags?: Record<string, string>): void
```
Include: message sanitization, context redaction for sensitive fields, ErrTracker breadcrumb support.

### 2. Implement Progress Log Service
Port 2201.js's `appendProgressLog()` and incremental reader:
```typescript
interface ProgressEntry {
  type: "worker_started" | "worker_completed" | "worker_failed" | "worker_paused" | "mission_paused" | "milestone_validation_triggered" | "worker_selected_feature";
  timestamp: string;
  workerSessionId?: string;
  featureId?: string;
  exitCode?: number;
  reason?: string;
  spawnId?: string;
  milestone?: string;
}
// Append to missionDir/progress_log.jsonl
// Incremental read with file offset tracking
// Derived worker state computation from log stream
```

### 3. Implement Event Notification Bus
Port 0084.js's event schemas and 2542.js's emission pattern:
```typescript
type MissionEvent =
  | { type: "mission_state_changed"; state: MissionState }
  | { type: "mission_features_changed"; features: Feature[] }
  | { type: "mission_progress_entry"; progressLog: ProgressEntry[] }
  | { type: "mission_heartbeat"; timestamp: string }
  | { type: "mission_worker_started"; workerSessionId: string }
  | { type: "mission_worker_completed"; workerSessionId: string; exitCode: number };

// Emitter pattern
eventBus.emit("project-notification", { notification: event });
```

### 4. Implement Heartbeat
Port 2542.js's heartbeat loop:
- 180-second interval during worker polling
- Event-bus only (not persisted to progress_log)
- Separate from IDE client heartbeat (75s timeout, TCP-based)

### 5. Tool Output Formatting
Port 2509.js's text truncation utilities:
```typescript
function formatReadOutput(content: string, offset: number, limit: number, maxChars: number): string
function truncateString(str: string, maxLen: number): string
function truncateMiddle(str: string, maxLen: number): string  // head...tail
function truncateMultiline(text: string, maxChars: number, maxLines: number): { text: string; isTruncated: boolean }
```

## Known Gaps

- **OpenTelemetry log levels (0124.js)**: The `logLevel`/`DiagLogLevel` patterns in module 0124.js relate to OpenTelemetry diagnostic logging: not directly mission-specific but relevant for understanding the full observability stack. Not fully traced.
- **Terminal LogService (3774.js)**: The `LogService` class with `trace/debug/info/warn/error` levels (3774.js, line ~5924) is for the xterm.js terminal emulator, not the mission system. Mentioned for completeness but not fully documented.
- **Log scrubbing configuration (2345.js)**: Module 2345.js references `structured_logging` and `log_scrubbing` patterns as project health checks. These are meta-level assessments, not the runtime logging system.
- **ErrTracker integration depth**: The `captureException`/`captureMessage`/`addBreadcrumb` calls in 0093.js were noted but the full ErrTracker configuration and DSN resolution were not traced.
- **Exact `Pf()` sanitization logic**: The `Pf()` function that sanitizes log messages was not fully deobfuscated: its exact redaction rules (regex patterns, key matching) need further investigation.
