# Protocol: Mission Lifecycle
## Status: DEFINITIVE (cross-validated)

Cross-referenced observed protocol samples (state.json, progress_log.jsonl, worker-transcripts.jsonl) with architecture notes (mission-orchestrator-core.md, mission-parallel-runner.md, mission-retry-policy.md, mission-reports.md). All schemas and event types validated against real production data.

---

## 1. Mission State Machine

### 1.1 States

| State | Description | Observed in Runtime |
|-------|-------------|---------------------|
| `awaiting_input` | Mission proposed, waiting for user acceptance | ⚠️ Not directly observed; inferred from `mission_accepted` event preceding `mission_run_started` |
| `initializing` | Mission being set up (directory, files) | ⚠️ Not directly observed in observed protocol samples; architecture notes mention this state |
| `running` | Active execution loop: workers being spawned | Confirmed (features transition `pending` → `in_progress` → `completed`) |
| `paused` | User paused or SIGINT received | ✅ Observed: `state.json` shows `"state": "paused"` |
| `orchestrator_turn` | Control returned to orchestrator for decision | ✅ Confirmed by RE: set on failure/escalation (2542.js, 2539.js) |
| `completed` | All features done, mission finished | ✅ Confirmed by RE: all features `completed` or `cancelled` |

### 1.2 State Transitions

```
┌──────────────┐
│ awaiting_input│ ← Mission created (proposal accepted by user)
└──────┬───────┘
       │ mission_accepted event
       ▼
┌──────────────┐
│ initializing │ ← Mission directory created, files populated
└──────┬───────┘
       │ mission_run_started event
       ▼
┌──────────────┐     SIGINT / pause     ┌─────────┐
│   running    │ ──────────────────────► │ paused  │
└──┬──┬──┬─────┘                        └────┬────┘
   │  │  │                                   │ mission_resumed event
   │  │  │  ◄────────────────────────────────┘
   │  │  │
   │  │  ├─► orchestrator_turn (worker failed / issues found)
   │  │  │     └─► orchestrator decides → start_mission_run → running
   │  │  │
   │  │  └─► orchestrator_turn (validator completed with returnToOrchestrator=true)
   │  │        └─► orchestrator reviews → start_mission_run → running
   │  │
   │  └─► completed (all features completed/cancelled)
   │
   └─► paused (no pending features, not all completed: edge case)
```

### 1.3 Triggers

| From | To | Trigger | Source |
|------|----|---------|--------|
| `awaiting_input` | `initializing` | User accepts mission proposal | RE: orchestrator flow |
| `initializing` | `running` | `start_mission_run` tool invoked | RE: 2542.js `start()` |
| `running` | `paused` | SIGINT signal or abort signal | ✅ Runtime: `worker_paused` → `mission_paused` events |
| `paused` | `running` | `start_mission_run` with `resumeWorkerSessionId` | ✅ Runtime: `mission_resumed` event |
| `running` | `orchestrator_turn` | Worker failure, issues in handoff, or validator completion | RE: 2542.js, 2539.js |
| `orchestrator_turn` | `running` | Orchestrator calls `start_mission_run` again | RE: 2544.js |
| `running` | `completed` | All features `completed` or `cancelled` | RE: 2542.js `areAllFeaturesCompleted()` |
| `completed` | `running` | Stale state detected (pending features found) | RE: 2542.js runLoop |

### 1.4 state.json Schema

```json
{
  "missionId": "mis_0fe1958c",
  "state": "paused",
  "workingDirectory": "<workspace>\\SampleProject",
  "createdAt": "2026-04-20T05:09:49.607Z",
  "updatedAt": "2026-04-20T22:14:09.898Z",
  "lastReviewedHandoffCount": 11
}
```

### 1.5 state.json Field Documentation

| Field | Type | Required | Source | Notes |
|-------|------|----------|--------|-------|
| `missionId` | string | Yes | Runtime | Short ID with `mis_` prefix. NOT a UUID. ⚠️ RE expected UUID format |
| `state` | string enum | Yes | Both | One of: `awaiting_input`, `initializing`, `running`, `paused`, `orchestrator_turn`, `completed` |
| `workingDirectory` | string | Yes | Both | Absolute path to project directory |
| `createdAt` | ISO 8601 string | Yes | Both | Mission creation timestamp |
| `updatedAt` | ISO 8601 string | Yes | Both | Last state update timestamp |
| `lastReviewedHandoffCount` | integer | No | Runtime | Tracks orchestrator progress through handoffs. ⚠️ Not in architecture notes |

