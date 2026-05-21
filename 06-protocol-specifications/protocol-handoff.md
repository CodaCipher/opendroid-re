# Protocol: Handoff

## Status: DEFINITIVE (cross-validated)

Cross-referenced observed protocol samples from `protocol-samples/runtime-findings.md` (11 sanitized handoff samples) with architecture notes from `mission-handoff.md` (Zod schemas in 0082.js, 1234.js, 2539.js) and `mission-worker-lifecycle.md` (worker spawn/monitor/crash flows in 2542.js, 2543.js, 2544.js).

---

## JSON Schema

The handoff protocol has two layers: the **EndFeatureRun tool parameters** (submitted by worker) and the **persisted handoff file** (written to disk). Both share the same core structure but differ in envelope fields.

### EndFeatureRun Tool Parameters (RE: 1234.js `Qz$`)

```typescript
interface EndFeatureRunParams {
  successState: "success" | "partial" | "failure";    // REQUIRED
  returnToOrchestrator: boolean;                       // REQUIRED
  featureId?: string;                                  // optional, falls back to currentFeatureId
  commitId?: string;                                   // required if successState === "success"
  validatorsPassed?: boolean;                          // required true if successState === "success"
  handoff: HandoffObject;                              // REQUIRED
}
```

### Persisted Handoff File

**Filename format:** `{ISO-timestamp}__{featureId}__{workerSessionId}.json`
**Location:** `{missionDir}/handoffs/`

```typescript
interface HandoffFile {
  timestamp: string;           // ISO 8601, when handoff was created
  workerSessionId: string;     // UUID, the worker's session ID
  featureId: string;           // kebab-case feature identifier
  milestone: string;           // milestone the feature belongs to
  commitId: string;            // git commit hash (full or short SHA)
  successState: "success" | "partial" | "failure";
  returnToOrchestrator: boolean;
  handoff: HandoffObject;
}
```

### Handoff Object (RE: 1234.js `Z11`, 0082.js `h$H`)

```typescript
interface HandoffObject {
  salientSummary: string;                                    // 20-500 chars, 1-4 sentences, no newlines
  whatWasImplemented: string;                                // min 50 chars
  whatWasLeftUndone: string;                                 // empty string "" if complete
  verification: VerificationObject;
  tests: TestsObject;
  discoveredIssues: DiscoveredIssue[];
  skillFeedback?: SkillFeedback;                              // OPTIONAL, mainly from scrutiny validators
}

interface VerificationObject {
  commandsRun: CommandRun[];
  interactiveChecks: InteractiveCheck[];
}

interface CommandRun {
  command: string;       // the shell command that was run
  exitCode: number;      // process exit code
  observation: string;   // what was observed in the output
}

interface InteractiveCheck {
  action: string;        // what was done (clicked, navigated, typed)
  observed: string;      // what was seen as result
}

interface TestsObject {
  added: TestEntry[];
  updated?: string[];    // OPTIONAL - list of test file paths modified
  coverage: string;      // human-readable coverage description
}

interface TestEntry {
  file: string;          // test file path
  cases: TestCase[];
}

interface TestCase {
  name: string;          // test case name
  verifies: string;      // what behavior this test verifies
}

interface DiscoveredIssue {
  severity: "blocking" | "non_blocking" | "suggestion";
  description: string;
  suggestedFix?: string;  // OPTIONAL
}

interface SkillFeedback {
  followedProcedure: boolean;
  deviations: Deviation[];
  suggestedChanges?: string[];  // OPTIONAL
}

interface Deviation {
  step: string;              // which skill step was deviated from
  whatIDidInstead: string;   // what the worker actually did
  why: string;               // reason for deviation
}
```

### worker_completed Event (Embedded in progress_log.jsonl)

The handoff is also embedded in `worker_completed` progress log events with additional envelope fields:

```typescript
interface WorkerCompletedEvent {
  timestamp: string;              // ISO 8601
  type: "worker_completed";
  workerSessionId: string;        // UUID
  featureId: string;
  successState: "success" | "partial" | "failure";
  returnToOrchestrator: boolean;
  commitId?: string;
  exitCode: number;               // process exit code (NOT in handoff file)
  validatorsPassed?: boolean;     // NOT in handoff file, only in progress log
  handoff?: HandoffObject;        // full embedded handoff
}
```

