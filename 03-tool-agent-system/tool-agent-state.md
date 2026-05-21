# Agent State Machine / Working State: Tool System Architecture Notes

## Overview

OpenDroid implements a lightweight agent state machine that governs the working state of each agent session. The core state enum (`uJ`) defines five states: **idle**, **streaming_assistant_message**, **waiting_for_tool_confirmation**, **executing_tool**, and **compacting_conversation**. State transitions are driven by the agent execution loop (3399.js) and propagated to consumers via an event bus (`v$`) using the `"working-state-changed"` event. The `SessionController` class (2575.js) serves as the central state holder, with its `setWorkingState()` method acting as the single source of truth. A parallel decomp session state enum (`oT`) manages orchestrator/worker lifecycle states (awaiting_input, initializing, running, paused, orchestrator_turn, completed). The system does NOT use a formal state machine library: transitions are imperative and scattered across the agent runner loop, JSON-RPC adapter, and TUI hooks.

## Module Map

| Module | Size (KB) | Role | Vendor/App |
|--------|-----------|------|------------|
| 0059.js | ~1 | State enum declarations (`uJ`, `oT`, `fX`, etc.) + shared constants | App |
| 0060.js | ~2 | Enum IIFE definitions: working state (`uJ`), proceed action (`rT`), tool confirmation action (`jY`), run status (`P$H`), MCP state (`wn`), decomp role (`fX`), decomp state (`oT`), todo status (`j7`), run result (`vu`), issue severity (`NUA`) | App |
| 0061.js | ~1 | Working state groupings: `kE0` (active states), `bE0` (active + waiting states) | App |
| 0919.js | ~8 | Zod schemas for daemon JSON-RPC methods, including `workingState: AH.nativeEnum(uJ)` and `newState: AH.nativeEnum(uJ)` in session notification schemas | App |
| 2575.js | ~15 | `SessionController` class: holds `workingState` property, `setWorkingState()`/`getWorkingState()` methods, emits `"working-state-changed"` events, session CRUD | App |
| 3399.js | ~40 | Core agent runner loop (React-based): drives state transitions: idle → streaming → executing_tool → compacting_conversation → idle | App |
| 3419.js | ~5 | `SharedAgentRunner` (cIA): wraps agent loop with event callbacks, emits `working-state-changed` for streaming_assistant_message and idle on entry/exit | App |
| 3428.js | ~18 | `JsonRpcAdapter` (bfL): bridges event bus to JSON-RPC notifications, handles tool confirmation state transitions, ask-user flow | App |
| 3708.js | ~35 | `JsonRpcCliRunner` (VBL): CLI JSON-RPC entry point, manages interrupt/pause mechanism, sets working state on message processing | App |
| 3761.js | ~12 | `OpenDroidSessionManager` (hZH): daemon-side session state persistence, stores `workingState` per session, broadcasts state changes to all connections | App |
| 4040.js | ~3 | `useSessionController` React hook (BH0): exposes working state to TUI components, subscribes to `"working-state-changed"` | App |
| 4065.js | ~120 | Main TUI app: consumes `workingState`, computes `isAgentStateIdle`, manages message queue drain when idle | App |

## Architecture

### State Enum Definition

The working state enum `uJ` is defined in module 0060.js as a TypeScript-style string enum:

```
uJ.Idle                        = "idle"
uJ.StreamingAssistantMessage   = "streaming_assistant_message"
uJ.WaitingForToolConfirmation  = "waiting_for_tool_confirmation"
uJ.ExecutingTool               = "executing_tool"
uJ.CompactingConversation      = "compacting_conversation"
```

A related decomp session state enum `oT` manages mission orchestrator/worker lifecycle:

```
oT.AwaitingInput    = "awaiting_input"
oT.Initializing     = "initializing"
oT.Running          = "running"
oT.Paused           = "paused"
oT.OrchestratorTurn = "orchestrator_turn"
oT.Completed        = "completed"
```

### State Transition Flow

The primary agent execution loop (in 3399.js) drives these transitions:

