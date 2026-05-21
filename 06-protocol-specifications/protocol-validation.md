# Protocol: Validation System
## Status: DEFINITIVE (cross-validated)

Cross-referenced runtime validation-state.json, validation-contract.md, and handoff data with architecture notes from `mission-validation-contract.md` (Modules 2542.js, 2541.js, 2175.js, 2176.js, 2201.js, 0082.js). The architecture notes is extremely comprehensive: the deepest of any protocol area analyzed: and aligns closely with runtime observations. Key discrepancies center on assertion status values and runtime-only artifacts.

---

## 1. Three-File Artifact Model

The validation system revolves around three files co-located in the mission directory:

| File | Format | Purpose | Created By |
|------|--------|---------|------------|
| `validation-contract.md` | Markdown | Defines testable behavioral assertions | Orchestrator during planning |
| `validation-state.json` | JSON | Tracks assertion pass/fail status | Updated by user-testing-validator |
| `features.json` | JSON | Maps features → assertions via `fulfills` | Orchestrator during planning |

**Invariant:** Every assertion ID in `validation-contract.md` must be claimed by exactly one feature's `fulfills` array in `features.json`. This is a coverage gate: unclaimed assertions indicate a planning gap.

---

## 2. validation-state.json Schema

### JSON Schema

```json
{
  "assertions": {
    "<ASSERTION_ID>": {
      "status": "<STATUS_ENUM>"
    }
  }
}
```

### Field Documentation

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `assertions` | object | Yes | Map of assertion ID → status object |
| `assertions.<ID>` | object | Yes | Status wrapper for a single assertion |
| `assertions.<ID>.status` | string enum | Yes | Current assertion status |

### Assertion ID Format

Pattern: `VAL-{CATEGORY}-{NUMBER}`

- `VAL`: Fixed prefix (uppercase)
- `{CATEGORY}`: Feature area category (uppercase, 2-6 chars)
- `{NUMBER}`: Zero-padded sequence number (001, 002, etc.)
- Separator: Hyphen (`-`)

**Observed categories from observed protocol samples:**
- `GLOBE`, `SAT`, `AIR`, `SHADE`, `HUD`, `UI`, `LLM`, `WE` (external-data), `ANIM`, `SFX`, `LOAD`, `CROSS` (sample mission)
- `HANDOFF`, `FEATURE`, `VALID`, `Msample itemION`, `SESSION`, `CONFIG`, `IPC` (Protocol Reconstruction mission: this mission)

**Example IDs:**
```
VAL-GLOBE-001, VAL-SAT-001, VAL-HANDOFF-001, VAL-CROSS-001, VAL-VALID-002
```

### Status Enum Values

| Status | Description | Source |
|--------|-------------|--------|
| `"pending"` | Initial state, not yet tested | Runtime ✅, RE ✅ |
| `"passed"` | Behavior confirmed working | RE ✅ (Module 2175.js) |
| `"failed"` | Behavior doesn't match spec, or prerequisite broken | RE ✅ (Module 2175.js) |
| `"blocked"` | ⚠️ Collapsed to `"failed"` in validation-state.json per RE. See discrepancy below. | Feature spec ✅, RE ⚠️ |
| `"skipped"` | ⚠️ Listed in feature description but not confirmed in RE code. See discrepancy below. | Feature spec ✅, RE ❌ |

**Runtime observation:** Only `"pending"` status was observed in the actual sample mission data (all 118 assertions remained pending due to browser automation issues). No `"passed"`, `"failed"`, `"blocked"`, or `"skipped"` values appear in observed protocol samples.

### Example 1: validation-state.json (Runtime: Sample Runtime Mission, excerpt)

```json
{
  "assertions": {
    "VAL-GLOBE-001": { "status": "pending" },
    "VAL-GLOBE-002": { "status": "pending" },
    "VAL-SAT-001": { "status": "pending" },
    "VAL-AIR-001": { "status": "pending" },
    "VAL-SHADE-001": { "status": "pending" },
    "VAL-HUD-001": { "status": "pending" },
    "VAL-UI-001": { "status": "pending" },
    "VAL-LLM-001": { "status": "pending" },
    "VAL-WE-001": { "status": "pending" },
    "VAL-ANIM-001": { "status": "pending" },
    "VAL-SFX-001": { "status": "pending" },
    "VAL-LOAD-001": { "status": "pending" },
    "VAL-CROSS-001": { "status": "pending" }
  }
}
```

