# mission-state-machine: Mission System RE

## Overview

The Mission System implements a dual-layer state machine: **mission-level state** (managed via `state.json`) and **feature-level state** (managed via `features.json`). The mission runner (`MissionRunner` class in Module 2542.js) drives transitions by spawning workers sequentially, each processing one feature. State is persisted to the filesystem under a `missionDir` (typically `~/.opendroid/missions/{sessionId}/`) and recovered on restart by reading the progress log and state files. Validation state is tracked separately in `validation-state.json` with per-assertion pass/fail/blocked status. The system emits `project-notification` events on every state change for real-time TUI updates.

## Module Map

| Module | Size (KB) | Role | Notes |
|--------|-----------|------|-------|
| 2201.js | 19.0 | **MissionFileService**: state persistence class | Core class `PMH` with readState/writeState/readFeatures/writeFeatures, getFilePaths, progress log, handoffs |
| 2542.js | 19.0 | **MissionRunner**: state machine driver | `runLoop()`, `spawnWorker()`, `resumeWorker()`, `waitForWorkerCompletion()`, pause/resume, validation injection |
| 2175.js | 135.8 | **Orchestrator prompt**: state machine documentation | Prompt templates with full features.json schema, validation-state.json schema, state transition rules |
| 3970.js | ~5 | **State watcher**: file polling for TUI | `WaD()` file stat tracking, incremental polling with cache |
| 3973.js | 6.6 | **Handoff viewer**: TUI handoff rendering | Displays `salientSummary`, `whatWasLeftUndone`, `discoveredIssues` from handoff |
| 3971.js | 7.1 | **Feature detail viewer**: TUI feature rendering | `xxM()` maps status → icon/color, `vxM()` maps status → label |
| 0919.js | 7.3 | **Daemon session schemas**: Zod validation | Session API schemas with status/state fields |
| 2857.js | 7.3 | **Worker prompt templates**: prompt assembly | TodoWrite, spec mode, tool usage guidelines |
| 3384.js | 7.3 | **Task result renderer**: TUI task display | Task completed/failed status rendering |
| 2856.js | 6.8 | **Skill execution**: tool dispatch | `NZf()` async tool execution with status handling |

## Architecture / Flow

### Mission-Level State Machine (`state.json`)

The mission-level state is stored in `state.json` at `{missionDir}/state.json`. The `state` field transitions through these values:

```
initializing → running ↔ orchestrator_turn
                 ↓              ↓
              paused         running (re-entry)
                 ↓
              completed
```

**States:**
| State | Meaning | Transitions From |
|-------|---------|-----------------|
| `initializing` | Mission dir created, artifacts being set up | (initial creation) |
| `running` | MissionRunner actively spawning/monitoring workers | `initializing`, `orchestrator_turn`, `paused` (resume) |
| `paused` | User or SIGINT paused the mission | `running` |
| `orchestrator_turn` | Runner returned control to orchestrator for decision | `running` (worker failure, handoff with `returnToOrchestrator=true`) |
| `completed` | All features completed or cancelled | `running` (all features done) |

**State transitions are driven by `MissionRunner.runLoop()` (Module 2542.js, line 160):**

1. Loop reads `state.json` via `readState()`
2. If `state === "completed"`:
   - Check for pending features (race condition guard) → if found, set `state = "running"` and continue
   - If truly done, break loop
3. If `state === "paused"` → break loop
4. If `state === "orchestrator_turn"` → break loop
5. If `state === "running"` → spawn or resume worker
6. After worker completes:
   - If success and all features done → `state = "completed"`
   - If success and features remain → continue loop (spawn next)
   - If failure or `returnToOrchestrator` → `state = "orchestrator_turn"`
   - If aborted → `state = "paused"`

**State persistence (Module 2201.js, line 73-100):**
```js
// Module 2201.js:73: writeState()
async writeState(H) {
  H.updatedAt = new Date().toISOString();
  await sM.writeFile(this.stateFilePath, JSON.stringify(H, null, 2));
}

// Module 2201.js:93-100: updateState()
async updateState(H) {
  let A = await this.readState();
  let L = { ...A, ...H };
  if ("workerStates" in L) delete L.workerStates;
  await this.writeState(L);
  if (H.state !== void 0)
    v$.emit("project-notification", {
      notification: { type: "mission_state_changed", state: L.state },
    });
  return L;
}
```

### Feature-Level State Machine (`features.json`)

Each feature in the `features` array has a `status` field:

```
pending → in_progress → completed
                     → (reset to pending on failure/crash)
pending → cancelled (terminal)
```