```
┌──────────────────────────────────────────────────────────┐
│                    IDLE (entry/exit)                      │
└─────────────┬────────────────────────────────────────────┘
              │ User sends message / agent starts
              ▼
┌──────────────────────────────────────────────────────────┐
│         streaming_assistant_message                       │
│  - LLM is generating response tokens                     │
│  - Set on entry to agent run (3419.js, 3708.js)          │
└─────────────┬────────────────────────────────────────────┘
              │ LLM response contains tool_use blocks
              ▼
┌──────────────────────────────────────────────────────────┐
│              executing_tool                               │
│  - Tools are being dispatched and awaiting results       │
│  - executionStartTime recorded                           │
└─────────────┬────────────────────────────────────────────┘
              │ Tool requires permission (sandbox/ask_user)
              ▼
┌──────────────────────────────────────────────────────────┐
│       waiting_for_tool_confirmation                       │
│  - User must approve/deny tool execution                 │
│  - Permission request forwarded to client (3428.js)      │
│  - AskUser prompt forwarded to client                    │
└─────────────┬────────────────────────────────────────────┘
              │ User approves → back to executing_tool
              │ Tool execution completes → back to streaming
              ▼
┌──────────────────────────────────────────────────────────┐
│        compacting_conversation                            │
│  - Token budget exceeded, conversation summarized        │
│  - After compaction → returns to streaming               │
└─────────────┬────────────────────────────────────────────┘
              │ Agent turn complete / error / abort
              ▼
┌──────────────────────────────────────────────────────────┐
│                    IDLE (reset)                           │
│  - Set in finally block of agent run (3708.js, 3419.js) │
│  - Set on error recovery (3399.js)                       │
│  - Set on hook-blocked execution (3399.js)               │
└──────────────────────────────────────────────────────────┘
```

### Event Bus Propagation

State changes propagate through an EventEmitter (`v$`) via the `"working-state-changed"` event:

1. **Producer**: `SessionController.setWorkingState()` (2575.js, line 370): only emits if state actually changed (dedup guard)
2. **Consumers**:
   - `JsonRpcAdapter` (3428.js, line 42): forwards as `droid_working_state_changed` JSON-RPC notification
   - `useSessionController` hook (4040.js, line 24): updates React state for TUI
   - `OpenDroidSessionManager` (3761.js, line 184): persists to daemon session state map and broadcasts to all daemon connections
   - `uH0` message queue (4065.js): uses `isAgentStateIdle` to gate message draining

### Tool Loop Integration

The agent loop in 3399.js integrates state with tool execution:

1. **Streaming start**: State set to `streaming_assistant_message` when LLM call begins
2. **Tool detection**: After streaming completes, if `toolUses.length > 0`, state transitions to `executing_tool`
3. **Tool execution**: `wA(toolUses, ...)` executes tools; results appended as tool messages
4. **Permission gates**: If a tool requires confirmation, the permission handler pauses execution; state transitions to `waiting_for_tool_confirmation`
5. **Compaction**: If total tokens exceed threshold (`daH`), state transitions to `compacting_conversation`, conversation is summarized, then returns to `streaming_assistant_message`
6. **Loop continuation**: After tool results, the loop restarts (LLM receives tool results as context)

### Pause/Resume Mechanism

The interrupt/pause mechanism is implemented in 3708.js (`JsonRpcCliRunner`):

- **interruptAgent** property holds a callback that can abort the current agent run
- **pendingInterrupt** flag queues an interrupt if agent isn't yet running when interrupt arrives
- **Trigger**: `droid.interrupt_session` JSON-RPC method from client
- **Behavior**: Calls `interruptAgent()` callback which triggers `AbortController.abort()` on the LLM stream; state transitions to idle
- **Worker special case**: After interrupt in worker mode, process exits (`process.exit(0)`)
- **SIGINT**: Also handled at process level, triggers shutdown sequence

### Session Lifecycle

Session state persistence is managed by `OpenDroidSessionManager` (3761.js):

1. **Session creation**: `getOrCreateSessionState(sessionId)` initializes with `workingState: "idle"`, `updatedAt: timestamp`
2. **State broadcast**: `broadcastForSession()` intercepts `droid_working_state_changed` notifications and updates the session state map, then broadcasts to ALL daemon connections (not just the active one)
3. **Client registration**: `registerClient()` stores the droid client and active WebSocket listener per session
4. **Timeout**: Sessions have configurable timeout (`LOCAL_DROID_SESSION_TIMEOUT_MS = 1800000` = 30 min for local); extended on activity via `session_notification` events
5. **Cleanup**: On WebSocket disconnect, session's active listener is cleared; session timeout scheduled

### Working State Groupings

