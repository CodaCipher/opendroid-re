# mission-worker-lifecycle: Mission System RE

## Overview (3-5 sentences)

The worker lifecycle is managed through a cooperative protocol between the MissionRunner (2542.js), the start_mission_run tool (2544.js), and daemon RPC calls. Workers are spawned as autonomous daemon sessions via `spawnWorkerSession`, receive a bootstrap prompt with feature assignment and skill instructions, then run independently until they call EndFeatureRun. The runner monitors workers through a 1-second polling loop on `progress_log.jsonl`, daemon session notifications (`session_inactivity`, `ProcessExitError`), and periodic daemon health checks every 5 seconds. Crash recovery resets the feature status to `pending` and returns control to the orchestrator, which can retry by calling `start_mission_run` again. Pause/resume works by interrupting the worker session, saving its `workerSessionId` in state, then re-loading that session with a continuation prompt when the orchestrator resumes.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 2542.js | 19.0 KB | **MissionRunner** (`toA`) | Core run loop, `spawnWorker()`, `resumeWorker()`, `waitForWorkerCompletion()`, crash detection, daemon health check |
| 2543.js | ~4 KB | **Worker lifecycle helpers** (`FNI`) | `pauseMissionRunner()` (`Hx`), `killWorkerSession()` (`JNI`), feature requeue (`JUf`), inactivity timeout config (`FUf`) |
| 2544.js | ~20 KB | **start_mission_run tool** (`HtA`) | Tool implementation with streaming status updates, worker file watching, handoff collection, abort handling, system messages for orchestrator |
| 0087.js | 8.1 KB | **RPC schema definitions** (`v$H`) | Zod schemas for daemon RPC methods: `droid.kill_worker_session`, `droid.interrupt_session`, worker state schema (`tM0`), mission snapshot schema (`aM0`) |
| 1234.js | 8.0 KB | **Tool parameter schemas** (`nGH`) | Zod schemas for `start_mission_run` params, `EndFeatureRun` params, handoff schema (`Z11`), worker handoff summary (`Gz$`) |
| 2575.js | 9.9 KB | **SessionController** (`jtA`) | Session CRUD, model switching, `killWorkerSession()` on controller, mission model settings, working state management |
| 3985.js | 7.7 KB | **Worker status TUI component** (`daD`) | React component rendering worker timeline: status icons, duration, feature description, filter tabs (all/active/completed/failed) |
| 2541.js | ~4 KB | **Worker prompt builder** (`GNI`) | Constructs bootstrap message for new workers and resume message for interrupted workers |

## Architecture / Flow

### Worker Lifecycle State Machine

```
                    ┌─────────────────────┐
                    │  Feature: pending    │
                    │  (in features.json)  │
                    └──────────┬───────────┘
                               │
                    MissionRunner.spawnWorker()
                               │
                    ┌──────────▼───────────┐
                    │  Feature: in_progress │
                    │  currentWorkerSessionId│
                    │  set in features.json  │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Worker Running       │
                    │  (daemon session)     │
                    │  Monitored via:       │
                    │  - progress_log poll  │
                    │  - daemon notifs      │
                    │  - health checks      │
                    └──┬─────┬──────┬──────┘
                       │     │      │
           ┌───────────┘     │      └───────────┐
           │                 │                   │
    ┌──────▼──────┐  ┌──────▼──────┐   ┌────────▼────────┐
    │  Completed  │  │   Failed    │   │     Paused      │
    │  (worker    │  │  (crash,    │   │  (user SIGINT,  │
    │  calls      │  │  inactivity,│   │   abort signal) │
    │  EndFeature │  │  daemon     │   │                 │
    │  Run)       │  │  unreachable│   │                 │
    └──────┬──────┘  └──────┬──────┘   └────────┬────────┘
           │                │                    │
           ▼                ▼                    ▼
    Feature: completed  Feature: pending    State: paused
    State: continues    State: orchestrator   workerSessionId
    to next feature     _turn                 saved for resume
```

### spawnWorker Flow (2542.js:416-520)

