# tui-panel-permission-dialog: TUI Architecture Notes

## Overview

The Permission/Confirmation Dialog is the central TUI component that renders tool approval requests and collects user decisions. It is implemented as the `SiD` React component in **3872.js**, which renders a dynamic multi-type confirmation UI using Ink `<Box>`/`<Text>` primitives. The dialog handles seven distinct confirmation types (edit, create, exec, apply_patch, ask_user, exit_spec_mode, propose_mission, start_mission_run), each with specialized rendering and option sets. Keyboard navigation uses Ink's `useInput` hook with ↑/↓ arrows, Enter, Esc, and Ctrl+O (toggle diff view). The permission flow is a multi-layer event bus architecture: `useToolExecution` hook (3146.js) emits `permission-request` events → protocol adapters (3428.js JsonRpcAdapter, 3417.js AcpAdapter) route to the client (3407.js) → the TUI renders the dialog → user response triggers `permission-response` event → tool execution proceeds or cancels.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|-----|------------|
| 3872.js | 11.5 KB | PermissionDialog UI component (`SiD`), diff rendering, option selector | App |
| 3141.js | ~8 KB | Option generation (`UYH`), outcome processing (`ee`), confirmation prep (`yWD`) | App |
| 3146.js | ~30 KB | `useToolExecution` hook: permission flow orchestration, event bus emit/respond | App |
| 3416.js | ~2 KB | Single-tool confirmation helper (`vYH`), tool kind classifier (`lDM`) | App |
| 3421.js | ~8 KB | Autonomy mode definitions (Auto Off/Spec/Low/Medium/High), ACP Daemon | App |
| 3399.js | ~50 KB | Agent state management: sets `tool_confirmation` state when pending | App |
| 3428.js | ~15 KB | JsonRpcAdapter: pending permission requests, protocol layer | App |

## Architecture

### Component Hierarchy

```
SiD (3872.js): Main ConfirmationDialog component
├── CfA: AskUser questions component (confirmationType === "ask_user")
├── ZiD: MissionProposal component (confirmationType === "propose_mission")
├── NiD: ExitSpecMode component (confirmationType === "exit_spec_mode")
├── PF: DiffView component (renders unified diffs for edit/create/apply_patch)
├── f3: SelectInput component (renders option list with selection highlight)
└── nl: Label generator function (generates display text per confirmationType)
```

### Permission Flow (End-to-End)

```
1. Agent generates tool calls → useToolExecution (3146.js)
2. yWD() prepares confirmation data (tool name, type, details)
3. UYH() generates option list based on confirmation types + autonomy level
4. Event "permission-request" emitted via v$ event bus
5. Protocol adapters (3428.js, 3417.js) route to client
6. Agent state (3399.js) transitions to "tool_confirmation"
7. SiD (3872.js) renders dialog with options + diff preview
8. User navigates (↑/↓), selects (Enter), or cancels (Esc)
9. Callback resolves Promise → "permission-response" emitted
10. ee() processes outcome → approved tool IDs returned
11. Tools execute or cancel
```

### Confirmation Types

| Type | Description | Special Rendering |
|------|-------------|-------------------|
| `edit` | File edit operation | Diff view with old/new content |
| `create` | File creation | Diff view from empty |
| `exec` | Shell command execution | Shows command, deny/allow list status |
| `apply_patch` | Patch application | Diff view with patch content |
| `ask_user` | User question (ask_user tool) | Renders CfA questions component |
| `exit_spec_mode` | Exit spec mode + implement | Renders NiD with implementation options |
| `propose_mission` | Mission proposal review | Renders ZiD with approve/edit/reject |
| `start_mission_run` | Start mission with conflicts | Warning about running missions |

### Autonomy Levels (3421.js)

```js
L7D = {
  normal: { name: "Auto (Off)", description: "Auto-approves only read operations" },
  spec: { name: "Spec", description: "Build feature specs (read-only)" },
  "auto-low": { name: "Auto (Low)", description: "Auto-approves file edits and low-risk actions during the session" },
  "auto-medium": { name: "Auto (Medium)", description: "Auto-approves medium-risk actions during the session" },
  "auto-high": { name: "Auto (High)", description: "Auto-approves all actions" },
}
```

## Key Components

### SiD: Main Confirmation Dialog (3872.js, line 80)

```js
function SiD({ confirmationDetails: H, isFocused: A = true, width: L = 60, ideClient: $ }) {
  let { tools: I, onConfirm: D } = H,
    [E, f] = aZH.useState(0),       // selectedIndex
    [M, U] = aZH.useState([]),       // diffOpenFiles
    [P, B] = aZH.useState(false),    // showFullDiff
  // ...
  // Dispatches to specialized components based on confirmationType:
  if (I.some((DH) => DH.confirmationType === "ask_user") && I.length === 1)
    return b0.jsxDEV(CfA, { questions: VH.parsed?.questions || [], ... });
  if (Y && I.length === 1)  // propose_mission
    return b0.jsxDEV(ZiD, { title: VH.title, proposal: VH.proposal, ... });
  if (w && I.length === 1)  // exit_spec_mode
    return b0.jsxDEV(NiD, { title: VH.title, plan: VH.plan, ... });
  // Otherwise: batch confirmation with diff preview + option list
}
```

