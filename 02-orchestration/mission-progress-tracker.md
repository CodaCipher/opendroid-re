# mission-progress-tracker: Mission System RE

## Overview

This report documents how OpenDroid's mission system tracks milestone progress, detects completion, auto-injects validation features, and manages sealed milestones. The core tracking logic lives in the MissionFileService (module 2201.js) and MissionRunner (module 2542.js), with event schemas defined in module 0082.js and UI rendering in module 3978.js.

**Note on module map:** The feature spec listed modules 0082, 2478, 2553, 3979, 3970, 0930. Grep analysis revealed that 2478.js (file-edit tool), 2553.js (CLI arg validation), and 0930.js (shell command parsing) do not contain milestone/progress logic. The actual core modules for this feature are 0082.js, 2201.js, 2542.js, 2176.js, 3978.js, 3970.js, and 0087.js. The feature spec module list appears stale.

---

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 0082.js | 109 lines | Event/feature schemas | Defines `milestone_validation_triggered` event, feature schema with `status`, `milestone` fields |
| 2201.js | 626 lines | MissionFileService | Core persistence: milestone completion check, validation planning state, progress log |
| 2542.js | 811 lines | MissionRunner | Main loop: triggers milestone check after each worker, injects validation features |
| 2176.js | ~150 lines | Skill registry | Defines `HMH`: validator skill names excluded from implementation counting |
| 3978.js | 1045 lines | UI formatting | Formats progress events (including `milestone_validation_triggered`) for TUI display |
| 3970.js | 157 lines | React hook | Watches mission files (state, features, progressLog) for changes, drives live UI updates |
| 0087.js | 278 lines | State schemas | Defines `features.json` and `progressLog` array schemas |

---

## Architecture / Flow

### Milestone Progress Calculation

1. **Feature grouping by milestone**: `MissionFileService.getMilestoneFeatures(milestone)` (2201.js:223) filters `features.json` by the `milestone` field.
2. **Implementation completion check**: `isMilestoneImplementationComplete(milestone)` (2201.js:235) retrieves all milestone features, **filters out validator features** (those with `skillName` in `HMH`), then checks if every remaining feature has `status === "completed"` or `status === "cancelled"`.
3. **Cancelled features count as done**: The completion check explicitly treats `cancelled` as a terminal done state (2201.js:240).
4. **Overall mission completion**: `areAllFeaturesCompleted()` (2201.js:178) checks if **all** features across **all** milestones are `completed` or `cancelled`.

### Completion Detection Flow

The MissionRunner main loop (2542.js) checks milestone completion at two points:

1. **After worker resume/completion** (2542.js:248): `if (f.featureId) await this.checkMilestoneCompletionAndInjectValidation(f.featureId)`
2. **After spawning a new worker** (2542.js:282): `if (I.featureId) await this.checkMilestoneCompletionAndInjectValidation(I.featureId)`

When no pending features exist, the runner also calls `checkAllMilestonesForValidation()` (2542.js:264) to sweep all milestones.

### Validator Auto-Injection Trigger

`checkMilestoneCompletionAndInjectValidation(featureId)` (2542.js:316) performs the injection:

1. Gets the feature's milestone
2. Calls `isMilestoneImplementationComplete(milestone)`: returns false if any non-validator feature is not completed/cancelled
3. Calls `hasValidationPlannerRun(milestone)`: checks `state.json` `milestonesWithValidationPlanned` array to prevent duplicate injection
4. Reads model settings for `skipScrutiny` and `skipUserTesting` flags
5. If both skips are true: marks validation planned and returns early
6. Otherwise, constructs two feature objects:
   - `user-testing-validator-{milestone}` (skillName = `efH` = "user-testing-validator")
   - `scrutiny-validator-{milestone}` (skillName = `sfH` = "scrutiny-validator")
7. Inserts both at the **top** of the features array via `insertFeatureAtTop()` (2201.js:191)
8. Calls `markMilestoneValidationPlanned(milestone)` to record the milestone in `state.json`
9. Appends a `milestone_validation_triggered` entry to `progress_log.jsonl`

### Sealed Milestones

The "sealed" concept is enforced through the `milestonesWithValidationPlanned` array in `state.json`:

- A milestone is considered "sealed" once `markMilestoneValidationPlanned()` has been called (2201.js:247)
- `hasValidationPlannerRun(milestone)` returns true after sealing (2201.js:242)
- The orchestrator prompt (2175.js:1114, 1273) explicitly states: "Once a milestone's validators pass, that milestone is **sealed**. Never add features to a completed milestone."
- Post-seal restrictions are **policy-level** (enforced by orchestrator guidance in agent-guidelines.md / 2175.js), not hardcoded in the runner: the runner itself does not block adding features to sealed milestones; it only prevents duplicate validation injection.

