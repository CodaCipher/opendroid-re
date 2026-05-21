# mission-cli-ui-bridge: Mission System RE

## Overview

The CLI-UI bridge is the event-driven notification layer that connects the mission runner (backend) to the TUI status display (frontend). The mission runner emits structured events through a project-scoped EventEmitter (`v$`), which the `start_mission_run` tool consumes and streams as tool updates to the TUI. The bridge defines six mission-specific event types: `mission_state_changed`, `mission_features_changed`, `mission_progress_entry`, `mission_heartbeat`, `mission_worker_started`, and `mission_worker_completed`. All events are validated against Zod schemas defined in module 0084.js and dispatched as `session_notification` JSON-RPC messages to the TUI layer.

## Module Map

| Module | Size (KB) | Role | Notes |
|--------|-----------|------|-------|
| 0084.js | ~7.5 | Event schema definitions | Zod schemas for all 6 mission event types + other session notifications |
| 2542.js | 135.8 | MissionRunner class (event emitter) | Core runner that emits all mission events via `v$.emit()` |
| 2543.js | ~10 | Worker kill + feature reset | Emits `mission_worker_completed` on worker kill; resets feature state |
| 2544.js | ~15 | StartMissionRun tool (event consumer + status formatter) | Consumes events, polls state, formats `soA()` status for TUI streaming |
| 2201.js | ~19 | MissionFileService (state persistence) | Emits `mission_state_changed` and `mission_features_changed` on state writes |
| 2478.js | ~5 | File edit helper + feature notification | Emits `mission_features_changed` when mission features change outside runner |
| 1186.js | ~3.5 | Telemetry/metrics exporter | AttributeEnhancingExporter: session/user/org metrics (tangential) |
| 0934.js | ~2 | TUI model config | `getTuiModelConfig()`: model display resolution for TUI layer |

## Architecture / Flow

### Event Emission Architecture

The bridge uses a **unidirectional event flow**: MissionRunner → EventEmitter → TUI

```
MissionRunner (2542.js)
    │
    ├── state change ──► v$.emit("project-notification", { type: "mission_state_changed" })
    ├── feature change ──► v$.emit("project-notification", { type: "mission_features_changed" })
    ├── progress log ──► v$.emit("project-notification", { type: "mission_progress_entry" })
    ├── heartbeat ──► v$.emit("project-notification", { type: "mission_heartbeat" })
    ├── worker start ──► v$.emit("project-notification", { type: "mission_worker_started" })
    └── worker done ──► v$.emit("project-notification", { type: "mission_worker_completed" })
         │
         ▼
StartMissionRun tool (2544.js): subscribes via v$.on("project-notification", handler)
         │
         ▼
Tool update stream: yields { type: "update", value: statusPayload }
         │
         ▼
TUI Layer (01-terminal-ui territory): renders status panel
```

### Event Types and Schemas

All 6 mission event types are defined in **0084.js** as Zod schemas and assembled into a discriminated union:

| Event Type | Schema Variable | Payload Fields | Trigger |
|-----------|-----------------|---------------|---------|
| `mission_state_changed` | `BM0` | `state: oT` (enum) | Mission state transition (running→paused→completed→orchestrator_turn) |
| `mission_features_changed` | `WM0` | `features: pNH[]` | Feature array change (status update, new feature, reorder) |
| `mission_progress_entry` | `TM0` | `progressLog: lNH[]` | New entry appended to progress_log.jsonl |
| `mission_heartbeat` | `VM0` | `timestamp: string` | Every 180 seconds while worker is running |
| `mission_worker_started` | `XM0` | `workerSessionId: string` | New worker spawned by MissionRunner |
| `mission_worker_completed` | `GM0` | `workerSessionId: string, exitCode: number` | Worker session finishes (success, failure, or kill) |

### State Query Interface

The **StartMissionRun** tool (2544.js) provides the state query interface for the TUI. It does NOT rely solely on events: it also **actively polls** mission state:

1. **Polling loop**: Every 500ms, reads `state.json` + `features.json` + `progress_log.jsonl` from disk
2. **Change detection**: Compares serialized JSON to detect state/feature/progress changes
3. **Status aggregation**: Calls `soA(state, features, workerActivity, workerToolCount)` to build composite status
4. **Stream yield**: Pushes `{ type: "update", value: eoA(statusPayload) }` into the tool output stream