⚠️ **RE discrepancy**: architecture notes (mission-orchestrator-core.md) documents additional fields in state.json: `currentFeatureId`, `currentWorkerSessionId`, `workerSessionIds[]`, `completedFeatures`, `totalFeatures`, `orchestratorActedSinceReturn`, `milestonesWithValidationPlanned[]`. These fields exist in the code (2201.js) but were NOT observed in the runtime state.json. The runtime state.json appears to be a simplified version.

---

## 2. progress_log.jsonl Event Catalog

### 2.1 Event Types (Observed: 9 types)

Each line is a JSON object with `timestamp` (ISO 8601) and `type` fields.

#### 2.1.1 mission_accepted
Mission started by user. **Precedes** `mission_run_started`.

```json
{
  "timestamp": "2026-04-20T04:31:51.567Z",
  "type": "mission_accepted",
  "title": "Sample Mission"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"mission_accepted"` |
| `title` | string | Yes | Mission title from mission.md |

#### 2.1.2 mission_run_started
Orchestrator begins execution.

```json
{
  "timestamp": "2026-04-20T05:09:49.613Z",
  "type": "mission_run_started",
  "message": "Starting sample mission execution..."
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"mission_run_started"` |
| `message` | string | Yes | Descriptive message |

#### 2.1.3 worker_selected_feature
Orchestrator picked a feature for execution.

```json
{
  "timestamp": "2026-04-20T05:09:52.238Z",
  "type": "worker_selected_feature",
  "workerSessionId": "<run-id>",
  "featureId": "project-scaffold"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"worker_selected_feature"` |
| `workerSessionId` | UUID string | Yes | Session ID assigned to worker |
| `featureId` | string | Yes | Feature being assigned |

#### 2.1.4 worker_started
Worker session begins execution.

```json
{
  "timestamp": "2026-04-20T05:09:52.254Z",
  "type": "worker_started",
  "workerSessionId": "<run-id>",
  "spawnId": "worker_32eaaf3d",
  "featureId": "project-scaffold"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"worker_started"` |
| `workerSessionId` | UUID string | Yes | Worker session ID |
| `spawnId` | string | Yes | Internal spawn identifier (e.g., `"worker_32eaaf3d"`). ⚠️ Not in architecture notes |
| `featureId` | string | Yes | Feature being executed |

#### 2.1.5 worker_completed
Worker finished execution. Contains full embedded handoff object.

```json
{
  "timestamp": "2026-04-20T05:22:02.013Z",
  "type": "worker_completed",
  "workerSessionId": "<run-id>",
  "featureId": "project-scaffold",
  "successState": "success",
  "returnToOrchestrator": false,
  "commitId": "<handoff-hash>",
  "exitCode": 0,
  "validatorsPassed": true,
  "handoff": {
    "timestamp": "2026-04-20T05:22:02.013Z",
    "workerSessionId": "<run-id>",
    "featureId": "project-scaffold",
    "milestone": "foundation",
    "commitId": "6692b644...",
    "successState": "success",
    "returnToOrchestrator": false,
    "handoff": { "...": "...full handoff object..." }
  }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"worker_completed"` |
| `workerSessionId` | UUID string | Yes | Worker session ID |
| `featureId` | string | Yes | Feature that was executed |
| `successState` | string enum | Yes | `"success"` or `"failure"` |
| `returnToOrchestrator` | boolean | Yes | Whether control returns to orchestrator |
| `commitId` | string | Yes | Git commit hash (full or short) |
| `exitCode` | integer | Yes | Worker process exit code |
| `validatorsPassed` | boolean | Yes | Whether validators passed. ⚠️ Not in architecture notes |
| `handoff` | object | Yes | Full embedded handoff JSON (see protocol-handoff.md) |

#### 2.1.6 worker_paused
Worker was paused (user interrupted).

```json
{
  "timestamp": "2026-04-20T14:52:33.197Z",
  "type": "worker_paused",
  "workerSessionId": "7328fc5c-...",
  "featureId": "zustand-stores"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"worker_paused"` |
| `workerSessionId` | UUID string | Yes | Worker session ID |
| `featureId` | string | Yes | Feature that was being executed |