### nl: Confirmation Label Generator (3872.js, line 60)

```js
function nl(H) {
  let { toolName: A, confirmationType: L, details: $ } = H;
  switch (L) {
    case "edit": return `Edit ${$.fileName}`;
    case "create": return `Create ${$.fileName}`;
    case "exit_spec_mode": return "Exit Spec Mode and proceed with implementation";
    case "propose_mission": return "Review mission proposal";
    case "start_mission_run": {
      let I = $, D = I.runningMissionCount === 1 ? "mission is" : "missions are";
      return `${I.runningMissionCount} other ${D} currently running. Start this mission anyway?`;
    }
    case "exec": {
      let I = $, D = VV.isCommandDenied(I.fullCommand),
        E = VV.isCommandAllowed(I.fullCommand);
      return `${D ? "Run (denylisted):" : E ? "Run (allowlisted):" : "Run:"} ${I.fullCommand}`;
    }
    case "apply_patch": return `Apply patch to ${$.fileName}`;
    default: return `${A} operation`;
  }
}
```

### Keyboard Input Handler (3872.js, line ~340)

```js
jL(  // useInput
  async (DH, VH) => {
    if (V && (DH === "[111;5u" || DH === "\x0F")) {  // Ctrl+O
      B((IH) => !IH);  // toggle showFullDiff
      return;
    }
    if (VH.escape) await x("cancel");           // Esc → cancel
    else if (VH.upArrow) f(IH => (IH <= 0 ? N.length - 1 : IH - 1));   // ↑ navigate
    else if (VH.downArrow) f(IH => (IH >= N.length - 1 ? 0 : IH + 1)); // ↓ navigate
    else if (VH.return) await x(N[E].value);    // Enter → select
  },
  { isActive: A && W.isEnabled },
);
```

### Option Rendering (3872.js, line ~870)

```js
b0.jsxDEV(f3, {  // SelectInput component
  items: N,       // option array from UYH()
  selectedIndex: E,
  helpText: P
    ? "Use ↑↓ to navigate, Enter to select, Esc to cancel, Ctrl+O to collapse"
    : V && !Q
      ? "Use ↑↓ to navigate, Enter to select, Esc to cancel, Ctrl+O to expand diff"
      : "Use ↑↓ to navigate, Enter to select, Esc to cancel",
})
```

### UYH: Option Generator (3141.js, line ~30)

```js
function UYH({ hasExitSpecMode, hasStartMissionRun, toolCount, toolConfirmationInfoInputs, hasDeniedCommands }) {
  // Exit spec mode: offers proceed + auto-approve levels
  if (H) return [
    { label: "Proceed with implementation", value: "proceed_once", selectedColor: "#E3992A" },
    { label: "Proceed, and allow file edits (Low)", value: "proceed_auto_run_low", selectedColor: "#E3992A" },
    { label: "Proceed, and allow reversible commands (Medium)", value: "proceed_auto_run_medium", selectedColor: "#E3992A" },
    { label: "Proceed, and allow all commands (High)", value: "proceed_auto_run_high", selectedColor: "#E54048" },
    { label: "No, keep iterating on spec", value: "cancel", selectedColor: "#E54048" },
  ];
  // Single tool:
  //   "Yes, allow" (proceed_once) + "Yes, and always allow [type]" (proceed_always) + "No, cancel"
  // Batch tools:
  //   "Yes, allow all" (proceed_once) + "Yes, and always allow [type]" (proceed_always) + "No, cancel all"
}
```

### ee: Outcome Processor (3141.js, line ~120)

```js
function ee({ outcome, tools, approvedToolIds, updateAction, selectedOptionName }) {
  if (outcome === "cancel") return [];                           // nothing approved
  if (outcome === "proceed_always") { o8().setAutonomyLevel(...); return all; }  // set mode
  if (outcome === "proceed_edit") { o8().setAutonomyMode("normal"); return all; }
  if (outcome includes "proceed_auto_run_*") { o8().setAutonomyMode(level); return all; }
  // proceed_once: return specific approvedToolIds
}
```

### useToolExecution Hook (3146.js, line ~30)

```js
// Main permission request emission:
v$.emit("permission-request", {
  requestId: FH,
  toolUses: EH.toolUses,
  options: EH.options,
  sessionId: OH,
});
// Response listener:
PO("permission-response", (z) => {
  if (z.requestId === FH) { resolve(z.approvedToolIds); }
});
```

## Theme / Style Tokens

| Token | Usage | Value (from theme) |
|-------|-------|--------------------|
| `o.primary` | Section headers ("File Edits", "Commands"), "Approve N operations" | Theme primary color |
| `o.border` | Diff box border: `borderStyle: "round"`, `borderColor: o.border` | Theme border color |
| `o.text.muted` | "...N more lines, press Ctrl+O to expand", collapse hint | Theme muted text color |
| `#E3992A` | Selected/highlighted option color for approval actions | Hardcoded amber |
| `#E54048` | Selected option color for cancel/danger actions, denylisted commands | Hardcoded red |
| `"round"` | Diff container border style | Constant |