The composite status payload (`soA()` in 2544.js, line ~110) contains:
```js
{
  missionState: string,           // from state.json
  startedAt: string,              // mission creation timestamp
  currentFeatureId: string,       // active feature
  currentWorkerSessionId: string, // active worker session
  currentWorkerToolCount: number, // tool invocations by current worker
  completedFeatures: number,      // count of completed features
  totalFeatures: number,          // total feature count
  features: [{                    // per-feature summary
    id, description, status, milestone, skillName
  }],
  workerActivity: [{              // last 4 worker actions
    type: "thinking"|"tool_call"|"text",
    summary: string
  }]
}
```

### Worker Activity Monitoring

The bridge monitors worker activity by watching the worker's JSONL conversation log file (2544.js):

- File path: `{sessions_dir}/{sessionId}/{sessionId}.jsonl`
- Uses `fs.watch()` for real-time file change detection
- Parses last 4 assistant messages to extract thinking snippets, tool calls, and text
- Extracts tool names and truncated parameters for display
- Debounces file reads to avoid thrashing

### Heartbeat Protocol

Mission heartbeat (2542.js, line ~787):
- Emitted every **180 seconds** (`TUf = 180000`) while a worker is running
- Payload: `{ type: "mission_heartbeat", timestamp: ISO_string }`
- Purpose: Signal to TUI that the mission runner is alive and the worker is still processing
- Also checks daemon (opendroidd) reachability every 5 seconds: if unreachable 3 times consecutively, fails the worker

## Key Findings

### KF1: Dual-Channel Status Update (Event + Poll)
The TUI receives status updates through **two channels simultaneously**:
1. **Event-driven**: `v$.on("project-notification")` fires immediately when state changes
2. **Poll-driven**: 500ms polling loop reads state.json/features.json for periodic sync
Both channels converge in the `StartMissionRun` tool's update queue, which deduplicates consecutive status updates.

### KF2: EventEmitter is In-Process (Not IPC)
The `v$` EventEmitter is shared in the same Node.js process: this is NOT an IPC bridge. The mission runner and the tool execution share the same process memory. The "bridge" to the TUI happens at the tool stream level (tool updates → JSON-RPC → TUI).

### KF3: Progress Log is the Source of Truth
While `state.json` and `features.json` hold current state, the `progress_log.jsonl` file is the authoritative event log. Every worker start, completion, failure, pause, and milestone validation trigger is recorded as a timestamped entry. The TUI reads this log to build `workerActivity` and derive worker states.

### KF4: Feature Change Events Fire from Multiple Sources
`mission_features_changed` is emitted from:
- **2201.js** (`MissionFileService.writeFeatures`): on any features.json write
- **2542.js** (`waitForWorkerCompletion`): during polling when features change
- **2544.js** (`StartMissionRun`): initial feature load
- **2478.js** (`emitMissionFeaturesIfApplicable`): when features change outside the runner

### KF5: Daemon Reachability Check is Part of the Bridge
The mission runner performs periodic daemon health checks (opendroidd TCP connection on configured host:port). If unreachable for 3 consecutive checks (5s interval = 15s total), it emits `worker_failed` and returns control to the orchestrator.

## Code Examples

### Event Emission: Mission State Changed (2542.js, line 632-633)
```js
// MissionRunner.waitForWorkerCompletion(): polling detects state change
v$.emit("project-notification", {
  notification: { type: "mission_state_changed", state: X.state },
});
```

### Event Emission: Mission Features Changed (2201.js, ErrTracker init section6-257)
```js
// MissionFileService.updateFeature(): after writing features.json
v$.emit("project-notification", {
  notification: { type: "mission_features_changed", features: E.features },
});
```

### Event Emission: Heartbeat (2542.js, line 786-787)
```js
// MissionRunner.waitForWorkerCompletion(): every 180 seconds
v$.emit("project-notification", {
  notification: { type: "mission_heartbeat", timestamp: new Date(Q).toISOString() },
});
```

### Event Emission: Worker Completed (2542.js, line 716-721)
```js
// MissionRunner.waitForWorkerCompletion(): worker finished
v$.emit("project-notification", {
  notification: {
    type: "mission_worker_completed",
    workerSessionId: H,
    exitCode: X.exitCode,
  },
});
```

### Status Aggregation for TUI (2544.js, soA function)
```js
function soA(H, A, L, $) {
  let I = A.filter((D) => D.status === "completed").length;
  return {
    missionState: H?.state,
    startedAt: H?.createdAt,
    currentFeatureId: H?.currentFeatureId,
    currentWorkerSessionId: H?.currentWorkerSessionId,
    currentWorkerToolCount: $,
    completedFeatures: I,
    totalFeatures: A.length,
    features: A.map((D) => ({
      id: D.id, description: D.description, status: D.status,
      milestone: D.milestone, skillName: D.skillName,
    })),
    workerActivity: L,
  };
}
```

