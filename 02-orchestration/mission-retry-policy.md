# mission-retry-policy: Mission System RE

## Overview (3-5 sentences)

The OpenDroid mission system implements a **delegated retry policy** where the mission runner itself does NOT automatically retry failed features. Instead, when a worker fails, the runner escalates to the orchestrator (sets state to `orchestrator_turn`) and returns control, providing detailed failure context including the reason, feature ID, and handoff data. The orchestrator (LLM) then decides whether to re-run the same feature, create a fix feature, or abandon it. The only automatic "retry-like" behavior is **feature re-queuing** (resetting a feature from `in_progress` back to `pending`) when a worker crashes or the daemon becomes unreachable. At the AWS/HTTP transport level (module 0454), a standard exponential backoff with capacity-based rate limiting is used for API retries, but this is infrastructure-level, not mission-level.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 2542.js | ~811 lines | MissionRunner core loop | Worker spawn, failure detection, feature re-queuing, daemon health monitoring |
| 2544.js | ~654 lines | StartMissionRun tool | Blocking call, orchestrator messaging on failure, resume/retry instructions |
| 0454.js | ~255 lines | AWS SDK retry strategy | Standard + adaptive retry modes, exponential backoff, capacity tokens |
| 2539.js | ~280 lines | EndFeatureRun tool | Worker handoff submission, status update, orchestrator escalation logic |
| 0082.js | ~120 lines | Schema definitions | Progress log event types (worker_failed, worker_completed, etc.) |
| 0472.js | ~80 lines | AWS credential provider | maxRetries/timeout defaults for AWS metadata service |
| 2175.js | ~2200 lines | Orchestrator system prompt | Retry instructions for orchestrator on daemon failure, restart guidance |

## Architecture / Flow

### Failure Detection Chain

```
Worker crashes/exits
    │
    ├─► Daemon notification (session_inactivity, ProcessExitError)
    │       │
    │       └─► waitForWorkerCompletion() detects via subscribeToSessionNotifications()
    │               │
    │               └─► S9H() resets feature to "pending"
    │               └─► Logs "worker_failed" to progress_log
    │               └─► Returns { success: false, returnToOrchestrator: true }
    │
    ├─► Progress log polling (every 1 second)
    │       │
    │       └─► Checks for "worker_completed" entry with workerSessionId match
    │
    └─► Daemon health check (every 5 seconds)
            │
            └─► TCP connect to daemon host:port
            └─► 3 consecutive failures → abort mission, escalate
```

### Retry Decision Flow

```
Worker fails (any cause)
    │
    ▼
MissionRunner.runLoop()
    │
    ├─► S9H() resets feature status → "pending"
    ├─► State → "orchestrator_turn"
    ├─► Break loop, return control
    │
    ▼
StartMissionRun returns to orchestrator with systemMessage
    │
    ├─► opendroidd error → "call start_mission_run once to retry, then stop"
    ├─► General failure → "investigate, then call start_mission_run"
    ├─► Handoff issues → "create features or update descriptions, then call start_mission_run"
    │
    ▼
Orchestrator decides:
    ├─► Re-run same feature (it's "pending" again)
    ├─► Create fix feature
    ├─► Abandon (mark as cancelled or skip)
    └─► Resume specific worker session (resumeWorkerSessionId param)
```

### Key State: "pending" Reset (S9H function)

The `S9H` function in Module 2542.js is the core "restart feature" mechanism. It:
1. Checks feature is `in_progress` with matching `currentWorkerSessionId`
2. Resets status to `pending`
3. Clears `currentWorkerSessionId`
This makes the feature eligible for re-execution on the next `start_mission_run` call.

## Key Findings

### 1. No Automatic Retry: Delegated to Orchestrator

The mission runner does **not** have a retry counter or max-retries configuration for failed features. Every failure immediately escalates to the orchestrator. This is by design: the orchestrator (LLM) has the intelligence to determine whether a failure is transient (retry same feature) or fundamental (create a fix feature).

**Evidence**: Module 2542.js, lines 300-307: `runLoop()` after `spawnWorker()` returns:
```js
if (!I.success) {
    await this.missionFileService.updateState({
        state: "orchestrator_turn",
        currentFeatureId: null,
        currentWorkerSessionId: null,
    });
    BH("[MissionRunner] Worker failed, returning to orchestrator");
    break;
}
```

### 2. Feature Re-Queuing: The "restartFeature" Mechanism

When a worker crashes or the daemon becomes unreachable, the system calls `S9H()` (Module 2542.js, lines 7-18) to reset the feature back to `pending`:

```js
// Module 2542.js:7-18
async function S9H(H) {
  let { missionFileService: A, featureId: L, workerSessionId: $ } = H;
  if (!L) return;
  let I = await A.getFeature(L);
  if (!I) return;
  if (I.status !== "in_progress") return;
  if ($ && I.currentWorkerSessionId !== $) return;
  await A.updateFeature(L, { status: "pending", currentWorkerSessionId: null });
}
```