### Progress Events / Notifications

Progress flows through multiple layers:

1. **Persistence layer**: `appendProgressLog(entry)` (2201.js:395) writes JSON lines to `progress_log.jsonl` in the mission directory
2. **Notification emission**: After appending, it emits `project-notification` events:
   - `mission_progress_entry` with full progressLog array (2201.js:408)
   - `mission_worker_started` / `mission_worker_completed` for worker lifecycle (2201.js:412-420)
3. **File watching**: The React hook in 3970.js watches `state.json`, `features.json`, and `progress_log.jsonl` via stat comparison (`WaD`, `TaD` at 3970.js:75), polling for changes and refreshing the UI snapshot
4. **UI rendering**: Module 3978.js formats events for display, e.g.:
   - `milestone_validation_triggered` → `"Milestone validation: {milestone}"` (3978.js:91)
   - `worker_completed` → `"Worker #{id} completed [{featureId}] ✓"` (3978.js:89)

---

## Key Findings

### 1. Milestone completion excludes validators from counting

The `HMH` array (2176.js:113) contains `["scrutiny-validator", "user-testing-validator"]`. When `isMilestoneImplementationComplete()` checks a milestone, it filters out any feature whose `skillName` is in this list. This prevents validator features from being counted as "implementation work": otherwise a milestone would never complete because the validators themselves are pending.

### 2. Validation injection is idempotent

The `milestonesWithValidationPlanned` array in `state.json` ensures validation features are only injected once per milestone. Even if the runner checks the same milestone multiple times, `hasValidationPlannerRun()` blocks re-injection.

### 3. Validation features are inserted at TOP of array

`insertFeatureAtTop()` (2201.js:191) uses `Array.unshift()`, placing validators before any other pending features. This means validation runs immediately after the last implementation feature completes, without waiting for unrelated pending features from other milestones.

### 4. Progress log is append-only JSONL

The progress log uses line-delimited JSON (`progress_log.jsonl`), appended via `fs.appendFile()`. The UI reads it incrementally with a stat-based cache (`progressLogCache` in 2201.js) to avoid re-parsing the entire file on every update.

### 5. Sealed milestone enforcement is policy, not code

The runner does not have a "sealed" boolean per milestone. Sealing is tracked implicitly via `milestonesWithValidationPlanned`. The actual restriction on adding new features to sealed milestones is enforced by the orchestrator's system prompt (2175.js), not by the MissionFileService or MissionRunner.

---

## Code Examples

### Milestone implementation completion check (2201.js:235-240)

```js
// Module 2201.js, line 235
async isMilestoneImplementationComplete(H) {
  let A = await this.getMilestoneFeatures(H);
  if (A.length === 0) return false;
  let L = A.filter(($) => !HMH.includes($.skillName));
  if (L.length === 0) return false;
  return L.every(($) => $.status === "completed" || $.status === "cancelled");
}
```

### Validation feature injection (2542.js:316-378)

```js
// Module 2542.js, line 316
async checkMilestoneCompletionAndInjectValidation(H) {
  let A = await this.missionFileService.getFeature(H);
  if (!A?.milestone) return;
  let L = A.milestone;
  if (!(await this.missionFileService.isMilestoneImplementationComplete(L))) return;
  if (await this.missionFileService.hasValidationPlannerRun(L)) return;
  // ... constructs scrutiny-validator-{L} and user-testing-validator-{L} features
  if (!W) await this.missionFileService.insertFeatureAtTop(E);   // user-testing
  if (!B) await this.missionFileService.insertFeatureAtTop(M);   // scrutiny
  await this.missionFileService.markMilestoneValidationPlanned(L);
  await this.missionFileService.appendProgressLog({
    timestamp: new Date().toISOString(),
    type: "milestone_validation_triggered",
    milestone: L,
    featureId: V,
  });
}
```

### Progress log append with notification emission (2201.js:395-420)

```js
// Module 2201.js, line 395
async appendProgressLog(H) {
  let A = { ...H, timestamp: H.timestamp || new Date().toISOString() },
    L = `${JSON.stringify(A)}\n`;
  try {
    await sM.appendFile(this.progressLogPath, L);
  } catch {
    await sM.writeFile(this.progressLogPath, L);
  }
  let $ = await this.readProgressLog();
  v$.emit("project-notification", {
    notification: { type: "mission_progress_entry", progressLog: $ },
  });
  // ... worker-started / worker-completed notifications
}
```

