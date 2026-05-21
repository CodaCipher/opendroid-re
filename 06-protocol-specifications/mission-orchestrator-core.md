# mission-orchestrator-core: Mission System RE

## Overview (3-5 sentences)

The mission orchestrator core is built around two primary classes: `toA` (MissionRunner, in module 2542.js) which drives the sequential feature execution loop, and `PMH` (MissionFileService, in module 2201.js) which manages all mission directory state persistence. The runner operates as a stateful polling loop: it reads `state.json` and `features.json` from a `missionDir`, spawns worker sessions via the daemon's `spawnWorkerSession` RPC, then waits for completion by monitoring the progress log. Workers are dispatched one at a time in array order: the first pending feature in `features.json` always runs next. When a milestone's implementation features complete, the runner auto-injects scrutiny and user-testing validation features at the top of the queue.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 2542.js | 19.0 KB | **MissionRunner class** (`toA`) | Core run loop, worker spawn/resume/wait, milestone validation injection |
| 2201.js | 13.6 KB | **MissionFileService** (`PMH`) | missionDir resolution, state.json/features.json/progress_log.jsonl CRUD, handoff persistence |
| 2541.js | ~4 KB | **Worker prompt builder** (`GNI`) | Constructs worker bootstrap message with feature assignment, skill names, session rules |
| 3399.js | 28.6 KB | **Agent/LLM integration** | Orchestrator reminder injection, mission context injection into system prompt, tool execution |
| 1185.js | 42.3 KB | **SessionService** (`OGH`) | Session CRUD, orchestrator upgrade/downgrade, `spawnWorkerSession` routing |
| 3978.js | 12.7 KB | **Mission UI component** (`baD`) | React TUI rendering for mission status, feature progress, worker timeline |
| 2175.js | 135.8 KB | **Prompt templates & skill definitions** | Full orchestrator/worker/validator system prompts, skill templates, mission artifact specs |
| 0201.js | ~0.2 KB | **OpenDroid home resolver** (`C$`) | `OPENDROID_HOME_OVERRIDE` env var or `os.homedir()` |

## Architecture / Flow

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Orchestrator (AGI mode)                │
│  SessionService.upgradeToOrchestratorSession()           │
│  Calls start_mission_run tool → MissionRunner.start()    │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│              MissionRunner (toA): 2542.js               │
│                                                          │
│  start(abortSignal, resumeWorkerSessionId?)              │
│    → cleanupOrphanedWorker()                             │
│    → runLoop(resumeWorkerSessionId?)                     │
│                                                          │
│  runLoop:                                                │
│    while(isRunning):                                     │
│      1. readState() → check mission state                │
│      2. getNextPendingFeature() → first pending feature  │
│      3. spawnWorker() → create worker session via daemon │
│      4. waitForWorkerCompletion() → poll progress log    │
│      5. checkMilestoneCompletionAndInjectValidation()    │
│      6. Repeat or break (completed/paused/orchestrator)  │
└──────────┬──────────────────────────┬────────────────────┘
           │                          │
           ▼                          ▼