**Feature statuses (Module 3971.js, line 50-70):**
| Status | Icon | Color | Meaning |
|--------|------|-------|---------|
| `pending` | ○ | text.muted | Feature queued, awaiting execution |
| `in_progress` | ● | primary | Worker currently executing this feature |
| `completed` | ✓ | success | Feature finished successfully |
| `cancelled` | ✗ | error | Feature cancelled by user/orchestrator (terminal) |

**Status transitions (Module 2542.js):**
- `pending → in_progress`: When `spawnWorker()` selects the feature (line 464: `updateFeature(A.id, { status: "in_progress", currentWorkerSessionId: B, ... })`)
- `in_progress → pending`: On failure/crash, function `S9H()` resets the feature (line 12-14: `updateFeature(L, { status: "pending", currentWorkerSessionId: null })`)
- `in_progress → completed`: Worker submits successful handoff via `EndFeatureRun`
- `pending → cancelled`: Orchestrator cancels a feature (terminal state, feature moved to bottom)

**Feature ordering:** Features execute in array order. Completed/cancelled features move to the bottom via `moveStrandedDoneFeaturesToBottom()` (Module 2201.js, line 223-237). New urgent features are inserted at the top via `insertFeatureAtTop()`.

### validation-state.json Management

**Schema (from Module 2175.js, line 717-729):**
```json
{
  "assertions": {
    "VAL-AUTH-001": { "status": "pending" },
    "VAL-AUTH-002": { "status": "pending" },
    "VAL-CROSS-001": { "status": "pending" }
  }
}
```

**Assertion statuses:**
| Status | Meaning |
|--------|---------|
| `"pending"` | Not yet tested |
| `"passed"` | Verified by user-testing validator |
| `"failed"` | Test failed or blocked |

**Update triggers (Module 2175.js, line 2109):**
- User-testing validator updates on assertion test completion:
  - `pass` → set status to `"passed"`, record `validatedAtMilestone`
  - `fail` → set status to `"failed"`, record issues
  - `blocked` → set status to `"failed"`, record blocking reason
- On validation failure, assertions moved to new features remain `"pending"` in validation-state.json
- End-of-mission gate: ALL assertions must be `"passed"` before mission declared complete

### missionDir Layout

The mission directory is located at `~/.opendroid/missions/{sessionId}/` (Module 2201.js, line 44):
```
missionDir/
├── state.json              # Mission-level state (state, currentFeatureId, workerSessionIds)
├── features.json           # Feature array with status, fulfills, skillName
├── validation-contract.md  # Assertion definitions
├── validation-state.json   # Assertion pass/fail/blocked tracking
├── agent-guidelines.md               # Worker guidance (boundaries, conventions)
├── mission.md              # Mission proposal (accepted by user)
├── progress_log.jsonl      # Append-only event log (worker_started, worker_completed, etc.)
├── handoffs.jsonl          # Append-only handoff log
├── handoffs/               # Individual handoff JSON files
│   └── {timestamp}__{featureId}__{sessionId}.json
├── worker-transcripts.jsonl # Worker transcript skeletons
├── working_directory.txt   # CWD for workers
└── model-settings.json     # Model overrides (orchestrator, worker, validation)
```

**File paths are resolved by `PMH` class (Module 2201.js):**
- `getFilePaths()` returns `{ state, features, progressLog, agentsMd, missionMd }` for file watchers
- Handoff files: `P4.join(this.missionDir, "handoffs", "{timestamp}__{featureId}__{sessionId}.json")`
- Progress log: append-only JSONL with entries: `worker_started`, `worker_completed`, `worker_failed`, `worker_paused`, `mission_paused`, `milestone_validation_triggered`, `worker_selected_feature`

### State Recovery on Restart

When the system restarts, `MissionRunner` recovers state through:

1. **Read `state.json`** to determine current mission state
2. **Check for orphaned workers** via `cleanupOrphanedWorker()` (Module 2542.js, line 109):
   - If `currentWorkerSessionId` exists in state → interrupt it, log `worker_failed`, reset feature to `pending`
3. **Resume interrupted worker** via `getInterruptedWorkerSessionId()` (Module 2201.js, line 341):
   - Scans progress log for `worker_paused` entries without matching `worker_completed`
   - Falls back to `worker_started` without matching completion
   - Returns session ID for resume
4. **Resume flow** (`runLoop` with `resumeWorkerSessionId`):
   - Calls `loadSession()` on the interrupted session
   - Sends resume message: "You were interrupted mid-work. Continue where you left off."
   - Enters `waitForWorkerCompletion()` polling loop