---

## Field Documentation

### Top-Level Handoff File Fields

| Field | Type | Required | Source | Notes |
|-------|------|----------|--------|-------|
| `timestamp` | string (ISO 8601) | Yes | Runtime + RE | When the handoff was created. Format: `"2026-04-20T05:22:02.013Z"` |
| `workerSessionId` | string (UUID) | Yes | Runtime + RE | The worker's session UUID. Cross-references features.json `workerSessionIds[]`. |
| `featureId` | string | Yes | Runtime + RE | Kebab-case feature identifier. Matches a feature `id` in features.json. |
| `milestone` | string | Yes | Runtime | Milestone the feature belongs to (e.g. `"foundation"`, `"data-layers"`). |
| `commitId` | string | Yes (on success) | Runtime + RE | Git commit hash. Can be full SHA or abbreviated. Required when `successState === "success"`. |
| `successState` | enum | Yes | Runtime + RE | `"success"`, `"partial"`, or `"failure"`. RE defines 3 values; runtime only shows `"success"` and `"failure"`. |
| `returnToOrchestrator` | boolean | Yes | Runtime + RE | Whether orchestrator should take back control. Auto-forced `true` when `discoveredIssues` or `whatWasLeftUndone` exist (RE: 2539.js). |
| `handoff` | object | Yes | Runtime + RE | The structured handoff payload. See HandoffObject below. |

### Handoff Object Fields

| Field | Type | Required | Source | Notes |
|-------|------|----------|--------|-------|
| `salientSummary` | string (20-500 chars) | Yes* | Runtime + RE | 1-4 sentences, no newlines. RE marks `optional()` in 0082.js but enforces required in 2539.js (Zod `.min(20).max(500)` with refinements). |
| `whatWasImplemented` | string (≥50 chars) | Yes | Runtime + RE | Concrete description of what was built. Zod `.min(50)`. |
| `whatWasLeftUndone` | string | Yes | Runtime + RE | Empty string `""` if fully complete. Non-empty forces `returnToOrchestrator = true` (RE: 2539.js line ~45). |
| `verification` | object | Yes | Runtime + RE | Contains `commandsRun` and `interactiveChecks` arrays. |
| `verification.commandsRun` | CommandRun[] | Yes | Runtime + RE | Array of shell commands with exit codes and observations. Can be empty `[]`. |
| `verification.interactiveChecks` | InteractiveCheck[] | Yes | Runtime + RE | Array of manual UI checks. Empty `[]` in all observed observed protocol samples. |
| `tests` | object | Yes | Runtime + RE | Contains `added`, optionally `updated`, and `coverage`. |
| `tests.added` | TestEntry[] | Yes | Runtime + RE | Array of test file entries with cases. Can be empty `[]`. |
| `tests.updated` | string[] | Optional | Analysis + Samples | List of modified test file paths. Only observed in scrutiny validator handoffs. |
| `tests.coverage` | string | Yes | Runtime + RE | Human-readable coverage description. E.g. `"No tests added - this is a scaffold feature"`. |
| `discoveredIssues` | DiscoveredIssue[] | Yes | Runtime + RE | Array of issues. Empty `[]` if none. Non-empty forces `returnToOrchestrator = true`. |
| `skillFeedback` | object | Optional | Analysis + Samples | Structured feedback about skill procedure compliance. Only observed in scrutiny validator handoffs. |

### DiscoveredIssue Fields

| Field | Type | Required | Source | Notes |
|-------|------|----------|--------|-------|
| `severity` | enum | Yes | Runtime + RE | `"blocking"`, `"non_blocking"`, or `"suggestion"`. |
| `description` | string | Yes | Runtime + RE | Issue description. |
| `suggestedFix` | string | Optional | Analysis + Samples | How to fix. Not always present. |

### SkillFeedback Fields

| Field | Type | Required | Source | Notes |
|-------|------|----------|--------|-------|
| `followedProcedure` | boolean | Yes | Analysis + Samples | Whether the worker followed the skill procedure. |
| `deviations` | Deviation[] | Yes | Analysis + Samples | Array of deviation records. Empty `[]` if `followedProcedure === true`. |
| `suggestedChanges` | string[] | Optional | Analysis + Samples | Suggestions for improving the skill. |

