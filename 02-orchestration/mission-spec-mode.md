# mission-spec-mode: Mission System RE

## Overview

OpenDroid's **spec mode** is a constrained interaction mode where the agent is forbidden from making any system modifications until the user approves an implementation plan. The agent researches, crafts a spec, and then calls `ExitSpecMode` to present the plan for approval. The companion `ProposeMission` tool serves a similar role at the mission level, presenting a multi-feature mission plan before spawning workers. Together these tools form a gated execution model that prevents autonomous code changes without explicit user consent.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 1226.js | 55 lines | `exit-spec-mode` tool definition | Defines `Propose Specification` tool schema, description, and constraints |
| 1235.js | ~30 lines | `propose-mission` tool definition | Defines `Propose Mission` tool for multi-feature mission proposals |
| 0060.js | 96 lines | Confirmation enums | `ExitSpecMode`, `ProposeMission`, `WaitingForToolConfirmation`, outcome types |
| 0084.js | 242 lines | Event schemas | Zod schemas for `exit_spec_mode` and `propose_mission` confirmation events |
| 0939.js | 930 lines | Settings service | `specModeModel`, `specModeReasoningEffort`, `specSaveDir`, listeners |
| 1185.js | 1823 lines | Session service | `isSpecMode()` (interactionMode === "spec"), mode cycling, settings persistence |
| 2532.js | 139 lines | ExitSpecMode executor | Saves plan to file, opens editor, generates approval message |
| 3139.js | 207 lines | Confirmation UI factory | Returns confirmation details for `ExitSpecMode` and `ProposeMission` |
| 2572.js | 987 lines | Conversation manager | Spec mode error message constants (`QRI`, `JRI`, `wRI`, `KRI`) |
| 3146.js | 712 lines | Tool execution hook | Blocks non-readonly tools in spec mode, handles approval/rejection |
| 3399.js | 1303 lines | System prompt builder | Injects "Spec mode is active" system reminder into LLM context |
| 2575.js | 409 lines | Session controller | `switchSpecModeModel()`, bridges settings ↔ session |

> **Note:** The mission brief listed modules 1206, 0910, 2479, 3876, 0912, 1226. Only 1226 was directly relevant; the rest were stale mappings (1206 = WebSearch, 0910 = daemon RPC constants, 2479 = apply_patch, 3876 = process manager UI, 0912 = terminal schemas). The actual spec-mode modules were discovered via grep for `spec mode`, `ExitSpecMode`, `propose_mission`, and `isSpecMode`.

## Architecture / Flow

### Spec Mode Entry

1. **User toggles mode**: The user switches interaction mode from `auto` → `spec` (or the orchestrator enters it). This is done via `SessionService.cycleInteractionMode()` (1185.js:214) or `setInteractionMode("spec")` (1185.js:131).
2. **Model switch**: If a `specModeModel` is configured (0939.js:687), the session switches to that model for the spec-writing phase (2575.js:277 `switchSpecModeModel()`). If none is set, the default model is used.
3. **System reminder injection**: On every LLM request, `useAgent` in 3399.js (line 378) checks `BA().isSpecMode()`. If true, it injects a system reminder:
   > "Spec mode is active. The user indicated that they do not want you to execute yet: you MUST NOT make any edits, run any non-readonly tools... Instead, you should: 1. Answer the user's query with generated spec using the ExitSpecMode tool. 2. When you're done researching, present your spec by calling the ExitSpecMode tool..."

### Spec Mode Gating (Tool Blocking)

While in spec mode, the tool execution layer enforces restrictions:

- **3146.js (line ~216)**: If spec mode is active and a tool is NOT in the readonly allowlist (`PYH`), the tool call is cancelled with error `wRI`:
  > "Error: Tool cancelled: Spec mode is active - file modifications or tools that modify the system state in any way are not allowed"
- **2572.js (lines 11-17)**: Defines the constant messages:
  - `QRI`: generic spec mode block message
  - `JRI`: tool cancelled variant
  - `wRI`: error variant (used in 3146.js)
  - `KRI`: "Plan not approved - remaining in Spec Mode. Provide feedback to refine the spec."

### ExitSpecMode / Plan Proposal Flow

1. **Agent calls `ExitSpecMode`**: The agent finishes research and calls the tool (1226.js), passing:
   - `plan` (markdown string, required)
   - `title` (optional)
   - `optionNames` (optional array of option labels)

2. **Confirmation UI generated**: 3139.js (line 89) handles the `ExitSpecMode` case:
   - Returns `{ type: "exit_spec_mode", plan, title, optionNames, onConfirm: async () => {} }`
   - In non-interactive CLI mode, confirmation is skipped (`return false`)

