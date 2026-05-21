# Protocol: Feature State Machine

## Status: DEFINITIVE (cross-validated)

Cross-referenced observed protocol samples from `protocol-samples/runtime-findings.md` (sanitized features.json from sample mission with 15+ features, 11 handoff files) with architecture notes from `mission-feature-registry.md` (Zod schemas in 0082.js, MissionFileService CRUD in 2201.js, MissionRunner in 2542.js, TUI rendering in 3971.js/3973.js) and `mission-state-machine.md` (dual-layer state machine in 2542.js, state recovery, validation injection).

---

## JSON Schema

### features.json (Top-Level)

```typescript
interface FeaturesFile {
  features: Feature[];
}
```

The file contains a single top-level `features` array. Each entry is a `Feature` object.

### Feature Object (RE: 0082.js Zod schema `pNH`, Runtime: confirmed)

```typescript
interface Feature {
  id: string;                          // REQUIRED: unique kebab-case identifier
  description: string;                 // REQUIRED: detailed description of what to implement
  skillName: string;                   // REQUIRED: which skill/droid handles this feature
  preconditions: string[];             // REQUIRED: what must be true before starting
  expectedBehavior: string[];          // REQUIRED: success criteria list
  verificationSteps: string[];         // REQUIRED: commands/steps to verify (can be [])
  fulfills: string[];                  // OPTIONAL: validation assertion IDs (can be []/omitted)
  milestone: string;                   // OPTIONAL: vertical slice grouping
  status: FeatureStatus;               // REQUIRED: current execution state
  workerSessionIds: string[];          // OPTIONAL: all session IDs that worked on this feature
  currentWorkerSessionId: string|null; // OPTIONAL: currently active worker session
  completedWorkerSessionId: string|null; // OPTIONAL: session that completed the feature
}
```

### FeatureStatus Enum

```typescript
type FeatureStatus = "pending" | "in_progress" | "completed" | "cancelled";
```

⚠️ **No `"failed"` status exists in the runtime.** Failed features are reset to `"pending"` for re-execution. The original feature description mentioned `"failed"` as a state but neither RE code nor observed protocol samples supports it.

---

## Field Documentation

| # | Field | Type | Required | Default | Source | Notes |
|---|-------|------|----------|---------|--------|-------|
| 1 | `id` | string | Yes |: | Analysis+Samples | Unique kebab-case identifier (e.g. `"feature-alpha"`, `"feature-beta-fix"`, `"scrutiny-validator-foundation"`) |
| 2 | `description` | string | Yes |: | Analysis+Samples | Detailed description of what to implement. Used by worker as primary instructions. |
| 3 | `skillName` | string | Yes |: | Analysis+Samples | Which skill/droid handles this feature. Observed values: `"frontend-worker"`, `"backend-worker"`, `"scrutiny-validator"`, `"user-testing-validator"`, `"protocol-analyzer"`. |
| 4 | `preconditions` | string[] | Yes | `[]` | Analysis+Samples | List of conditions that must be met before starting. Can be empty `[]`. RE (2201.js) normalizes missing to `[]`. |
| 5 | `expectedBehavior` | string[] | Yes | `[]` | Analysis+Samples | Expected outcomes after completion. Can be empty `[]`. |
| 6 | `verificationSteps` | string[] | Yes | `[]` | Analysis+Samples | Commands or manual steps to verify. Can be empty `[]`. Validator features typically have `[]`. |
| 7 | `fulfills` | string[] | No | omitted | Analysis+Samples | Array of validation assertion IDs (e.g. `["VAL-SAT-001", "VAL-SAT-002"]`). Maps to `validation-state.json` keys. Each assertion ID appears in exactly one feature's `fulfills`. Infrastructure features may have empty or omitted `fulfills`. |
| 8 | `milestone` | string | No | omitted | Analysis+Samples | Milestone this feature belongs to (e.g. `"foundation"`, `"data-layers"`, `"shaders"`, `"protocol-reports"`). Not all features have a milestone. |
| 9 | `status` | enum | Yes | `"pending"` | Analysis+Samples | Current execution state. One of: `"pending"`, `"in_progress"`, `"completed"`, `"cancelled"`. RE normalizes missing to `"pending"`. |
| 10 | `workerSessionIds` | string[] | No | `[]` | Analysis+Samples | Accumulated array of ALL worker session IDs that have worked on this feature. Grows across retries/iterations. E.g. `["6d65c86c-...", "559200b2-..."]` for a feature that was worked on twice. |
| 11 | `currentWorkerSessionId` | string\|null | No | `null` | Analysis+Samples | UUID of the currently active worker, or `null` if no worker is assigned. Set on spawn, cleared on completion/failure. |
| 12 | `completedWorkerSessionId` | string\|null | No | `null` | Analysis+Samples | Session ID that completed the feature. ⚠️ Never observed as non-null in observed protocol samples. Possibly set only on final completion or may be unused. |