This is the closest thing to "restartFeature": it's not a separate API but a side effect of failure handling. The feature goes from `in_progress` → `pending`, making it eligible for the next `start_mission_run` call.

### 3. Daemon Health Monitoring with Threshold

The runner monitors daemon health via TCP connection checks every 5 seconds (`VUf = 5000`). If the daemon is unreachable for **3 consecutive checks** (`XUf = 3`), the mission aborts and escalates:

```js
// Module 2542.js:740-780
if (!PUf() && Q - D >= VUf) {
    D = Q;
    let { host: w, port: C } = M4H();
    if (!(await BUf(w, C))) E++;
    else E = 0;
    if (E >= XUf) {
        // ... logs worker_failed, requeues feature, returns to orchestrator
        return {
            success: false,
            returnToOrchestrator: true,
            featureId: f,
            message: "opendroidd appears to be unavailable while a worker was running..."
        };
    }
}
```

**Constants** (Module 2542.js, lines 805-809):
| Constant | Value | Meaning |
|----------|-------|---------|
| `WUf` | 5000ms | Default timeout for opendroidd operations |
| `TUf` | 180000ms | Heartbeat emission interval (3 min) |
| `VUf` | 5000ms | Daemon health check interval |
| `XUf` | 3 | Consecutive daemon failure threshold |
| `GUf` | 600000ms | Worker inactivity timeout default (10 min) |

### 4. Opendroidd Retry: Single Attempt Only

For opendroidd (daemon) connectivity errors, the system message to the orchestrator explicitly limits retry to **one attempt**:

```js
// Module 2544.js:548
`Next steps: call start_mission_run again once to retry: it will attempt to reconnect 
or restart the daemon automatically. If the same opendroidd error occurs again on the retry, 
do NOT attempt further retries. Instead, tell the user: "The mission daemon failed to start 
and could not be automatically recovered. Please run /quit to exit Droid, restart it, 
then re-enter mission mode and resume the mission."`
```

### 5. AWS-Level Retry Strategy (Infrastructure)

Module 0454.js implements a full AWS SDK retry strategy with:
- **Standard mode**: Exponential backoff with jitter, base delay 100ms, max delay 20s
- **Adaptive mode**: Adds cubic rate limiting (beta=0.7, smooth=0.8)
- **Capacity system**: 500 initial tokens, retry costs 5 (normal) or 10 (transient)
- **Default max attempts**: 3

```js
// Module 0454.js:136-170
var TXH = 100,       // base delay ms
    qXA = 20000,     // max delay ms
    OlL = 500,       // throttling base delay ms
    CXA = 500,       // initial capacity tokens
    ZlL = 5,         // retry cost
    zlL = 10,        // timeout retry cost
    NlL = 1;         // no-retry increment

YlL = ({ retryDelay: H, retryCount: A, retryCost: L }) => ({
    getRetryCount: () => A,
    getRetryDelay: () => Math.min(qXA, H),
    getRetryCost: () => L,
});
```

This is NOT mission-level retry: it's for AWS API calls (S3, DynamoDB, etc.) used internally.

### 6. Handoff-Based Escalation Triggers

The EndFeatureRun tool (Module 2539.js) determines escalation based on handoff content:

```js
// Module 2539.js:44-46
let P = (f?.discoveredIssues?.length ?? 0) > 0,
    B = f?.whatWasLeftUndone?.trim(),
    W = B !== void 0 && B !== "" && B.toLowerCase() !== "none",
    V = U || P || W;  // returnToOrchestrator = explicit OR has issues OR has undone work
```

Three escalation triggers:
1. Worker explicitly sets `returnToOrchestrator: true`
2. Handoff contains `discoveredIssues` (non-empty array)
3. Handoff contains non-trivial `whatWasLeftUndone`

### 7. Worker Inactivity Timeout

The default inactivity timeout is 600 seconds (10 minutes), configurable via `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS` env var:

```js
// Module 2542.js:69-75
function FUf() {
  let H = process.env[$8.Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS],
    A = Number(H);
  if (Number.isFinite(A) && A > 0) return A;
  return GUf;  // 600000ms = 10 minutes
}
```

When a worker is inactive beyond this threshold, the daemon cleans up the session, and the runner receives a `session_inactivity` notification, which triggers the standard failure path.

### 8. Validation Failure → Back to Pending

For validation features (scrutiny-validator, user-testing-validator), the system prompt in Module 2175.js states:

```
When a validator fails, it goes back to pending. Create fix features, then call 
start_mission_run: the validator will re-run and only re-validate what failed.
```

This means validators follow the same "reset to pending, re-run" pattern as regular features.

## Code Examples

### Example 1: Feature Re-Queue on Worker Failure

```js
// Module 2542.js:654-677: Daemon event handler during worker execution
V = zX().subscribeToSessionNotifications(H, async (X) => {
    if (X.type === "session_inactivity") {
        await W({
            reason: "Worker session cleaned up by daemon after inactivity timeout.",
            spawnId: `daemon_inactivity_${H}`,
            notification: X,
        });
        return;
    }
    if (X.type === "error" && X.errorType === "ProcessExitError") {
        let Q = `Worker process exited unexpectedly: ${X.message}`;
        await W({ reason: Q, spawnId: `daemon_process_exit_${H}`, notification: X });
    }
});
```

### Example 2: Orchestrator Messaging on Failure

```js
// Module 2544.js:530-565: Determining message based on failure type
if (SH)  // opendroidd error
    MH = `...call start_mission_run again once to retry...do NOT attempt further retries...`;
