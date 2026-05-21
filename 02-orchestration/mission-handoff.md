# mission-handoff: Mission System RE

## Overview

This report reverse-engineers the inter-worker handoff protocol in OpenDroid's mission system. The handoff is the structured JSON payload a worker submits via the `EndFeatureRun` tool when finishing a feature. It captures what was built, what remains incomplete, discovered issues, skill feedback, and verification evidence. The orchestrator consumes these handoffs to decide whether to continue to the next feature, return control to the orchestrator for decision-making, or block until actionable items are addressed.

The handoff schema is strongly typed with Zod validators, persisted to per-worker JSON files in the mission directory, and rendered in the TUI for user inspection.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 0082.js | ~110 lines | Handoff schema definitions (Zod) | Defines `h$H` (handoff object), `jf0` (discoveredIssue), `gf0` (skillFeedback), `nf0` (worker_completed event) |
| 1234.js | ~240 lines | Tool input/output schemas | Defines `Z11` (handoff schema for EndFeatureRun), `z11` (worker handoff summary), dismiss-handoff-items schemas |
| 2539.js | ~230 lines | EndFeatureRun tool execution | Validates handoff fields, updates feature status, writes handoff JSON, generates transcript skeleton |
| 2544.js | ~654 lines | start_mission_run orchestrator logic | Consumes handoffs, blocks on unaddressed items, builds workerHandoffs array |
| 3973.js | ~699 lines | TUI rendering | Renders salientSummary, whatWasLeftUndone, discoveredIssues in the CLI |
| 1237.js | ~35 lines | dismiss-handoff-items tool registration | Defines the tool for explicitly dismissing handoff items |

**Note on target modules:** The feature spec lists modules 3921, 3925, 3980, 3972, 2517, 0940 as targets. Grep searches for handoff keywords in these modules returned no hits. The actual handoff implementation is concentrated in modules 0082, 1234, 2539, 2544, 3973, and 1237. Modules 3921-0940 may contain adjacent orchestrator or reporting logic not directly tagged with handoff keywords.

## Architecture / Flow

### 1. Handoff Submission (Worker → EndFeatureRun Tool)

When a worker completes a feature, it calls the `EndFeatureRun` tool with:
- `successState`: `"success"`, `"partial"`, or `"failure"`
- `returnToOrchestrator`: boolean
- `featureId`: optional (falls back to current feature in state)
- `commitId`: required if success
- `validatorsPassed`: required true if success
- `handoff`: the structured handoff object (Z11 schema)

**Module 2539.js, lines 30-219**: The `roA` class executes `EndFeatureRun`:
1. Validates the worker session is linked to a mission (`decompMissionId`).
2. Validates `successState === "success"` requires `commitId` and `validatorsPassed === true`.
3. Parses the handoff with `h$H.parse(f)` (Zod schema from 0082.js).
4. Validates `salientSummary`:
   - Required (not empty)
   - 20–500 characters
   - No newlines
   - 1–4 sentences (computed by splitting on `[.!?]+\s+`)
5. Computes `V = returnToOrchestrator || hasDiscoveredIssues || hasUnfinishedWork`.
6. Updates feature status: `"success"` → `"completed"`, otherwise → `"pending"`.
7. Appends a `worker_completed` event to the progress log.
8. Writes the handoff to a per-worker JSON file via `ensureWorkerHandoffJson`.
9. Generates a transcript skeleton from the worker's message events.
10. Updates mission state: `"orchestrator_turn"` if `V` is true, else continues to next feature.

### 2. Handoff JSON Schema

**Module 1234.js, lines 148-189**: The `Z11` schema (handoff object):