### Event Subscription in StartMissionRun (2544.js, line 440-442)
```js
v$.on("project-notification", IH);
// ...
let i = () => {
  (LH(), v$.off("project-notification", IH));
};
```

### Worker Activity Extraction (2544.js, ZUf function)
```js
// Parses worker's JSONL conversation log for activity summary
if (M.name === "Execute" && M.input?.command)
  U = `Execute: ${gm(String(M.input.command), 200)}`;
else if (M.name === "Read" && M.input?.file_path)
  U = `Read: ${gm(String(M.input.file_path), 200)}`;
// Returns last 4 activity items
```

### Event Schema Definitions (0084.js, lines 112-119)
```js
(BM0 = AH.object({ type: AH.literal("mission_state_changed"), state: AH.nativeEnum(oT) })),
(WM0 = AH.object({ type: AH.literal("mission_features_changed"), features: AH.array(pNH) })),
(TM0 = AH.object({ type: AH.literal("mission_progress_entry"), progressLog: AH.array(lNH) })),
(VM0 = AH.object({ type: AH.literal("mission_heartbeat"), timestamp: AH.string() })),
(XM0 = AH.object({ type: AH.literal("mission_worker_started"), workerSessionId: AH.string() })),
(GM0 = AH.object({
  type: AH.literal("mission_worker_completed"),
  workerSessionId: AH.string(),
  exitCode: AH.number(),
})),
```

### Kill Worker: Event Emission (2543.js, line 96-97)
```js
v$.emit("project-notification", {
  notification: { type: "mission_worker_completed", workerSessionId: A, exitCode: 1 },
});
```

## Integration Points

- **01-terminal-ui (TUI Rendering)**: The TUI layer consumes the tool update stream from `StartMissionRun` and renders the mission status panel. The `soA()` payload shape defines the TUI data contract. The 6 event types (`mission_state_changed`, etc.) are the event contract for real-time rendering. Worker activity display (thinking, tool calls) is also 01-terminal-ui territory.
- **03-tool-agent-system (Tool System)**: The `StartMissionRun` tool is a standard tool execution with streaming updates. The tool framework (ToolResult type, yield-based streaming) is 03-tool-agent-system's concern.
- **05-infrastructure (Infrastructure)**: The daemon reachability check (`BUf()` in 2542.js) probes opendroidd's TCP port. The `v$` EventEmitter is process-level infrastructure. The worker JSONL log path resolution depends on session directory structure.

## Implementation Notes

### Event System
1. Define 6 mission event types as a discriminated union (TypeScript enum + Zod)
2. Use a process-level EventEmitter (`v$`) with `"project-notification"` channel
3. Wrap events in `{ notification: { type: <event_type>, ...payload } }` envelope
4. All events validated by Zod schemas before emission

### Status Polling
1. Implement 500ms polling loop that reads state.json + features.json + progress_log.jsonl
2. Serialize and compare to detect changes (JSON.stringify comparison)
3. Build composite status via `soA()`-equivalent function
4. Push updates into a tool output stream with deduplication

### Worker Activity
1. Watch worker's JSONL conversation log via `fs.watch()`
2. Parse last 4 assistant messages for thinking/tool_call/text blocks
3. Extract tool names and truncated parameters
4. Include in status payload as `workerActivity` array

### Heartbeat
1. Emit heartbeat every 180 seconds while worker is active
2. Include ISO timestamp
3. Concurrently check daemon reachability (TCP probe every 5s, fail after 3 misses)

## Known Gaps

- **Module 1186.js** (metrics exporter): Only tangentially related: the AttributeEnhancingExporter adds session/user/org attributes to telemetry metrics. Not directly part of the CLI-UI bridge but may be relevant for 05-infrastructure telemetry.
- **Module 1190.js** (language mapping): Completely unrelated to the bridge: it's a file extension to language name map. The original feature spec's module assignment was partially off.
- **Module 0934.js** (TUI model config): Contains `getTuiModelConfig()` and model resolution logic. Only briefly examined: its full relationship to TUI rendering should be explored in 01-terminal-ui.
- **Event delivery to actual TUI process**: The bridge described here is within the Node.js backend. The final hop from backend tool stream → Electron TUI process was not traced (likely IPC/websocket, 01-terminal-ui territory).
- **pNH schema** (feature shape for events): The feature array in `mission_features_changed` uses `pNH` schema which was defined in a dependency not fully explored: it mirrors the features.json feature schema.
