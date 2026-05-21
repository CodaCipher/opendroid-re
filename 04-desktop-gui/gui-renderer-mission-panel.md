# gui-renderer-mission-panel: GUI Architecture Notes

## Overview

This report documents the **Mission Panel** and **Session Sidebar** UI subsystems in OpenDroid's Electron renderer. The mission panel displays autonomous multi-agent mission progress (features, milestones, workers, token usage) in a right-hand pane. The session sidebar (`i4s`) provides the left-hand navigation tree with collapsible groups, session rows, and virtual scrolling. Data flows from the daemon via WebSocket JSON-RPC notifications into a singleton `MissionStore` per session, then into React components via `useSyncExternalStore`.

## File Map

| File | Size | Role |
|------|------|------|
| `renderer-main.js` | ~large renderer bundle | Main renderer bundle: contains all components documented here |
| `theme.css` | 29 KB | Theme tokens (`--surface-*`, `--border-*`, `--text-*`) |

## Architecture

### Component Hierarchy

```
App
└── Allotment (B9) split pane
    ├── Sidebar Pane (left)
    │   └── i4s (SessionManagerSidebar)
    │       ├── U7n (sidebar container)
    │       ├── H7n (collapse toggle)
    │       └── f5s (virtual list)
    │           ├── A5s (group header)
    │           └── z5s (session row)
    └── Detail Pane (right)
        └── cXs (SessionDetailPage)
            ├── lXs (EmbeddedEditorLayout wrapper) OR split pane
            │   └── YYs (session detail layout)
            │       ├── Hgr (chat / main pane)
            │       └── Right pane (conditional)
            │           ├── Qpr (MissionPanel) when H===_b.Mission
            │           ├── ogr (DiffPanel) when H===_b.DiffPanel
            │           └── VSCode iframe when H===_b.VSCode
            └── lWs (tabbed mission+terminal) in non-embedded mode
                ├── Qpr (mission tab)
                └── Jpr (terminal tabs)
```

### Data Flow

```
Daemon WebSocket
    ↓ JSON-RPC notifications
DaemonSessionController.handleNotification()
    ↓ ki.Msample itemION_* cases
MissionStore (cWs) per session
    ↓ useSyncExternalStore
uWs() hook
    ↓ props
Qpr / lWs / YYs components
```

## Key Findings

### 1. Session Sidebar (`i4s` / `ebo`)

**Container:** `U7n`: `data-testid="session-manager"`, `bg="sidebar-bg"`, `direction="column"`, right border `1px solid var(--border-1)`.

**Group Header (`A5s`):**
- Collapsible folder icon + chevron + group name
- `aria-expanded` for accessibility
- "New Session" button with tooltip
- `isDimmed` styling when group has no active sessions

**Session Row (`z5s`):**
- `React.memo` with 61-slot cache (`Ae.c(61)`)
- `data-session-id`, `data-sidebar-nav`, `role="button"`
- Keyboard accessible: Enter/Space triggers click
- Status indicator via `U5s`: dot, spinner, or text for "Waiting for your response", "Droid is thinking...", unread messages
- States: `isActive`, `isLegacy`, `isPinned`, `isSubsession`, `isWorking`
- Hover actions: Pin (`j5s`) and Archive (`B5s`)
- Time formatter: `Mdr` (timeago)

**Virtual List (`f5s`):**
- `estimateSize`, `getItemKey`, `overscan: 10`
- Animated enter/exit delays
- Keyboard nav `zn` on `[data-sidebar-nav]`: ArrowUp/ArrowDown

**Collapse Toggle (`H7n`):**
- Tooltip: "Expand sidebar" / "Collapse sidebar"
- Shortcut hint via `y6("B")`

### 2. Mission Store (`cWs`)

A per-session singleton store using `useSyncExternalStore` pattern:

```js
class cWs {
  state = UE.AwaitingInput
  features = []
  progressLog = []
  workers = new Map()        // workerSessionId → {startedAt, completedAt, exitCode}
  tokenUsageBySessionId = new Map()
  listeners = new Set()
}
```

**State enum (`UE`):**
| Enum | Value |
|------|-------|
| AwaitingInput | awaiting_input |
| Initializing | initializing |
| Running | running |
| Paused | paused |
| OrchestratorTurn | orchestrator_turn |
| Completed | completed |