Total assertions in sample mission: **118** (matching validation contract count).

### Example 2: validation-state.json (Runtime: This Mission, excerpt)

```json
{
  "assertions": {
    "VAL-HANDOFF-001": { "status": "pending" },
    "VAL-HANDOFF-002": { "status": "pending" },
    "VAL-HANDOFF-003": { "status": "pending" },
    "VAL-HANDOFF-004": { "status": "pending" },
    "VAL-FEATURE-001": { "status": "pending" },
    "VAL-FEATURE-002": { "status": "pending" },
    "VAL-FEATURE-003": { "status": "pending" },
    "VAL-VALID-001": { "status": "pending" },
    "VAL-VALID-002": { "status": "pending" },
    "VAL-VALID-003": { "status": "pending" },
    "VAL-Msample itemION-001": { "status": "pending" },
    "VAL-Msample itemION-002": { "status": "pending" },
    "VAL-Msample itemION-003": { "status": "pending" },
    "VAL-Msample itemION-004": { "status": "pending" },
    "VAL-SESSION-001": { "status": "pending" },
    "VAL-SESSION-002": { "status": "pending" },
    "VAL-SESSION-003": { "status": "pending" },
    "VAL-CONFIG-001": { "status": "pending" },
    "VAL-CONFIG-002": { "status": "pending" },
    "VAL-CONFIG-003": { "status": "pending" },
    "VAL-IPC-001": { "status": "pending" },
    "VAL-IPC-002": { "status": "pending" },
    "VAL-IPC-003": { "status": "pending" },
    "VAL-CROSS-001": { "status": "pending" },
    "VAL-CROSS-002": { "status": "pending" },
    "VAL-CROSS-003": { "status": "pending" }
  }
}
```

Total assertions in this mission: **26** (7 areas × 3-4 assertions each).

---

## 3. Assertion Lifecycle

### State Machine

```
                    ┌──────────────────────────────────────────┐
                    │                                          │
                    ▼                                          │
              ┌──────────┐                                    │
              │ pending  │◄───────────────────────────────────┤
              └────┬─────┘  (re-run: validator resets to      │
                   │         pending after fix features)       │
                   │                                          │
         ┌─────────┼──────────┐                               │
         │         │          │                               │
         ▼         ▼          ▼                               │
    ┌─────────┐ ┌────────┐ ┌─────────┐                       │
    │ passed  │ │ failed │ │ skipped │                       │
    │ (final) │ │        │ │(final?) │                       │
    └─────────┘ └───┬────┘ └─────────┘                       │
                    │                                          │
                    │ (fix features created,                   │
                    │  validator re-runs)                      │
                    │                                          │
                    └──────────────────────────────────────────┘
```

### Transition Table

| From | To | Trigger | Side Effects |
|------|----|---------|--------------|
| `pending` | `passed` | User-testing-validator confirms assertion behavior works | `validatedAtMilestone` recorded in synthesis |
| `pending` | `failed` | User-testing-validator finds behavior doesn't match spec | Issue recorded in synthesis, fix features may be created |
| `pending` | `failed` | User-testing-validator finds prerequisite broken ("blocked") | Blocking reason recorded; collapsed to `failed` in state file |
| `failed` | `pending` | Orchestrator resets for re-run after fix features complete | Previous synthesis.json preserved for reference |
| `passed` |: | Terminal state. Assertion does not revert. |: |
| `pending` | `skipped` | ⚠️ Not traced in RE. Possibly when assertion is inapplicable. | [INFERRED] |

### Key Rules

1. **Terminal states**: `passed` is terminal: once passed, an assertion never reverts.
2. **`failed` is recoverable**: The orchestrator can reset failed assertions to `pending` for re-runs.
3. **Blocked collapses to failed**: In `validation-state.json`, blocked assertions are stored as `"failed"` with a blocking reason. The synthesis report tracks them separately for UX but the authoritative state file uses only `failed`.
4. **Re-run scoping**: On re-run, validators read prior `synthesis.json` and only validate previously failed/blocked assertions + new assertions from fix features.

---

## 4. Auto-Injection Rules

### Trigger Condition

The function `checkMilestoneCompletionAndInjectValidation()` (Module 2542.js, lines 316-383) fires after every worker completion. It checks **three conditions**: all must pass for injection:

| # | Condition | Logic |
|---|-----------|-------|
| 1 | Feature has a milestone | `feature.milestone` is truthy |
| 2 | All implementation features complete | Filter out validator skills (`HMH`), check all remaining are `completed` or `cancelled` |
| 3 | Validation not yet planned | `state.milestonesWithValidationPlanned` doesn't include this milestone |

### Implementation Complete Check

```js
// Module 2201.js, lines 235-241
isMilestoneImplementationComplete(milestone) {
  let features = getMilestoneFeatures(milestone);
  if (features.length === 0) return false;
  let impl = features.filter(f => !HMH.includes(f.skillName)); // Exclude validators
  if (impl.length === 0) return false;
  return impl.every(f => f.status === "completed" || f.status === "cancelled");
}
```

Key constant: `HMH = ["scrutiny-validator", "user-testing-validator"]` (Module 2176.js, line 120).

### Injected Feature Objects

When conditions pass, two features are created:

#### scrutiny-validator-{milestone}

```json
{
  "id": "scrutiny-validator-{milestone}",
  "description": "Scrutiny validation for milestone \"{milestone}\"...",
  "skillName": "scrutiny-validator",
  "preconditions": ["All implementation features for milestone \"{milestone}\" are complete"],
  "expectedBehavior": [
    "Validators pass (test, typecheck, lint)",
    "Review subagents spawned for each feature",
    "Findings synthesized into scrutiny report"
  ],
  "verificationSteps": [],
  "fulfills": [],
  "milestone": "{milestone}",
  "status": "pending"
}
```

#### user-testing-validator-{milestone}

```json
{
  "id": "user-testing-validator-{milestone}",
  "description": "User testing validation for milestone \"{milestone}\"...",
  "skillName": "user-testing-validator",
  "preconditions": ["All implementation features for milestone \"{milestone}\" are complete"],
  "expectedBehavior": [
    "Testable assertions determined from fulfills mapping",
    "Environment set up (services started, data seeded)",
    "Flow validator subagents spawned and completed",
    "Results synthesized, validation-state.json updated"
  ],
  "verificationSteps": [],
  "fulfills": [],
  "milestone": "{milestone}",
  "status": "pending"
}
```

### Insertion Order & Execution Sequence

Both features use `insertFeatureAtTop()`, but insertion order determines execution:

1. User-testing feature inserted first (goes to top)
2. Scrutiny feature inserted second (pushes user-testing down, goes to top)
3. **Result: Scrutiny runs first, user-testing runs second**

### Skip Flags

Model settings (`model-settings.json`) can contain skip flags:

| Flag | Type | Default | Effect |
|------|------|---------|--------|
| `skipScrutiny` | boolean | `false` | If `true`, skip scrutiny validator injection |
| `skipUserTesting` | boolean | `false` | If `true`, skip user-testing validator injection |

**Full skip**: If **both** are `true`, validation is entirely skipped and milestone is marked as planned immediately (no validators injected).

**Code path** (Module 2542.js, lines 361-368):
```js
if (skipScrutiny && skipUserTesting) {
  await markMilestoneValidationPlanned(milestone);
  return;  // No validators injected
}
if (!skipUserTesting) await insertFeatureAtTop(userTestingFeature);
if (!skipScrutiny) await insertFeatureAtTop(scrutinyFeature);
```

### Double-Injection Prevention

The `markMilestoneValidationPlanned(milestone)` call updates `state.milestonesWithValidationPlanned` to track which milestones have already had validation injected. This prevents the function from creating duplicate validator features on subsequent worker completions.

### Progress Event

After injection, a progress event is emitted:

```json
{
  "type": "milestone_validation_triggered",
  "milestone": "{milestone}",
  "featureId": "scrutiny-validator-{milestone}"
}
```

**Runtime confirmation:** This event type appears in sample runtime progress_log.jsonl:
```
{"timestamp":"2026-04-20T14:58:46.438Z","type":"milestone_validation_triggered","milestone":"foundation","featureId":"scrutiny-validator-foundation"}
```

---

## 5. Two-Phase Validation Flow

### Phase 1: Scrutiny Validation (Code Quality)

**Skill:** `scrutiny-validator`
**Subagents spawned:** `scrutiny-feature-reviewer` (one per completed implementation feature)

#### Pipeline

```
1. Hard gate validators (test, typecheck, lint)
   └── Any fail → immediate EndFeatureRun with failure
2. Spawn scrutiny-feature-reviewer subagent per feature
   └── Uses Task tool for each
   └── On re-run: only reviews failed features + fix features
3. Synthesize results
   └── Determine pass/fail
   └── Write synthesis.json
4. EndFeatureRun with returnToOrchestrator: true
```