#### 2.1.7 mission_paused
Mission paused by user. Always follows `worker_paused`.

```json
{
  "timestamp": "2026-04-20T14:52:33.225Z",
  "type": "mission_paused"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"mission_paused"` |

#### 2.1.8 mission_resumed
Mission resumed by user.

```json
{
  "timestamp": "2026-04-20T14:52:58.649Z",
  "type": "mission_resumed",
  "resumeWorkerSessionId": "7328fc5c-..."
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"mission_resumed"` |
| `resumeWorkerSessionId` | UUID string | Yes | Worker session to resume |

#### 2.1.9 milestone_validation_triggered
Milestone validation begins.

```json
{
  "timestamp": "2026-04-20T14:58:46.438Z",
  "type": "milestone_validation_triggered",
  "milestone": "foundation",
  "featureId": "scrutiny-validator-foundation"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `timestamp` | ISO 8601 string | Yes | Event timestamp |
| `type` | string | Yes | `"milestone_validation_triggered"` |
| `milestone` | string | Yes | Milestone being validated |
| `featureId` | string | Yes | Validator feature ID (e.g. `"scrutiny-validator-foundation"`) |

### 2.2 Additional Event Types (analysis-only, not observed in runtime)

| Event Type | Source | Notes |
|------------|--------|-------|
| `worker_failed` | RE: 2542.js, 0082.js | Logged when worker crashes or daemon becomes unreachable. Fields: `workerSessionId`, `featureId`, `reason`, `spawnId` |

---

## 3. Mission Creation Flow

### 3.1 Directory Structure

```
~/.opendroid/missions/{mission-uuid}/
├── state.json                 # Mission state (paused/running/completed)
├── features.json              # Feature definitions (the plan)
├── validation-state.json      # Assertion tracking
├── mission.md                 # Mission description (Markdown)
├── agent-guidelines.md                  # Agent guidance / coding conventions
├── validation-contract.md     # Full validation contract (Markdown)
├── model-settings.json        # Per-mission model config
├── runtime-custom-models.json # Custom model definitions snapshot
├── working_directory.txt      # Working directory for the mission
├── progress_log.jsonl         # Timeline of all events (JSONL)
├── worker-transcripts.jsonl   # Full worker transcripts (JSONL)
├── handoffs/                  # Worker handoff JSON files
├── contract-work/             # Contract surface descriptions
└── evidence/                  # Test evidence (screenshots, etc.)
```

### 3.2 Creation Sequence

1. **User proposes mission** via chat → `awaiting_input` state
2. **User accepts** → `mission_accepted` event logged to progress_log.jsonl
3. **System creates** mission directory under `~/.opendroid/missions/{uuid}/`
4. **System populates** initial files: `mission.md`, `agent-guidelines.md`, `features.json`, `validation-state.json`, `validation-contract.md`, `working_directory.txt`, `state.json`
5. **Orchestrator session** created/upgraded with `tags: [{name: "mission-session", metadata: {role: "orchestrator", missionId: "..."}}]`
6. **state.json** initialized with `state: "initializing"`, `createdAt`, `workingDirectory`
7. **mission_run_started** event logged when `start_mission_run` tool invoked
8. **State** → `"running"`

---

## 4. Worker Spawn Protocol

### 4.1 Spawn Sequence

```
MissionRunner.runLoop()
    │
    ├─► getNextPendingFeature(): first feature with status === "pending"
    │
    ├─► worker_selected_feature event logged (assigns workerSessionId)
    │
    ├─► spawnWorkerSession() RPC to daemon
    │     Parameters:
    │     - cwd: working directory
    │     - baseSessionId: mission UUID
    │     - modelId: from model-settings.json
    │     - interactionMode: "auto"
    │     - autonomyLevel: "high"
    │     - reasoningEffort: from model-settings.json
    │     - inactivityTimeoutMs: 600000 (10 min default)
    │
    ├─► Feature status → "in_progress", currentWorkerSessionId set
    │
    ├─► worker_started event logged (includes spawnId)
    │
    ├─► Bootstrap message sent to worker session:
    │     "You are a worker assigned to implement feature {id}..."
    │     + feature JSON + skill invocation instructions
    │
    └─► waitForWorkerCompletion(): polls progress_log.jsonl every 1000ms