3. **User review**: The UI presents the plan. The user can:
   - **Approve** (`proceed_once`) → spec mode exits, agent proceeds with implementation
   - **Approve & Edit** (`proceed_edit`) → plan is saved and opened in editor; agent waits for user to finish editing
   - **Cancel** (`cancel`) → agent remains in spec mode (`KRI` message injected)

4. **Tool execution on approval**: 3146.js (lines 315-326):
   - Sets `confirmationOutcome` to `"proceed_edit"` or `"proceed_once"`
   - Passes `selectedExitSpecModeOptionIndex` if options were presented
   - Passes `exitSpecModeComment` if user left a comment

5. **Executor saves plan**: 2532.js (`doA.execute()`):
   - Saves plan to `specSaveDir` (default: `~/.opendroid/specs/`)
   - Filename generated via `zgH(B, V, A.plan)` (slugified title + timestamp)
   - If `proceed_edit`, opens file in IDE or external editor
   - Returns result with `approved: true`, `filePath`, `isEdited`, `selectedOptionIndex`, `selectedOptionName`

### ProposeMission Flow

`ProposeMission` (1235.js) works similarly but at the mission level:

1. **Agent calls `ProposeMission`**: passes `proposal` (markdown) and optional `title`
2. **Confirmation UI**: 3139.js (line 100) returns `{ type: "propose_mission", proposal, title }`
3. **User approval**: Same confirmation outcomes as ExitSpecMode
4. **Mission creation**: On acceptance, the system creates `missionDir` and writes `mission.md`, `features.json`, `validation-contract.md`, etc. (handled by the mission runner, referenced in 2175.js)
5. **System notification**: 3146.js (line 395-400): If accepted, injects a system notification with mission directory path via `VNI(b.missionDir)`

### Interaction Mode State Machine

From 1185.js (lines 204-212):

```
auto  ──cycle──→  spec  ──cycle──→  auto
                  ↑
                  └── (agi bypasses spec mode)
```

- `cycleInteractionMode()` rotates: `auto` → `spec` → `auto`
- Orchestrator sessions auto-enable `agi` mode (1185.js:188), which bypasses spec mode restrictions (3399.js:380 checks `!GB` where `GB = isAGIMode()`)

## Key Findings

### 1. Spec Mode is Just `interactionMode === "spec"`

The entire spec mode mechanism hinges on a single string comparison (1185.js:174):

```js
isSpecMode() {
  return this.getInteractionMode() === "spec";
}
```

This means spec mode is not a separate runtime state machine: it's a session setting that propagates through:
- System prompt injection (3399.js)
- Tool blocking (3146.js)
- Model selection (3399.js:296-299 uses specModeModel when in spec mode)

### 2. Spec Mode Has a Dedicated Model Configuration

Users can configure a separate model for spec mode (0939.js:687-716):
- `specModeModel`: model ID used during spec drafting
- `specModeReasoningEffort`: reasoning effort for spec mode
- If unset, falls back to the default model (0939.js:689)

### 3. Plan Files Are Persisted to Disk

The `ExitSpecMode` executor (2532.js) always writes the plan to a file:
- Directory: `settings.general.specSaveDir` or `~/.opendroid/specs/`
- Template detection: checks for `SPEC_TEMPLATE.md` in project root or user home (2532.js:15-25 `HUf()`)
- Telemetry: records `SPEC_SAVED` or `SPEC_SAVE_FAILED` events

### 4. Two Parallel Proposal Tools Exist

| Tool | Scope | When Used |
|------|-------|-----------|
| `ExitSpecMode` | Single feature / task | Agent has finished crafting an implementation plan for a coding task |
| `ProposeMission` | Multi-feature mission | Orchestrator decomposing a large task into sequential worker features |

Both share the same confirmation UI infrastructure (3139.js) and approval flow (3146.js).

### 5. Spec Mode Does NOT Block Read-Only Tools

The allowlist `PYH` in 3146.js permits certain tools even in spec mode (e.g., `Read`, `LS`, `WebSearch`, `FetchUrl`). Only destructive tools (`Edit`, `Create`, `Execute`, `ApplyPatch`) are blocked.

## Code Examples

### System prompt injection when spec mode is active
```js
// Module 3399.js, lines 378-396
let E_ = BA().isSpecMode(),
  GB = BA().isAGIMode();
if (E_ && !GB) {
  let $L = F$(BA().getModel()).modelProvider !== "openai",
    D$ = nf().isAskUserToolEnabled(),
    kL = `