┌─────────────────────┐   ┌─────────────────────────────┐
│ MissionFileService   │   │ Daemon RPC (zX)             │
│ (PMH): 2201.js     │   │ - spawnWorkerSession()      │
│ - state.json        │   │ - addUserMessage()          │
│ - features.json     │   │ - interruptSession()        │
│ - progress_log.jsonl│   │ - loadSession()             │
│ - handoffs.jsonl    │   │ - subscribeToSessionNotifs()│
│ - handoffs/*.json   │   └─────────────────────────────┘
└─────────────────────┘
```

### start_mission_run Flow

1. **Orchestrator calls `start_mission_run` tool** (blocking call: tool stays open)
2. **MissionRunner.start()** is invoked with an optional `resumeWorkerSessionId`
3. If not resuming: **cleanupOrphanedWorker()**: resets any stale worker session in state.json
4. **SIGINT handler** registered for pause support
5. Enters **runLoop()**: the main event loop

### Main Event Loop (`runLoop`)

The loop operates as a state machine driven by `state.json`:

```
State: running
  → getNextPendingFeature() → found? → spawnWorker() → waitForWorkerCompletion()
  → no pending features → checkAllMilestonesForValidation() → inject validation features → continue
  → still no features → areAllFeaturesCompleted()?
     → yes → state = "completed", break
     → no → state = "orchestrator_turn", break

State: completed
  → check for pending features (stale state?) → resume as "running"
  → areAllFeaturesCompleted()? → if not, resume as "running"

State: paused → break (exit loop)

State: orchestrator_turn → break (exit loop, return control to orchestrator)

Any other state → break with warning
```

### Feature Sequencing Logic

- **Array order execution**: `getNextPendingFeature()` (2201.js:336) returns the **first** feature with `status === "pending"`: features are executed strictly in the order they appear in `features.json`
- **Validation injection**: When a milestone's implementation features are all done, `checkMilestoneCompletionAndInjectValidation()` inserts scrutiny and user-testing features at the **top** of features.json via `insertFeatureAtTop()`, so they run before any remaining features
- **Completed/cancelled features** are treated as done: `areAllFeaturesCompleted()` checks for `completed || cancelled`
- Workers that fail or need orchestrator attention set state to `orchestrator_turn`, breaking the loop

### Orchestrator Dispatch to Workers

1. **getNextPendingFeature()** retrieves the next feature object from features.json
2. **Model selection**: Validation workers (scrutiny/user-testing) use `getMissionValidationWorkerModel()`, implementation workers use `getMissionWorkerModel()`
3. **Daemon call**: `zX().spawnWorkerSession({ cwd, baseSessionId, modelId, interactionMode: "auto", autonomyLevel: "high", reasoningEffort, inactivityTimeoutMs })`: this creates a new session in the daemon
4. **Feature status update**: feature is set to `in_progress` with `currentWorkerSessionId`
5. **Bootstrap message**: `GNI(missionDir, feature, workerSessionId)` constructs the initial user message containing feature JSON, skill invocation instructions, and session rules
6. **addUserMessage**: The bootstrap prompt is sent to the worker session
7. **waitForWorkerCompletion**: Polls progress log every 1000ms for `worker_completed` entry

### missionDir Resolution

```
missionDir = path.join(getOpenDroidHome(), "missions", baseSessionId)
```

Where:
- `getOpenDroidHome()` = `C$()` (0201.js:8) = `process.env.OPENDROID_HOME_OVERRIDE || os.homedir()`
- On default Windows: `C:\Users\{user}\.opendroid\missions\{baseSessionId}\`

**missionDir contains:**
| File | Purpose |
|------|---------|
| `state.json` | Mission-level state (state, currentFeatureId, workerSessionIds, etc.) |
| `features.json` | Feature array with status, skillName, milestone, fulfills |
| `progress_log.jsonl` | Appended event log (worker_started, worker_completed, etc.) |
| `handoffs.jsonl` | Appended handoff entries |
| `handoffs/*.json` | Individual worker handoff files |
| `worker-transcripts.jsonl` | Worker transcript skeletons |
| `agent-guidelines.md` | Worker/orchestrator guidance |
| `mission.md` | Mission proposal document |
| `working_directory.txt` | Repository path |
| `model-settings.json` | Per-mission model overrides |

## Key Findings

### 1. MissionRunner is a Singleton Loop, Not an Event Emitter

The MissionRunner (`toA`, 2542.js) is a synchronous-style polling loop, not event-driven. It polls `state.json` and `progress_log.jsonl` on each iteration with a 1000ms sleep between polls (`_Uf(1000, this.abortSignal)`). This design is simple and robust: no complex event subscription needed: but means the runner process must stay alive for the entire mission duration.

### 2. Worker Sessions are Daemon-Managed RPC Calls

Workers are NOT child processes or threads. They are **daemon sessions** created via `spawnWorkerSession` RPC. The runner monitors them by:
- Polling `progress_log.jsonl` for `worker_completed` events
- Subscribing to daemon session notifications (`session_inactivity`, `ProcessExitError`)
- Checking daemon reachability every 5 seconds via TCP port probe

If the daemon becomes unreachable for 3 consecutive checks (15 seconds), the worker is marked failed and the mission returns to orchestrator.

### 3. Milestone Validation Auto-Injection is Idempotent

`checkMilestoneCompletionAndInjectValidation()` (2542.js:275) uses `milestonesWithValidationPlanned` in `state.json` to track which milestones have already had validation injected. This prevents duplicate injection on resume. The injection inserts scrutiny BEFORE user-testing (scrutiny goes first via `insertFeatureAtTop`, then user-testing goes first via another `insertFeatureAtTop`), so user-testing runs after scrutiny.

### 4. Orchestrator Upgrade Requires AGI Mode

SessionService (1185.js:1219) enforces that only sessions in `agi` interaction mode can be upgraded to `orchestrator` type. The upgrade changes the session's `decompSessionType` to `"orchestrator"`, which triggers:
- Mission context injection into system prompts (3399.js:417)
- Orchestrator reminder messages ("Do NOT implement code yourself")
- Access to mission-specific tools (start_mission_run, EndFeatureRun)

### 5. Graceful Pause via SIGINT + AbortSignal

The runner supports pausing via:
- `SIGINT` signal → calls `pause()` which interrupts the current worker session, logs `worker_paused`, and sets state to `paused`
- `abortSignal` from the tool call → same effect
- On resume, the orchestrator is told which `workerSessionId` to resume, and `start_mission_run` is called again with that ID

### 6. Worker Inactivity Timeout Configurable via Environment

`FUf()` (2542.js:27) reads `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS` from environment, defaulting to `GUf` (600000ms = 10 minutes). This timeout is passed to `spawnWorkerSession` and the daemon enforces it: if a worker is inactive for this duration, the daemon cleans it up and emits `session_inactivity`, which the runner handles as a worker failure.

## Code Examples

### MissionRunner Class and Entry Point (2542.js:64-97)

```js
// Module 2542.js:64-97
class toA {
  baseSessionId;
  missionFileService;
  isRunning = false;
  abortSignal = null;
  constructor(H) {
    this.baseSessionId = H;
    this.missionFileService = Hf(H); // PMH: MissionFileService
  }
  async start(H, A) {
    if (this.isRunning) {
      BH("[MissionRunner] Already running, ignoring start request");
      return;
    }
    if (H?.aborted) {
      BH("[MissionRunner] Abort signal already aborted, not starting");
      return;
    }
    this.isRunning = true;
    this.abortSignal = H ?? null;
    if (!A) await this.cleanupOrphanedWorker();
    let L = async () => {
      BH("[MissionRunner] Received interrupt signal, pausing...");
      await this.pause();
    };
    process.on("SIGINT", L);
    try {
      await this.runLoop(A);
    } finally {
      process.off("SIGINT", L);
      this.isRunning = false;
      this.abortSignal = null;
    }
  }
```

### Main Event Loop (2542.js:176-302)

```js
// Module 2542.js:176-220 (condensed)
async runLoop(H) {
  let A = H; // resumeWorkerSessionId
  while (this.isRunning) {
    if (this.abortSignal?.aborted) {
      await this.pause();
      break;
    }
    let L = await this.missionFileService.readState();
    if (!L) { BH("[MissionRunner] No state file found, stopping"); break; }
    
    if (L.state === "completed") {
      let E = await this.missionFileService.getNextPendingFeature();
      if (E) {
        await this.missionFileService.updateState({ state: "running" });
        continue; // Resume: pending features found
      }
      if (!(await this.missionFileService.areAllFeaturesCompleted())) {
        await this.missionFileService.updateState({ state: "running" });
        continue;
      }
      break; // Truly completed
    }
    if (L.state === "paused") break;
    if (L.state === "orchestrator_turn") break;
    if (L.state !== "running") break; // Unexpected state
    
    // Resume handling...
    // Spawn next worker...
    // Check milestone completion...
  }
}
```

### Feature Sequencing (2201.js:336-342)

```js
// Module 2201.js:336-342
async getNextPendingFeature() {
  let H = await this.readFeatures();
  if (!H) return null;
  return H.features.find((A) => A.status === "pending") ?? null;
}
```

### missionDir Resolution (2201.js:32-37)

```js
// Module 2201.js:32-37
class PMH {
  baseSessionId;
  missionDir;
  constructor(H) {
    this.baseSessionId = H;
    this.missionDir = P4.join(kG(), "missions", this.baseSessionId);
  }
}
```

### Worker Bootstrap Prompt (2541.js:55-168)

```js
// Module 2541.js:55-168 (condensed)
function GNI(H, A, L) {
  // H = missionDir, A = feature object, L = workerSessionId
  let $ = HMH.includes(A.skillName); // is validation worker?
  let E = $ 
    ? "1. Invoke the '${A.skillName}' skill\n2. Call EndFeatureRun when done"
    : "1. First, invoke the 'mission-worker-base' skill\n" +
      "2. Then, invoke the '${A.skillName}' skill\n" +
      "3. Call EndFeatureRun when done";
  
  return `<system-reminder>
You are a worker assigned to implement feature "${A.id}".
## Worker Session
Your worker session id is: ${L ?? "(unknown)"}
## Mission Files
The following files are in ${H}:
- mission.md, validation-contract.md, validation-state.json, features.json, agent-guidelines.md
${E}
REMEMBER TO CALL ENDFEATURERUN WHEN YOU ARE DONE.
</system-reminder>
## Your Assigned Feature
${JSON.stringify({ id: A.id, description: A.description, skillName: A.skillName, 
  milestone: A.milestone, preconditions: A.preconditions, 
  expectedBehavior: A.expectedBehavior, verificationSteps: A.verificationSteps }, null, 2)}`;
}
```

### Milestone Validation Injection (2542.js:275-367)

```js
// Module 2542.js:275-367 (condensed)
async checkMilestoneCompletionAndInjectValidation(H) {
  let A = await this.missionFileService.getFeature(H);
  if (!A?.milestone) return;
  let L = A.milestone;
  if (!(await this.missionFileService.isMilestoneImplementationComplete(L))) return;
  if (await this.missionFileService.hasValidationPlannerRun(L)) return;
  
  // Create validation feature objects
  let E = { id: `${efH}-${L}`, skillName: efH, /* user-testing-validator */ };
  let f = { id: `${sfH}-${L}`, skillName: sfH, /* scrutiny-validator */ };
  
  // Insert at top of features.json (user-testing first, then scrutiny on top)
  if (!W) await this.missionFileService.insertFeatureAtTop(E);
  if (!B) await this.missionFileService.insertFeatureAtTop(M);
  await this.missionFileService.markMilestoneValidationPlanned(L);
}
```

## Integration Points

### 01-terminal-ui (TUI)
- **3978.js** contains the React-based mission status TUI component (`baD`) that renders mission state, feature progress bars, worker timelines, and event logs
- Mission runner emits `project-notification` events: `mission_state_changed`, `mission_features_changed`, `mission_progress_entry`, `mission_heartbeat`, `mission_worker_completed`, `milestone_validation_triggered`
- TUI subscribes to these events for real-time display updates

### 03-tool-agent-system (Tool)
- `start_mission_run` is exposed as a tool that the orchestrator LLM calls: the tool implementation creates an abort controller and calls `MissionRunner.start(abortSignal)`
- `EndFeatureRun` is another tool that workers call to submit handoffs
- The daemon's `zX()` API provides the RPC layer for session management

### 04-desktop-gui (GUI)
- Mission status could be rendered in the Electron app using the same notification events
- The React component in 3978.js may be reusable for GUI rendering

### 05-infrastructure (Infra)
- `kG()` → `.opendroid/` directory is the base for mission storage
- `C$()` → `OPENDROID_HOME_OVERRIDE` env var or `os.homedir()` determines the root path
- Daemon reachability check uses TCP port probe via `UUf.createConnection`
- Worker sessions stored at `{opendroidHome}/sessions/{cwdHash}/{sessionId}.jsonl`

## Implementation Notes

### 1. MissionRunner Core
**Priority: P0: Required for any mission execution**

Implement a `MissionRunner` class with:
- `start(abortSignal, resumeWorkerSessionId?)` entry point
- `runLoop()` state machine with states: `running`, `paused`, `completed`, `orchestrator_turn`
- Sequential feature execution via `getNextPendingFeature()`
- Worker spawn via RPC/IPC to the agent runtime
- 1-second polling interval for worker completion detection
- SIGINT/abort handling for graceful pause

### 2. MissionFileService
**Priority: P0: Required for state persistence**

Implement file-based state management:
- `state.json`: Mission-level state with `state`, `currentFeatureId`, `currentWorkerSessionId`, `workerSessionIds[]`, `completedFeatures`, `totalFeatures`
- `features.json`: Feature array with `status`, `skillName`, `milestone`, `fulfills`, `workerSessionIds`
- `progress_log.jsonl`: Append-only event log for audit trail
- File paths resolved relative to `{opendroidHome}/missions/{baseSessionId}/`
- All mutations emit notification events for UI synchronization

### 3. Worker Dispatch
**Priority: P0: Required for multi-agent execution**

Key design decisions to replicate:
- Workers are autonomous agent sessions, not child processes
- Each worker gets a bootstrap message with: feature JSON, skill invocation order, session rules
- `interactionMode: "auto"`, `autonomyLevel: "high"` for maximum independence
- Reasoning effort set to `"high"` if model supports it
- Inactivity timeout defaulting to 10 minutes

### 4. Milestone Validation Injection
**Priority: P1: Required for quality gates**

- Track `milestonesWithValidationPlanned[]` in state.json for idempotency
- Insert scrutiny before user-testing (both at top of features.json)
- Skip individual validators via `model-settings.json` or global settings
- Log `milestone_validation_triggered` events

### 5. Daemon Reachability
**Priority: P1: Required for robustness**

- Periodic TCP health check to daemon port (every 5s)
- 3 consecutive failures (15s) → worker marked failed, mission pauses
- `session_inactivity` and `ProcessExitError` notifications from daemon

## Known Gaps

- **Module 3978.js full analysis**: Only read the first 200 lines (TUI component). The full component has ~1045 lines with detailed rendering logic. 01-terminal-ui territory.
- **Module 1185.js session lifecycle**: Only read the orchestrator upgrade/downgrade and spawn functions. The full session management (1823 lines) includes session persistence, token tracking, and cloud sync: relevant for 05-infrastructure.
- **Module 3399.js agent streaming**: Only analyzed the orchestrator context injection portion. The full module (1303 lines) contains LLM streaming, tool execution, and message formatting: relevant for 03-tool-agent-system.
- **Constants `GUf` (inactivity timeout)** and **`QUf` (spawn timeout)** were not fully traced to their definitions. Line 811 of 2542.js shows `QUf` but the value was cut off.
- **Worker transcript management** (`worker-transcripts.jsonl`) was identified in PMH but the write path was not fully traced.
- **Cloud session sync** in SessionService (1185.js) was not explored.