1. **Generate spawn ID**: `worker_{random_8chars}`
2. **Get next pending feature**: `missionFileService.getNextPendingFeature()`: first feature with `status === "pending"`
3. **Select model**: Validation workers use `getMissionValidationWorkerModel()`, implementation workers use `getMissionWorkerModel()`. If model supports `"high"` reasoning effort, use it.
4. **Daemon pre-warm**: `It()`: check if opendroidd is reachable; if not, throw error
5. **Spawn session**: `zX().spawnWorkerSession({ cwd, baseSessionId, modelId, interactionMode: "auto", autonomyLevel: "high", reasoningEffort, inactivityTimeoutMs })`: creates a new daemon session
6. **Update feature status**: Set to `in_progress` with `currentWorkerSessionId`
7. **Update mission state**: Set `currentFeatureId`, `currentWorkerSessionId`, append to `workerSessionIds[]`
8. **Log selection**: `worker_selected_feature` entry in progress log
9. **Build bootstrap prompt**: `GNI(missionDir, feature, workerSessionId)`: includes feature JSON, skill invocation order, session rules
10. **Send bootstrap**: `zX().addUserMessage({ sessionId, text: bootstrapPrompt })`: worker receives initial instructions
11. **Begin monitoring**: `waitForWorkerCompletion(workerSessionId)`

### resumeWorker Flow (2542.js:521-609)

1. **Get feature ID** from current state's `currentFeatureId`
2. **Daemon pre-warm**: Check opendroidd reachability
3. **Load session**: `zX().loadSession(workerSessionId)`: restore the worker's conversation history
4. **Send resume prompt**: Includes "You were interrupted mid-work. Continue where you left off." plus instructions to check git status and review existing work
5. **Begin monitoring**: `waitForWorkerCompletion(workerSessionId)`

### waitForWorkerCompletion Flow (2542.js:611-800)

This is the core monitoring loop:

1. **Poll progress log** every 1000ms via `_Uf(1000, abortSignal)`: look for `worker_completed` entry matching `workerSessionId`
2. **Emit notifications** when state/features/progress change:
   - `mission_state_changed`: mission state transitions
   - `mission_features_changed`: feature array modifications
   - `mission_progress_entry`: new progress log entries
3. **Subscribe to daemon notifications** via `zX().subscribeToSessionNotifications(workerSessionId, callback)`:
   - `session_inactivity` → Worker cleaned up by daemon after inactivity timeout → treat as failure
   - `ProcessExitError` → Worker process crashed → treat as failure
4. **Daemon health check** every 5 seconds (`VUf = 5000`):
   - TCP port probe via `BUf(host, port)` 
   - 3 consecutive failures (`XUf = 3`) → daemon unreachable → log `worker_failed`, requeue feature, return to orchestrator
5. **Heartbeat** every 180 seconds (`TUf = 180000`):
   - Emit `mission_heartbeat` notification for UI freshness
6. **On worker_completed found**:
   - Emit `mission_worker_completed` notification
   - `zX().interruptSession(workerSessionId)`: clean up the session
   - Return result with success/failure and `returnToOrchestrator` flag

### pauseMissionRunner Flow (2543.js:38-80)

1. **Attempt MissionRunner.pause()**: sets `isRunning = false`, breaks the run loop
2. **Interrupt worker session** via `zX().interruptSession(currentWorkerSessionId)`: non-blocking fire-and-forget
3. **Log `worker_paused`** in progress log with `workerSessionId` and `featureId`
4. **Set state to `paused`** in state.json
5. **Log `mission_paused`** in progress log

### killWorkerSession Flow (2543.js:83-120)

1. **Log `worker_failed`** in progress log with reason "Killed by user"
2. **Emit `mission_worker_completed`** notification with exitCode 1
3. **Requeue feature**: `JUf()` resets feature status from `in_progress` back to `pending`, clears `currentWorkerSessionId`
4. **Set state to `orchestrator_turn`** with `currentFeatureId: null`, `currentWorkerSessionId: null`
5. **Interrupt session** via daemon RPC (fire-and-forget)

### Crash Detection and Recovery

Three crash scenarios are handled:

**1. Worker Inactivity Timeout** (detected via daemon notification):
- Daemon emits `session_inactivity` when worker is idle for `QUf` ms (default `GUf` = 600000ms = 10 min, configurable via `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS`)
- Handler logs `worker_failed`, requeues feature (`S9H` resets to `pending`), returns to orchestrator