#### Synthesis Output

Path: `.opendroid/validation/{milestone}/scrutiny/synthesis.json`

```json
{
  "milestone": "foundation",
  "round": 1,
  "status": "pass",
  "validatorsRun": {
    "test": { "passed": true, "command": "npm test", "exitCode": 0 },
    "typecheck": { "passed": true, "command": "tsc --noEmit", "exitCode": 0 },
    "lint": { "passed": true, "command": "eslint .", "exitCode": 0 }
  },
  "reviewsSummary": {
    "total": 5,
    "passed": 4,
    "failed": 1,
    "failedFeatures": ["checkout-reserve-inventory"]
  },
  "blockingIssues": [],
  "appliedUpdates": [],
  "suggestedGuidanceUpdates": [],
  "rejectedObservations": [],
  "previousRound": null
}
```

### Phase 2: User-Testing Validation (Behavioral)

**Skill:** `user-testing-validator`
**Subagents spawned:** `user-testing-flow-validator` (one per testable assertion group)

#### Pipeline

```
1. Collect testable assertions
   └── From fulfills arrays of completed features
   └── Cross-reference with validation-state.json for "pending" only
2. Setup environment
   └── Start services, seed data
   └── Assign isolation resources per flow validator
3. Spawn user-testing-flow-validator subagents in parallel
   └── Uses Task tool for each
4. Synthesize results
   └── Determine pass/fail/blocked per assertion
   └── Update validation-state.json
   └── Write synthesis.json
5. EndFeatureRun with returnToOrchestrator: true
```

#### Assertion Status Update Rules

From Module 2175.js, lines 2104-2112:

| Test Result | validation-state.json Update | Additional |
|-------------|------------------------------|------------|
| `pass` | `"passed"` | Record `validatedAtMilestone` |
| `fail` | `"failed"` | Record issue description |
| `blocked` | `"failed"` | Record blocking reason |

#### Synthesis Output

Path: `.opendroid/validation/{milestone}/user-testing/synthesis.json`

```json
{
  "milestone": "foundation",
  "round": 1,
  "status": "pass",
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
  "appliedUpdates": [],
  "previousRound": null
}
```

### Flow Diagram

```
All implementation features complete
         │
         ▼
  ┌──────────────────┐
  │ Auto-injection   │ ── Creates scrutiny + user-testing features
  │ trigger fires    │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐     Fail → Return to orchestrator
  │ Scrutiny         │ ──→ Fix features created
  │ (code review)    │     Validator reset to pending
  └────────┬─────────┘     Re-run after fixes
           │ Pass
           ▼
  ┌──────────────────┐     Fail → Return to orchestrator
  │ User Testing     │ ──→ Fix features created
  │ (behavioral)     │     Validator reset to pending
  └────────┬─────────┘     Re-run after fixes
           │ Pass
           ▼
  ┌──────────────────┐
  │ Milestone Sealed │ ── No more features can be added
  └──────────────────┘
```

---

## 6. Milestone Sealing Rules

Once a milestone passes both scrutiny and user-testing validation:

1. **Milestone is sealed**: no new features can be added to it
2. **All assertions for the milestone** are `passed` in `validation-state.json`
3. **The `milestonesWithValidationPlanned`** set is updated to prevent re-injection
4. New work goes into follow-up milestones

### End-of-Mission Gate

Before declaring a mission complete, the system checks `validation-state.json`. **ALL assertions must be `"passed"`**. Any `"pending"` or `"failed"` assertions block mission completion.

---

## 7. validation-contract.md Format

### Purpose

The validation contract defines **testable behavioral assertions** that specify what "done" means for the mission. It serves as:
- The specification for validator features to test against
- The source of truth for assertion IDs that appear in `validation-state.json` and `features.json`
- A human-readable quality contract between the mission planner and the validation system

### Format

The file is Markdown with this structure:

```markdown
# Validation Contract: {Mission Name}

This contract defines "done" for the {Mission Name} mission. Each assertion
specifies a testable quality criterion for a protocol report. Validation is
report-content-based (not code-based).

## Area: {Area Name}

### VAL-{CATEGORY}-{NUMBER}: {Assertion Title}
{Behavioral description of what must be true}
Tool: {validation tool, e.g., file-read, browser-test}
Evidence: {what evidence confirms this assertion}

### VAL-{CATEGORY}-{NUMBER}: {Assertion Title}
...

## Area: {Another Area}
...
```