```

### 4.2 Session Hierarchy

Worker sessions are linked to the orchestrator via `callingSessionId`:

```
Orchestrator session (interactionMode: "agi")
  │
  ├─► Worker session 1 (callingSessionId: orchestrator-session-id)
  │     └─► Subagent sessions (callingSessionId: worker-session-id)
  │
  ├─► Worker session 2 (callingSessionId: orchestrator-session-id)
  │
  └─► Validator session (callingSessionId: orchestrator-session-id)
        └─► Scrutiny subagent (callingSessionId: validator-session-id)
```

### 4.3 callingSessionId / callingToolUseId

These fields link subagent/worker sessions to their parent:

| Field | Location | Type | Description |
|-------|----------|------|-------------|
| `callingSessionId` | sessions-index.json entry | string (optional) | Parent session that spawned this one |
| `callingToolUseId` | sessions-index.json entry | string (optional) | Tool call ID that spawned this session |

Example from sessions-index.json:
```json
{
  "sessionId": "14724ccb-...",
  "callingSessionId": "orchestrator-session-uuid",
  "callingToolUseId": "call_function_oplxzp4kozqm_1",
  "tags": [{"name": "exec"}, {"name": "subagent"}]
}
```

### 4.4 Worker Bootstrap Message

The worker receives an initial user message constructed by `GNI()` (2541.js):

```
<system-reminder>
You are a worker assigned to implement feature "{featureId}".
## Worker Session
Your worker session id is: {workerSessionId}
## Mission Files
The following files are in {missionDir}:
- mission.md, validation-contract.md, validation-state.json, features.json, agent-guidelines.md
1. First, invoke the 'mission-worker-base' skill
2. Then, invoke the '{skillName}' skill
3. Call EndFeatureRun when done
REMEMBER TO CALL ENDFEATURERUN WHEN YOU ARE DONE.
</system-reminder>
## Your Assigned Feature
{feature JSON with id, description, skillName, milestone, preconditions, expectedBehavior, verificationSteps}
```

For validation workers, the instructions differ:
```
1. Invoke the '{skillName}' skill
2. Call EndFeatureRun when done
```
(Skips `mission-worker-base`: validators have their own startup procedure.)

### 4.5 Model Selection

| Worker Type | Model Source | Static Analysis Source |
|-------------|-------------|-----------|
| Implementation workers | `model-settings.json` → `workerModel` | 2542.js `getMissionWorkerModel()` |
| Validation workers | `model-settings.json` → `validationWorkerModel` | 2542.js `getMissionValidationWorkerModel()` |

model-settings.json example:
```json
{
  "workerModel": "custom:MiniMax-M2.7-0",
  "workerReasoningEffort": "high",
  "validationWorkerModel": "custom:built-in model-[Z.AI]-0",
  "validationWorkerReasoningEffort": "none"
}
```

---

## 5. Pause/Resume Mechanics

### 5.1 Pause Flow

```
User presses pause (or SIGINT)
    │
    ├─► MissionRunner.pause() called
    │     ├─► interruptSession(workerSessionId) via daemon RPC
    │     ├─► worker_paused event logged to progress_log.jsonl
    │     ├─► mission_paused event logged
    │     ├─► state.json → state: "paused"
    │     └─► Feature status remains "in_progress" (NOT reset to pending)
    │
    └─► runLoop() breaks
```

⚠️ **Key difference from failure**: On pause, feature status stays `in_progress`. On failure, it resets to `pending`.

### 5.2 Resume Flow

```
User presses resume
    │
    ├─► start_mission_run tool called with resumeWorkerSessionId
    │
    ├─► MissionRunner.start(abortSignal, resumeWorkerSessionId)
    │     ├─► Does NOT call cleanupOrphanedWorker() (resume path)
    │     └─► runLoop(resumeWorkerSessionId)
    │
    ├─► mission_resumed event logged with resumeWorkerSessionId
    │
    └─► Worker continues from where it left off