**2. Worker Process Crash** (detected via daemon notification):
- Daemon emits `ProcessExitError` when the worker process exits unexpectedly
- Handler constructs reason: `"Worker process exited unexpectedly: {message}"`, logs `worker_failed`, requeues feature, returns to orchestrator

**3. Daemon Unreachable** (detected via health check):
- Runner checks daemon reachability every 5s via TCP port probe
- After 3 consecutive failures (15 seconds total), the daemon is considered dead
- Logs `worker_failed` with reason about opendroidd unavailability, requeues feature, returns to orchestrator with instruction to ask user to restart Droid

### Worker Status Display (3985.js)

The TUI component `YvM()` reconstructs worker status from progress log entries:
- **`worker_selected_feature`** → maps worker to feature ID
- **`worker_started`** → alternative feature association
- **`worker_completed`** → maps worker to success state (`success`/`partial`/`failure`)
- **`worker_paused`** → identifies the paused worker (searched backwards from end)
- Duration calculated from `workerStates[sessionId].startedAt` to `completedAt` (or `Date.now()`)

## Key Findings

### 1. Workers are Daemon Sessions, Not Child Processes

Workers are created via `zX().spawnWorkerSession()`: an RPC call to the opendroid daemon. They run as independent agent sessions with `interactionMode: "auto"` and `autonomyLevel: "high"`. The MissionRunner has no direct process control; it monitors via progress log polling and daemon notifications. This means:
- Workers outlive the runner process (if runner crashes, workers may still be running)
- Worker cleanup happens via `interruptSession()` RPC, not process kill
- Orphaned workers are detected and cleaned up at start via `cleanupOrphanedWorker()`

### 2. Feature Requeue is the Primary Crash Recovery Mechanism

When a worker fails for any reason (inactivity, crash, daemon unreachable), the response is always the same pattern (2542.js, function `S9H`/`JUf`):
1. Reset feature status from `in_progress` to `pending`
2. Clear `currentWorkerSessionId` on the feature
3. Set mission state to `orchestrator_turn`
4. The next call to `start_mission_run` will pick up the requeued feature

There is **no automatic retry counter** or **backoff strategy** in the worker lifecycle layer: retries are delegated to the orchestrator's judgment.

### 3. Worker Activity Monitoring via File Watching

The `start_mission_run` tool (2544.js) implements real-time worker activity monitoring:
- **File watcher**: `fs.watch()` on the worker's session JSONL file (`{opendroidHome}/sessions/{cwdHash}/{sessionId}.jsonl`)
- **Activity extraction**: `ZUf()` reads the last 512KB of the session file, parses recent messages, extracts thinking summaries, tool calls, and text responses
- **Streaming updates**: Activity is pushed to the orchestrator as `{ type: "status", details: JSON.stringify(stateSnapshot) }` update events
- **Throttled**: Only keeps last 4 activity items, polls on file change events

### 4. Handoff Gating Prevents Blind Continuation

The `start_mission_run` tool (2544.js:266-295) implements a handoff review gate:
- Before resuming from `orchestrator_turn` state, checks if the last worker handoff has `discoveredIssues` or non-empty `whatWasLeftUndone`
- If either exists, returns an **error** to the orchestrator with instructions to address items first
- Uses `lastReviewedHandoffCount` in state.json to track which handoffs have been reviewed
- This ensures the orchestrator doesn't blindly re-start without handling actionable worker output

### 5. Inactivity Timeout is Environment-Configurable

The `FUf()` function (2543.js:1, referenced as `QUf` at line 450 of 2542.js) reads `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS` from environment, defaulting to `GUf` (600000ms = 10 minutes). This timeout is passed to `spawnWorkerSession` and the daemon enforces it: if a worker is inactive for this duration, the daemon cleans it up and emits `session_inactivity`.

### 6. Worker Bootstrap Prompt is Feature-Aware

The `GNI()` function (2541.js) constructs different prompts based on whether the worker is a validation worker or implementation worker:
- **Implementation workers**: `"1. Invoke 'mission-worker-base' skill\n2. Invoke '{skillName}' skill\n3. Call EndFeatureRun"`
- **Validation workers** (scrutiny/user-testing): `"1. Invoke '{skillName}' skill\n2. Call EndFeatureRun"`: skips the base skill
- The prompt includes full feature JSON, mission file paths, and session ID

