# tui-panel-model-selector: TUI Architecture Notes

## Overview

The ModelSelector is a React Ink panel that provides an interactive, keyboard-driven model picker for switching AI models within OpenDroid's TUI. It supports three selection targets: orchestrator, worker, and validation: each with independent model and reasoning-effort settings. The panel renders a categorized list of built-in models (Anthropic, OpenAI, Google, xAI, OpenDroid "Core") alongside user-defined custom models, with a collapsible "show all models" toggle and a "set as default" action. Selection flows through a two-stage pipeline: model selection → reasoning effort selection (for models supporting extended thinking). Model metadata is sourced from a central catalog (0257.js) with 38+ model definitions, and the active model state is managed by a session-level store (`BA()`) persisted to settings via `vA()`.

## Module Map
| Module | Size | Role | Vendor/App |
|-------|-------|-----|------------|
| 3980.js | ~7 KB | ModelSelector UI component (`Hi`): renders model list, handles selection, toggle-builtins, spec-config, set-as-default | App |
| 3982.js | ~3 KB | ModelSelector orchestrator wrapper (`QMA`): routes to model or reasoning picker based on target (orchestrator/worker/validation) | App |
| 3826.js | ~610 lines | Mission Control panel: hosts `mission_model_selector` view, provides `showModelSelector`/`showMissionModelPicker` callbacks | App |
| 3987.js | ~30 lines | Slash command handler (`/model`): dispatches to model selector, reasoning selector, or settings | App |
| 0257.js | ~650 lines | Model catalog: 38+ model definitions with metadata (name, provider, reasoning, cost, feature flags) | App |
| 0071.js | ~50 lines | Model ID constants enum (`JH`) + provider/tier/reasoning enums | App |
| 0251.js | ~35 lines | Default model settings: default model IDs, recommended models, reasoning defaults | App |
| 0250.js | ~30 lines | Orchestrator recommendation string (`mTH`) | App |
| 0934.js | ~50 lines | `F$()`: getTuiModelConfig helper, resolves model ID to display config | App |
| 4025.js | ~610 lines | Settings panel (`ZeD`): reads/writes `sessionDefaultSettings.model` and `reasoningEffort` | App |

## Architecture

### Component Hierarchy

```
4065.js (App coordinator)
  ├── showModelSelector(flag) → hM(flag), l1(true)  [session-level model selector]
  ├── showMissionModelPicker() → hM(false), k1(true) [mission-level picker]
  └── renders:
      ├── Hi (3980.js): Model list renderer [session context]
      └── QMA (3982.js): Orchestrator/worker/validation wrapper
          ├── Hi (3980.js): Model list
          └── Ai (reasoning picker): Reasoning effort selector

3826.js (Mission Control)
  └── view: "mission_model_selector" → QMA (3982.js)
      └── Hi (3980.js)
```

### Model Selection Flow

1. **Trigger**: User types `/model` command (3987.js) or presses keyboard shortcut
2. **Dispatch**: The command checks session type:
   - If orchestrator → `showMissionModelPicker()` (mission-level)
   - If worker → `showModelSelector(flag)` (session-level)
   - Falls back to reasoning effort selector or settings
3. **Render**: `Hi` component (3980.js) builds item list from:
   - `lh()` → available models list
   - `vA().getModelPolicy()` → admin policy for allowed/disabled
   - `vA().getCustomModels()` → user custom models
4. **Categorization**: Models split into:
   - **Recommended** (`JhH`): 7 top-tier models always shown first
   - **Other OpenDroid**: Remaining built-in models (collapsed by default, togglable)
   - **Custom Models**: User-added models, shown under separate header
5. **Selection**: `wD()` hook (focus manager) handles arrow-key navigation
6. **Two-stage selection**: If model supports multiple reasoning efforts → transitions to reasoning picker (`Ai`), otherwise auto-selects the single effort level
7. **State update**: `setModel(target, modelId)` + `setReasoningEffort(target, effort)` called on session store (`NBH()`)

### State Management

- **Session store** (`BA()`): `getModel()`, `setModel()`, `getReasoningEffort()`, `setReasoningEffort()`, `getDecompSessionType()`
- **Config store** (`vA()`): `getModelPolicy()`, `getCustomModels()`, `getSessionDefaultSettingManagementInfo()`, `getSettings()`
- **Mission Control** (`NBH()`): `orchestratorConfig`, `workerConfig`, `validatorConfig`: per-target model/reasoning state

## Key Components

### 1. `Hi`: ModelSelector Component (3980.js)

The main rendering component that displays the model list:

```js
// 3980.js, line 7-98
function Hi({
  currentModel: H,
  onSelect: A,
  onCancel: L,
  showSpecModeOption: $ = false,
  onSpecModeConfig: I,
  compatibleModelsOnly: D,
  isSelectingSpecModel: E = false,
  isOrchestratorModelSelector: f = false,
  title: M,
  onSetAsDefault: U,
  defaultDescription: P,
}) {
```

Item list construction with categories:

```js
// 3980.js, line 43-66: Item list builder
let x = dAH.useMemo(() => {
  let l = [];
  // Optional spec-config entry at top
  if ($ && I) (l.push({ type: "spec-config" }), l.push({ type: "sep" }));
  // OpenDroid models header
  l.push({ type: "header", label: "── OpenDroid Provided Models ──" });
  // Recommended models (JhH): always visible
  let LH = q.map((IH) => {
    let i = ds(IH, V);  // policy check
    return { type: "model", id: IH, disabled: !i.allowed };
  });
  // Other built-in models: collapsible
  let DH = N.map((IH) => {
    let i = ds(IH, V);
    return { type: "model", id: IH, disabled: !i.allowed };
  });
  // Custom models
  let VH = Y.map((IH) => {
    let i = ds(IH.id, V, IH);
    return { type: "model", id: IH.id, disabled: !i.allowed };
  });
  // ...
  // Toggle for showing more built-in models
  if (N.length > 0)
    l.push({ type: "toggle-builtins", expanded: B, hiddenCount: N.length });
  // ...
}, [w, Y, q, N, B, V, $, I, U]);
```

Focus/selection management via `wD()` hook:

```js
// 3980.js, line 70-89
let { selectedIndex: e } = wD({
  items: x,
  initialIndex: g >= 0 ? g : 0,
  isSelectable: (l) => {
    if (l.type === "sep" || l.type === "header") return false;
    if (l.type === "set-as-default") return true;
    if (l.type === "spec-config") return true;
    if (l.type === "toggle-builtins") return true;
    if (l.type === "model") return !l.disabled;
    return false;
  },
  onSelect: (l) => {
    if (l.type === "model" && !l.disabled) A(l.id);
    else if (l.type === "set-as-default" && U) U();
    else if (l.type === "spec-config" && I) I();
    else if (l.type === "toggle-builtins") W(!l.expanded);
  },
  onCancel: L,
});
```

Model row rendering with multiplier display:

```js
// 3980.js, line 189-224: Per-model rendering
let DH = l.id,       // model ID
    VH = LH === e,   // is focused/selected
    IH = DH === H,   // is current model
    i = l.disabled,  // disabled by admin policy
    HH = VH ? o.primary : i ? o.text.muted : void 0,  // color
    PH = F$(DH),     // get model config
    QH = PH.shortDisplayName;
// Appends reasoning effort label for current model
if (IH && PH.supportedReasoningEfforts.length > 1) {
  let JH = E ? BA().getSpecModeReasoningEffort() : BA().getReasoningEffort();
  QH = `${QH} (${YW(JH)})`;  // e.g. "Sonnet 4.6 (High)"
}
// Cost multiplier display: "(1.2x label)"
let EH = PH.modelId ? whH(PH.modelId) : void 0;   // tokenMultiplier
let KH = PH.modelId ? AjL(PH.modelId) : void 0;    // promoLabel
// Visual markers:
// "> " prefix for focused item
// " [current]" for current model
// " [disabled by admin]" for policy-disabled
```

### 2. `QMA`: Orchestrator/Target Wrapper (3982.js)

Routes between model list and reasoning picker based on target:

```js
// 3982.js, line 10-68
function QMA({ target: H, onDone: A, onCancel: L }) {
  let {
    loading: $,
    orchestratorConfig: I,
    workerConfig: D,
    validatorConfig: E,
    setModel: f,
    setReasoningEffort: M,
  } = NBH();  // Mission Control state hook

  // Resolve current model/reasoning for target
  let V = H === "orchestrator" ? (I?.model ?? "")
        : H === "validation"   ? (E?.model ?? "")
        :                        (D?.model ?? "");
  let X = H === "orchestrator" ? (I?.reasoningEffort ?? "high")
        : H === "validation"   ? (E?.reasoningEffort ?? "high")
        :                        (D?.reasoningEffort ?? "high");

  // Two-stage flow: model → reasoning (if multi-effort model)
  let w = useCallback(async (q, N) => {
    // If only one reasoning effort, skip picker
    let x = F$(q).supportedReasoningEfforts;
    if (x.length <= 1) {
      let v = x[0] ?? "none";
      Q(q, v);  // direct selection
      return;
    }
    // Otherwise, transition to reasoning picker
    (W(q), P("reasoning"));
  }, [Q]);
}
```

### 3. `/model` Command Handler (3987.js)