---

## Examples

### Example 1: Pending Implementation Feature (Sanitized from Runtime)

Source: `protocol-samples/runtime-findings.md`: features.json entry

```json
{
  "id": "feature-alpha",
  "description": "Implement item tracking: fetch external dataset from external API via backend proxy...",
  "skillName": "frontend-worker",
  "preconditions": ["interactive canvas renders", "backend proxy for /api/items returns external dataset data"],
  "expectedBehavior": ["180+ items visible on canvas...", "Click item shows cyan orbit track polyline..."],
  "verificationSteps": ["npm test: --grep 'item'", "Manual: Click sample item, verify orbit track + info panel", "curl http://localhost:3001/api/items returns 180+ records"],
  "fulfills": ["VAL-SAT-001", "VAL-SAT-002", "VAL-SAT-003"],
  "milestone": "data-layers",
  "status": "pending",
  "workerSessionIds": [],
  "currentWorkerSessionId": null,
  "completedWorkerSessionId": null
}
```

**Key observations:**
- `workerSessionIds` is `[]`: no workers have attempted this feature yet
- `currentWorkerSessionId` and `completedWorkerSessionId` are both `null`
- `fulfills` links to 3 validation assertions
- `verificationSteps` includes both automated and manual checks

### Example 2: Completed Feature with Multiple Worker Sessions (Sanitized from Runtime)

Source: `protocol-samples/runtime-findings.md` + `features.json` (current mission)

```json
{
  "id": "protocol-handoff",
  "description": "Cross-reference runtime handoff JSON data with RE handoff analysis...",
  "skillName": "protocol-analyzer",
  "preconditions": [
    "protocol-samples/runtime-findings.md exists with Handoff JSON Schema section",
    "mission-handoff.md exists with handoff architecture notes",
    "mission-worker-lifecycle.md exists with worker lifecycle notes"
  ],
  "expectedBehavior": [
    "Report at protocol-handoff.md with status DEFINITIVE",
    "Complete handoff JSON schema with all 15+ fields documented",
    "At least one sanitized handoff example from observed protocol samples",
    "Discrepancy table capturing all Inferred vs Observed differences",
    "Worker lifecycle connection documented (start → work → commit → handoff)"
  ],
  "verificationSteps": [
    "Verify protocol-handoff.md exists and is under 800 lines",
    "Verify report has sections: JSON Schema, Field Documentation, Examples, Discrepancies",
    "Verify at least one example matches runtime-findings.md sanitized"
  ],
  "fulfills": ["VAL-HANDOFF-001", "VAL-HANDOFF-002", "VAL-HANDOFF-003", "VAL-HANDOFF-004"],
  "milestone": "protocol-reports",
  "status": "completed",
  "workerSessionIds": [
    "<feature-id-1>",
    "<feature-id-2>"
  ],
  "currentWorkerSessionId": null,
  "completedWorkerSessionId": null
}
```

**Key observations:**
- `workerSessionIds` contains 2 entries: this feature was worked on by two different worker sessions (likely one initial attempt + one retry/continuation)
- `status` is `"completed"` despite having 2 worker sessions (one may have been a failure that was reset)
- `completedWorkerSessionId` is still `null` even though completed: ⚠️ this field may be unused or only set in specific conditions
- `currentWorkerSessionId` is `null`: no active worker

### Example 3: In-Progress Feature (from Current Session)

Source: Current mission `features.json`

```json
{
  "id": "protocol-feature-state",
  "description": "Cross-reference runtime features.json with RE feature registry analysis...",
  "skillName": "protocol-analyzer",
  "preconditions": [
    "protocol-samples/runtime-findings.md exists with Feature Registry section",
    "mission-feature-registry.md exists",
    "mission-state-machine.md exists"
  ],
  "expectedBehavior": [
    "Report at protocol-feature-state.md with status DEFINITIVE",
    "Complete features.json schema with all 12+ fields documented",
    "Feature state machine diagram/table with all states and transitions",
    "Dynamic feature creation mechanism documented with runtime example"
  ],
  "verificationSteps": [
    "Verify protocol-feature-state.md exists and is under 800 lines",
    "Verify state machine section covers pending, in_progress, completed, failed, cancelled",
    "Verify dynamic feature creation section with feature-beta-fix example"
  ],
  "fulfills": ["VAL-FEATURE-001", "VAL-FEATURE-002", "VAL-FEATURE-003"],
  "milestone": "protocol-reports",
  "status": "in_progress",
  "workerSessionIds": [
    "<worker-session-id>"
  ],
  "currentWorkerSessionId": null,
  "completedWorkerSessionId": null
}
```