### VS Code Diff Integration

The dialog supports opening diffs in VS Code via two paths:
1. **MCP**: If `ideClient.isConnected()`, uses `$.openDiff(filePath, newContent)`
2. **CLI fallback**: If `OPENDROID_EDITOR_MCP_PORT` is set, spawns `code --diff` via child_process

```js
// Module 3872.js, line ~130
if ($?.isConnected())
  DH.forEach((i) => {
    let HH = i.details;
    if (HH.oldContent !== void 0 && HH.newContent !== void 0 && HH.filePath)
      $.openDiff(HH.filePath, HH.newContent)
        .then(() => { U((PH) => [...PH, HH.filePath]); });
  });
```

## Keyboard / Input Handling

| Key | Action | Context |
|-----|--------|---------|
| `↑` (upArrow) | Move selection up (wraps to bottom) | Always active when focused |
| `↓` (downArrow) | Move selection down (wraps to top) | Always active when focused |
| `Enter` (return) | Select current option | Always active when focused |
| `Esc` (escape) | Cancel all operations | Always active when focused |
| `Ctrl+O` (`\x0F` / `[111;5u`) | Toggle expand/collapse diff view | Only when file edits present |

The input handler is registered via `jL` (Ink's `useInput`) with `{ isActive: isFocused && isEnabled }` guard, where `isEnabled` comes from `gX()` (a global focus/context manager).

### Option Values

| Option Value | Description |
|-------------|-------------|
| `proceed_once` | Approve once (for current batch) |
| `proceed_always` | Approve + set permanent autonomy level for this tool type |
| `proceed_edit` | Approve with spec editing (exit_spec_mode) |
| `proceed_auto_run_low` | Approve + set auto-low mode |
| `proceed_auto_run_medium` | Approve + set auto-medium mode |
| `proceed_auto_run_high` | Approve + set auto-high mode |
| `cancel` | Reject/cancel all pending operations |

## Integration Points

- **tui-theme-system** (3773.js, 0939.js): Theme tokens `o.primary`, `o.border`, `o.text.muted` consumed by dialog rendering
- **tui-react-runtime** (2206.js): React `useState` hooks for selection state, diff expansion
- **tui-ink-core** (2228.js): Ink `useInput` hook for keyboard handling, `<Box>`/`<Text>` JSX rendering
- **tui-logo-animation** (4065.js): Consumes `SiD` component via `pendingConfirmation` state; shows "Waiting for tool confirmation..." in status overlay
- **tui-renderer-main-loop**: Coordinates `tool_confirmation` state with the main render loop
- **02-orchestration (mission system)**: `propose_mission` and `start_mission_run` confirmation types cross into mission orchestration boundary
- **03-tool-agent-system (tool system)**: Tool execution sandbox and ask_user tool feed into the confirmation flow

## Implementation Notes

1. **Port `SiD` component** as a generic `<ConfirmationDialog>` with configurable option schemas
2. **Decouple event bus**: Replace `v$` (event emitter) with a standard pub/sub or callback pattern
3. **Theme tokens**: Map `o.primary`, `o.border`, `o.text.muted` to your theme system
4. **Autonomy levels**: The 5-level auto-approve hierarchy (off/spec/low/medium/high) is portable as a config schema
5. **Diff rendering**: The `PF` component (unified diff) can be reused; the VS Code diff integration may need adaptation for your IDE
6. **Keyboard bindings**: ↑/↓/Enter/Esc/Ctrl+O are standard TUI patterns: portable to any Ink-based or blessed-based renderer
7. **Confirmation types**: The enum (`edit`, `create`, `exec`, `apply_patch`, `ask_user`, `exit_spec_mode`, `propose_mission`) should be extended with your custom tool types

## Module Reference

| Module | Lines | Key Content |
|--------|-------|-------------|
| 3872.js | 1-899 | SiD (line 80), nl (line 60), CTL VS Code diff (line 14), keyboard handler (~340), option rendering (~870) |
| 3141.js | 1-200+ | UYH option generator (~30), ee outcome processor (~120), yWD batch prep (~90), H1L system message (line 5) |
| 3146.js | 1-712 | useToolExecution hook (~30), permission-request emit (~165), permission-response handler (~162) |
| 3416.js | 1-50 | lDM tool kind classifier (line 1), vYH single-tool helper (line 14), uYH outcome mapper (line 36) |
| 3421.js | 1-199 | L7D autonomy mode definitions (line 28), nDM mode list (line 37), ACP Daemon class (line 43) |
| 3399.js | 1-1303 | jA state transition callback (~1165), pendingConfirmation state (~151, ~1291) |
| 3428.js | 1-450+ | pendingPermissionRequests Map (line 14), requestPermission (electronAPI section0), respondToPermission (~322), clearAll (~364) |
