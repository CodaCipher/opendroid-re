# mission-validation-contract: Mission System RE

## Overview

The validation contract system is OpenDroid's quality gate for mission execution. It operates on a three-file artifact model (`validation-contract.md`, `validation-state.json`, `features.json`) authored during mission planning, then enforced automatically at milestone boundaries. When all implementation features in a milestone complete, the `MissionRunner` (Module 2542.js) auto-injects two validation features: `scrutiny-validator` and `user-testing-validator`: which spawn subagents to verify code quality and user-facing behavior respectively. The system tracks assertion-level pass/fail/blocked state in `validation-state.json`, enabling re-run-on-failure semantics, milestone sealing, and an end-of-mission gate requiring all assertions to be `"passed"`.

## Module Map

| Module | Size (KB) | Role | Notes |
|--------|-----------|------|-------|
| 2542.js | 19.0 | MissionRunner core | Contains `checkMilestoneCompletionAndInjectValidation()`, auto-injection logic, skip flags |
| 2541.js | 6.6 | Mission prompt templates | Worker system prompt with fulfills/assertion context; `VNI()`, `GNI()` builders |
| 2175.js | 135.8 | Validation skill prompts | Full scrutiny-validator + user-testing-validator skill text; validation contract documentation |
| 2176.js | 4.8 | Built-in droids registry | Defines `sfH`/`efH` constants, `HMH` array, `L7I` built-in skill droids |
| 2201.js | 8.5 | MissionFileService | `insertFeatureAtTop()`, `isMilestoneImplementationComplete()`, `hasValidationPlannerRun()`, feature schema normalization |
| 0082.js | 3.8 | Zod validation schemas | Feature schema (`pNH`), handoff schema (`h$H`), progress log event types including `milestone_validation_triggered` |
| 3383.js | 6.1 | TUI mission rendering | `UIM = ["scrutiny-validator", "user-testing-validator"]` for display; milestone rendering |
| 3978.js | 5.0 | TUI progress events | Renders `milestone_validation_triggered` events in TUI |
| 1184.js | 6.4 | Droid config validation | `k5.validate()`: validates droid metadata (name, tools, model), NOT mission validation contracts |
| 2518.js | 5.8 | Tool classification | `Z9f()` tool category mapping: related to droid tools, not validation contracts |
| 3382.js | 5.6 | Shell executor | PowerShell/cmd background process spawner: infrastructure, not validation |
| 2335.js | 5.6 | Shell executor errors | Background process error handling: infrastructure, not validation |

## Architecture / Flow

### Three-File Artifact Model

The validation contract system revolves around three files in `missionDir`:

1. **`validation-contract.md`**: Markdown document defining testable assertions grouped by feature area. Each assertion has a unique ID (e.g., `VAL-AUTH-001`). Created by the orchestrator during mission planning using a subagent-per-area strategy with multiple review passes.

2. **`validation-state.json`**: JSON file tracking assertion status:
   ```json
   {
     "assertions": {
       "VAL-AUTH-001": { "status": "pending" },
       "VAL-AUTH-002": { "status": "passed" },
       "VAL-CROSS-001": { "status": "failed" }
     }
   }
   ```
   Initialized with all assertions as `"pending"`. Updated by user-testing-validator synthesis with `"passed"`, `"failed"`, or `"blocked"` results.

3. **`features.json`**: Feature array where each feature has a `fulfills` field listing assertion IDs it completes. Coverage gate: every assertion ID in `validation-contract.md` must be claimed by exactly one feature's `fulfills`.

### Assertion Schema

Assertions are defined in `validation-contract.md` with IDs following the pattern `VAL-{AREA}-{SEQ}` (e.g., `VAL-AUTH-001`). Key properties:

- **Area-grouped**: Per-feature assertions + cross-feature assertions
- **Status lifecycle**: `pending` → `passed` | `failed` | `blocked`
- **`fulfills` semantics**: "completes" not "contributes to": only the leaf feature that makes an assertion fully testable claims it
- **Uniqueness**: Each assertion ID appears in exactly one feature's `fulfills` across the entire `features.json`

The Zod schema for features (Module 0082.js, line 13):
```js
fulfills: AH.array(AH.string()).optional()
```