```js
Z11 = p.object({
  salientSummary: p.string().min(20).max(500)
    .refine((H) => !H.includes("\n"), { message: "salientSummary must be 1–4 sentences (no newlines)" })
    .refine((H) => { let A = G11(H); return A >= 1 && A <= 4; }, { message: "salientSummary must be 1–4 sentences" })
    .describe("1–4 sentence salient summary of what happened in this session"),
  whatWasImplemented: p.string().min(50)
    .describe("Concrete description of what was built (min 50 characters)"),
  whatWasLeftUndone: p.string()
    .describe("Anything incomplete or deferred. Must leave empty if everything is truly complete."),
  verification: J11.describe("Verification performed: commands with results + any manual checks"),
  tests: C11.describe("Tests written or updated"),
  discoveredIssues: p.array(q11).describe("Issues found during implementation. Empty array if none."),
  skillFeedback: O11.optional().describe("Feedback on the skill procedure. Fill this out to help improve future workers."),
})
```

**Sub-schemas:**
- `J11` (verification): `{ commandsRun: [{ command, exitCode, observation }], interactiveChecks: [{ action, observed }] }`
- `C11` (tests): `{ added: [{ file, cases: [{ name, verifies }] }], updated: [string], coverage: string }`
- `q11` (discoveredIssue): `{ severity: "blocking" | "non_blocking" | "suggestion", description: string, suggestedFix?: string }`
- `O11` (skillFeedback): `{ followedProcedure: boolean, deviations: [{ step, whatIDidInstead, why }], suggestedChanges?: [string] }`

**Module 0082.js, lines 44-50**: The `h$H` schema mirrors this exactly:
```js
h$H = AH.object({
  salientSummary: AH.string().optional(),
  whatWasImplemented: AH.string(),
  whatWasLeftUndone: AH.string(),
  verification: kf0,
  tests: vf0,
  discoveredIssues: AH.array(jf0),
  skillFeedback: gf0.optional(),
})
```

**Module 0082.js, lines 81-83**: The `worker_completed` event embeds the handoff:
```js
nf0 = uY.extend({
  type: AH.literal("worker_completed"),
  workerSessionId: AH.string(),
  featureId: AH.string(),
  successState: mNH,
  returnToOrchestrator: AH.boolean(),
  commitId: AH.string().optional(),
  exitCode: AH.number(),
  validatorsPassed: AH.boolean().optional(),
  handoff: h$H.optional(),
})
```

### 3. Orchestrator Consumption

**Module 2544.js, lines 261-310**: When `start_mission_run` is called while the mission state is `"orchestrator_turn"`, the system checks the most recent `worker_completed` event:

```js
let SH = [...(await I.readProgressLog())]
  .reverse()
  .find((RH) => RH.type === "worker_completed" && RH.handoff !== void 0);
if (SH?.handoff) {
  let { handoff: RH } = SH,
    F = RH.discoveredIssues && RH.discoveredIssues.length > 0,
    z = RH.whatWasLeftUndone &&
        RH.whatWasLeftUndone.trim() !== "" &&
        RH.whatWasLeftUndone.toLowerCase() !== "none";
  if (F || z) {
    let y = [];
    if (F) y.push(`${RH.discoveredIssues.length} discovered issue(s)`);
    if (z) y.push("incomplete work");
    yield {
      type: "result",
      isError: true,
      errorType: "invalidParameterLLMError",
      userError: "Cannot resume without addressing worker handoff items",
      llmError: `...If discoveredIssues and/or whatWasLeftUndone exist, either create new features or update existing feature descriptions...`
    };
    return;
  }
}
```

**Key behavior:** If `discoveredIssues` or `whatWasLeftUndone` exist, `start_mission_run` **blocks** and returns an error to the orchestrator. The orchestrator must either create follow-up features, update existing descriptions, or dismiss the items via `dismiss-handoff-items` before the mission can resume.

**Module 2544.js, lines 531-630**: The orchestrator also receives:
- `workerHandoffs`: array of summaries for all unreviewed handoffs since last run
- `latestWorkerHandoff`: the full JSON of the most recent handoff, including `handoffFile` path

The `lastReviewedHandoffCount` in state tracks which handoffs have been seen, ensuring each is only presented once.

### 4. discoveredIssues Escalation Flow

**Module 2539.js, lines 34-35**:
```js
P = (f?.discoveredIssues?.length ?? 0) > 0;
```

Any discovered issue forces `V = true` (line ~45), which causes:
1. Feature status set to `"pending"` (not `"completed"`) even if `successState === "success"`
2. Mission state set to `"orchestrator_turn"`
3. Orchestrator must address issues before resuming