**Key observations:**
- `status` is `"in_progress"`: worker is currently executing
- `workerSessionIds` has 1 entry: the current worker
- `currentWorkerSessionId` is `null` (interesting: it should be set when worker is spawned, but may be set/cleared at different times in the lifecycle)

### Example 4: Validator Feature Pattern

Source: Runtime observations of scrutiny/user-testing validator features

Validator features follow a specific pattern:
- `skillName`: `"scrutiny-validator"` or `"user-testing-validator"`
- `verificationSteps`: `[]`: validators use their own internal logic
- `fulfills`: `[]`: validators test assertions across all features, not specific ones
- `milestone`: The milestone being validated (e.g. `"foundation"`)
- Auto-injected by `checkMilestoneCompletionAndInjectValidation()` after all implementation features complete

---

## State Machine: Feature Lifecycle

### States

| State | Icon (TUI) | Color | Meaning | Persistence |
|-------|------------|-------|---------|-------------|
| `pending` | ○ | text.muted | Feature queued, awaiting execution | Default for new features |
| `in_progress` | ● | primary | Worker currently executing this feature | Set on spawn, cleared on completion/failure |
| `completed` | ✓ | success | Feature finished successfully | Terminal state |
| `cancelled` | ✗ | error | Feature cancelled by user/orchestrator | Terminal state |

⚠️ **No `"failed"` state.** When a worker fails, the feature is reset to `"pending"` for re-execution. This is a deliberate design choice: the system retries failed features automatically.

### Transition Diagram

```
                    ┌──────────────────────────────────────────┐
                    │         Orchestrator / User               │
                    │  (scope change, obsolete feature)         │
                    ▼                                          │
┌──────────┐  spawn worker  ┌──────────────┐   success +     ┌───────────┐
│ pending  │ ──────────────▶│ in_progress  │ ──────────────▶ │ completed │
└──────────┘                └──────────────┘  clean handoff   └───────────┘
     ▲                           │                                 │
     │                           │                                 │
     │      ┌────────────────────┤                                 │
     │      │                    │                                 │
     │      │ worker fail/       │ success with                   │
     │      │ crash/timeout      │ discoveredIssues               │
     │      │                    │ or whatWasLeftUndone            │
     │      ▼                    ▼                                 │
     │  resetFeature()    Feature stays pending                    │
     │  (S9H in 2542.js)  State → orchestrator_turn               │
     │  status → "pending"                                        │
     │                                                             │
     └─────────────────────────────────────────────────────────────┘
                    moveToBottom()

┌──────────┐
│ cancelled│  (terminal: from "pending" only, via orchestrator/user)
└──────────┘
     │
     ▼
  moveToBottom()
```

### Transition Table

| # | From | To | Trigger | Module (RE) | Side Effects |
|---|------|----|---------|-------------|-------------|
| 1 | `pending` | `in_progress` | `spawnWorker()` selects feature | 2542.js:433-442 | `currentWorkerSessionId` set, `workerSessionIds[]` appended with new session ID, `worker_started` logged |
| 2 | `in_progress` | `completed` | Worker calls `EndFeatureRun` with `successState: "success"`, no issues, no unfinished work | 2542.js:303-320 | `moveFeatureToBottom()`, `worker_completed` logged, handoff file written |
| 3 | `in_progress` | `pending` | Worker spawn fails or crashes | 2542.js:14-22 (`S9H`) | `currentWorkerSessionId` set to `null`, `worker_failed` logged, feature requeued |
| 4 | `in_progress` | `pending` | Worker success but `returnToOrchestrator=true` OR has discoveredIssues OR has whatWasLeftUndone | 2544.js (handoff gating) | Feature NOT marked completed, stays pending for orchestrator review, state → `orchestrator_turn` |
| 5 | `pending` | `cancelled` | User or orchestrator cancels feature (scope change, obsolete) | 2175.js:1109-1113 | `moveFeatureToBottom()`, assertions removed from validation-state.json |
| 6 | `completed` | any | **Not allowed**: terminal state |: | Completed features never transition again |
| 7 | `cancelled` | any | **Not allowed**: terminal state |: | Cancelled features never transition again |