**Key methods:**
- `setState(e)`, `setFeatures(e)`, `setProgressLog(e)`
- `addWorker(sessionId)`, `completeWorker(sessionId, exitCode)`
- `setTokenUsageBySessionId(map)`, `setSessionTokenUsage(sessionId, usage)`
- `getAggregatedTokenUsage()`: sums input/output/cache tokens across all workers
- `hasMission()`: true if any features/progressLog/workers exist or state ≠ AwaitingInput

**Hook:** `uWs(t)` returns `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)` bound to `hyt(t)` (singleton factory cached in `fNn` Map).

### 3. Mission Panel (`Qpr`)

**Props:**
- `features`, `progressLog`, `tokenUsage`, `workers`
- `onWorkerClick`, `onPause`, `onResume`, `onKillWorker`
- `isWorkerView`, `activeWorkerSessionId`, `isWorkerInProgress`, `state`

**Computed values:**
- Title: first `progressLog` entry with `type === "mission_accepted"`
- Start time: first `progressLog` entry with `type === "mission_accepted" || "mission_run_started"`
- `completedTasks`: features filtered by `status === BQ.Completed`
- `totalTasks`: `features.length`
- `totalMilestones`: unique `feature.milestone` names (default "Unassigned")
- `isWorkerKilled`: active worker has `completedAt != null`

**Sub-components:**
- `qQs` (header): title, Kill Worker / Pause / Resume buttons, elapsed time tooltip, token usage tooltip, SVG ring progress indicator
- `FQs` (feature list): groups by milestone via `zQs` reducer, tracks `expandedFeatureId`, scrollable
- `UQs` (milestone group): flag icon (color by status), maps to `LQs`
- `LQs` (feature row): index, ID, expand toggle, border style by status (`InProgress`→dashed, `Completed`→border-4, else→solid). Expanded view shows skillName, description, preconditions, acceptance criteria, verification steps, worker session links

### 4. Session Detail Layout (`YYs`)

**Props:** `sessionId`, `viewingWorkerSessionId`, `machineId`, `backendSession`, `missionState`, `codeServer`, `missionActions`, etc.

**Right pane mode enum (`_b`):**
| Mode | UI |
|------|-----|
| None | No right pane |
| VSCode | Embedded code-server iframe |
| DiffPanel | `ogr` diff viewer component |
| Mission | `Qpr` mission panel |

**Toggles:**
- `XYs(t)`: Mission toggle (`_b.Mission ↔ _b.None`)
- `ZYs(t)`: VSCode toggle (`_b.VSCode ↔ _b.None`)

**Computed flags:**
- `isMissionRunning = missionState?.state === UE.Running`
- `hasMission = missionState?.hasMission`
- `isMissionVisible = rightPaneMode === _b.Mission`
- `isOrchestratorSession = (hasMission || decompSessionType === HC.Orchestrator) && !isWorkerView`

**Layout:** `B9` (Allotment) split pane with `minSize: 350`. Left pane is `Hgr` (chat). Right pane conditionally renders VSCode, Mission (`Qpr`), or DiffPanel (`ogr`).

### 5. Session Detail Page (`cXs`)

**Route:** `/_authenticated/allotment/sessions/$sessionId`

**State management:**
- `viewingWorkerSessionId`: local React state, null when viewing orchestrator
- Fetches session via `oXs` (`GET /api/sessions/v2/{id}`)
- `missionState = uWs(t)`: live store subscription

**Actions:**
- `handleWorkerClick(sessionId)`: navigates to worker session via TanStack Router
- `handleKillWorker(workerSessionId)`: calls `daemonClient.killWorkerSession(t, workerSessionId)`
- `onBackToOrchestrator()`: clears `viewingWorkerSessionId`
- `onPause()`: calls `daemonClient.interruptSession(n)`

**PR merge readiness:** Integrated via `sXs()` hook + `Vgr` context. Background polling every 60s (`tXs`).

### 6. Data Flow from Daemon to UI

WebSocket notifications handled in `DaemonSessionController.handleNotification()`:

| Notification (`ki.*`) | MissionStore Action | React Event (`or.*`) |
|-----------------------|---------------------|----------------------|
| `Msample itemION_STATE_CHANGED` | `setState(mapped)` | `MissionStateChanged` |
| `Msample itemION_FEATURES_CHANGED` | `setFeatures(e.features)` | `MissionFeaturesChanged` |
| `Msample itemION_PROGRESS_ENTRY` | `setProgressLog(e.progressLog)` | `MissionProgressEntry` |
| `Msample itemION_HEARTBEAT` |: |: |
| `Msample itemION_WORKER_STARTED` | `addWorker(e.workerSessionId)` | `MissionWorkerStarted` |
| `Msample itemION_WORKER_COMPLETED` | `completeWorker(e.workerSessionId, e.exitCode)` | `MissionWorkerCompleted` |
| `SESSION_TOKEN_USAGE_CHANGED` | `setSessionTokenUsage(e.sessionId, e.tokenUsage)` | `SessionTokenUsageChanged` |