## Code Examples

### spawnWorker: Worker Creation (2542.js:416-470)

```js
// Module 2542.js:416-470
async spawnWorker() {
  let H = `worker_${rf().slice(0, 8)}`,
    A = await this.missionFileService.getNextPendingFeature();
  if (!A)
    return { success: false, returnToOrchestrator: true, message: "No pending features available" };
  let L = HMH.includes(A.skillName),  // is validation worker?
    $ = vA(),
    I = L ? $.getMissionValidationWorkerModel() : $.getMissionWorkerModel(),
    f = (F$(I).supportedReasoningEfforts ?? []).includes("high") ? "high" : n3(I);
  // ... daemon pre-warm ...
  B = await OaH({
    label: "spawnWorkerSession",
    promise: zX().spawnWorkerSession({
      cwd: P,
      baseSessionId: this.baseSessionId,
      modelId: I,
      interactionMode: "auto",
      autonomyLevel: "high",
      reasoningEffort: f,
      inactivityTimeoutMs: QUf,
    }),
  });
  // Update feature + state + progress log + send bootstrap prompt
  await this.missionFileService.updateFeature(A.id, {
    status: "in_progress",
    currentWorkerSessionId: B,
    workerSessionIds: [...(A.workerSessionIds ?? []), B],
  });
  // ...
  let Q = GNI(this.missionFileService.getMissionDir(), A, B);
  await OaH({ label: "addUserMessage", promise: zX().addUserMessage({ sessionId: B, text: Q }) });
  return this.waitForWorkerCompletion(B);
}
```

### waitForWorkerCompletion: Crash Detection (2542.js:696-800)

```js
// Module 2542.js:696-800 (condensed)
let V = zX().subscribeToSessionNotifications(H, async (X) => {
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
// ... polling loop ...
// Daemon health check every 5s:
if (!PUf() && Q - D >= VUf) {
  D = Q;
  if (!(await BUf(host, port))) E++;
  else E = 0;
  if (E >= XUf) {  // 3 consecutive failures
    // Log worker_failed, requeue feature, return to orchestrator
  }
}
// Heartbeat every 180s:
if (Q - I >= TUf) {
  v$.emit("project-notification", {
    notification: { type: "mission_heartbeat", timestamp: new Date(Q).toISOString() }
  });
}
```

### pauseMissionRunner (2543.js:38-80)

```js
// Module 2543.js:38-80
async function Hx(H) {
  let A = Hf(H), L = aoA.get(H);
  if (L) {
    if ((await L.pause(), (await A.readState())?.state === "paused")) return;
  }
  let I = await A.readState();
  if (I.currentWorkerSessionId) {
    try {
      Promise.resolve(zX().interruptSession(I.currentWorkerSessionId)).catch((D) => {
        gH("[pauseMissionRunner] Failed to interrupt worker session", {
          workerSessionId: I.currentWorkerSessionId ?? void 0, cause: D
        });
      });
    } catch (D) { /* ... */ }
    await A.appendProgressLog({
      timestamp: new Date().toISOString(),
      type: "worker_paused",
      workerSessionId: I.currentWorkerSessionId,
      featureId: I.currentFeatureId ?? void 0,
    });
  }
  await A.updateState({ state: "paused" });
  await A.appendProgressLog({ timestamp: new Date().toISOString(), type: "mission_paused" });
}
```

### killWorkerSession (2543.js:83-120)

```js
// Module 2543.js:83-120
async function JNI({ missionId: H, workerSessionId: A }) {
  let L = Hf(H);
  await L.appendProgressLog({
    timestamp: new Date().toISOString(),
    type: "worker_failed",
    workerSessionId: A,
    spawnId: `killed_${A}`,
    reason: "Killed by user",
  });
  v$.emit("project-notification", {
    notification: { type: "mission_worker_completed", workerSessionId: A, exitCode: 1 },
  });
  if (I.currentFeatureId)
    await JUf({ missionFileService: L, featureId: I.currentFeatureId, workerSessionId: A });
  await L.updateState({
    state: "orchestrator_turn",
    currentFeatureId: null, currentWorkerSessionId: null, currentWorkerPid: null,
  });
  Promise.resolve(zX().interruptSession(A)).catch(() => {});
}
```

### Worker Status Reconstruction (3985.js:53-130)