```

### 5.3 Orchestrator Turn → Resume

When the mission enters `orchestrator_turn`:

1. Runner breaks out of loop, returns to `start_mission_run` tool
2. Tool constructs system message for orchestrator based on failure type
3. Orchestrator reviews handoff/progress, decides action
4. Orchestrator calls `start_mission_run` again
5. Runner resumes from top of runLoop: picks up next pending feature

---

## 6. Milestone Validation Flow

### 6.1 Two-Phase Validation

```
All implementation features in milestone completed
    │
    ├─► checkMilestoneCompletionAndInjectValidation() triggered
    │     ├─► Checks isMilestoneImplementationComplete(milestone)
    │     ├─► Checks hasValidationPlannerRun(milestone): idempotent guard
    │     │
    │     ├─► Creates scrutiny-validator-{milestone} feature
    │     │     skillName: "scrutiny-validator"
    │     │     returnToOrchestrator: true (always)
    │     │
    │     ├─► Creates user-testing-validator-{milestone} feature
    │     │     skillName: "user-testing-validator"
    │     │     returnToOrchestrator: true (always)
    │     │
    │     ├─► Inserts at TOP of features.json:
    │     │     user-testing inserted first, then scrutiny inserted on top
    │     │     → order: [scrutiny-validator, user-testing-validator, ...remaining]
    │     │
    │     └─► milestone_validation_triggered event logged
    │
    ├─► scrutiny-validator runs first:
    │     ├─► Runs validators (test, typecheck, lint): hard gate
    │     ├─► Spawns parallel scrutiny-feature-reviewer subagents
    │     ├─► Writes synthesis.json to .opendroid/validation/{milestone}/scrutiny/
    │     └─► returnToOrchestrator: true → orchestrator_turn
    │
    ├─► Orchestrator reviews scrutiny results
    │     ├─► If FAIL: creates fix features → start_mission_run → re-validate
    │     └─► If PASS: start_mission_run → user-testing-validator runs
    │
    └─► user-testing-validator runs:
          ├─► Determines testable assertions from features[].fulfills
          ├─► Spawns parallel user-testing-flow-validator subagents
          ├─► Writes synthesis.json to .opendroid/validation/{milestone}/user-testing/
          └─► returnToOrchestrator: true → orchestrator_turn
```

### 6.2 Idempotency

`milestonesWithValidationPlanned[]` in internal state tracks which milestones have had validation injected. This prevents duplicate injection on resume (2542.js:275).

### 6.3 Validation Feature Schema

Validator features follow the same features.json schema but with specific patterns:

```json
{
  "id": "scrutiny-validator-foundation",
  "description": "...",
  "skillName": "scrutiny-validator",
  "preconditions": [],
  "expectedBehavior": [],
  "verificationSteps": [],
  "fulfills": [],
  "milestone": "foundation",
  "status": "pending",
  "workerSessionIds": [],
  "currentWorkerSessionId": null,
  "completedWorkerSessionId": null
}
```

Key differences from implementation features:
- `verificationSteps: []`: validators use internal skill logic, not external verification
- `fulfills: []`: validators don't map to validation assertions
- `skillName` is `"scrutiny-validator"` or `"user-testing-validator"`

---

## 7. Worker Spawn and Preemption Mechanics

### 7.1 Parallel Execution Model

| Level | Execution | Mechanism |
|-------|-----------|-----------|
| Feature-level | **Sequential** | One feature at a time in array order |
| Tool-level | **Parallel** | Multiple `Task` tool calls via `Promise.all` |

Feature-level parallelism does NOT exist. The mission runner executes workers one at a time (1236.js:12):

> "The tool call remains open while the mission runner executes workers sequentially."

### 7.2 Parallel Subagent Spawning

Within a worker or orchestrator turn, multiple `Task` tool calls can be issued in a single assistant message. The system executes them concurrently via `Promise.all`:

```js
// 2362.js:448-484 (condensed)
return {
  role: "user",
  content: await Promise.all(
    L.map(async (I) => {
      let D = H.tools.find(E => E.name === I.name);
      let f = await D.run(E);
      return { type: "tool_result", tool_use_id: I.id, content: f };
    })
  )
};
```

Each parallel subagent:
- Runs as independent OS process (`child_process.spawn`)
- Gets `--calling-session-id` and `--calling-tool-use-id` for traceability
- Gets `--depth` incremented from parent
- Has isolated context window

### 7.3 Model Capability Gating

Parallel tool calls are gated by `parallelToolCalls` flag per model (0257.js):

| Model | `parallelToolCalls` |
|-------|---------------------|
| gpt-5-codex | `false` |
| gpt-5.1-codex | `false` |
| gpt-5.2-codex | `true` |
| gpt-5.3-codex | `true` |

When `false`, the LLM emits tool calls sequentially. When `true`, it may emit multiple in one response.

### 7.4 Preemption

Preemption occurs via:
1. **User pause** → SIGINT → `pause()` → interrupt worker → state = `paused`
2. **Worker failure** → feature reset to `pending` → state = `orchestrator_turn`
3. **Daemon unreachable** → 3 consecutive failures (15s) → worker failed → state = `orchestrator_turn`
4. **Worker inactivity** → timeout (default 10 min) → daemon cleans up → `session_inactivity` event

---

## 8. Error Recovery and Retry Policy

### 8.1 Core Principle: Delegated Retry

The mission runner does NOT automatically retry failed features. Every failure escalates to the orchestrator LLM for decision.

### 8.2 Failure Detection

| Detection Method | Interval | Failure Threshold | Response |
|-----------------|----------|-------------------|----------|
| Progress log polling | 1000ms | `worker_completed` event match | Process handoff |
| Daemon session notifications | Event-driven | `session_inactivity` or `ProcessExitError` | Feature re-queue |
| Daemon health check (TCP) | 5000ms | 3 consecutive failures (15s) | Worker failed, mission abort |

### 8.3 Feature Re-Queue (S9H Function)

On worker crash/daemon failure, the `S9H()` function resets the feature:

```js
// 2542.js:7-18
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