```js
// 3987.js, line 8-42
JdD = {
  name: "model",
  description: "Open model selector",
  execute: async (H, A) => {
    let { showModelSelector, showMissionModelPicker, showReasoningEffortSelector,
           showSettingsSelector, showSpecModeConfigurator } = A;
    let U = BA().getDecompSessionType() === "orchestrator";
    if (U && showMissionModelPicker) return (showMissionModelPicker(), { handled: true });
    if (showModelSelector) return (showModelSelector(!U), { handled: true });
    // Falls through to spec mode config, reasoning picker, or settings
  },
};
```

### 4. Model Catalog (0257.js): Excerpt

```js
// 0257.js: Recommended models list (JhH)
JhH = [
  "claude-opus-4-6",
  "claude-sonnet-4-6",
  "gpt-5.3-codex",
  "gemini-3.1-pro-preview",
  "glm-5",
  "kimi-k2.5",
  "minimax-m2.5",
];

// 0257.js: Example model entry
"claude-sonnet-4-6": {
  id: "claude-sonnet-4-6",
  name: "Claude Sonnet 4.6",
  shortName: "Sonnet 4.6",
  provider: "anthropic",
  apiProviders: ["anthropic", "bedrock_anthropic", "vertex_anthropic"],
  reasoningEffort: { supported: ["off", "low", "medium", "high", "max"], default: "high" },
  tier: "standard",
  cost: { tokenMultiplier: 1.2 },
  matchPatterns: [/sonnet[^a-z0-9]*4[.-]?6/i, /claude-sonnet-4-6/i],
}
```

### 5. Model ID Constants (0071.js)

```js
// 0071.js: Enum for model IDs
JH.CLAUDE_SONNET_4_6 = "claude-sonnet-4-6";
JH.CLAUDE_OPUS_4_6 = "claude-opus-4-6";
JH.GPT_5_1 = "gpt-5.1";
JH.GPT_5_2 = "gpt-5.2";
JH.GLM_5 = "glm-5";
// ... 38+ entries

// Reasoning effort enum
U.None = "none"; U.Off = "off"; U.Low = "low";
U.Medium = "medium"; U.High = "high";
U.ExtraHigh = "xhigh"; U.Max = "max"; U.Dynamic = "dynamic";

// Provider enum
f.ANTHROPIC = "anthropic"; f.OPENAI = "openai";
f.GOOGLE = "google"; f.OPENDROID = "opendroid"; f.XAI = "xai";
```

### 6. getTuiModelConfig Helper (0934.js)

```js
// 0934.js: Resolves any model ID to TUI display config
function F$(H) {
  if (BK(H)) return WK(H);       // Built-in → lookup catalog
  let A = vA().getCustomModels();
  let L = TT(H, A);              // Custom model match
  if (L) return {                 // Custom model config
    id: L.model, modelProvider: L.provider,
    displayName: L.displayName || L.model,
    supportedReasoningEfforts: I ? ["off", "low", "medium", "high"] : ["none"],
    isCustom: true,
  };
  if (H.startsWith("custom:"))    // Generic custom
    return { id: I, isCustom: true, noImageSupport: true, ... };
  return WK(gM);                  // Fallback to default
}
```

## Theme / Style Tokens

The ModelSelector uses theme tokens from the global `o` (theme) object:

| Token | Usage | Context |
|-------|-------|---------|
| `o.primary` | Focused/selected item color, current model highlight, model name display | Text foreground for active items |
| `o.text.muted` | Dimmed items, headers ("── OpenDroid Provided Models ──"), disabled items, US inference notice | Headers, footer |
| `o.text.secondary` | Unfocused items, "[current]" badge, "Multipliers represent cost..." | Secondary text |
| `o.warning` | Orchestrator recommendation warning text (`mTH`), shown when `isOrchestratorModelSelector=true` | Warning banner |
| `o.error` | Used in parent Mission Control for error states | Error display |
| `dimColor: true` | Applied to sub-descriptions (spec mode model name, "Will save:" text, "Droid Core model inference is US-based") | Sub-annotations |

Orchestrator-specific warning message (0250.js):
```
"GPT-5.2, GPT-5.3 Codex, Opus 4.6, or Opus 4.6 Fast with High or Extra High thinking are recommended for mission orchestration. Other models may not perform as well."
```

## Keyboard / Input Handling

The ModelSelector uses the `wD()` focus management hook for keyboard navigation:

1. **Arrow keys (↑/↓)**: Move selection through the item list. Items are filtered by `isSelectable`: headers and separators are skipped.
2. **Enter**: Triggers `onSelect` callback on the focused item:
   - `type: "model"` → calls `onSelect(modelId)` → starts model switch flow
   - `type: "toggle-builtins"` → toggles expanded/collapsed state for hidden built-in models
   - `type: "set-as-default"` → calls `onSetAsDefault()` → saves current model as session default
   - `type: "spec-config"` → opens Spec Mode model configurator