**Progress log event types drive recovery:**
| Event Type | Logged When | Key Fields |
|-----------|-------------|------------|
| `worker_started` | Worker session created | `workerSessionId`, `spawnId`, `featureId` |
| `worker_selected_feature` | Feature assigned to worker | `workerSessionId`, `featureId` |
| `worker_completed` | Worker finished successfully | `workerSessionId`, `exitCode`, `successState`, `returnToOrchestrator`, `handoff` |
| `worker_failed` | Worker crashed/errored | `workerSessionId`, `spawnId`, `exitCode`, `reason` |
| `worker_paused` | Worker interrupted mid-work | `workerSessionId`, `featureId` |
| `mission_paused` | Mission paused by user/SIGINT | (no extra fields) |
| `milestone_validation_triggered` | Validation features injected | `milestone`, `featureId` |

## Key Findings

### 1. Dual-Layer State Machine
Mission state (`state.json`) and feature state (`features.json`) form independent but coordinated state machines. The mission state drives the runner loop (what to do next), while feature state drives worker assignment (what work is available).

### 2. Terminal States Are Explicit
`completed` and `cancelled` are terminal for features. The runner treats both as "done" for milestone completion checks. The runner never transitions a completed feature back to pending: only `in_progress` features are reset on failure via `S9H()`.

### 3. Orchestrator Turn Gate
The `orchestrator_turn` state is a critical gate: the runner stops and returns control to the orchestrator. This happens on: worker failure, worker requesting `returnToOrchestrator=true`, or no pending features but mission not complete.

### 4. Validation Auto-Injection
When all implementation features in a milestone complete, `checkMilestoneCompletionAndInjectValidation()` (Module 2542.js, line 309) automatically inserts `scrutiny-validator-{milestone}` and `user-testing-validator-{milestone}` at the top of the feature array. This is guarded by `milestonesWithValidationPlanned` to prevent duplicate injection.

### 5. Append-Only Audit Trail
The progress log (`progress_log.jsonl`) is append-only and serves as the audit trail. Recovery after crashes relies on scanning this log for unmatched `worker_started`/`worker_paused` entries.

### 6. Feature Re-ordering on Completion
Completed features move to the bottom of the array automatically. This ensures the "next pending" feature is always at the top, implementing a FIFO-with-priority system (urgent fixes can be inserted at top).

## Code Examples

### State File Creation (Module 2201.js, line 87-100)
```js
// Module 2201.js:87: createInitialState()
async createInitialState(H, A) {
  let L = new Date().toISOString();
  let $ = {
    missionId: `mis_${rf().slice(0, 8)}`,
    baseSessionId: this.baseSessionId,
    state: "initializing",
    workingDirectory: A,
    currentFeatureId: null,
    currentWorkerSessionId: null,
    currentWorkerPid: null,
    workerSessionIds: [],
    completedFeatures: 0,
    totalFeatures: H,
    createdAt: L,
    updatedAt: L,
  };
  await this.writeState($);
  return $;
}
```

### Feature Status Transition: Spawn (Module 2542.js, line 464)
```js
// Module 2542.js:464: Setting feature to in_progress on worker spawn
await this.missionFileService.updateFeature(A.id, {
  status: "in_progress",
  currentWorkerSessionId: B,
  workerSessionIds: [...(A.workerSessionIds ?? []), B],
});
```