Transition: `in_progress` → `pending` (makes feature eligible for re-execution).

### 8.4 Orchestrator Decision Matrix

After failure escalation, the orchestrator receives a system message and decides:

| Failure Type | System Message Instruction | Orchestrator Options |
|-------------|---------------------------|---------------------|
| opendroidd error | "call start_mission_run once to retry: do NOT attempt further retries" | Single retry, then tell user to restart |
| General worker failure | "investigate what happened, then call start_mission_run again" | Re-run same feature or create fix |
| Handoff has issues | "create new features or update existing feature descriptions" | Create fix features, then re-run |
| Validator failure | "Create fix features, then call start_mission_run" | Fix features → re-validate |

### 8.5 EndFeatureRun Escalation Triggers

The EndFeatureRun tool (2539.js) determines escalation based on handoff:

```js
let hasIssues = handoff?.discoveredIssues?.length > 0;
let hasUndone = whatWasLeftUndone?.trim() !== "" && whatWasLeftUndone?.toLowerCase() !== "none";
let shouldEscalate = returnToOrchestrator || hasIssues || hasUndone;
```

Three triggers:
1. Worker explicitly sets `returnToOrchestrator: true`
2. Handoff contains non-empty `discoveredIssues` array
3. Handoff contains non-trivial `whatWasLeftUndone`

### 8.6 Validation Retry

Validators follow the same "reset to pending, re-run" pattern. The orchestrator prompt states:

> "When a validator fails, it goes back to pending. Create fix features, then call start_mission_run: the validator will re-run and only re-validate what failed."

Both validators support **incremental re-runs**:
- Scrutiny: reads prior `synthesis.json`, only reviews fix features + original failures
- User-testing: reads prior `synthesis.json`, only tests failed/blocked assertions + new assertions
- `round` field incremented on each re-run

### 8.7 Key Constants

| Constant | Value | Source | Purpose |
|----------|-------|--------|---------|
| Worker inactivity timeout | 600000ms (10 min) | 2542.js `GUf` | Default stall detection |
| Configurable via | `Msample itemION_WORKER_INACTIVITY_TIMEOUT_MS` | env var | Override default |
| Daemon check interval | 5000ms | 2542.js `VUf` | Health check frequency |
| Daemon failure threshold | 3 consecutive | 2542.js `XUf` | Failures before abort |
| Progress log poll | 1000ms | 2542.js `_Uf(1000)` | Worker completion check |
| Heartbeat interval | 180000ms (3 min) | 2542.js `TUf` | Progress emission to UI |
| Opendroidd default timeout | 5000ms | 2542.js `WUf` | RPC call timeout |

---

## 9. worker-transcripts.jsonl

### 9.1 Schema