### Key Transition Details

**Transition 1: pending → in_progress (Worker Spawn)**
```
spawnWorker() in MissionRunner (2542.js):
1. Generate spawnId: "worker_{random8hex}"
2. Call getNextPendingFeature() → first feature with status === "pending"
3. Create worker session via daemon API
4. Update feature: { status: "in_progress", currentWorkerSessionId: sessionUUID, workerSessionIds: [...existing, sessionUUID] }
5. Log worker_selected_feature + worker_started events
6. Send bootstrap prompt to worker session
```

**Transition 3: in_progress → pending (Failure Recovery)**
```
S9H() in 2542.js:
1. Check feature exists and status === "in_progress"
2. Verify currentWorkerSessionId matches (prevents wrong-session reset)
3. Update feature: { status: "pending", currentWorkerSessionId: null }
4. Feature re-enters queue for next spawn cycle
```

⚠️ The workerSessionIds array is NOT cleared on failure: it retains history of all attempts, enabling retry tracking.

**Transition 4: in_progress → pending (Success with Issues)**
```
When EndFeatureRun is called with:
- successState: "success" BUT
- returnToOrchestrator: true, OR
- discoveredIssues: non-empty, OR
- whatWasLeftUndone: non-empty

→ Feature is NOT marked completed
→ Feature stays in "pending" (or is reset to pending)
→ Mission state → "orchestrator_turn"
→ Orchestrator must review and address items
```

---

## Feature Ordering Mechanics

### Selection Algorithm

```typescript
// 2201.js:174-177
async getNextPendingFeature(): Feature | null {
  const features = await this.readFeatures();
  if (!features) return null;
  return features.features.find(f => f.status === "pending") ?? null;
}
```

**Array order is the priority order.** The first feature with `status === "pending"` is selected next.

### Re-ordering Operations

| Operation | Method | When | Effect |
|-----------|--------|------|--------|
| Insert at top | `insertFeatureAtTop()` | Urgent/fix features created | Feature placed at index 0 (highest priority) |
| Move to bottom | `moveFeatureToBottom()` | Feature completed/cancelled | Feature moved to end of array |
| Compact stranded | `moveStrandedDoneFeaturesToBottom()` | After feature array changes | Completed/cancelled features that appear mid-array are compacted to bottom, preserving relative order of active features |

### Priority FIFO System

```
Initial order:
  [feature-A: pending, feature-B: pending, feature-C: pending]

After feature-A completes:
  [feature-B: pending, feature-C: pending, feature-A: completed]

After scrutiny finds bug → fix feature inserted at top:
  [feature-X-fix: pending, feature-B: pending, feature-C: pending, feature-A: completed]

Execution order: feature-X-fix → feature-B → feature-C
```

This implements a **FIFO-with-priority** system: normal features run in array order, but urgent fixes jump to the front.

---

## Milestone Completion & Validation Injection

### Milestone Completion Detection

```typescript
// 2201.js:237-244
async isMilestoneImplementationComplete(milestone: string): boolean {
  const features = await this.getMilestoneFeatures(milestone);
  if (features.length === 0) return false;
  const nonValidators = features.filter(f => !HMH.includes(f.skillName)); // exclude validators
  if (nonValidators.length === 0) return false;
  return nonValidators.every(f => f.status === "completed" || f.status === "cancelled");
}
```

A milestone is "implementation complete" when ALL non-validator features are in a terminal state (`completed` or `cancelled`).

### Validation Auto-Injection

After milestone implementation completes, `checkMilestoneCompletionAndInjectValidation()` (2542.js:309-360) automatically:

1. Checks if `milestonesWithValidationPlanned` already includes this milestone (guard against duplicate injection)
2. Creates `scrutiny-validator-{milestone}` feature with:
   - `skillName`: `"scrutiny-validator"`
   - `milestone`: the completed milestone name
   - `verificationSteps`: `[]`
   - `fulfills`: `[]`
   - Inserted at top of feature array
3. Creates `user-testing-validator-{milestone}` feature with:
   - `skillName`: `"user-testing-validator"`
   - `milestone`: the completed milestone name
   - `verificationSteps`: `[]`
   - `fulfills`: `[]`
   - Inserted at top of feature array (above scrutiny)
4. Marks milestone as validation-planned via `markMilestoneValidationPlanned()`