else if (RH && bH)  // general worker failure
    MH = `...investigate what happened, then call start_mission_run again...`;
else if (ZH)  // handoff has actionable items
    MH = `...create new features or update existing feature descriptions...`;
else  // normal return
    MH = `...Review the progress_log.jsonl file...then call start_mission_run again...`;
```

### Example 3: EndFeatureRun Success/Failure Path

```js
// Module 2539.js:178-204: Handoff processing
if (V || v === "failure" || v === "partial")
    ((VH = "orchestrator"),
      await C.updateState({
        state: "orchestrator_turn",
        currentFeatureId: null,
        currentWorkerSessionId: null,
        completedFeatures: v === "success" ? (q?.completedFeatures || 0) + 1 : q?.completedFeatures || 0,
        orchestratorActedSinceReturn: false,
      }));
else
    ((VH = "continue"),
      await C.updateState({
        currentFeatureId: null,
        currentWorkerSessionId: null,
        completedFeatures: (q?.completedFeatures || 0) + 1,
      }));
```

On success with no issues → state stays `running`, next worker starts automatically.
On failure/partial/issues → state becomes `orchestrator_turn`, runner stops.

## Integration Points

- **01-terminal-ui (TUI)**: Mission state change events (`mission_state_changed`, `mission_heartbeat`) emitted via `v$` event bus: consumed by TUI for status display
- **03-tool-agent-system (Tool)**: Task tool / subagent invocation patterns used during validator retry flows
- **04-desktop-gui (GUI)**: Progress log entries (`worker_failed`, `worker_completed`) consumed by GUI status panels
- **05-infrastructure (Infra)**: Daemon health check (`BUf()` TCP connect) relies on opendroidd process management; inactivity timeout env var `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS`
- **features.json**: Feature status transitions `in_progress` → `pending` (on failure) or `completed` (on success)
- **validation-state.json**: Updated by validators during retry validation cycles
- **progress_log.jsonl**: `worker_failed` entries with reason, `worker_completed` entries with handoff

## Implementation Notes

### Retry Policy Implementation
1. **No automatic retry needed**: The system delegates retry decisions to the orchestrator LLM
2. **Feature re-queuing**: Implement `S9H()` equivalent: on worker crash, reset feature to `pending`
3. **Daemon health monitoring**: TCP check every 5s, abort after 3 consecutive failures
4. **Inactivity timeout**: Default 10 min, configurable via env var
5. **Opendroidd retry**: Single attempt with explicit "do not retry again" instruction

### Key Constants to Port
| Constant | Value | Purpose |
|----------|-------|---------|
| Daemon check interval | 5000ms | How often to verify daemon is alive |
| Daemon failure threshold | 3 | Consecutive failures before abort |
| Heartbeat interval | 180000ms | Progress emission to UI |
| Inactivity timeout | 600000ms | Worker stall detection |
| AWS retry base delay | 100ms | Infrastructure retry backoff base |
| AWS retry max delay | 20000ms | Infrastructure retry backoff cap |
| AWS max attempts | 3 | Infrastructure retry limit |

## Known Gaps

1. **Module 0454.js adaptive retry mode**: The cubic rate limiter math (`cubicThrottle`, `cubicSuccess`) was read but not fully traced; the adaptive mode is secondary to the standard mode for mission-level understanding
2. **Module 0483.js**: Grep hit for `maxRetries ?? 3` at line 72; may contain additional retry configuration not analyzed
3. **Module 2597.js**: Contains `PPf = "backoff"` constant at line 7; likely a string literal used in telemetry or configuration, not a full backoff implementation
4. **Modules 1244, 1204, 1208, 2526, 0945, 3982**: Listed in feature spec but are unrelated to retry policy (1244 = Slack tool, 1204 = grep tool, 1208/2526/0945/3982 have zero retry-related content). This feature's module list was stale/incorrect.