```js
// Module 3985.js:53-130 (condensed)
function YvM(H, A, L, $, I) {
  // H = workerSessionIds, A = workerStates, L = progressLog, $ = features, I = missionState
  let M;  // paused worker session ID
  if (I === "paused")
    for (let X = L.length - 1; X >= 0; X--) {
      let Q = L[X];
      if (Q.type === "worker_paused" && "workerSessionId" in Q) { M = Q.workerSessionId; break; }
    }
  // Reverse-iterate progress log to find feature assignments and completions
  for (let X = L.length - 1; X >= 0; X--) {
    let Q = L[X];
    if (!("workerSessionId" in Q)) continue;
    if (Q.type === "worker_selected_feature" && !E.has(w)) (E.set(w, Q.featureId), P--);
    else if (Q.type === "worker_started" && Q.featureId && !E.has(w)) (E.set(w, Q.featureId), P--);
    else if (Q.type === "worker_completed" && !f.has(w)) (f.set(w, Q.successState), W--);
  }
  // Build worker status objects with sessionId, featureId, duration, status
  for (let X = 0; X < H.length; X++) {
    let q = "running";
    if (I === "paused" && Q === M) q = "paused";
    if (w?.completedAt) {
      let v = f.get(Q);
      if (v === "success") q = "success";
      else if (v === "partial") q = "partial";
      else q = "failed";
    }
    D.push({ sessionId: Q, shortId: Q.slice(0, 8), featureId: C, workerNumber: X + 1, status: q });
  }
}
```

### EndFeatureRun Schema (1234.js:155-215)

```js
// Module 1234.js:155-215 (condensed)
let Z11 = p.object({
  salientSummary: p.string().min(20).max(500)
    .refine((H) => !H.includes("\n"), { message: "must be 1-4 sentences (no newlines)" })
    .describe("1-4 sentence salient summary of what happened in this session"),
  whatWasImplemented: p.string().min(50)
    .describe("Concrete description of what was built (min 50 characters)"),
  whatWasLeftUndone: p.string()
    .describe("Anything incomplete or deferred. Must leave empty if truly complete."),
  verification: J11.describe("Verification performed: commands with results + any manual checks"),
  tests: C11.describe("Tests written or updated"),
  discoveredIssues: p.array(q11).describe("Issues found. Empty array if none."),
  skillFeedback: O11.optional(),
});

let Qz$ = p.object({
  successState: p.enum(["success", "partial", "failure"]),
  returnToOrchestrator: p.boolean(),
  featureId: p.string().optional(),
  commitId: p.string().optional(),
  validatorsPassed: p.boolean().optional(),
  handoff: Z11,
});
```

### Worker State Schema (0087.js:143-148)

```js
// Module 0087.js:143-148
let tM0 = AH.object({
  startedAt: AH.string(),
  completedAt: AH.string().optional(),
  exitCode: AH.number().optional(),
});
// Used in mission snapshot:
let aM0 = AH.object({
  state: AH.nativeEnum(oT),
  features: AH.array(pNH),
  progressLog: AH.array(lNH),
  workerSessionIds: AH.array(AH.string()),
  workerStates: AH.record(tM0).optional(),
  tokenUsage: hn.optional(),
  tokenUsageBySessionId: AH.record(hn).optional(),
});
```

## Integration Points

### 01-terminal-ui (TUI)
- **3985.js** renders the worker timeline in the TUI using `YvM()` to reconstruct status from progress log entries
- Status icons: `●` (running), `P` (paused), `✓` (success), `!` (partial), `✗` (failed)
- Filter tabs: all / active / completed / failed
- Worker activity display reads from session JSONL files

### 03-tool-agent-system (Tool)
- `start_mission_run` is the primary tool (2544.js `HtA` class): a streaming tool that yields status updates
- `EndFeatureRun` is called by workers to submit handoffs (1234.js defines the schema)
- `droid.kill_worker_session` RPC method (0087.js:166) allows explicit worker termination
- `droid.interrupt_session` RPC method (0087.js:165) for graceful pause

### 04-desktop-gui (GUI)
- Worker status data shape (`soA()` in 2544.js) could be consumed by GUI components
- `mission_worker_completed`, `mission_state_changed`, `mission_heartbeat` notifications for real-time updates
- Worker activity (tool calls, thinking, text) extracted from session JSONL files