```json
{
  "workerSessionId": "<run-id>",
  "featureId": "project-scaffold",
  "milestone": "foundation",
  "skeleton": "## Tool: Skill\n{\"skill\":\"mission-worker-base\"}\n\n## Tool: Read\n{...}\n\n## Tool Result\n...\n\n## Tool: EndFeatureRun\n{...}",
  "timestamp": "2026-04-20T05:22:02.043Z"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `workerSessionId` | UUID string | Yes | Worker's session ID |
| `featureId` | string | Yes | Feature being worked on |
| `milestone` | string | Yes | Milestone name |
| `skeleton` | string | Yes | Full text transcript: NOT JSON. Uses `## Tool: ToolName` headers |
| `timestamp` | ISO 8601 string | Yes | When transcript was recorded |

The skeleton is the raw audit trail containing the COMPLETE tool call history including `EndFeatureRun` with full handoff JSON.

---

## 10. Discrepancies (Inferred vs Observed)

| Field/Behavior | Architecture Notes Says | Observed Samples Show | Resolution |
|---------------|-----------------|-------------------|------------|
| `missionId` format | UUID format (directory name) | `mis_` prefix + hex string (`mis_0fe1958c`) | Runtime is authoritative. Short ID is the actual missionId; directory UUID is the baseSessionId |
| `spawnId` field | Not documented | Present in `worker_started` events (`"worker_32eaaf3d"`) | Runtime-only field for internal worker tracking |
| `validatorsPassed` field | Not documented | Present in `worker_completed` events (`true`/`false`) | Runtime-only field indicating validator pass status |
| `lastReviewedHandoffCount` | Not documented | Present in state.json (value: 11) | Runtime-only field tracking orchestrator handoff review progress |
| state.json fields | Contains `currentFeatureId`, `currentWorkerSessionId`, `completedFeatures`, `totalFeatures`, `milestonesWithValidationPlanned[]` | Only contains `missionId`, `state`, `workingDirectory`, `createdAt`, `updatedAt`, `lastReviewedHandoffCount` | RE describes internal code state (2201.js). Runtime shows persisted subset |
| `initializing` state | Listed as valid state | Never observed in runtime | State may be transient (set then immediately changed to `running`) |
| `awaiting_input` state | Listed as valid state | Never directly observed (mission_accepted event seen but state not captured) | State exists but is transient before progress_log captures it |
| Feature-level parallelism | Explicitly sequential | Confirmed sequential: workers run one at a time | Both agree |
| Validation injection order | scrutiny first, user-testing second | RE code shows: user-testing inserted first at top, then scrutiny inserted at top → final order: scrutiny first | RE is authoritative on mechanism; end result is scrutiny runs first |
| Retry policy | No automatic retry, delegated to orchestrator | No counter-retry evidence in runtime | Both agree: delegated retry confirmed |
| worker_failed event | Documented in RE (0082.js) | Not observed in runtime (no workers crashed during observation) | Event type exists but was not triggered |

---

## 11. Open Questions

1. **state.json persistence gap**: RE shows rich internal state fields but runtime shows minimal state.json. Are the additional fields (`currentFeatureId`, `completedFeatures`, etc.) persisted elsewhere or only in memory?
2. **missionId generation**: How is the `mis_` prefix ID generated? Is it derived from the UUID or independently assigned server-side?
3. **Feature cancellation**: `cancelled` status is mentioned in RE but never observed in runtime. What triggers cancellation?
4. **Worker process exit handling**: When a worker exits with non-zero code but doesn't crash, is the behavior different from daemon unreachability?
5. **Max worker session depth**: The `--depth` parameter is incremented for subagents. What is the maximum allowed depth?

---

## 12. OpenDroid Implementation Notes

### Required Implementations

1. **MissionRunner**: Stateful polling loop with 1s interval, SIGINT/abort support
2. **MissionFileService**: File-based state CRUD for mission directory
3. **Worker spawn**: RPC call to agent runtime with bootstrap message
4. **Feature re-queue**: `S9H()` equivalent: reset `in_progress` → `pending` on failure
5. **Daemon health monitor**: TCP check every 5s, abort after 3 failures
6. **Validation injection**: Milestone completion check, idempotent injection at top of features.json
7. **Progress log**: Append-only JSONL with all event types documented above

### Key Design Decisions

- **Sequential feature execution**: no feature-level parallelism
- **Delegated retry**: no automatic retry counter, orchestrator decides
- **Pause preserves state**: paused workers resume in-place, failed workers reset to pending
- **Validation always returns to orchestrator**: even on success, validators set `returnToOrchestrator: true`
- **Inactivity timeout**: 10 minutes default, configurable via env var
