# mission-feature-registry: Mission System RE

## Overview

The feature registry is the central data structure that drives mission execution. It is persisted as `features.json` in the mission directory and consumed by the MissionRunner, orchestrator, and TUI. This report documents the complete schema, feature ordering algorithm, status transitions, milestone boundaries, and fulfills tracking derived from the Zod schemas, MissionFileService CRUD operations, and MissionRunner orchestration logic.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 0082.js | 109 lines | Zod schemas | Feature schema, Handoff schema, MissionEvent discriminated union |
| 2175.js | 2602 lines | Orchestrator prompt template | Documents features.json schema, ordering rules, cancellation, sealed milestones |
| 2201.js | 626 lines | MissionFileService | CRUD for features.json: read, update, getNextPendingFeature, moveFeatureToBottom |
| 2542.js | 811 lines | MissionRunner | runLoop that selects next pending feature, spawns workers, handles completion |
| 3971.js | 662 lines | TUI feature detail view | Renders feature fields (status, milestone, preconditions, expectedBehavior, verificationSteps) |
| 3973.js | 699 lines | TUI status filters | Status enum labels: pending, in_progress, completed, cancelled |

> **Note on target modules**: The original feature spec listed modules 2311, 3989, 0914, 1146, 1207, 0944. These modules contain TUI ASCII art, mission research preview UI, daemon automation RPCs, MCP auth, fetch URL tool, and wake lock logic: none directly implement the feature registry. The modules above (0082, 2175, 2201, 2542, 3971, 3973) are the actual implementations substituted via grep-first discovery.

## Architecture / Flow

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  orchestrator   │────▶│  features.json   │◄────│  MissionFileSvc │
│  (plans edits)  │     │  (missionDir)    │     │  (2201.js)      │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │                           │
                               ▼                           ▼
                        ┌──────────────┐          ┌─────────────────┐
                        │  TUI render  │          │  MissionRunner  │
                        │  (3971.js)   │          │  (2542.js)      │
                        └──────────────┘          └─────────────────┘
                                                         │
                                                         ▼
                                                  ┌─────────────┐
                                                  │  Worker     │
                                                  │  (spawned)  │
                                                  └─────────────┘
```

1. **Orchestrator** creates and edits `features.json` during mission planning.
2. **MissionFileService** (2201.js) provides typed CRUD operations over the JSON file.
3. **MissionRunner** (2542.js) reads the registry via `getNextPendingFeature()` to decide what to run next.
4. **TUI** (3971.js + 3973.js) renders the feature list with status icons and detail panes.
5. **Validation** uses `fulfills` arrays to map features to contract assertions.

## Key Findings

### 1. features.json Schema (Zod-validated, 0082.js lines 8-25)

The runtime schema is defined in 0082.js using Zod (`AH` alias):

```js
// Module 0082.js, line 8
pNH = AH.object({
  id: AH.string(),
  description: AH.string(),
  status: AH.nativeEnum(j7),        // j7 = { pending, in_progress, completed, cancelled }
  skillName: AH.string(),
  preconditions: AH.array(AH.string()),
  expectedBehavior: AH.array(AH.string()),
  verificationSteps: AH.array(AH.string()),
  fulfills: AH.array(AH.string()).optional(),
  milestone: AH.string().optional(),
  workerSessionIds: AH.array(AH.string()).optional(),
  currentWorkerSessionId: AH.string().nullable().optional(),
  completedWorkerSessionId: AH.string().nullable().optional(),
})
```

**Field semantics** (from 2175.js lines 735-790):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Unique identifier |
| `description` | string | yes | What to build |
| `skillName` | string | yes | Which worker skill handles this feature |
| `milestone` | string | no | Vertical slice; triggers validation when all features complete |
| `preconditions` | string[] | yes | What must be true before starting |
| `expectedBehavior` | string[] | yes | Success criteria |
| `verificationSteps` | string[] | yes | How to verify |
| `fulfills` | string[] | no | Validation contract assertion IDs this feature completes |
| `status` | enum | yes | `pending`, `in_progress`, `completed`, `cancelled` |
| `workerSessionIds` | string[] | no | All session IDs that worked on this feature |
| `currentWorkerSessionId` | string\|null | no | Active worker session |
| `completedWorkerSessionId` | string\|null | no | Session that completed the feature |

**Normalization** (2201.js lines 88-107): `readFeatures()` normalizes each feature via `PMH.normalizeFeature()`, ensuring missing arrays default to `[]` and `status` defaults to `"pending"`.

### 2. Feature Ordering and Sequencing

**Algorithm** (2201.js lines 174-177):
```js
async getNextPendingFeature() {
  let H = await this.readFeatures();
  if (!H) return null;
  return H.features.find((A) => A.status === "pending") ?? null;
}
```

- **Array order matters**: The first feature with `status === "pending"` is selected.
- **Insertion at top**: Urgent/blocking features are inserted at the top via `insertFeatureAtTop()` (2201.js lines 192-199).
- **Completed features sink**: On completion, `moveFeatureToBottom()` (2201.js lines 200-207) moves the feature to the end of the array.
- **Stranded cleanup**: `moveStrandedDoneFeaturesToBottom()` (2201.js lines 208-220) compacts completed/cancelled features that appear mid-array to the bottom, preserving relative order of active features.

**Orchestrator guidance** (2175.js lines 1095-1105):
> "Features are executed in array order: first pending feature runs next. Place foundational features first. Group features by milestone. When adding urgent/blocking features, insert them at the TOP of the array. Completed features automatically move to the bottom."

### 3. Status Transitions

The feature status enum has **four states** (3973.js lines 11-17):
```js
UMA = ["all", "pending", "in_progress", "completed", "cancelled"];
gxM = {
  all: "All",
  pending: "Pending",
  in_progress: "In Progress",
  completed: "Completed",
  cancelled: "Cancelled",
};
```

**Transition diagram**:

```
                    ┌─────────────────────────────────────────┐
                    │         orchestrator / user             │
                    │  (cancel scope change, user request)    │
                    ▼                                         │