---

## Examples

### Example 1: Successful Implementation Worker Handoff (Sanitized from Runtime)

Source: `protocol-samples/runtime-findings.md`: `project-scaffold` feature handoff

```json
{
  "timestamp": "2026-04-20T05:22:02.013Z",
  "workerSessionId": "<run-id>",
  "featureId": "project-scaffold",
  "milestone": "foundation",
  "commitId": "<handoff-hash>",
  "successState": "success",
  "returnToOrchestrator": false,
  "handoff": {
    "salientSummary": "Scaffolded React+TS+Vite frontend...",
    "whatWasImplemented": "Created sample-app/ with Vite React+TS app...",
    "whatWasLeftUndone": "",
    "verification": {
      "commandsRun": [
        {
          "command": "npm run build in sample-app/",
          "exitCode": 0,
          "observation": "Built successfully: dist/index.html, ..."
        }
      ],
      "interactiveChecks": []
    },
    "tests": {
      "added": [],
      "coverage": "No tests added - this is a scaffold feature"
    },
    "discoveredIssues": []
  }
}
```

**Key observations from this example:**
- Clean success: `whatWasLeftUndone: ""`, `discoveredIssues: []`, `returnToOrchestrator: false`
- `interactiveChecks: []`: all observed observed protocol samples has empty interactive checks
- `commitId` is a full SHA-1 hash (40 chars)
- No `skillFeedback` field present: this is a regular implementation worker, not a validator

### Example 2: Validator Handoff Pattern (from Runtime Observations)

Source: `protocol-samples/runtime-findings.md`: scrutiny validator observations

Validator handoffs (scrutiny-validator, user-testing-validator) differ from implementation handoffs:
- `returnToOrchestrator: true`: validators always return control to orchestrator
- `skillFeedback` field present with `followedProcedure` and `deviations`
- `tests.updated` may be present (list of modified test files)
- `successState` may be `"failure"` if validation finds blocking issues

### Example 3: worker_completed Event (from progress_log.jsonl)

Source: `protocol-samples/runtime-findings.md`: progress log event

```json
{
  "timestamp": "2026-04-20T05:22:02.013Z",
  "type": "worker_completed",
  "workerSessionId": "<run-id>",
  "featureId": "project-scaffold",
  "successState": "success",
  "returnToOrchestrator": false,
  "commitId": "6692b644...",
  "exitCode": 0,
  "validatorsPassed": true,
  "handoff": {
    "salientSummary": "Scaffolded React+TS+Vite frontend...",
    "whatWasImplemented": "Created sample-app/ with Vite React+TS app...",
    "whatWasLeftUndone": "",
    "verification": {
      "commandsRun": [
        {
          "command": "npm run build in sample-app/",
          "exitCode": 0,
          "observation": "Built successfully: dist/index.html, ..."
        }
      ],
      "interactiveChecks": []
    },
    "tests": {
      "added": [],
      "coverage": "No tests added - this is a scaffold feature"
    },
    "discoveredIssues": []
  }
}
```

**Note:** The progress log event includes `exitCode` and `validatorsPassed` fields that are NOT in the handoff JSON file: they are envelope metadata for the event.

---

## Handoff File Naming Convention

Format: `{ISO-timestamp}__{featureId}__{workerSessionId}.json`

The ISO timestamp uses colons replaced or preserved (runtime shows standard ISO format). The double underscore `__` separates the three components. This naming allows:
- Chronological sorting by filename
- Association with a specific feature
- Association with a specific worker session

---

## State Machine: Handoff Processing

