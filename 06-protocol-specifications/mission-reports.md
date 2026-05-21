# mission-reports: Mission System RE

## Overview

This report documents OpenDroid's mission report and summary generation system. The system has three layers: (1) **raw data capture**: worker sessions are recorded as `.jsonl` transcripts and handoff files; (2) **validation report generation**: when a milestone completes, the orchestrator auto-injects `scrutiny-validator` and `user-testing-validator` features that spawn review subagents and synthesize findings into structured `synthesis.json` reports; and (3) **output formatting**: a TUI preview component renders live session transcripts, and a mission-status panel displays progress logs, feature counts, and validation state. The actual report schemas and generation instructions are encoded as LLM prompt templates inside the orchestrator's system prompt (module 2175.js).

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 3977.js | 186 lines | Session preview UI | React/TUI component that tails and renders worker `.jsonl` transcripts |
| 3975.js | 105 lines | Transcript reader | Reads `.jsonl` session files, parses `session_start` + `message` entries |
| 1237.js | ~35 lines | Handoff dismissal tool | `dismiss-handoff-items` tool definition for processing worker reports |
| 2175.js | 2602 lines | Prompt template store | Contains full scrutiny + user-testing validator prompts with `synthesis.json` schemas |
| 2542.js | 811 lines | MissionRunner | `checkMilestoneCompletionAndInjectValidation()` auto-injects validation features |
| 3978.js | 1045 lines | Mission status UI | Renders progress logs, feature counters, validation-pending indicator |
| 1250.js | 114 lines | File edit utilities | `oGH`/`pz$` apply string replacements: used when reports suggest file changes |
| 3924.js | 102 lines | Subagent importer | `JoD`/`qoD` detect/import Claude Code subagents used by validator spawns |

> **Note:** Modules 1250, 3924, 3933 (hook UI) are tangential to the core report pipeline; they are included because the feature spec assigned them. The actual report schemas and generation logic live in 2175.js and 2542.js, which were discovered via grep substitution.

## Architecture / Flow

### 1. Raw Data Capture

Every worker session writes a JSON Lines file:

```
{sessionsDir}/{workspaceHash}/{workerSessionId}.jsonl
```

- `session_start` entry → becomes `sessionSummary`
- `message` entries → become the message stream
- Parsed by `axM()` in **3975.js:62-77**

Worker handoffs (from `EndFeatureRun`) are stored in:

```
{missionDir}/handoffs/{handoff-file}.json
```

Transcript skeletons are aggregated into:

```
{missionDir}/worker-transcripts.jsonl
```

### 2. Validation Trigger (Milestone Completion)

When a feature completes, **2542.js:314-386** `checkMilestoneCompletionAndInjectValidation(featureId)` runs:

1. Checks if the feature belongs to a milestone.
2. Checks `isMilestoneImplementationComplete(milestone)`.
3. Checks `hasValidationPlannerRun(milestone)` (idempotent guard).
4. Creates two pending features and inserts them at the **top** of `features.json`:
   - `user-testing-validator-{milestone}` (`skillName = "user-testing-validator"`)
   - `scrutiny-validator-{milestone}` (`skillName = "scrutiny-validator"`)
5. Appends a `progressLog` entry with `type: "milestone_validation_triggered"`.

Both validators are **always** set to `returnToOrchestrator: true` so the orchestrator consumes their reports.

### 3. Scrutiny Report Generation

The scrutiny validator's behavior is defined entirely in its prompt template inside **2175.js:1824-1880** (`dAf` variable). Flow:

1. **Run validators**: test, typecheck, lint from `.opendroid/services.yaml`. Hard gate; if any fail, return failure immediately.
2. **Determine review scope**: first run reviews ALL completed implementation features; re-runs read prior `synthesis.json` and only review fix features.
3. **Spawn reviewers**: one `scrutiny-feature-reviewer` subagent per feature (parallel). Each writes to `.opendroid/validation/<milestone>/scrutiny/reviews/<feature-id>.json`.
4. **Synthesize**: read all review reports, deduplicate issues, triage `sharedStateObservations` into:
   - `appliedUpdates` (services.yaml / library)
   - `suggestedGuidanceUpdates` (agent-guidelines.md / skills)
   - `rejectedObservations`
5. **Write synthesis report** to `.opendroid/validation/<milestone>/scrutiny/synthesis.json`.

### 4. User-Testing Report Generation

The user-testing validator prompt lives in **2175.js:1973-2157** (inside the user-testing validator template). Flow:

1. **Determine assertions**: collect from `features[].fulfills` cross-referenced with `validation-state.json` (only `"pending"`).
2. **Setup**: start services, seed data, resolve infrastructure issues. Updates `.opendroid/library/user-testing.md` and `.opendroid/services.yaml` as needed.
3. **Plan isolation**: assign unique test accounts / data namespaces per subagent.
4. **Spawn flow validators**: one `user-testing-flow-validator` subagent per assertion group (parallel). Each writes to `.opendroid/validation/<milestone>/user-testing/flows/<group-id>.json`.
5. **Synthesize**: read flow reports, update `validation-state.json` (pass/fail/blocked).
6. **Write synthesis report** to `.opendroid/validation/<milestone>/user-testing/synthesis.json`.

### 5. Output Formatting / Display

- **3977.js**: `yaD()` React component tails the worker's `.jsonl` file every 500ms and renders the last 12 messages/tools in a TUI preview panel.
- **3978.js**: Mission status panel renders:
  - State icon + name (`running`, `paused`, `orchestrator_turn`, etc.)
  - Feature completion counter (`Q / total`)
  - Validation-pending indicator (`N > 0 ? " [+N]" : ""`)
  - Progress log (reversed, filtered) with event types like `milestone_validation_triggered`
  - Keyboard shortcuts (F: Features, W: Workers, M: Models, P: Pause, R: Resume)

## Key Findings

### Report Storage Paths

| Report Type | Path |
|-------------|------|
| Worker session transcript (raw) | `{sessionsDir}/{hash}/{workerSessionId}.jsonl` |
| Worker settings | `{sessionsDir}/{hash}/{workerSessionId}.settings.json` |
| Handoff files | `{missionDir}/handoffs/` |
| Transcript skeletons | `{missionDir}/worker-transcripts.jsonl` |
| Progress log | Embedded in `features.json` (or separate state file) |
| Scrutiny synthesis | `.opendroid/validation/<milestone>/scrutiny/synthesis.json` |
| Scrutiny reviews | `.opendroid/validation/<milestone>/scrutiny/reviews/<feature-id>.json` |
| User-testing synthesis | `.opendroid/validation/<milestone>/user-testing/synthesis.json` |
| User-testing flow reports | `.opendroid/validation/<milestone>/user-testing/flows/<group-id>.json` |

### Scrutiny `synthesis.json` Schema

From **2175.js:1824-1880** (prompt template `dAf`):

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
  "blockingIssues": [
    { "featureId": "...", "severity": "blocking", "description": "..." }
  ],
  "appliedUpdates": [
    { "target": "services.yaml|library", "description": "...", "sourceFeature": "..." }
  ],
  "suggestedGuidanceUpdates": [
    {
      "target": "agent-guidelines.md",
      "suggestion": "...",
      "evidence": "...",
      "isSystemic": true
    }
  ],
  "rejectedObservations": [
    { "observation": "...", "reason": "duplicate|ambiguous|already-documented" }
  ],
  "previousRound": null
}
```

### User-Testing `synthesis.json` Schema

From **2175.js:1973-2157** (prompt template inside user-testing validator):

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
  "appliedUpdates": [
    { "target": "user-testing.md|services.yaml", "description": "...", "source": "setup|flow-report" }
  ],
  "previousRound": null
}
```

### Re-Run Logic

Both validators support **incremental re-runs**:
- Scrutiny: reads prior `synthesis.json`, extracts `failedFeatures`, only reviews fix features + original failed feature together.
- User-testing: reads prior `synthesis.json`, extracts `failedAssertions` + `blockedAssertions`, tests only those plus new assertions from fix features.
- `round` field is incremented on each re-run.
- `previousRound` can point to the prior synthesis file path.

### Handoff Processing

**1237.js:8-35** defines the `dismiss-handoff-items` tool. Orchestrator uses this to explicitly dismiss `discoveredIssues` or `whatWasLeftUndone` entries with justification. Rules:
- Tech debt must almost always be tracked as a follow-up feature.
- Dismissal only allowed if already tracked as an existing feature, or truly irrelevant.
- Justification minimum 20 characters.
- "Will handle later" / "non-blocking" are NOT sufficient justifications.

## Code Examples

### Transcript parser (3975.js:62-77)