### Milestone Completion Detection

When a worker completes a feature, `MissionRunner` (Module 2542.js, line 307) calls:

```
this.checkMilestoneCompletionAndInjectValidation(featureId)
```

This function (lines 316-383) performs:

1. **Get feature's milestone**: `A = await this.missionFileService.getFeature(H)`
2. **Check implementation complete**: `isMilestoneImplementationComplete(L)` filters out validator skill names (`HMH = [sfH, efH]`) and checks all remaining features are `completed` or `cancelled`
3. **Check if validation already planned**: `hasValidationPlannerRun(L)` checks `state.milestonesWithValidationPlanned`
4. **If all checks pass**: Inject validation features

(Module 2542.js, lines 316-383; Module 2201.js, lines 235-252)

### Validator Auto-Injection

The auto-injection creates two feature objects dynamically (Module 2542.js, lines 318-383):

**User-testing feature:**
```js
{
  id: `${efH}-${L}`,  // "user-testing-validator-{milestone}"
  description: `User testing validation for milestone "${L}"...`,
  skillName: efH,     // "user-testing-validator"
  preconditions: [`All implementation features for milestone "${L}" are complete`],
  expectedBehavior: [
    "Testable assertions determined from fulfills mapping",
    "Environment set up (services started, data seeded)",
    "Flow validator subagents spawned and completed",
    "Results synthesized, validation-state.json updated"
  ],
  verificationSteps: [],
  milestone: L,
  status: "pending"
}
```

**Scrutiny feature:**
```js
{
  id: `${sfH}-${L}`,  // "scrutiny-validator-{milestone}"
  description: `Scrutiny validation for milestone "${L}"...`,
  skillName: sfH,     // "scrutiny-validator"
  preconditions: [`All implementation features for milestone "${L}" are complete`],
  expectedBehavior: [
    "Validators pass (test, typecheck, lint)",
    "Review subagents spawned for each feature",
    "Findings synthesized into scrutiny report"
  ],
  verificationSteps: [],
  milestone: L,
  status: "pending"
}
```

**Skip flags** (Module 2542.js, lines 361-368):
```js
let U = await this.missionFileService.readModelSettings(),
    P = vA(),
    B = U?.skipScrutiny ?? P.getMissionSkipScrutiny(),
    W = U?.skipUserTesting ?? P.getMissionSkipUserTesting();
if (B && W) {
  // Skip all validation (experimental)
  await this.missionFileService.markMilestoneValidationPlanned(L);
  return;
}
if (!W) await this.missionFileService.insertFeatureAtTop(userTestingFeature);
if (!B) await this.missionFileService.insertFeatureAtTop(scrutinyFeature);
```