```
                    ┌─────────────────────────┐
                    │  Worker calls            │
                    │  EndFeatureRun           │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Validate handoff schema  │
                    │  (Zod parse h$H)          │
                    │  • salientSummary 20-500  │
                    │  • whatWasImpl min 50     │
                    │  • commitId required      │
                    │  • validatorsPassed true  │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Compute forced return    │
                    │  V = returnToOrchestrator │
                    │    || hasDiscoveredIssues │
                    │    || hasUnfinishedWork   │
                    └────┬──────────┬──────────┘
                         │          │
              V = false  │          │  V = true
                         │          │
              ┌──────────▼──┐  ┌───▼────────────────┐
              │ Feature:    │  │ Feature: pending    │
              │ completed   │  │ (even on success!)  │
              │ State:      │  │ State:              │
              │ continues   │  │ orchestrator_turn   │
              └──────────┘  └───┬───────────────────┘
                                │
                    ┌───────────▼───────────────┐
                    │  Orchestrator reviews:     │
                    │  • discoveredIssues?       │
                    │  • whatWasLeftUndone?      │
                    │  • Create fix features?    │
                    │  • Dismiss items?          │
                    └───────────┬───────────────┘
                                │
                    ┌───────────▼───────────────┐
                    │  start_mission_run:        │
                    │  Check last handoff for    │
                    │  unaddressed items         │
                    │  Block if items remain     │
                    └───────────────────────────┘
```

### State Transitions

| From | To | Trigger | Side Effects |
|------|----|---------|-------------|
| Feature: pending | Feature: in_progress | `spawnWorker()` | `currentWorkerSessionId` set, `workerSessionIds[]` appended |
| Feature: in_progress | Feature: completed | EndFeatureRun success, no issues | State continues to next feature |
| Feature: in_progress | Feature: pending | EndFeatureRun success WITH issues | State → `orchestrator_turn` |
| Feature: in_progress | Feature: pending | Worker crash/timeout/inactivity | `worker_failed` logged, feature requeued |
| Feature: in_progress | Feature: pending | User pause/kill | `worker_paused` logged, state → `paused` |
| State: orchestrator_turn | State: running | `start_mission_run` (after addressing items) | Picks next pending feature |
| State: orchestrator_turn | BLOCKED | `start_mission_run` with unaddressed items | Error returned to orchestrator |

---

## returnToOrchestrator Semantics

The `returnToOrchestrator` field is **worker-set but system-enforced**:

| Condition | Worker Sets | System Forces | Result |
|-----------|-------------|---------------|--------|
| Clean success | `false` | no | Feature → completed, next feature picked |
| Worker requests review | `true` | no | State → `orchestrator_turn` |
| Success with discoveredIssues | any | `true` | Feature stays `pending`, orchestrator must address |
| Success with whatWasLeftUndone | any | `true` | Feature stays `pending`, orchestrator must address |
| Validator completion | `true` (always) | no | State → `orchestrator_turn` for validation review |
| Worker failure | N/A | `true` | State → `orchestrator_turn`, feature requeued |

**Static Analysis Source:** Module 2539.js computes `V = returnToOrchestrator || hasDiscoveredIssues || hasUnfinishedWork` and uses `V` to decide whether to set orchestrator_turn.

---

## Worker Lifecycle Connection

The handoff is the terminal event in the worker lifecycle:

```
start → spawn → bootstrap → execute → commit → handoff → cleanup
  │                                                  │
  │  spawnWorker()              EndFeatureRun tool    │
  │  ├─ Generate spawnId        ├─ Validate schema    │
  │  ├─ Get pending feature     ├─ Update feature     │
  │  ├─ Select model            ├─ Write handoff JSON │
  │  ├─ Create daemon session   ├─ Generate skeleton  │
  │  ├─ Update feature status   ├─ Append progress    │
  │  └─ Send bootstrap prompt   └─ interruptSession   │
```

### Timeline of a Worker Session

1. **`worker_selected_feature`** logged (orchestrator picks feature)
2. **`worker_started`** logged (spawn with `spawnId`)
3. Worker invokes `mission-worker-base` skill (startup)
4. Worker invokes assigned skill (implementation)
5. Worker writes code, runs tests, commits
6. Worker calls **`EndFeatureRun`** tool
7. **Validation**: schema checked, commitId/validatorsPassed verified
8. **State update**: feature status, mission state updated
9. **Persistence**: handoff JSON written, transcript skeleton generated
10. **`worker_completed`** logged (with full handoff embedded)
11. **`interruptSession`** called to clean up daemon session

### Crash Recovery (No Handoff)