**Module 2544.js, lines 571-595**: discoveredIssues count is surfaced in the handoff summary:
```js
hH = z.handoff.discoveredIssues.length,
NH = z.handoff.whatWasLeftUndone.trim(),
qH = NH && NH !== "" && NH.toLowerCase() !== "none" ? 1 : 0,
// ...
return {
  featureId: z.featureId,
  resultState: t,
  discoveredIssuesCount: hH,
  unfinishedWorkCount: qH,
  whatWasImplemented: z.handoff.whatWasImplemented,
  handoffFile: G,
};
```

### 5. whatWasLeftUndone Drives Follow-up Features

**Module 2544.js, lines 557-560**: The orchestrator prompt explicitly instructs:
> "If discoveredIssues and whatWasLeftUndone exist, either create new features or update existing feature descriptions if the issue belongs to a pending feature's scope."

This is the primary mechanism for follow-up feature creation. When a worker reports unfinished work:
- The orchestrator evaluates whether it fits an existing pending feature
- If not, it creates a new feature entry in `features.json`
- The mission runner then picks up the new feature in subsequent runs

**Module 2175.js, lines 999-1001**: The orchestrator's own instructions reinforce this:
> "When any handoff contains `discoveredIssues` or `whatWasLeftUndone`:** For discoveredIssues and whatWasLeftUndone (tech debt - MUST be tracked):**"

### 6. Dismiss Handoff Items Tool

**Module 1237.js, lines 8-30**: The `dismiss-handoff-items` tool allows the orchestrator to explicitly dismiss handoff items with justification:

```js
id: "dismiss-handoff-items",
description: `Explicitly dismiss items from a worker's handoff. Use sparingly...`,
inputSchema: wz$,  // { dismissals: [{ type, sourceFeatureId, summary, justification }] }
```

Dismissal rules:
- Tech debt should almost always be tracked, not dismissed
- Valid dismissals: already tracked as existing feature, or truly irrelevant/non-issue
- "Low priority" or "non-blocking" is NOT sufficient justification
- Justification minimum: 20 characters

**Module 0082.js, lines 96-98**: Dismissal events are logged:
```js
tf0 = uY.extend({
  type: AH.literal("handoff_items_dismissed"),
  dismissals: AH.array(dNH).optional(),
})
```

### 7. TUI Rendering

**Module 3973.js, lines 641-666**: The TUI renders handoff fields with color coding:
- `salientSummary`: rendered as "Summary"
- `whatWasLeftUndone`: rendered as "What Was Left Undone" with `warning` color if non-empty, else `muted`
- `discoveredIssues`: rendered via `mxM` component (issue list with severity badges)

## Key Findings

1. **Handoff is mandatory and validated.** The `salientSummary` field is the only optional field at the schema level (`AH.string().optional()` in 0082.js), but the `EndFeatureRun` executor (2539.js) enforces it as required with strict validation (20-500 chars, 1-4 sentences, no newlines).

2. **Success with issues is not completion.** If a worker returns `successState: "success"` but includes `discoveredIssues` or `whatWasLeftUndone`, the feature status is set to `"pending"` (not `"completed"`), forcing orchestrator review.

3. **`returnToOrchestrator` is auto-computed.** The worker sets `returnToOrchestrator`, but the executor also forces it true if `discoveredIssues` or `whatWasLeftUndone` exist. This ensures the orchestrator always sees actionable items.

4. **Handoffs are persisted and tracked.** Each handoff is written to a per-worker JSON file (`ensureWorkerHandoffJson`). The `lastReviewedHandoffCount` state field ensures the orchestrator only sees new handoffs on each `start_mission_run` call.

5. **Transcript skeletons are generated.** After writing the handoff, the system generates a transcript skeleton from the worker's message events (2539.js, lines 207-219). This provides the orchestrator with a condensed view of the worker session.

## Code Examples

### EndFeatureRun parameter schema (Module 1234.js, lines 195-205)
```js
Qz$ = p.object({
  successState: p.enum(["success", "partial", "failure"])
    .describe("Whether the feature implementation succeeded"),
  returnToOrchestrator: p.boolean()
    .describe("Whether to return control to orchestrator (true = needs attention)"),
  featureId: p.string().optional().describe("Feature ID (uses currentFeatureId if not provided)"),
  commitId: p.string().optional().describe("Git commit ID (required if success)"),
  validatorsPassed: p.boolean().optional()
    .describe("Whether all validators passed (required true if success)"),
  handoff: Z11.describe("Structured handoff information for the team"),
})
```

### discoveredIssue schema (Module 0082.js, lines 37-40)
```js
jf0 = AH.object({
  severity: Rf0,  // enum: "blocking", "non_blocking", "suggestion"
  description: AH.string(),
  suggestedFix: AH.string().optional(),
})
```

### Orchestrator blocks on unaddressed handoff (Module 2544.js, lines 287-296)
```js
userError: "Cannot resume without addressing worker handoff items",
llmError: `Your action is needed before the mission run can resume.

The previous worker handoff includes items that still need to be addressed: ${y.join(", ")}.

If discoveredIssues and/or whatWasLeftUndone exist, either create new features or update existing feature descriptions if the issue belongs to a pending feature's scope.

Skip only if it is already tracked as an existing feature (cite the ID), or truly irrelevant and will never need to be fixed.

Review the handoff from feature "${SH.featureId}", take the appropriate action, then call start_mission_run again to continue.`
```

### Return value of start_mission_run with handoffs (Module 2544.js, lines 631-645)
```js
yield {
  type: "result",
  isError: false,
  value: {
    started: true,
    workerHandoffs: XH,
    latestWorkerHandoff: FH,
    systemMessage: `<system>${MH}</system>`,
  },
};
```

## Integration Points

- **01-terminal-ui (TUI)**: Module 3973.js renders handoff fields in the TUI. The `workerHandoffs` array and `latestWorkerHandoff` are passed through `start_mission_run` results to the orchestrator, which interacts with the TUI layer.
- **03-tool-agent-system (Tool)**: `EndFeatureRun` is a tool invocation (module 2539.js, 0966.js). The `dismiss-handoff-items` tool is registered as a top-level tool (1237.js).
- **04-desktop-gui (GUI)**: Handoff JSON files are stored in the mission directory; GUI could read them for visualization.
- **05-infrastructure (Infra)**: `ensureWorkerHandoffJson` writes to disk in the mission directory. Transcript skeletons are also persisted.

## Implementation Notes

1. **Schema**: Port the Zod schemas from 0082.js and 1234.js to your language of choice. The handoff object is the central contract.
2. **EndFeatureRun tool**: Implement as a callable tool that validates the handoff, updates feature status, writes handoff JSON, and generates transcript skeletons.
3. **Orchestrator blocking logic**: In `start_mission_run`, check the last `worker_completed` event for unaddressed `discoveredIssues`/`whatWasLeftUndone`. Block with a clear error message until addressed.
4. **Dismiss tool**: Implement `dismiss-handoff-items` with strict justification rules (min 20 chars, no "low priority" dismissals).
5. **Persistence**: Store handoffs as per-worker JSON files in `missionDir/handoffs/`. Track `lastReviewedHandoffCount` in state.
6. **TUI rendering**: Render `salientSummary`, `whatWasLeftUndone` (with warning color if present), and `discoveredIssues` (with severity badges).

## Known Gaps

- **Target modules 3921, 3925, 3980, 3972, 2517, 0940**: These modules did not contain handoff-keyword hits in grep. They may contain related orchestrator/reporting logic under different identifiers. A follow-up grep with broader keywords (e.g., `worker_completed`, `progressLog`, `ensureWorkerHandoffJson`) could reveal their role.
- **Transcript skeleton generation**: The `_NI(i)` function in 2539.js (line 213) that converts message events to skeletons was not traced into its implementation module.
- **Handoff file path resolution**: The exact disk path format for `ensureWorkerHandoffJson` was not read (it returns a `handoffFile` string).
- **Skill feedback aggregation**: No evidence was found of how `skillFeedback` from multiple workers is aggregated or used to update skill files.