Spec mode is active. The user indicated that they do not want you to execute yet: you MUST NOT make any edits, run any non-readonly tools...
1. Answer the user's query with generated spec using the ExitSpecMode tool.
2. When you're done researching, present your spec by calling the ExitSpecMode tool...`;
  if (D$) kL += `\n3. Use the AskUser tool to gather requirements...`;
  if ($L) kL += `\n\nIMPORTANT: Do not make calls to Edit, Create, and ApplyPatch tools when spec mode is active.`;
  CE.push({ type: "text", text: Dl(kL) });
}
```

### Tool blocking in spec mode
```js
// Module 3146.js, lines 216-218
let SH = wRI;
(A(ZH.id, SH), IH.set(ZH.id, { type: "tool_result", toolUseId: ZH.id, content: SH }));
```
Where `wRI` = `"Error: Tool cancelled: Spec mode is active - file modifications or tools that modify the system state in any way are not allowed"`

### ExitSpecMode tool definition
```js
// Module 1226.js, lines 23-43
M2 = eI({
  id: "exit-spec-mode",
  llmId: OKA,
  uiGroupId: "planning",
  displayName: "Propose Specification",
  description: `Use this tool only when you are in spec mode and have finished crafting a concrete implementation plan...
  
  When NOT to use this tool:
  • Pure research / investigation tasks...
  • If the system message "Spec mode is active" is **NOT** present.
  `,
  executionLocation: "client",
  inputSchema: U11,  // { plan: string, optionNames?: string[], title?: string }
  outputSchemas: { result: _11 },  // { approved: boolean }
  isVisibleToUser: true,
  isTopLevelTool: true,
  requiresConfirmation: true,
  toolkit: "Base",
  isToolEnabled: true,
});
```

### Spec mode state check
```js
// Module 1185.js, lines 174-176
isSpecMode() {
  return this.getInteractionMode() === "spec";
}
```

### Plan persistence in ExitSpecMode executor
```js
// Module 2532.js, lines 28-40
let B = $.getSpecSaveDir(),
  W = YGH(B);
await ps.mkdir(W, { recursive: true });
let V = (A.title || "").trim();
((I = await zgH(B, V, A.plan)), await ps.writeFile(I, A.plan, "utf-8"));
```

## Integration Points

- **01-terminal-ui (TUI)**: The confirmation UI for ExitSpecMode/ProposeMission is rendered by the TUI layer. The `requiresConfirmation: true` flag on both tools (1226.js, 1235.js) triggers TUI confirmation panels. The `isNonInteractiveCLIMode()` check in 3139.js bypasses UI in headless mode.
- **03-tool-agent-system (Tool)**: ExitSpecMode and ProposeMission are Base toolkit tools registered alongside `AskUser`, `WebSearch`, etc. The tool blocking logic (3146.js) references the same `PYH` allowlist used by other permission systems.
- **04-desktop-gui (GUI)**: The `ideClient` in 2532.js opens the spec file in the IDE (VS Code / Cursor) when `proceed_edit` is chosen. This uses the GUI↔IDE bridge.
- **05-infrastructure (Infra)**: Session settings (`interactionMode`, `specModeModel`) are persisted to `~/.opendroid/sessions/` and optionally synced to cloud (1185.js:136). `specSaveDir` defaults to `~/.opendroid/specs/`.

## Implementation Notes

1. **Core abstraction**: A session-level `interactionMode` enum (`auto` | `spec` | `agi`) with `isSpecMode()` predicate.
2. **Prompt injection**: Before sending messages to the LLM, check `isSpecMode()` and inject the spec-mode system reminder (sanitized text from 3399.js).
3. **Tool gating**: Maintain a readonly tool allowlist. In spec mode, intercept tool calls for non-readonly tools and return the `wRI` error message without executing.
4. **ExitSpecMode tool**: Define a tool with schema `{ plan: string, title?: string, optionNames?: string[] }`. On execution, save to disk, optionally open editor, return `{ approved: true, filePath, ... }`.
5. **ProposeMission tool**: Define a tool with schema `{ proposal: string, title?: string }`. On acceptance, create `missionDir` and write mission files.
6. **Confirmation flow**: Both tools need `requiresConfirmation: true`. The UI shows plan markdown, options, and Approve/Edit/Cancel buttons.
7. **Spec mode model**: Allow configuring a separate model/reasoning-effort for spec mode. Store in session settings.

## Known Gaps

- The exact content of `VNI()` (mission acceptance notification) was not fully traced: it lives in a module not yet read.
- The `PYH` readonly tool allowlist definition was not located: it likely lives in a permission/module registry module.
- `ProposeMission` mission directory creation logic (writing `mission.md`, `features.json`, etc.) is referenced in 2175.js but not traced into the actual file-writing code.
- The module catalog mappings for this feature were stale (4 of 6 listed modules were irrelevant). Suggest updating `.opendroid/library/module-catalog.md` with the corrected module list above.