**State mapping (daemon → UI):**
```js
{
  awaiting_input: UE.AwaitingInput,
  initializing: UE.Initializing,
  running: UE.Running,
  paused: UE.Paused,
  orchestrator_turn: UE.OrchestratorTurn,
  completed: UE.Completed
}
```

**Hydration on session load:** `loadSession()` receives `n.mission` object and populates the store:
```js
u.setState(n.mission.state)
u.setFeatures(n.mission.features)
u.setProgressLog(n.mission.progressLog)
u.setTokenUsageBySessionId(n.mission.tokenUsageBySessionId ?? {})
// Workers hydrated from n.mission.workerSessionIds + n.mission.workerStates
```

## Code Examples

### MissionStore class snippet
```js
// renderer-main.js, char ~7.4M
class cWs {
  constructor(e) {
    this.state = UE.AwaitingInput
    this.features = []
    this.progressLog = []
    this.workers = new Map()
    this.tokenUsageBySessionId = new Map()
    this.listeners = new Set()
    this.sessionId = e
    this.snapshot = this.buildSnapshot()
  }
  // ...
  hasMission() {
    return this.features.length > 0
      || this.progressLog.length > 0
      || this.workers.size > 0
      || this.state !== UE.AwaitingInput
  }
}
```

### Daemon notification handler for mission state
```js
// renderer-main.js, char ~6.6M
case ki.Msample itemION_STATE_CHANGED: {
  const i = this.getMissionStoreForSession(n)
  if (i) {
    const l = {
      awaiting_input: UE.AwaitingInput,
      initializing: UE.Initializing,
      running: UE.Running,
      paused: UE.Paused,
      orchestrator_turn: UE.OrchestratorTurn,
      completed: UE.Completed
    }[e.state]
    l !== void 0 && i.setState(l)
  }
  this.emit(or.MissionStateChanged, {sessionId: n, state: e.state})
  break
}
```

### Qpr (Mission Panel) composition
```js
// renderer-main.js, char ~7.0M
function Qpr(t) {
  const {features, progressLog, tokenUsage, workers, onWorkerClick,
         onPause, onResume, isWorkerView, activeWorkerSessionId,
         onKillWorker, state, isWorkerInProgress} = t
  const title = $Qs(progressLog)      // extract mission title
  const startTime = KQs(progressLog)  // extract start timestamp
  const completedTasks = features.filter(XQs).length
  const totalTasks = features.length
  const totalMilestones = new Set(features.map(YQs)).size
  // ... renders qQs header + FQs feature list
}
```

### YYs right-pane mode switch
```js
// renderer-main.js, char ~7.6M
const [H, W] = B.useState(_b.None)  // rightPaneMode
const je = () => W(XYs)             // toggle Mission
const ke = () => { df(nl.WEB_VSCODE_OPEN_COUNT, F); W(ZYs) }  // toggle VSCode
const isMissionRunning = h?.state === UE.Running
const hasMission = h?.hasMission
const isMissionVisible = H === _b.Mission
// Conditionally renders Qpr:
tt = h?.hasMission ? m.jsx(Qpr, {...}) : null
```

## Theme / Style Tokens

| Token | Usage |
|-------|-------|
| `--sidebar-bg` | Sidebar background |
| `--border-1` | Borders, dividers |
| `--surface-1` / `--surface-2` / `--surface-3` | Card/pane backgrounds |
| `--text-heading` / `--text-subheading` / `--text-muted` | Text hierarchy |
| `--text-success` / `--text-error` / `--text-warning` | Status colors |
| `var(--border-3)` / `var(--border-4)` | Feature row active/completed borders |

## IPC Channels / WebSocket Protocol

**Direction:** Daemon → Renderer (WebSocket JSON-RPC notifications)