If a worker crashes before calling EndFeatureRun:
- No handoff file is written
- `worker_failed` event logged in progress log
- Feature reset to `pending` (requeue)
- State → `orchestrator_turn` for orchestrator to decide retry

---

## Discrepancies (Inferred vs Observed)

| # | Field/Behavior | Architecture Notes Says | Observed Samples Show | Resolution |
|---|----------------|------------------|-------------------|------------|
| 1 | `successState` enum | `"success"`, `"partial"`, `"failure"` (3 values, RE: 1234.js `p.enum`) | Only `"success"` and `"failure"` observed. `"partial"` never seen. | `"partial"` is valid but no runtime example exists. Mark as [INFERRED-USED]: RE code clearly supports it. |
| 2 | `skillFeedback` in handoff object | Defined as optional field `O11` in Zod schema (0082.js, 1234.js) | Only observed in scrutiny validator handoffs, never in implementation worker handoffs | Field is truly optional. Implementation workers typically omit it. ⚠️ RE does not explain this usage pattern difference. |
| 3 | `validatorsPassed` location | Part of EndFeatureRun params (`Qz$`) and worker_completed event (`nf0` in 0082.js) | Appears in progress_log `worker_completed` events, NOT in persisted handoff JSON files | ⚠️ `validatorsPassed` is an envelope field, NOT part of the `handoff` sub-object. The handoff file does not contain it. |
| 4 | `salientSummary` optionality | 0082.js marks `AH.string().optional()`, but 1234.js and 2539.js enforce `.min(20).max(500)` with refinements | Always present and non-empty in observed protocol samples | ⚠️ Schema contradiction in RE: 0082.js says optional, 1234.js/2539.js says required with strict validation. Runtime confirms: always required in practice. |
| 5 | `tests.updated` field | Defined in RE schema `C11` as part of tests object | Mentioned as "seen in scrutiny validator" in runtime notes but never shown in the sanitized example | Field exists but is rarely used. Mark as optional. ⚠️ No runtime sanitized example available. |
| 6 | `missionId` format | RE in 0082.js references `decompMissionId` but format unclear | Runtime shows `mis_` prefix with hex string (`mis_0fe1958c`) in state.json, while directory name is UUID | ⚠️ The `missionId` in handoff context uses the `mis_` prefixed format, not the UUID directory name. This distinction matters for cross-referencing. |
| 7 | `interactiveChecks` population | RE defines full schema with `action` and `observed` fields | Always empty `[]` in all 11 observed runtime handoffs | Field is defined and validated but never populated in this dataset. Possibly used in missions with browser automation. |
| 8 | Feature status after success-with-issues | RE (2539.js) explicitly sets `pending`, not `completed` | No direct runtime observation of this edge case in the sanitized examples | RE code is authoritative: `successState === "success"` with issues → feature stays `pending`. ⚠️ Not confirmed by runtime but RE logic is clear. |
| 9 | Handoff file path format | RE references `ensureWorkerHandoffJson` but exact path not fully traced | Runtime confirms: `{missionDir}/handoffs/{timestamp}__{featureId}__{workerSessionId}.json` | Observed protocol samples resolves this gap definitively. |
| 10 | `worker_completed` event embedding | RE (0082.js `nf0`) defines event with optional `handoff` field | Runtime confirms full handoff object is embedded in `worker_completed` progress log events | Both RE and runtime agree. The handoff appears in both the file and the progress log. |
| 11 | `exitCode` field | RE (0082.js `nf0`) includes `exitCode: AH.number()` in worker_completed event | Runtime confirms `exitCode: 0` in progress log events | ⚠️ `exitCode` is in the progress log event but NOT in the handoff file. These are different persistence formats. |
| 12 | Handoff gating mechanism | RE (2544.js:261-310) blocks `start_mission_run` if unaddressed items exist | Runtime shows `lastReviewedHandoffCount: 11` in state.json confirming tracking | Both agree. `lastReviewedHandoffCount` tracks which handoffs the orchestrator has seen. |

---

## Handoff File vs Progress Log Event

The handoff data exists in two places with different schemas:

| Aspect | Handoff File (JSON) | Progress Log Event |
|--------|--------------------|--------------------|
| Location | `handoffs/{ts}__{fid}__{sid}.json` | Line in `progress_log.jsonl` |
| `exitCode` | ❌ Not present | ✅ Present |
| `validatorsPassed` | ❌ Not present | ✅ Present |
| `type` field | ❌ Not present | ✅ `"worker_completed"` |
| `milestone` | ✅ Present | ❌ Not present |
| `handoff` sub-object | ✅ Direct fields | ✅ Nested under `handoff` key |
| Purpose | Persistent record, orchestrator review | Timeline event, state tracking |

---

## Orchestrator Handoff Consumption Flow

### Step 1: Handoff Written
- Worker calls EndFeatureRun
- System validates, writes JSON file, logs progress event
- Feature status and mission state updated

### Step 2: Orchestrator Receives Summary
On next `start_mission_run` call (RE: 2544.js:531-630):
- `workerHandoffs`: array of summaries for unreviewed handoffs since last run
  - Each summary: `{ featureId, resultState, discoveredIssuesCount, unfinishedWorkCount, whatWasImplemented, handoffFile }`
- `latestWorkerHandoff`: full JSON of most recent handoff
- `lastReviewedHandoffCount` incremented in state.json

### Step 3: Blocking Check
If `discoveredIssues` or `whatWasLeftUndone` exist (RE: 2544.js:261-310):
- Returns error: `"Cannot resume without addressing worker handoff items"`
- Orchestrator must: create follow-up features, update existing descriptions, or dismiss items
- `dismiss-handoff-items` tool available with strict justification rules (min 20 chars)

### Step 4: Follow-up Feature Creation
The orchestrator evaluates:
- Does the unfinished work fit an existing pending feature? → Update its description
- Is it new work? → Create new feature entry in features.json (e.g., `feature-beta-fix`)
- Is it a false alarm? → Dismiss with justification (rare, must justify)

---

## dismiss-handoff-items Tool (RE: 1237.js)

```typescript
interface DismissHandoffItemsParams {
  dismissals: Dismissal[];
}

interface Dismissal {
  type: string;              // type of item being dismissed
  sourceFeatureId: string;   // feature that generated the handoff
  summary: string;           // brief summary of the item
  justification: string;     // min 20 chars, why it's safe to dismiss
}
```

Rules enforced by the system:
- Tech debt should almost always be tracked, NOT dismissed
- Valid dismissals: already tracked as existing feature (cite ID), or truly irrelevant
- "Low priority" or "non-blocking" is NOT sufficient justification
- Dismissal events logged as `handoff_items_dismissed` in progress log

---

## Transcript Skeleton Generation

After writing the handoff, the system generates a transcript skeleton (RE: 2539.js:207-219):

```typescript
interface TranscriptEntry {
  workerSessionId: string;   // UUID
  featureId: string;         // kebab-case
  milestone: string;
  skeleton: string;          // Full text transcript of tool calls/results
  timestamp: string;         // ISO 8601
}
```

The skeleton format is structured text (NOT JSON):
```
## Tool: Skill
{"skill":"mission-worker-base"}

## Tool: Read
{...}

## Tool Result
...

## Tool: EndFeatureRun
{...}
```

This provides the orchestrator with a condensed audit trail of the worker session.

---

## Open Questions

1. **`"partial"` successState**: Defined in RE Zod enum but never observed in runtime. What triggers it? Is it used for timeout/partial completion? Marked as [INFERRED-USED].

2. **`interactiveChecks` population**: The field is always empty `[]` in observed data. The schema supports `action`/`observed` pairs. Unclear if this is populated in missions with browser automation or if it's a reserved/unused field. Marked as [RESERVED].

3. **`completedWorkerSessionId`**: Present in features.json schema but never observed as non-null in observed protocol samples. Its relationship to handoff tracking is unclear. May be set only on final feature completion.

4. **Skill feedback aggregation**: RE found no evidence of how `skillFeedback` from multiple workers is aggregated or used to update skill files. The feedback is collected but the consumer is unknown.

5. **Handoff file cleanup**: No evidence of old handoff files being cleaned up. With retries, a feature may accumulate multiple handoff files. Whether these are pruned is unknown.