Module 0061.js defines boolean groupings used by UI:

```js
kE0 = ["streaming_assistant_message", "executing_tool", "compacting_conversation"]
bE0 = [...kE0, "waiting_for_tool_confirmation"]
```

- `kE0` = "agent is actively doing work" (streaming, executing, compacting)
- `bE0` = "agent is busy" (includes waiting for user confirmation)

These are used to determine UI states like showing a spinner, disabling input, etc.

## Key Findings

### 1. State Enum (0060.js, lines 51-55)

```js
((D) => {
    D.Idle = "idle";
    D.StreamingAssistantMessage = "streaming_assistant_message";
    D.WaitingForToolConfirmation = "waiting_for_tool_confirmation";
    D.ExecutingTool = "executing_tool";
    D.CompactingConversation = "compacting_conversation";
  })((uJ ||= {}));
```

Five-state enum. String values match enum keys (kebab_case). No formal state machine guard: any state can transition to any other state.

### 2. SessionController State Management (2575.js, lines 9-10, 368-375)

```js
class jtA {
  workingState = "idle";

  getWorkingState() {
    return this.workingState;
  }

  setWorkingState(H) {
    if (this.workingState !== H) {
      this.workingState = H;
      let A = this.getSessionId();
      v$.emit("working-state-changed", { state: H, sessionId: A ?? "" });
    }
  }
}
```

Singleton pattern (`o8()` getter, line 398). Dedup guard prevents redundant emissions. State change is synchronous: event emission is immediate.

### 3. Tool Confirmation Gate (3428.js, lines 142, 185, 231, 275)

```js
// Permission request triggers waiting state
this.sessionController.setWorkingState("waiting_for_tool_confirmation");

// After permission granted, back to streaming
this.sessionController.setWorkingState("streaming_assistant_message");

// AskUser request also triggers waiting state
this.sessionController.setWorkingState("waiting_for_tool_confirmation");
```

Both permission approval and ask-user flows use the same `waiting_for_tool_confirmation` state.

### 4. Agent Runner State Transitions (3399.js, lines 639, 828-834)

```js
// Transition to streaming
(HH?.onWorkingStateChange?.("streaming_assistant_message"),
  v$.emit("working-state-changed", {
    state: "streaming_assistant_message",
    sessionId: $L.sessionId,
  }));

// Transition to executing_tool
(HH?.onWorkingStateChange?.("executing_tool"),
  v$.emit("working-state-changed", { state: "executing_tool", sessionId: sD }));
```

Dual notification: callback + event bus. The callback (`onWorkingStateChange`) is used by the immediate caller, while the event bus propagates to all subscribers.

### 5. Interrupt Mechanism (3708.js, lines 45-46, 493-498)

```js
class VBL {
  interruptAgent = null;
  pendingInterrupt = false;

  // In message handler:
  await cIA({ ... }, {
    onInterruptReady: (P) => {
      if (((this.interruptAgent = P), this.pendingInterrupt))
        ((this.pendingInterrupt = false),
          P().catch((B) => {
            uH(B, "[JsonRpc] Pending interrupt handler failed");
          }));
    },
  });

  // Finally block:
  ((this.isAgentLoopInProgress = false),
    this.sessionController.setWorkingState("idle"),
    (this.interruptAgent = null),
    (this.pendingInterrupt = false));
```

Interrupt callback is registered after agent starts. If interrupt arrives before agent is ready, it's queued. State always resets to idle in finally block.

### 6. Daemon State Persistence (3761.js, lines 31, 166, 184)

```js
// Session state structure
A = {
  workingState: "idle",
  updatedAt: Math.floor(Date.now() / 1000),
  activeListener: void 0,
  cleanupFunction: void 0,
  timeout: void 0,
};

// State broadcast intercept
if (L.success && L.data.params.notification.type === "droid_working_state_changed") {
  let { newState: E } = L.data.params.notification,
    f = this.getOrCreateSessionState(H);
  ((f.workingState = E),
    (f.updatedAt = Math.floor(Date.now() / 1000)),
    this.broadcastToAllDaemonConnections(A));
}
```