| Channel / Notification | Payload Shape | Purpose |
|------------------------|---------------|---------|
| `ki.Msample itemION_STATE_CHANGED` | `{state: string}` | Mission lifecycle state |
| `ki.Msample itemION_FEATURES_CHANGED` | `{features: Feature[]}` | Feature list update |
| `ki.Msample itemION_PROGRESS_ENTRY` | `{progressLog: ProgressEntry[]}` | Full progress log replacement |
| `ki.Msample itemION_HEARTBEAT` | `{}` | Keep-alive (no-op in UI) |
| `ki.Msample itemION_WORKER_STARTED` | `{workerSessionId: string}` | New worker spawned |
| `ki.Msample itemION_WORKER_COMPLETED` | `{workerSessionId: string, exitCode: number}` | Worker finished |
| `ki.SESSION_TOKEN_USAGE_CHANGED` | `{sessionId: string, tokenUsage: TokenUsage}` | Token consumption |

**Renderer → Daemon (via `daemonClient`):**
| Method | Purpose |
|--------|---------|
| `killWorkerSession(sessionId, workerSessionId)` | Terminate a worker |
| `interruptSession(sessionId)` | Pause mission execution |
| `addUserMessage(sessionId, {text})` | Send user input |

## Integration Points

- **Mission System (02-orchestration scope):** The daemon sends `Msample itemION_*` notifications. Mission planning, feature generation, and worker orchestration happen server-side: only the UI surface is documented here.
- **Daemon Client (04-desktop-gui main process scope):** `DaemonSessionController` bridges WebSocket JSON-RPC to React state. See `gui-main-process-daemon.md` for the WebSocket client implementation.
- **Session Manager (`i4s`):** Sidebar navigation documented in this report. Session list comes from TanStack Query (`GET /api/sessions/v2/{id}` and infinite query).
- **Diff Viewer (`ogr`):** Cross-referenced in `gui-renderer-diff-settings.md`. Mission panel shares the right pane with diff viewer and VSCode embed.
- **Terminal (`Jpr`/`Wpr`):** Mission panel co-exists with terminal tabs in `lWs` (non-embedded mode). Terminal state is managed separately by `DaemonSessionController`.

## Implementation Notes

1. **MissionStore (`cWs`):** Port as a Zustand or Valtio store. The `useSyncExternalStore` pattern in `uWs()` maps directly to `useStore()` in modern state libraries. Keep the singleton-per-session pattern (`hyt(t)` factory + `Map` cache).

2. **Feature List (`FQs`/`UQs`/`LQs`):** The grouping-by-milestone logic (`zQs` reducer) and expand/collapse state are straightforward to replicate. The `LQs` component's conditional rendering of preconditions/acceptance criteria/verification steps is a good reference for feature detail expansion.

3. **Right Pane Mode (`YYs`):** The `_b.None/VSCode/DiffPanel/Mission` enum and Allotment split pane can be ported using `react-resizable-panels` or similar. Toggle functions (`XYs`, `ZYs`) are simple state setters.

4. **Mission Header (`qQs`):** The SVG ring progress indicator (`WQs`) and tooltip content (`GQs`, `QQs`) are self-contained. Token formatting (`uNn`, `K8`) should be extracted as utilities.

5. **Sidebar (`i4s`):** Virtual scrolling via `@tanstack/react-virtual` (used as `f5s`). The keyboard navigation (`zn` on `[data-sidebar-nav]`) and collapsible group state (`Ue` Set) are important accessibility features to preserve.

6. **Daemon Notification Handlers:** The mapping from daemon string states to UI enums (see Data Flow section) must be kept in sync with the daemon's protocol. Any new states added server-side need entries in this mapping.

## Module Reference

| File | Bundle Region | Symbol / Function | Notes |
|------|--------------|-------------------|-------|
| renderer-main.js | Session management area | `A5s` | Collapsible group header |
| renderer-main.js | Session management area | `z5s` | Session row with status/actions |
| renderer-main.js | Session management area | `i4s` | Session manager sidebar |
| renderer-main.js | Session-manager bridge | `DaemonSessionController.handleNotification` | WebSocket → MissionStore bridge |
| renderer-main.js | Feature/Mission area | `LQs` | Feature row (expandable) |
| renderer-main.js | Feature/Mission area | `FQs` | Feature list grouped by milestone |
| renderer-main.js | Feature/Mission area | `qQs` | Mission header with stats/actions |
| renderer-main.js | Feature/Mission area | `Qpr` | Mission panel (composes qQs + FQs) |
| renderer-main.js | MissionStore area | `lWs` | Tabbed mission + terminal wrapper |
| renderer-main.js | MissionStore area | `cWs` | MissionStore class |
| renderer-main.js | MissionStore area | `uWs` | React hook for MissionStore |
| renderer-main.js | Session detail area | `YYs` | Session detail layout, right pane mode |
| renderer-main.js | Session detail area | `cXs` | Session detail page (route entry) |