┌─────────┐   spawn worker   ┌─────────────┐   success    ┌───────────┐
│ pending │ ────────────────▶│ in_progress │ ────────────▶│ completed │
└─────────┘                  └─────────────┘              └───────────┘
     ▲                            │                            │
     │                            │ worker fail / crash        │
     │                            ▼                            │
     │                     resetFeature()                      │
     │                     (S9H in 2542.js)                    │
     │                     status → "pending"                  │
     │                                                         │
     └─────────────────────────────────────────────────────────┘
                           moveToBottom()
```

**Key transition rules**:

| From | To | Trigger | Module |
|------|-----|---------|--------|
| pending | in_progress | MissionRunner spawns worker | 2542.js:433-442 |
| in_progress | pending | Worker spawn fails (S9H) or crash recovery | 2542.js:14-22, 470-477 |
| in_progress | completed | Worker returns success + EndFeatureRun | 2542.js:303-320 |
| pending | cancelled | User scope reduction or obsolete feature | 2175.js:1110 |
| completed | pending | **Not allowed**: terminal state |: |
| cancelled | pending | **Not allowed**: terminal state |: |

**Worker failure handling**: When a worker spawn fails or crashes, `S9H()` (2542.js lines 14-22) resets the feature:
```js
async function S9H({ missionFileService, featureId, workerSessionId }) {
  if (!featureId) return;
  let I = await missionFileService.getFeature(featureId);
  if (!I) return;
  if (I.status !== "in_progress") return;
  if (workerSessionId && I.currentWorkerSessionId !== workerSessionId) return;
  await missionFileService.updateFeature(featureId, {
    status: "pending",
    currentWorkerSessionId: null,
  });
}
```

### 4. Milestone Boundaries

**Milestone completion detection** (2201.js lines 237-244):
```js
async isMilestoneImplementationComplete(H) {
  let A = await this.getMilestoneFeatures(H);
  if (A.length === 0) return false;
  let L = A.filter(($) => !HMH.includes($.skillName));  // exclude validators
  if (L.length === 0) return false;
  return L.every(($) => $.status === "completed" || $.status === "cancelled");
}
```

- A milestone is "complete" when **all non-validator features** are `completed` or `cancelled`.
- Once complete, the MissionRunner auto-injects `scrutiny-validator` and `user-testing-validator` features (2542.js: checkMilestoneCompletionAndInjectValidation).
- **Sealed milestone** (2175.js line 1114): After validators pass, the milestone is sealed. New work must go into follow-up or `misc-*` milestones. Never add to a sealed milestone.

### 5. fulfills Tracking and Assertion Coverage

**`fulfills` semantics** (2175.js lines 787-790):
> "Only the leaf feature that makes an assertion fully testable claims it. Infrastructure/foundational features have empty or no `fulfills`. Each assertion ID should appear in exactly one feature's `fulfills` across the entire features.json."

**Coverage gate** (2175.js line 954):
> "Every assertion ID in `validation-contract.md` is claimed by exactly one feature's `fulfills`."

**Validation flow**:
1. `validation-state.json` tracks assertion status (`pending` → `passed`/`failed`/`blocked`).
2. When a milestone completes, validators read features' `fulfills` to determine which assertions to test (2175.js: scrutiny-validator and user-testing-validator prompts).
3. On override, assertions can be moved out of a sealed milestone's `fulfills` into an unsealed milestone feature (2175.js line 1170).

### 6. Feature Cancellation

**Cancellation rules** (2175.js lines 1109-1113):
> "Set status to `"cancelled"` when the user asks to drop/skip a feature, when a scope change makes a feature obsolete, or when discovery reveals a feature is no longer viable. Cancelled is a terminal state: the runtime skips cancelled features and treats them as done for milestone completion. When cancelling, move the feature to the bottom of the array (alongside completed ones)."

**Side effects**:
- Cancelled assertions are removed from `validation-state.json` (2175.js line 1079).
- `areAllFeaturesCompleted()` treats `cancelled` same as `completed` (2201.js lines 179-182).

## Code Examples

### Feature schema validation (0082.js)
```js
// Module 0082.js, lines 8-25
pNH = AH.object({
  id: AH.string(),
  description: AH.string(),
  status: AH.nativeEnum(j7),
  skillName: AH.string(),
  preconditions: AH.array(AH.string()),
  expectedBehavior: AH.array(AH.string()),
  verificationSteps: AH.array(AH.string()),
  fulfills: AH.array(AH.string()).optional(),
  milestone: AH.string().optional(),
  workerSessionIds: AH.array(AH.string()).optional(),
  currentWorkerSessionId: AH.string().nullable().optional(),
  completedWorkerSessionId: AH.string().nullable().optional(),
});
```

### Next pending feature selection (2201.js)
```js
// Module 2201.js, lines 174-177
async getNextPendingFeature() {
  let H = await this.readFeatures();
  if (!H) return null;
  return H.features.find((A) => A.status === "pending") ?? null;
}
```

### MissionRunner feature spawning (2542.js)
```js
// Module 2542.js, lines 418-442
async spawnWorker() {
  let H = `worker_${rf().slice(0, 8)}`;
  let A = await this.missionFileService.getNextPendingFeature();
  if (!A) return { success: false, returnToOrchestrator: true, message: "No pending features available" };
  // ... spawn worker session ...
  await this.missionFileService.updateFeature(A.id, {
    status: "in_progress",
    currentWorkerSessionId: B,
    workerSessionIds: [...(A.workerSessionIds ?? []), B],
  });
}
```

### TUI status rendering (3971.js)
```js
// Module 3971.js, lines 33-49
function xxM(H) {
  switch (H) {
    case "completed":   return { icon: "\u2713", color: o.success };
    case "in_progress": return { icon: "\u25CF", color: o.primary };
    case "pending":     return { icon: "\u25CB", color: o.text.muted };
    case "cancelled":   return { icon: "\u2717", color: o.error };
    default:            return { icon: "\u25CB", color: o.text.muted };
  }
}
```

## Integration Points

- **01-terminal-ui (TUI)**: 3971.js and 3973.js render feature lists and status filters in the terminal UI. Feature navigation (`session_viewer`) uses `featureId`, `completedWorkerSessionId`, and `currentWorkerSessionId`.
- **03-tool-agent-system (Tool)**: The `Task` tool spawns subagents for validation reviewers; validation uses `fulfills` to determine which assertions to test.
- **04-desktop-gui (GUI)**: Not directly referenced; feature registry is file-based, no GUI editor found.
- **05-infrastructure (Infra)**: `features.json` and `state.json` are persisted to disk via `fs/promises` in MissionFileService (2201.js). Progress log is appended as JSONL.

## Implementation Notes

To port the feature registry to OpenDroid:

1. **Schema**: Replicate the Zod schema from 0082.js. The 12 fields are well-defined and self-contained.
2. **Persistence**: Use `features.json` in the mission directory. MissionFileService's `readFeatures`/`writeFeatures` pattern with normalization is straightforward to port.
3. **Ordering**: Implement `getNextPendingFeature()` as `features.find(f => f.status === "pending")`. Maintain array-order semantics.
4. **Status machine**: Four states (pending → in_progress → completed/cancelled). Worker failures reset to pending, not "failed".
5. **Milestone detection**: Filter by `milestone`, exclude validator `skillName`s, check all are terminal.
6. **fulfills**: Enforce uniqueness at planning time. Each assertion ID appears in exactly one feature's `fulfills`.
7. **Cancellation**: Terminal state. Move to bottom of array. Remove assertions from `validation-state.json`.

## Known Gaps

- `checkMilestoneCompletionAndInjectValidation()` in 2542.js was referenced but not fully read: the exact validator injection logic (how scrutiny-validator and user-testing-validator features are constructed and appended) is not fully documented here. Suggest follow-up reading of 2542.js lines 520-600.
- The `j7` enum definition location was not precisely found (obfuscated in 0059.js/0060.js), though its values are confirmed via 3973.js and runtime behavior.
- `HMH` (validator skill name exclusion list) definition not traced: appears to be an array of validator skill names like `scrutiny-validator` and `user-testing-validator`.
- No evidence of "paused" as a feature status: it appears to be a mission-level state (`state.json`) and worker-level state, not a feature status.