**Execution order:** scrutiny-validator runs first, then user-testing-validator (both inserted at top, so LIFO insertion order means user-testing is above scrutiny, but scrutiny was inserted last and thus appears first in array after insertion: depends on exact insertion order in code).

### Sealed Milestones

After both validators pass (scrutiny → user-testing), the milestone is **sealed** (2175.js:1114):
- No new features can be added to a sealed milestone
- Follow-up work must go into a new milestone or `misc-*` milestone
- This prevents scope creep after validation

---

## Dynamic Feature Creation

### Mechanism

The orchestrator can dynamically create new features during mission execution. This happens in two scenarios:

1. **Bug fix features**: When scrutiny validation finds issues in a completed feature, the orchestrator creates a fix feature (e.g. `feature-beta-fix`) to address the bugs. The fix feature is inserted at the **top** of the feature array via `insertFeatureAtTop()` for highest priority.

2. **Scope change features**: When the user requests changes mid-mission, the orchestrator creates new features and inserts them at the appropriate position.

### Real Runtime Example: feature-beta-fix

Source: `protocol-samples/runtime-findings.md`: Key Findings section

> "The orchestrator can CREATE new features dynamically (e.g., `feature-beta-fix` was created after scrutiny found bugs)"

The flow:
1. Worker completes `feature-beta` feature → status: `completed`
2. Milestone reaches implementation completion → scrutiny validation triggered
3. Scrutiny validator runs → finds bugs in canvas fly-to functionality
4. Scrutiny validator returns `returnToOrchestrator: true` with `discoveredIssues`
5. Orchestrator reviews issues → creates `feature-beta-fix` feature
6. Fix feature inserted at top of feature array (highest priority)
7. Fix feature executed → addresses the bugs → status: `completed`
8. Validation re-triggered for the milestone

### Dynamic Feature Naming Convention

Fix features typically follow the pattern: `{original-feature-id}-fix`

Examples:
- `feature-beta` → `feature-beta-fix`
- Validator features: `scrutiny-validator-{milestone}`, `user-testing-validator-{milestone}`

### Feature Creation Rules

From orchestrator prompt (2175.js):
- Each assertion ID must appear in exactly one feature's `fulfills` across the entire features.json
- New features for a sealed milestone are forbidden: use a follow-up milestone instead
- When creating fix features, the `fulfills` should claim any assertions that failed validation
- Fix features should reference the original feature in their description

---

## Normalization

### Feature Normalization on Read (RE: 2201.js:88-107)

When `readFeatures()` is called, each feature is normalized:

```typescript
normalizeFeature(feature: Partial<Feature>): Feature {
  return {
    id: feature.id,
    description: feature.description ?? "",
    skillName: feature.skillName ?? "",
    status: feature.status ?? "pending",        // default to pending
    preconditions: feature.preconditions ?? [],  // default to empty array
    expectedBehavior: feature.expectedBehavior ?? [],
    verificationSteps: feature.verificationSteps ?? [],
    fulfills: feature.fulfills ?? [],            // default to empty array
    milestone: feature.milestone ?? undefined,
    workerSessionIds: feature.workerSessionIds ?? [], // default to empty array
    currentWorkerSessionId: feature.currentWorkerSessionId ?? null,
    completedWorkerSessionId: feature.completedWorkerSessionId ?? null,
  };
}
```

This ensures that features with missing optional fields are always fully populated when consumed by the runtime.

---

## Discrepancies (Inferred vs Observed)