### Milestone validation triggered event schema (0082.js:101-105)

```js
// Module 0082.js, line 101-105
(af0 = uY.extend({
  type: AH.literal("milestone_validation_triggered"),
  milestone: AH.string(),
  featureId: AH.string(),
})),
```

### Validator skill name exclusion list (2176.js:113)

```js
// Module 2176.js, line 113
(HMH = [sfH, efH]);
// where sfH = "scrutiny-validator", efH = "user-testing-validator"
```

---

## Integration Points

- **01-terminal-ui (TUI)**: Module 3978.js formats progress events for terminal display; module 3970.js provides the React hook that drives live file watching. The TUI consumes `project-notification` events of type `mission_progress_entry`, `mission_worker_started`, `mission_worker_completed`.
- **03-tool-agent-system (Tool)**: The `appendProgressLog` and `updateState` calls in 2201.js emit events that the tool layer (Task tool, EndFeatureRun) consumes indirectly through file persistence.
- **04-desktop-gui (GUI)**: Module 3979.js (mission settings UI) and 3970.js (file watcher hook) are React components that would feed a GUI progress display.
- **05-infrastructure (Infra)**: The mission directory layout (`missionDir/state.json`, `missionDir/progress_log.jsonl`, `missionDir/features.json`) is the persistence infra. The `milestonesWithValidationPlanned` field in `state.json` is the sealing mechanism.

### Data Flow Diagram (textual)

```
Worker completes feature
    ↓
MissionRunner.checkMilestoneCompletionAndInjectValidation() [2542.js]
    ↓
MissionFileService.isMilestoneImplementationComplete() [2201.js]
    ↓ (if true)
MissionFileService.hasValidationPlannerRun() [2201.js]
    ↓ (if false)
Construct validator features → insertFeatureAtTop() [2201.js]
    ↓
markMilestoneValidationPlanned() → writes state.json [2201.js]
    ↓
appendProgressLog() → writes progress_log.jsonl + emits notification [2201.js]
    ↓
React hook (3970.js) detects file change → refreshes UI snapshot
    ↓
UI formatter (3978.js) renders "Milestone validation: {name}"
```

---

## Implementation Notes

To port milestone progress tracking to OpenDroid:

1. **Implement `MissionFileService`** (or equivalent):
   - `getMilestoneFeatures(milestone)`: filter features by milestone
   - `isMilestoneImplementationComplete(milestone)`: filter out `skillName ∈ ["scrutiny-validator", "user-testing-validator"]` then check all `completed`/`cancelled`
   - `hasValidationPlannerRun(milestone)`: check `state.json` array field
   - `markMilestoneValidationPlanned(milestone)`: append to `state.json` array
   - `insertFeatureAtTop(feature)`: unshift into `features.json` array
   - `appendProgressLog(entry)`: append JSON line to `progress_log.jsonl`

2. **Implement `MissionRunner` milestone check**:
   - After every worker handoff (success/failure/resume), call `checkMilestoneCompletionAndInjectValidation(featureId)`
   - When no pending features remain, call `checkAllMilestonesForValidation()` as a sweep

3. **State schema additions**:
   - Add `milestonesWithValidationPlanned: string[]` to `state.json`
   - Ensure `features.json` items have `milestone`, `status`, `skillName` fields

4. **Event schema**:
   - Define `milestone_validation_triggered` event with `{ type, milestone, featureId }`

5. **Progress notifications**:
   - Emit events after appending to progress log
   - File watcher (stat polling or fs.watch) refreshes UI

6. **Sealed milestone policy**:
   - Store `milestonesWithValidationPlanned` array
   - Enforce "never add features to sealed milestone" at orchestrator/policy layer (not runner layer)

---

## Known Gaps

- **Progress bar / percentage calculation**: No explicit "X of Y features done" percentage logic was found in the decoded modules. The TUI likely derives this from counting `features.filter(f => f.status === "completed").length` on the client side.
- **Parallel milestone execution**: The current system processes milestones sequentially. Whether multiple milestones can be active simultaneously was not explored.
- **Re-validation after fix**: When a validator fails and fix features are created, the `milestonesWithValidationPlanned` state prevents re-injection. The exact flow for re-running validation after fixes was not fully traced: it may involve resetting the milestone's validation-planned status or the runner may skip the check because validators are already present.
- **Module 2478, 2553, 0930**: As noted above, these modules from the feature spec do not contain milestone/progress logic. The spec's module list appears stale; the actual progress-tracking modules are 2201, 2542, 2176, 3978, 3970, and 0087.