### 05-infrastructure (Infra)
- Session files stored at `{opendroidHome}/sessions/{cwdHash}/{sessionId}.jsonl`
- Daemon RPC layer (`zX()`) provides `spawnWorkerSession`, `loadSession`, `interruptSession`, `subscribeToSessionNotifications`
- TCP health check mechanism for daemon reachability
- `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS` env var for timeout configuration
- `OPENDROID_HOME_OVERRIDE` env var for path resolution

## Implementation Notes

### 1. Worker Spawner
**Priority: P0: Required for any multi-agent execution**

Implement `spawnWorker()`:
1. Get next pending feature from features.json
2. Select model based on feature type (validation vs implementation)
3. Create autonomous agent session with `interactionMode: "auto"`, `autonomyLevel: "high"`
4. Set `reasoningEffort: "high"` if model supports it
5. Pass `inactivityTimeoutMs` (default 10 min) to session creation
6. Update feature status to `in_progress` with session ID
7. Build bootstrap prompt with feature JSON, skill invocation order, session rules
8. Send bootstrap as first user message to the session

### 2. Worker Monitor
**Priority: P0: Required for reliability**

Implement `waitForWorkerCompletion()`:
- 1-second polling interval on progress log for `worker_completed` entry
- Subscribe to daemon notifications for `session_inactivity` and `ProcessExitError`
- Periodic daemon health check (every 5s, 3 strikes = unreachable)
- Heartbeat emission every 180s for UI freshness
- On completion: interrupt session, emit notification, return result

### 3. Crash Recovery
**Priority: P0: Required for robustness**

Implement the feature requeue pattern:
- On any worker failure: reset feature `status` to `pending`, clear `currentWorkerSessionId`
- Log `worker_failed` in progress log with reason and spawn ID
- Set mission state to `orchestrator_turn` for manual intervention
- No automatic retry: let orchestrator decide whether to retry

### 4. Pause/Resume
**Priority: P1: Required for user control**

Implement `pauseMissionRunner()`:
- Interrupt current worker session via daemon RPC
- Log `worker_paused` in progress log
- Set state to `paused`, save `workerSessionId` for resume

Implement `resumeWorker()`:
- Load previous worker session via daemon RPC
- Send continuation prompt with "Continue where you left off"
- Begin monitoring again

### 5. Worker File Watching
**Priority: P2: Nice-to-have for UX**

Implement the activity monitoring from 2544.js:
- Watch worker session JSONL file for changes
- Extract last 512KB, parse recent messages
- Show thinking summaries, tool calls, text responses (last 4 items)
- Stream updates to orchestrator UI

### 6. Handoff Gating
**Priority: P1: Required for quality**

Implement the handoff review gate:
- Before resuming, check if last worker handoff has `discoveredIssues` or `whatWasLeftUndone`
- Block continuation until orchestrator addresses items
- Track `lastReviewedHandoffCount` to avoid re-checking old handoffs

## Known Gaps
- **Module 3987.js** (TUI mission status navigator) was only partially analyzed: the session viewer and handoff viewer UI rendering was not fully traced. This is 01-terminal-ui territory.
- **Module 2173.js** (search/caching system for sessions) was identified as session indexing infrastructure, not directly part of the worker lifecycle: its bloom filter search is relevant for session management but not critical for lifecycle understanding.
- **`FUf()` in 2543.js** that reads `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS` was referenced but the actual function body was not read: the default value `GUf = 600000` was inferred from the orchestrator-core report.
- **`S9H()` (feature requeue function)** was referenced in multiple places but its exact implementation was not directly read: it's called in `spawnWorker` error handling and crash recovery paths. Its behavior (reset feature to pending) was confirmed via the `JUf()` function in 2543.js which performs the same operation.
- **Worker file watcher retry logic** in 2544.js (`DH` function, lines 400-440) retries 30 times with 200ms delay to find the session JSONL file: this is useful for slow filesystems but was not fully traced.
- **EndFeatureRun tool handler**: the server-side processing of EndFeatureRun (writing handoff to JSON file, updating features.json status to completed, appending to handoffs.jsonl) was not fully traced in this feature. It's likely in 2542.js or 2544.js but the exact handler was beyond the read scope.