### Assertion Structure

Each assertion has:

| Element | Required | Description |
|---------|----------|-------------|
| ID | Yes | `VAL-{CATEGORY}-{NUMBER}` format (e.g., `VAL-HANDOFF-001`) |
| Title | Yes | Short descriptive title after the ID |
| Behavioral description | Yes | What must be true for this assertion to pass |
| Tool | Yes | How to validate (e.g., `file-read`, `browser-test`) |
| Evidence | Yes | What constitutes passing evidence |

### Naming Conventions

- **Category**: Matches the protocol area or feature area (uppercase): `HANDOFF`, `FEATURE`, `VALID`, `Msample itemION`, `SESSION`, `CONFIG`, `IPC`, `CROSS`
- **Number**: Zero-padded 3-digit sequence starting from `001`
- **Area grouping**: Assertions are grouped under `## Area: {Name}` headers

### Example 3: Validation Contract Assertion (Runtime: This Mission)

```markdown
## Area: Validation Protocol

### VAL-VALID-001: validation-state.json schema documented
The report documents the exact schema of validation-state.json including assertion ID format
(VAL-{CATEGORY}-{NUMBER}), status enum values (pending, pass, fail, blocked, skipped),
and the structure.
Tool: file-read
Evidence: schema section for validation-state.json

### VAL-VALID-002: Validation lifecycle and auto-injection documented
The report documents the assertion lifecycle (pending → pass/fail/blocked/skipped),
the auto-injection rules for scrutiny-validator and user-testing-validator features,
and milestone sealing rules.
Tool: file-read
Evidence: lifecycle section covering assertion states, auto-injection, milestone sealing

### VAL-VALID-003: validation-contract.md format documented
The report documents the format and purpose of validation-contract.md files, including
assertion ID conventions, behavioral descriptions, tool specifications, and evidence
requirements.
Tool: file-read
Evidence: section on validation-contract.md format
```

---

## 8. Validator Feature Properties

Validator features differ from implementation features in several ways:

| Property | Implementation Feature | Validator Feature |
|----------|----------------------|-------------------|
| `skillName` | Worker skill (e.g., `frontend-worker`) | `scrutiny-validator` or `user-testing-validator` |
| `fulfills` | Array of assertion IDs | Empty `[]` (validators test, not fulfill) |
| `verificationSteps` | Manual steps or commands | Empty `[]` (uses skill logic) |
| `returnToOrchestrator` | Varies | Always `true` |
| Included in "implementation complete" check | Yes | No (filtered by `HMH` constant) |
| Can create fix features | No | No (only orchestrator creates) |
| Can be skipped | No | Yes (via `skipScrutiny`/`skipUserTesting` flags) |

### Validator Droid Definitions

Both validator features spawn subagents that are defined as built-in droids:

**scrutiny-feature-reviewer** (`~/.opendroid/droids/scrutiny-feature-reviewer.md`):
- Model: `inherit` (uses parent session's model)
- Output: JSON review report with `featureId`, `reviewedAt`, `commitId`, `status`, `codeReview`, `sharedStateObservations`

**user-testing-flow-validator** (`~/.opendroid/droids/user-testing-flow-validator.md`):
- Model: `inherit`
- Output: JSON test report with assertion results, evidence, frictions, blockers
- Assertion statuses: `"pass"`, `"fail"`, `"blocked"`, `"skipped"`

---

## 9. Re-Run Semantics

When a validator fails:

### Sequence

```
1. Validator completes with failure
   └── returnToOrchestrator: true
   └── Handoff describes what failed
2. Orchestrator reviews handoff
   └── May spawn subagent to analyze failure
3. Fix features created at top of features.json
   └── New features targeting specific failures
4. Fix features execute (normal worker flow)
5. Validator feature re-runs
   └── Still "pending" status (never set to "completed" on failure)
   └── Reads prior synthesis.json to scope work
6. Re-run validates:
   └── Scrutiny: failed features + fix features together
   └── User-testing: failed/blocked assertions + new assertions from fix features
```

### Round Tracking

Each synthesis.json includes a `round` number (starting at 1) and a `previousRound` field (null for first run, references prior synthesis for re-runs). This enables tracking validation history across iterations.

---

## 10. Runtime-Only Artifacts

These artifacts were observed in observed protocol samples but NOT covered by architecture notes:

### contract-work/ Directory

Contains Markdown "surface" descriptions for each feature area's validation:
```
contract-work/
├── primary-surface.md
├── secondary-surface.md
├── tertiary-surface.md
└── ...
```

These describe the testing surfaces (entry points, UI elements, API endpoints) available for validation. They help user-testing validators know what to test and how to interact with the system.

### evidence/ Directory

Stores validation artifacts organized by milestone and test group:
```
evidence/
└── foundation/
    ├── primary-navigation/
    │   └── initial-view.png
    └── user-testing/
        └── landing-view.png
```

Screenshots and artifacts from validation runs, providing proof that testing occurred.

### model-settings.json Validation Fields

```json
{
  "validationWorkerModel": "custom:built-in model-[Z.AI]-0",
  "validationWorkerReasoningEffort": "none"
}
```

Per-mission model configuration for validation workers, separate from implementation worker model. The validation model can use a different (typically cheaper/faster) model.

---

## 11. Discrepancies (Inferred vs Observed)

| # | Field/Behavior | Architecture Notes Says | Observed Samples Show | Resolution |
|---|---------------|------------------|-------------------|------------|
| 1 | Assertion status values | `"pending"`, `"passed"`, `"failed"` (blocked = failed) | Feature description lists: `"pending"`, `"pass"`, `"fail"`, `"blocked"`, `"skipped"`. Runtime only shows `"pending"`. | ⚠️ **UNRESOLVED**: RE code is authoritative: `"passed"`/`"failed"` are the actual values written to validation-state.json. `"blocked"` is collapsed to `"failed"`. `"skipped"` not traced in RE but listed in feature spec. Likely present but unobserved in observed protocol samples. |
| 2 | `"skipped"` status | Not mentioned in Module 2175.js code | Listed in feature description as valid status | ⚠️ **INFERRED**: Possibly used when assertions are inapplicable to a specific mission, or when skip flags are active. Not confirmed in observed protocol samples. |
| 3 | `"pass"` vs `"passed"` | RE uses `"passed"` in synthesis logic | Feature description uses `"pass"` | RE code is authoritative. The actual value written is `"passed"`. Feature description may be imprecise. |
| 4 | synthesis.json output | `.opendroid/validation/{milestone}/scrutiny|user-testing/synthesis.json` | No runtime evidence of these files existing | ⚠️ **GAP**: No runtime confirmation. sample mission had no successful validations. Path is RE-derived and likely correct. |
| 5 | Skip flags | `skipScrutiny`, `skipUserTesting` in model-settings.json | Not observed in runtime model-settings.json | Feature exists in code but not exercised in captured data. |
| 6 | `contract-work/` directory | Not mentioned in architecture notes | Present in runtime mission directory | ⚠️ **NEW**: Runtime-only artifact. Purpose: surface descriptions for validation. |
| 7 | `evidence/` directory | Not mentioned in architecture notes | Present in runtime mission directory with screenshots | ⚠️ **NEW**: Runtime-only artifact. Purpose: validation proof artifacts. |
| 8 | `validationWorkerModel` in model-settings.json | Not traced in RE modules | Present in runtime model-settings.json | ⚠️ **GAP**: RE did not trace this field but it's part of the model-settings schema. |

---

## 12. Open Questions

1. **`"skipped"` assertion status**: Is this a real status value written to validation-state.json, or only used internally by the user-testing-flow-validator subagent? RE code only shows `"passed"` and `"failed"`. [INFERRED: Likely internal to subagent only.]

2. **Assertion reset mechanism**: How exactly does the orchestrator reset `failed` assertions back to `pending` for re-runs? The write path was inferred from skill prompt text, not traced to a specific function. [INFERRED: Orchestrator updates validation-state.json directly when creating fix features.]

3. **`dismiss_handoff_items` mechanism**: Referenced in Module 0082.js (line 100, `dNH` schema) for handling validation overrides. Not deeply traced in architecture notes. [INFERRED: Allows manually dismissing validation failures.]

4. **synthesis.json persistence**: Are synthesis.json files from failed rounds preserved alongside successful rounds, or overwritten? The `previousRound` field suggests at least one prior round is kept. [INFERRED: Prior synthesis preserved as reference, new synthesis overwrites the primary file.]

5. **Parallel milestone validation**: Can two milestones be in validation simultaneously? The `milestonesWithValidationPlanned` set suggests milestones are validated independently, but concurrent execution is unclear from architecture notes. [INFERRED: Milestones validate sequentially since features execute in order.]