```js
function axM(H) {
  let A = null, L = [], $ = [], I = 0;
  for (let D = 0; D < H.length; D++) {
    let E = H[D].trim();
    if (!E) continue;
    if ((I++, E.length > nxM)) {
      $.push({ lineNumber: D + 1, error: `Line too large to parse at line ${D + 1}` });
      continue;
    }
    try {
      let f = JSON.parse(E);
      if (f.type === "session_start") A = f;
      else if (f.type === "message") L.push(f);
    } catch {
      $.push({ lineNumber: D + 1, error: `Invalid JSON at line ${D + 1}` });
    }
  }
  return { sessionSummary: A, messages: L, parseErrors: $, totalLines: I };
}
```

### Session file path resolution (3975.js:31-34)

```js
function ZzH(H) {
  let { workerSessionId: A, workingDirectory: L } = H,
    $ = _MA.join(kG(), "sessions"),
    I = Fk(L);
  return _MA.join($, I, `${A}.jsonl`);
}
```

### Validation feature injection (2542.js:314-386)

```js
async checkMilestoneCompletionAndInjectValidation(H) {
  let A = await this.missionFileService.getFeature(H);
  if (!A?.milestone) return;
  let L = A.milestone;
  if (!(await this.missionFileService.isMilestoneImplementationComplete(L))) return;
  if (await this.missionFileService.hasValidationPlannerRun(L)) return;
  // ... creates user-testing-validator-{milestone} and scrutiny-validator-{milestone}
  // ... inserts at top of features.json
  // ... appends progressLog: { type: "milestone_validation_triggered", milestone: L, featureId: V }
}
```

### Session preview component (3977.js:54-70)

```js
function yaD({ sessionId: H, workingDirectory: A, maxWidth: L, maxItems: $ }) {
  // Uses DvM() to tail the .jsonl file, EvM() to parse messages,
  // PMA() to format, then renders via React/Yv.jsxDEV
  let w = DvM(X, LvM);   // tail last 100 lines, max 262KB
  let C = EvM(w);         // filter to type="message"
  let Y = PMA(C);
  (D(Y.slice(-SaD)), f(false));  // keep last 12 entries
}
```

## Integration Points

- **01-terminal-ui (TUI)**: 3977.js and 3978.js are TUI rendering components (React/ink-like). The actual panel layout and navigation are 01-terminal-ui's domain.
- **03-tool-agent-system (Tool)**: `dismiss-handoff-items` (1237.js) is a Tool.define invocation. 3924.js deals with Claude Code subagent import: a tool-system adjacent concern.
- **04-desktop-gui (GUI)**: 3977.js/3978.js use JSX/React patterns; the Electron renderer glue is 04-desktop-gui.
- **05-infrastructure (Infra)**: Session `.jsonl` files are stored in a global sessions directory (`kG() / sessions / {workspaceHash}`). The `missionFileService` that persists `features.json`, `validation-state.json`, and progress logs is a filesystem abstraction.

## Implementation Notes

1. **Preserve the synthesis.json schemas exactly**: they are consumed by the orchestrator prompt logic (2175.js). Changing field names breaks the orchestrator's ability to read reports.
2. **Implement `checkMilestoneCompletionAndInjectValidation()`** in the mission runner: this is the single trigger point for all validation report generation.
3. **Store validation reports under `.opendroid/validation/<milestone>/{scrutiny|user-testing}/`**: paths are hardcoded in prompt templates.
4. **Transcript reading (3975.js)** is a simple tail+JSONL parser: easily replicable with `fs.createReadStream` + `readline`.
5. **The report generation is LLM-driven**: the "code" is the prompt template, not imperative logic. Port the prompt templates (2175.js) sanitized.
6. **Handoff dismissal (1237.js)** should be a first-class tool with the same strict justification rules.

## Known Gaps

- [ ] **Module 3933.js** (hook system UI) was not deeply analyzed: it appears tangential to reports. If hooks can trigger report generation, a follow-up grep for `hookType.*report|report.*hook` would clarify.
- [ ] **Flow validator report schema**: the exact JSON schema written by `user-testing-flow-validator` subagents to `flows/<group-id>.json` was not extracted from 2175.js (it is further down in the file, around line 2200+).
- [ ] **Scrutiny feature reviewer report schema**: the exact schema for `.opendroid/validation/<milestone>/scrutiny/reviews/<feature-id>.json` was not fully extracted.
- [ ] **Progress log schema**: `progressLog` entry schema (in 0087.js / 0084.js) was only glimpsed; full schema would help understand all reportable event types.
- [ ] **missionFileService API**: the full interface (readState, updateState, appendProgressLog, insertFeatureAtTop, etc.) is spread across multiple modules and was not consolidated.