3. **Escape**: Triggers `onCancel` → closes the model selector panel. In Mission Control context, `handleEsc` navigates back through the view stack.
4. **Tab (in Mission Control)**: Cycles between main views: main → features → workers → mission_models.

The `/model` slash command (3987.js) serves as the primary trigger for the panel. The command's dispatch logic checks session type and available callbacks to determine which selector variant to show.

## Integration Points

- **tui-theme-system**: Consumes theme tokens (`o.primary`, `o.text.muted`, `o.text.secondary`, `o.warning`) for consistent styling across all rendered items.
- **tui-renderer-main-loop (4065.js)**: The App coordinator provides `showModelSelector`, `showMissionModelPicker`, `showSpecModeConfigurator`, and `showReasoningEffortSelector` callbacks that control panel visibility. State flags `l1` (model selector visible), `k1` (mission model picker visible), `l$` (reasoning selector visible), `jM` (spec mode config visible).
- **tui-panel-session-selector**: Model selection is context-aware: the selected model applies to the current session. `BA().getDecompSessionType()` determines whether to show session or mission-level picker.
- **Settings system (4025.js)**: Default model persisted via `settings.general.sessionDefaultSettings.model` (default: `"claude-sonnet-4-20250514"`). Default reasoning effort via `sessionDefaultSettings.reasoningEffort` (default: `"high"`).
- **Model policy system**: `vA().getModelPolicy()` returns admin-configured restrictions that can disable specific models (`[disabled by admin]` label). Custom model management via `vA().getCustomModels()`.
- **Reasoning effort selector**: After model selection, if the model supports multiple reasoning efforts, transitions to a separate `Ai` component for effort level selection (off/low/medium/high/max).
- **Mission system boundary**: Mission Control (3826.js) hosts a dedicated `mission_model_selector` view that uses the same `QMA` wrapper but with mission-level state (`orchestratorConfig`, `workerConfig`, `validatorConfig`).

## Implementation Notes

To implement a ModelSelector panel in OpenDroid:

1. **Model catalog**: Create a centralized model registry with id, name, shortName, provider, reasoningEffort.supported, cost.tokenMultiplier. The OpenDroid catalog has 38+ models across 5 providers (anthropic, openai, google, xai, opendroid).

2. **Recommended list**: Maintain a small "recommended" subset (7 models in OpenDroid) shown at top, with remaining models collapsible behind a "show all models" toggle.

3. **Focus management**: Use a reusable `wD()`-like hook that takes an item list + `isSelectable` predicate + `onSelect`/`onCancel` callbacks. This handles arrow key navigation and Enter/Escape.

4. **Two-stage selection**: After model selection, check if `supportedReasoningEfforts.length > 1`. If so, show a reasoning effort sub-picker before confirming. Models with only `["none"]` skip directly.

5. **Policy layer**: Implement `getModelPolicy()` to support admin-level model restrictions. Models disabled by policy show `[disabled by admin]` and are not selectable.

6. **Default persistence**: Store user's default model in settings under `sessionDefaultSettings.model`. The "set as default" action in the selector writes to this.

7. **Cost display**: Show `tokenMultiplier` as `(Xx label)` next to model name. Use `dimColor` for the multiplier text.

8. **Three target contexts**: Support orchestrator, worker, and validation model targets with independent model + reasoning effort state.

## Module Reference

| Module | Lines Read | Key Content |
|--------|-----------|-------------|
| 3980.js | Full (~225 lines) | `Hi` component: ModelSelector UI, item list builder, focus management, rendering |
| 3982.js | Full (~68 lines) | `QMA` component: target router, model/reasoning two-stage flow |
| 3826.js | Lines 1-250+ | Mission Control: `mission_model_selector` view, navigation stack, `showModelSelector` integration |
| 3987.js | Full (~30 lines) | `/model` command handler: dispatch logic |
| 0257.js | Full (~650 lines) | Model catalog: 38+ model definitions, `JhH` recommended list, `QhH` all models, helper functions |
| 0071.js | Full (~50 lines) | Model ID enum, provider enum, reasoning effort enum, tier enum |
| 0251.js | Full (~35 lines) | Default model IDs, reasoning defaults, recommended orchestrator models |
| 0250.js | Lines 26-29 | Orchestrator warning message `mTH` |
| 0934.js | Full (~50 lines) | `F$()`: getTuiModelConfig, resolves model ID to display config |
| 4025.js | Lines 1-80 | Settings panel: default model/reasoning read from settings |