**Insertion order**: User-testing is inserted first, then scrutiny is inserted at top. Since both use `insertFeatureAtTop()`, scrutiny ends up at the TOP of the queue (it's inserted after user-testing, pushing it above). So **scrutiny runs before user-testing**.

**Progress event** emitted after injection (Module 0082.js, line 101):
```js
{ type: "milestone_validation_triggered", milestone: L, featureId: V }
```

### Scrutiny Validator

**When injected**: After all implementation features in a milestone complete.

**What it checks** (Module 2175.js, lines ~1824-1920, full scrutiny-validator skill):

1. **Hard gate validators**: Runs test suite, typecheck, and lint from `.opendroid/services.yaml`. If any fails → immediate `EndFeatureRun` with failure.
2. **Feature review**: Spawns one `scrutiny-feature-reviewer` subagent per completed implementation feature via Task tool.
3. **Re-run logic**: On re-run, reads prior `synthesis.json` to find `failedFeatures`, only reviews fix features.
4. **Synthesis**: Determines pass/fail, triages shared state observations, writes to `.opendroid/validation/<milestone>/scrutiny/synthesis.json`.

**Report output** (`.opendroid/validation/<milestone>/scrutiny/synthesis.json`):
```json
{
  "milestone": "<milestone>",
  "round": 1,
  "status": "pass" | "fail",
  "validatorsRun": {
    "test": { "passed": true, "command": "...", "exitCode": 0 },
    "typecheck": { "passed": true, "command": "...", "exitCode": 0 },
    "lint": { "passed": true, "command": "...", "exitCode": 0 }
  },
  "reviewsSummary": {
    "total": 5,
    "passed": 4,
    "failed": 1,
    "failedFeatures": ["checkout-reserve-inventory"]
  },
  "blockingIssues": [...],
  "appliedUpdates": [...],
  "suggestedGuidanceUpdates": [...],
  "rejectedObservations": [...],
  "previousRound": null
}
```

**Always returns to orchestrator** (`returnToOrchestrator: true`).

### User-Testing Validator

**When injected**: After scrutiny validator completes (both auto-injected, scrutiny runs first).

**What it checks** (Module 2175.js, lines ~1958-2185, full user-testing-validator skill):

1. **Determine testable assertions**: Collects `fulfills` arrays from completed features in milestone, cross-references with `validation-state.json` for `"pending"` assertions only.
2. **Setup**: Starts services, seeds data, resolves setup issues. Updates `.opendroid/library/user-testing.md`.
3. **Isolation**: Assigns unique test accounts/data namespaces per flow validator subagent.
4. **Flow validation**: Spawns `user-testing-flow-validator` subagents in parallel via Task tool.
5. **Synthesis**: Determines pass/fail/blocked per assertion, updates `validation-state.json`.

**Assertion status update rules** (Module 2175.js, line 2104-2112):
- `pass` → set status to `"passed"`, record `validatedAtMilestone`
- `fail` → set status to `"failed"`, record issues
- `blocked` → set status to `"failed"`, record blocking reason

**Report output** (`.opendroid/validation/<milestone>/user-testing/synthesis.json`):
```json
{
  "milestone": "<milestone>",
  "round": 1,
  "status": "pass" | "fail",
  "assertionsSummary": {
    "total": 10,
    "passed": 8,
    "failed": 1,
    "blocked": 1
  },
  "passedAssertions": ["VAL-AUTH-001", "VAL-AUTH-002"],
  "failedAssertions": [
    { "id": "VAL-CHECKOUT-003", "reason": "Payment form validation missing" }
  ],
  "blockedAssertions": [
    { "id": "VAL-DASHBOARD-001", "blockedBy": "Login broken" }
  ],
  "appliedUpdates": [...],
  "previousRound": null
}
```

**Always returns to orchestrator** (`returnToOrchestrator: true`).

### Pass/Fail/Blocked State Management

**Assertion states** (in `validation-state.json`):
- `"pending"`: Initial state, not yet tested
- `"passed"`: Behavior confirmed working
- `"failed"`: Behavior doesn't match specification OR prerequisite broken (blocked maps to failed)
- `"blocked"`: Not a separate state in `validation-state.json`; blocked assertions are recorded as `"failed"` with blocking reason

**Validator feature states** (in `features.json`):
- `"pending"` → `"running"` → `"completed"` | back to `"pending"` (on failure for re-run)
- Validators that fail go back to `"pending"` status so they re-run after fix features are completed

**Milestone sealing**: Once a milestone's validators pass, it is **sealed**. No features can be added to a sealed milestone. New work goes into follow-up milestones.

**End-of-mission gate** (Module 2175.js, line 1179):
> Before declaring mission complete, check `validation-state.json`. ALL assertions must be `"passed"`.

### Re-Run Semantics

When a validator fails:
1. Returns to orchestrator with failure details
2. Orchestrator spawns subagent to analyze failure
3. Fix features created at top of `features.json`
4. Same validator feature re-runs (it's still `"pending"`)
5. On re-run, validator reads previous `synthesis.json` and only validates what failed

**Scrutiny re-run**: Only reviews fix features + original failed features together.

**User-testing re-run**: Tests union of `failedAssertions`/`blockedAssertions` from prior synthesis + new assertions from fix features' `fulfills`.

## Key Findings

1. **Auto-injection is deterministic**: The `checkMilestoneCompletionAndInjectValidation()` function in Module 2542.js is the single point where validators are injected. It checks three conditions: (a) feature has a milestone, (b) all implementation features in milestone are complete, (c) validation hasn't been planned for this milestone yet.

2. **Validator skill names are constants**: `sfH` = `"scrutiny-validator"`, `efH` = `"user-testing-validator"`, `HMH = [sfH, efH]` (Module 2176.js, line 120). These are used both for injection filtering and for excluding validator features from "implementation complete" checks.

3. **Skip flags provide escape hatch**: Model settings can set `skipScrutiny` and `skipUserTesting` to bypass validation (experimental feature). Both must be true to skip entirely; individual validators can be skipped independently.

4. **Insertion order ensures scrutiny runs first**: Both features use `insertFeatureAtTop()`, but scrutiny is inserted second, so it ends up at the top of the queue.

5. **Blocked = Failed in state tracking**: There's no separate `"blocked"` status in `validation-state.json`. Blocked assertions are stored as `"failed"` with a blocking reason. The synthesis report tracks them separately for UX but the authoritative state file collapses them.

6. **Coverage gate is a pre-flight check**: The orchestrator must verify every assertion ID in `validation-contract.md` is claimed by exactly one feature's `fulfills` before starting mission execution. Unclaimed assertions = planning gap.

7. **Validator features are second-class**: They lack `fulfills` fields and `verificationSteps`, are excluded from `isMilestoneImplementationComplete()` checks, and always return to orchestrator.

## Code Examples

### Milestone Completion Detection & Auto-Injection (Module 2542.js, lines 316-383)

```js
async checkMilestoneCompletionAndInjectValidation(H) {
    let A = await this.missionFileService.getFeature(H);
    if (!A?.milestone) return;
    let L = A.milestone;
    if (!(await this.missionFileService.isMilestoneImplementationComplete(L))) return;
    if (await this.missionFileService.hasValidationPlannerRun(L)) return;

    // Create user-testing feature
    let D = `${efH}-${L}`, E = { id: D, skillName: efH, milestone: L, status: "pending", ... };
    // Create scrutiny feature
    let f = `${sfH}-${L}`, M = { id: f, skillName: sfH, milestone: L, status: "pending", ... };

    // Check skip flags
    let B = U?.skipScrutiny ?? P.getMissionSkipScrutiny();
    let W = U?.skipUserTesting ?? P.getMissionSkipUserTesting();
    if (B && W) { await this.missionFileService.markMilestoneValidationPlanned(L); return; }
    if (!W) await this.missionFileService.insertFeatureAtTop(E);
    if (!B) await this.missionFileService.insertFeatureAtTop(M);
    await this.missionFileService.markMilestoneValidationPlanned(L);
}
```

### Implementation Complete Check (Module 2201.js, lines 235-241)

```js
async isMilestoneImplementationComplete(H) {
    let A = await this.getMilestoneFeatures(H);
    if (A.length === 0) return false;
    let L = A.filter(($) => !HMH.includes($.skillName));  // Exclude validator skills
    if (L.length === 0) return false;
    return L.every(($) => $.status === "completed" || $.status === "cancelled");
}
```

### Feature Schema with fulfills (Module 2201.js, lines 113-130)

```js
static normalizeFeature(H) {
    return {
      id: H.id || "",
      description: H.description || "",
      skillName: H.skillName || "",
      preconditions: Array.isArray(H.preconditions) ? H.preconditions : [],
      expectedBehavior: Array.isArray(H.expectedBehavior) ? H.expectedBehavior : [],
      verificationSteps: Array.isArray(H.verificationSteps) ? H.verificationSteps : [],
      fulfills: Array.isArray(H.fulfills) ? H.fulfills : void 0,
      milestone: H.milestone,
      status: H.status || "pending",
      workerSessionIds: H.workerSessionIds || [],
      currentWorkerSessionId: H.currentWorkerSessionId ?? null,
      completedWorkerSessionId: H.completedWorkerSessionId ?? null,
    };
}
```

### Zod Feature Schema (Module 0082.js, lines 5-20)

```js
pNH = AH.object({
    id: AH.string(),
    description: AH.string(),
    status: AH.nativeEnum(j7),     // FeatureStatus enum
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

### Built-in Validator Droids (Module 2176.js, lines 87-110)

```js
L7I = [
    {
      metadata: { name: sfH, description: "Scrutiny validation: runs validators, spawns review subagents, synthesizes results. Auto-injected by system when milestone completes." },
      validationResult: { valid: true, errors: [], warnings: [] },
    },
    {
      metadata: { name: efH, description: "User testing validation: determines testable assertions, sets up env, spawns flow validators, synthesizes results. Auto-injected by system when milestone completes." },
      validationResult: { valid: true, errors: [], warnings: [] },
    },
],
(HMH = [sfH, efH]));
```

### Progress Event Emission (Module 0082.js, lines 101-104)

```js
af0 = uY.extend({
    type: AH.literal("milestone_validation_triggered"),
    milestone: AH.string(),
    featureId: AH.string(),
})
```

### Validation-State.json Schema (Module 2175.js, lines 719-733)

```json
{
  "assertions": {
    "VAL-AUTH-001": { "status": "pending" },
    "VAL-AUTH-002": { "status": "pending" },
    "VAL-CROSS-001": { "status": "pending" }
  }
}
```

## Integration Points

- **01-terminal-ui (TUI)**: Module 3383.js renders validator skill names in TUI via `UIM` array. Module 3978.js renders `milestone_validation_triggered` progress events. The TUI uses `milestone` fields from features for grouping.
- **03-tool-agent-system (Tool)**: Task tool (Module 2521.js) is used by validators to spawn subagents (`scrutiny-feature-reviewer`, `user-testing-flow-validator`). The subagent spawning infrastructure is tool-layer.
- **04-desktop-gui (GUI)**: Module 2201.js emits `project-notification` events (`mission_features_changed`, `mission_state_changed`) which the GUI layer likely consumes.
- **05-infrastructure (Infra)**: `opendroidd` daemon connectivity check in Module 2542.js (`await It()` before `OaH` spawn). Worker session spawning via `zX().spawnWorkerSession()`.

## Implementation Notes

To implement the validation contract system in OpenDroid:

1. **Three-file artifact model**: Implement `validation-contract.md` (Markdown), `validation-state.json` (JSON), `features.json` (JSON with `fulfills` arrays) as first-class mission artifacts.

2. **Milestone boundary detection**: Implement `checkMilestoneCompletionAndInjectValidation()` that fires when all non-validator features in a milestone reach terminal state. Key logic: filter features by `!HMH.includes(skillName)` to exclude validators.

3. **Validator auto-injection**: Dynamically create `scrutiny-validator-{milestone}` and `user-testing-validator-{milestone}` feature objects, insert at top of queue via `insertFeatureAtTop()`. Track planned state to prevent double-injection.

4. **Scrutiny validator**: Pipeline = run test/typecheck/lint → spawn per-feature review subagents → synthesize pass/fail → write `.opendroid/validation/{milestone}/scrutiny/synthesis.json`.

5. **User-testing validator**: Pipeline = collect pending assertions from `fulfills` → setup services → assign isolation resources → spawn flow validators in parallel → synthesize results → update `validation-state.json`.

6. **Re-run semantics**: On failure, validator stays `"pending"` in features.json. On re-run, read prior `synthesis.json` to scope work to failures only.

7. **End-of-mission gate**: Check all assertions in `validation-state.json` are `"passed"` before declaring mission complete.

8. **Skip flags**: Implement `skipScrutiny` and `skipUserTesting` in model settings for escape hatch.

## Known Gaps

- Module 1184.js (`k5` class) and Module 2518.js (`Z9f()`) are droid-level validation, not mission validation contracts: they were analyzed but found to be tangential. The feature spec listed them as targets, but their role is orthogonal to the mission validation contract system.
- Module 3382.js and Module 2335.js are shell executor infrastructure, not validation-related. Included in module map for completeness.
- The `mission-planning` skill's subagent-per-area assertion generation workflow was not traced to the code level: only documented from skill prompt text.
- The `dismiss_handoff_items` mechanism referenced in Module 0082.js (line 100, `dNH` schema) for handling validation overrides was not deeply traced.
- The `scrutiny-feature-reviewer` and `user-testing-flow-validator` subagent droids' internal logic was not analyzed: only their spawn pattern and output schema were documented.
- `validation-state.json` write path (how assertions are updated from `"failed"` back to `"pending"` for re-runs) was inferred from skill prompt text, not traced to a specific write function in the modules.