Daemon intercepts state change notifications to persist state in memory map. Broadcasts to all connected daemon clients (not just the originating session's client).

### 7. TUI React Hook Integration (4040.js, lines 24, 73, 92)

```js
// Event subscription
"working-state-changed": (x) => {
  f(x.state);
},

// setWorkingState exposed via hook
setWorkingState: Q,
```

React hook provides reactive state binding: TUI components re-render on state change via `useState`.

## Integration Points

- **Mission System (02-orchestration/5)**: Decomp session state enum `oT` manages orchestrator/worker lifecycle; `fX` enum defines orchestrator/worker roles. Session state `decompSessionType` field used to branch behavior in agent runner.
- **TUI (01-terminal-ui)**: `useSessionController` hook (4040.js) provides working state to Ink components; `isAgentStateIdle` (4065.js) gates message queue draining; `uH0` message queue hook uses idle state for flow control.
- **Tool Execution Loop**: State transitions are tightly coupled with tool execution phases: streaming detects tool calls, executing_tool dispatches tools, waiting_for_tool_confirmation gates on permission/ask_user.
- **Permission Engine (tool-permissions)**: Permission requests trigger transition to `waiting_for_tool_confirmation`; approval triggers return to `streaming_assistant_message`.
- **Sandbox (tool-sandbox)**: Sandbox violation events can trigger permission escalation flow, which transitions to `waiting_for_tool_confirmation`.
- **MCP (tool-mcp-client/server-mgmt)**: MCP tool calls follow the same state flow; MCP status changes (`wn` enum: connecting/connected/disconnected/failed/disabled) are separate from working state.
- **LLM Adapters (tool-llm-anthropic/openai)**: Streaming start/complete callbacks trigger state transitions; abort mechanism propagates through `AbortController.signal`.

## Implementation Notes

### Agent State API

The state machine should be ported as a standalone module with:

1. **State enum**: Port from 0060.js: define as TypeScript enum or const object
2. **StateController**: Singleton class with `getWorkingState()`, `setWorkingState(state)` methods and dedup guard
3. **Event bus**: Use EventEmitter or typed event system for `"working-state-changed"` propagation
4. **React hook**: `useSessionController()` hook pattern from 4040.js for TUI integration
5. **State grouping helpers**: `isAgentActive(state)` (kE0), `isAgentBusy(state)` (bE0) predicates from 0061.js

### Key Interfaces

```typescript
// Working state enum
enum WorkingState {
  Idle = "idle",
  StreamingAssistantMessage = "streaming_assistant_message",
  WaitingForToolConfirmation = "waiting_for_tool_confirmation",
  ExecutingTool = "executing_tool",
  CompactingConversation = "compacting_conversation",
}

// State change event
interface WorkingStateChangedEvent {
  state: WorkingState;
  sessionId: string;
}

// Session state (daemon persistence)
interface SessionState {
  workingState: WorkingState;
  updatedAt: number;
  activeListener?: WebSocket;
  cleanupFunction?: () => void;
  timeout?: NodeJS.Timeout;
  cwd?: string;
  decompSessionType?: string;
}
```

### State Transitions to Implement

1. `idle` → `streaming_assistant_message` (agent run starts)
2. `streaming_assistant_message` → `executing_tool` (tool calls detected in LLM response)
3. `streaming_assistant_message` → `idle` (no tool calls, agent done)
4. `executing_tool` → `waiting_for_tool_confirmation` (permission/ask_user required)
5. `waiting_for_tool_confirmation` → `streaming_assistant_message` (user approved)
6. `executing_tool` → `streaming_assistant_message` (tools done, loop continues)
7. `streaming_assistant_message` → `compacting_conversation` (token budget exceeded)
8. `compacting_conversation` → `streaming_assistant_message` (compaction done)
9. Any state → `idle` (error, abort, interrupt, finally block)

## Open Questions / Left Undone

- **No formal state machine validation**: Current implementation allows any-to-any transitions without guard conditions. Could be enhanced with explicit transition table.
- **Decomp state `oT` integration**: How `oT` (orchestrator/worker lifecycle) interacts with `uJ` (working state) is not fully traced: they appear to operate at different abstraction levels.
- **Token threshold `daH`**: The exact value of the compaction trigger threshold was not extracted.
- **Error recovery edge cases**: AgentAbortError vs hook-stopped vs regular error handling paths could be further detailed.
- **Worker exit behavior**: Worker sessions call `process.exit(0)` after completing or being interrupted: lifecycle implications for mission orchestration need further investigation.
- **Multi-session state**: How daemon manages concurrent sessions with independent working states could be explored further (3761.js `sessionStates` Map).