### Feature Reset on Failure (Module 2542.js, line 12-14)
```js
// Module 2542.js:12: S9H() resets feature to pending on worker failure
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

### Milestone Validation Injection (Module 2542.js, line 309-360)
```js
// Module 2542.js:309: checkMilestoneCompletionAndInjectValidation()
async checkMilestoneCompletionAndInjectValidation(H) {
  let A = await this.missionFileService.getFeature(H);
  if (!A?.milestone) return;
  let L = A.milestone;
  if (!(await this.missionFileService.isMilestoneImplementationComplete(L))) return;
  if (await this.missionFileService.hasValidationPlannerRun(L)) return;
  // ... injects scrutiny-validator and user-testing-validator features
  if (!W) await this.missionFileService.insertFeatureAtTop(E); // user-testing
  if (!B) await this.missionFileService.insertFeatureAtTop(M); // scrutiny
  await this.missionFileService.markMilestoneValidationPlanned(L);
}
```

### Feature Status Icon Mapping (Module 3971.js, line 50-70)
```js
// Module 3971.js:50: xxM() maps status to icon+color
function xxM(H) {
  switch (H) {
    case "completed":  return { icon: "✓", color: o.success };
    case "in_progress": return { icon: "●", color: o.primary };
    case "pending":    return { icon: "○", color: o.text.muted };
    case "cancelled":  return { icon: "✗", color: o.error };
    default:           return { icon: "○", color: o.text.muted };
  }
}
```

### State File Watcher (Module 3970.js, line 75-130)
```js
// Module 3970.js:75: WaD() creates file stat snapshots for change detection
let x = WaD([N.state, N.features, N.progressLog]);
// If file sizes/mtimes haven't changed, skip re-read (polling optimization)
if (!X && U.current && !TaD(U.current, x)) {
  M.current = setTimeout(() => { V(false); }, VaD);
  return;
}
```

## Integration Points

- **01-terminal-ui (TUI):** Module 3971.js and 3973.js are TUI rendering components that consume state. Events `mission_state_changed`, `mission_features_changed`, `mission_progress_entry`, `mission_worker_started`, `mission_worker_completed` are emitted for TUI consumption via `v$.emit("project-notification", ...)`.
- **03-tool-agent-system (Tool):** The `EndFeatureRun` tool (Tool system) is how workers submit final state transitions. The `Skill` tool dispatches skill injection. Module 2856.js handles tool execution errors.
- **04-desktop-gui (GUI):** Module 0919.js contains daemon session schemas (Zod validation) that bridge the state to the Electron app layer. The daemon API uses `daemon.initialize_session`, `daemon.load_session`, etc.
- **05-infrastructure (Infra):** Session management, daemon connectivity, model settings persistence. The `zX()` opendroidd client handles worker session lifecycle.

### State File Schema (from Module 2201.js + 2175.js)

**state.json:**
```json
{
  "missionId": "mis_xxxx",
  "baseSessionId": "uuid",
  "state": "running",
  "workingDirectory": "/path/to/repo",
  "currentFeatureId": "feature-id",
  "currentWorkerSessionId": "session-uuid",
  "currentWorkerPid": null,
  "workerSessionIds": ["session-1", "session-2"],
  "completedFeatures": 5,
  "totalFeatures": 15,
  "milestonesWithValidationPlanned": ["auth"],
  "createdAt": "2025-01-01T00:00:00.000Z",
  "updatedAt": "2025-01-01T01:00:00.000Z"
}
```

**features.json:**
```json
{
  "features": [{
    "id": "feature-id",
    "description": "...",
    "skillName": "worker",
    "preconditions": [],
    "expectedBehavior": [],
    "verificationSteps": [],
    "fulfills": ["VAL-XXX-001"],
    "milestone": "milestone-name",
    "status": "pending",
    "workerSessionIds": [],
    "currentWorkerSessionId": null,
    "completedWorkerSessionId": null
  }]
}
```

## Implementation Notes

1. **Implement `MissionFileService`**: a class with read/write methods for `state.json`, `features.json`, `progress_log.jsonl`, and `handoffs/` directory. Use atomic file writes.
2. **Implement `MissionRunner` loop**: the core state machine driver with `runLoop()`, `spawnWorker()`, `resumeWorker()`, `pause()`. Key: the loop is sequential (one worker at a time).
3. **Feature status enum**: `pending | in_progress | completed | cancelled`. No `failed` state exists in the runtime; failed features are reset to `pending` for re-execution.
4. **Mission state enum**: `initializing | running | paused | orchestrator_turn | completed`.
5. **Event system**: emit events on state changes for UI updates. Use a simple EventEmitter pattern.
6. **Validation auto-injection**: after milestone completion, inject validator features at top of array, guarded by `milestonesWithValidationPlanned` set in state.json.
7. **Recovery**: scan progress log for unmatched worker events to determine if a worker was interrupted. On restart, offer to resume or reset the feature to pending.

## Known Gaps

- Module 3985.js and 3987.js (worker lifecycle modules) were not read: they may contain additional state transition hooks
- The `restartFeature` mechanism mentioned in the feature description was not found as a standalone function; it appears to be handled by resetting status to `"pending"` via `S9H()` and re-inserting via `insertFeatureAtTop()`
- Full `EndFeatureRun` tool implementation is in 03-tool-agent-system (Tool system) territory: only the consumption side is documented here
- The `dismiss_handoff_items` tool mentioned in the orchestrator prompt was not traced to a specific module
- The feature `normalizeFeature()` method has no `"failed"` or `"paused"` status in its status mapping, suggesting these are runtime-only states not persisted to features.json