| # | Field/Behavior | Architecture Notes Says | Observed Samples Show | Resolution |
|---|----------------|------------------|-------------------|------------|
| 1 | `"failed"` as a feature status | **Not present** in Zod enum `j7` (0082.js) or TUI rendering (3973.js). Only 4 states: pending, in_progress, completed, cancelled | Runtime confirms only 4 states observed. Failed features reset to `"pending"`. | ⚠️ Feature description mentions "failed" as a state, but neither RE code nor runtime supports it. Failed features are **always** reset to `"pending"` for automatic retry. `"failed"` does NOT exist as a feature status. |
| 2 | `completedWorkerSessionId` | Defined in Zod schema as `AH.string().nullable().optional()` | Never observed as non-null in observed protocol samples. Even completed features show `null`. | ⚠️ Field exists in schema but appears unused in practice. Possibly intended for future use or set only in specific conditions not observed. |
| 3 | `currentWorkerSessionId` timing | Set during `spawnWorker()` and cleared on completion | Current session shows `in_progress` feature with `currentWorkerSessionId: null` | ⚠️ Timing mismatch: the field may be cleared before status is observable externally, or there's a race condition between spawn and status propagation. |
| 4 | Feature status values in runtime | RE says `pending`, `in_progress`, `completed`, `cancelled` | Runtime shows `pending`, `in_progress`, `completed`. `"cancelled"` not directly observed in SampleApp data (all features were either pending, in-progress, or completed). | `"cancelled"` is valid but not triggered in this mission. RE code and TUI rendering confirm it as a valid state. |
| 5 | `fulfills` optionality | RE Zod marks `.optional()`, meaning the field itself can be omitted | Observed protocol samples shows `fulfills` always present (even as empty `[]`) | RE normalization defaults to `[]`, so in practice the field is always present after reading. The Zod `.optional()` refers to the raw JSON, not the normalized form. |
| 6 | `milestone` optionality | RE Zod marks `.optional()` | Some features in runtime lack a milestone. Validator features in the current mission DO have milestones. | Field is truly optional. Not all features belong to a milestone. |
| 7 | Dynamic feature creation mechanism | RE describes `insertFeatureAtTop()` and orchestrator creating fix features | Runtime confirms `feature-beta-fix` was dynamically created | Both sources agree. The orchestrator has the ability to add features at any point during execution. |
| 8 | Validation injection order | RE (2542.js:309-360) inserts scrutiny first, then user-testing | Runtime shows `milestone_validation_triggered` event with `featureId: "scrutiny-validator-foundation"` | ⚠️ Insertion order in code: scrutiny inserted last (at top), user-testing inserted before it. But since both are inserted at top, user-testing ends up below scrutiny in the array. Execution order: scrutiny → user-testing. |
| 9 | `HMH` validator exclusion list | Referenced in RE (2201.js:237) as validator skill name exclusion list, but exact definition not traced | Runtime shows validator features with `skillName: "scrutiny-validator"` and `"user-testing-validator"` | `HMH` likely contains `["scrutiny-validator", "user-testing-validator"]` or similar patterns. These skill names are excluded from milestone completion checks. |

---

## Feature Schema vs Worker Session Tracking

The feature object tracks worker sessions through three fields:

```
workerSessionIds: ["sess-1", "sess-2", "sess-3"]  ← All attempts (accumulated)
currentWorkerSessionId: null                        ← Active worker (null when idle)
completedWorkerSessionId: null                      ← Final worker (never observed set)
```

### Session Tracking Lifecycle

```
Feature created:
  workerSessionIds: []
  currentWorkerSessionId: null
  completedWorkerSessionId: null

Worker 1 spawned:
  workerSessionIds: ["sess-1"]
  currentWorkerSessionId: "sess-1"
  completedWorkerSessionId: null

Worker 1 crashes → reset:
  workerSessionIds: ["sess-1"]      ← NOT cleared (retains history)
  currentWorkerSessionId: null       ← Cleared
  completedWorkerSessionId: null

Worker 2 spawned (retry):
  workerSessionIds: ["sess-1", "sess-2"]  ← Appended
  currentWorkerSessionId: "sess-2"
  completedWorkerSessionId: null

Worker 2 completes successfully:
  workerSessionIds: ["sess-1", "sess-2"]  ← Retained
  currentWorkerSessionId: null             ← Cleared
  completedWorkerSessionId: null           ← ⚠️ Still null (possibly unused)
```

This design allows the orchestrator to see the full retry history of each feature.

---

## Open Questions

1. **`completedWorkerSessionId` usage**: This field is defined in the Zod schema and present in observed protocol samples but never observed as non-null. Is it set only in specific completion paths? Is it used by the TUI for session navigation? Or is it vestigial? Marked as [UNUSED-POSSIBLY].

2. **Validator execution order**: The RE code inserts both scrutiny and user-testing validators at the top of the array. The exact execution order depends on insertion sequence. Clarification needed: does scrutiny always run before user-testing, or can the order vary?

3. **Feature cancellation scope**: RE says only `pending` features can be cancelled, but this is not explicitly enforced in the Zod schema. Can an `in_progress` feature be cancelled? The TUI would need to kill the active worker first.

4. **Max retries**: No evidence of a maximum retry count for features that repeatedly fail. The `workerSessionIds` array grows unboundedly with each retry attempt. Is there a circuit breaker?

5. **Concurrent worker limit**: architecture notes suggests the MissionRunner spawns one worker at a time (sequential execution). But the current mission features.json shows `currentWorkerSessionId: null` for an in-progress feature: is there a parallel execution mode?
