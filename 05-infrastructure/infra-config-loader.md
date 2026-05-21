# infra-config-loader: Architecture Notes

## Overview

OpenDroid's configuration loading system is a **hierarchical, multi-level settings manager** that resolves configuration from multiple sources in a defined precedence order. The core component is the `MD` class (module 0267), which extends a base settings provider and manages settings at five levels: **org**, **folder**, **project**, **user**, **dynamic**, and **builtin** defaults. Configuration is stored as JSON files (`settings.json`, `model-settings.json`, `config.json`, `mcp.json`) within `.opendroid/` directories, discovered by walking up from the current working directory to the user's home directory. The merge algorithm (`CjL` in module 0264) reverses the hierarchy array and applies level-specific merge strategies per settings domain (general, mcp, droids, skills, hooks, commands). CLI flags from Commander.js (module 0209) and environment variables (via dotenv in module 0005, OPENDROID_HOME_OVERRIDE, OPENDROID_ORG_MANAGED_SETTINGS_URL) can override file-based settings, establishing the full precedence chain: **env > CLI flags > dynamic config > user > project > folder > org > builtin defaults**.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 00-runtime.js | ~1KB | Runtime bootstrap; defines `sL = ".opendroid"` global constant | App |
| 0201.js | ~3KB | Home directory resolver (`C$()`); respects OPENDROID_HOME_OVERRIDE env var | App |
| 0209.js | ~27KB | Commander.js CLI framework; parses global flags and subcommands | Vendor (Commander.js) |
| 0257.js | ~25KB | Model registry (`_X`); defines all model IDs, providers, apiProviders, routing | App |
| 0258.js | ~10KB | Config file constants; defines settings.json, config.json, mcp.json, plugin dir names | App |
| 0260.js | ~4KB | Config path resolver (`by` class); discovers user/project/folder .opendroid paths | App |
| 0264.js | ~8KB | Settings merge functions; CjL (deep merge), KjL (persist merge), DVH (skills merge) | App |
| 0267.js | ~20KB | **Core SettingsManager** (`MD` class); hierarchical settings loading, caching, file watching | App |
| 0934.js | ~2KB | Model config resolver (`F$()`); resolves model ID to provider/config with custom model support | App |
| 0939.js | ~30KB | **SettingsService** (`p4$`); high-level settings API, model policy, session defaults | App |
| 0940.js | ~7KB | Settings initialization; creates singleton `TvH = new p4$()`, hooks executor | App |
| 1181.js | ~5KB | Config directory helpers; `kG()` returns .opendroid dir path, `ZgH()` discovers opendroid dirs | App |
| 2201.js | ~20KB | MissionFileService; manages model-settings.json per mission (orchestrator/worker/validation models) | App |
| 3134.js | ~35KB | Anthropic message preparation; model provider resolution (`F$`, `v7`), provider routing | App |
| 3708.js | ~30KB | JSON-RPC session controller; uses config for droid mode, model settings, session dispatch | App |
| 0005.js | ~7KB | dotenv config loader; loads .env and .env.local for environment-based configuration | Vendor (dotenv) |

## Architecture

### Configuration Precedence Chain

Settings are loaded in a **hierarchical cascade** with later levels overriding earlier ones:

```
1. Builtin defaults        (hardcoded in wjL() → pJ)
2. Org settings            (remote API or local file via OPENDROID_ORG_MANAGED_SETTINGS_LOCAL_PATH)
3. Folder settings         (.opendroid/ in ancestor directories, closest-first)
4. Project settings        (.opendroid/ in git project root)
5. Project plugin settings (plugin-specific settings at project level)
6. User settings           (~/.opendroid/settings.json)
7. User agent settings     (droid-specific settings at user level)
8. User plugin settings    (plugin-specific settings at user level)
9. Dynamic config          (runtime server-managed settings)
10. Environment variables  (OPENDROID_HOME_OVERRIDE, OPENDROID_ORG_MANAGED_SETTINGS_URL, OPENDROID_DISABLE_SETTINGS_PERSISTENCE)
11. CLI flags              (--provider, --model, etc. via Commander.js)
```

### Config Discovery and Loading

1. **Path Discovery** (`by` class, module 0260): Uses `DIH` resolver with `homeDirProvider: () => C$()` to discover `.opendroid/` directories. Walks from CWD upward to find project config, plus `~/.opendroid/` for user config.

2. **Home Directory** (`C$()`, module 0201): Returns `process.env.OPENDROID_HOME_OVERRIDE || os.homedir()`. The env var allows overriding the home directory for testing or custom installations.

3. **Config Directory** (`kG()`, module 1181): Returns `path.join(homeDir, ".opendroid")`. Also provides `ZgH()` which walks from a given directory upward to find the nearest `.opendroid/` directory and git root.

### Config File Structure

From module 0258, the config files within each `.opendroid/` directory are:

| File | Purpose |
|------|---------|
| `settings.json` | Main settings (general, mcp, droids, skills, hooks, commands) |
| `config.json` | Additional config (legacy/alternate) |
| `mcp.json` | MCP server definitions |
| `hooks.json` | Hook definitions |
| `plugins/` | Plugin directory |
| `droids/` | Custom droid definitions |
| `skills/` | Custom skill definitions |
| `commands/` | Custom command definitions |

### Settings Hierarchy Loading

The `loadAllLevelsWithAttribution()` method (module 0267, line ~295) loads all levels in parallel:

```js
[M, U, P, B, W, V, X, Q, w, C] = await Promise.all([
  this.loadOrgLevel(),                    // Org settings
  this.loadPluginSettings("org"),         // Org plugin settings
  this.loadUserFolder(),                  // User settings
  this.loadUserAgentFolder(),             // User agent/droid settings
  this.loadPluginSettings("user"),        // User plugin settings
  E,                                      // Folder settings (ancestor dirs)
  this.loadProjectFolder(),               // Project settings
  this.loadProjectAgentFolder(),          // Project agent settings
  this.loadPluginSettings("project"),     // Project plugin settings
  this.loadDynamicConfigLevel(),          // Dynamic (server-managed) config
]);
```

### Merge Algorithm

The `CjL` function (module 0264, line 181) performs deep merge by **reversing** the hierarchy array and applying domain-specific merge strategies:

```js
function CjL(H) {
  let A = [...H].reverse(), L = {};
  for (let $ of A)
    L = {
      general: GjL($.general, L.general),     // Deep merge with array union
      mcp: QjL($.mcp, L.mcp),                 // MCP server merge (project overrides user)
      droids: XjL($.droids, L.droids),         // Custom droid dedup by name
      skills: DVH($.skills, L.skills),         // Skills merge (enabled wins)
      hooks: FjL($.hooks, L.hooks),            // Hook merge (per-event-type)
      commands: TjL($.commands, L.commands),   // Command merge (by name dedup)
    };
  return L;
}
```

The `GjL` function (line 38) for `general` settings uses a deep merge strategy:
- **Primitives**: Higher-precedence level overrides lower
- **Arrays**: Union (append unique values from higher level)
- **Objects**: Recursive merge
- **`customModels`**: Merged by `id` field with deduplication

### Model Settings (Per-Mission)

`MissionFileService` (module 2201, line 574) manages per-mission model settings:

```
{missions}/{sessionId}/model-settings.json
```

Contains:
- `orchestratorModel`: Model for the orchestrator agent
- `workerModel`: Model for worker agents
- `validationWorkerModel`: Model for validation workers

### Model Resolution Chain

Model configuration follows this resolution path:
1. `F$(modelId)` (module 0934): Main resolver
2. Check custom models from settings (`vA().getCustomModels()`)
3. Check model registry `_X` (module 0257) for built-in models
4. Handle `custom:` prefixed models for generic chat completion APIs
5. Fall back to default model `gM`

## Key Findings

1. **Hierarchical settings with folder-level granularity**: Unlike typical dotfile systems, OpenDroid supports `.opendroid/settings.json` in any ancestor directory (not just project root and home). These are sorted by depth (closest to CWD = highest priority) and merged accordingly.

2. **Org settings from remote API**: Organization-level settings can be loaded from a remote URL (`OPENDROID_ORG_MANAGED_SETTINGS_URL`), a local file path (`OPENDROID_ORG_MANAGED_SETTINGS_LOCAL_PATH`), or fetched from the OpenDroid API server. This enables enterprise-level policy management.

3. **Domain-specific merge strategies**: Each settings domain has its own merge function: general settings use deep merge, MCP servers use project-overrides-user, skills use enabled-wins, hooks use per-event-type concatenation. This prevents accidental overwrites of complex nested configurations.

4. **File watching with cache invalidation**: The `VIH` class (module 0260) wraps folder sources with file watching capability. When files change, caches are invalidated and `settings-changed` events are emitted, enabling live config reload.

5. **Environment variable overrides**: Multiple env vars control config behavior: `OPENDROID_HOME_OVERRIDE` (home dir), `OPENDROID_ORG_MANAGED_SETTINGS_URL` (org settings URL), `OPENDROID_ORG_MANAGED_SETTINGS_LOCAL_PATH` (local org settings), `OPENDROID_DISABLE_SETTINGS_PERSISTENCE` (disable writes for testing), plus `.env`/`.env.local` file loading.

6. **Dynamic config level**: A server-managed "dynamic config" level sits between user and builtin defaults, allowing OpenDroid to push configuration changes without requiring user action.

## Code Examples

### Home Directory Resolution (0201.js, line 8)
```js
function C$() {
  return process.env.OPENDROID_HOME_OVERRIDE || iHE(); // iHE = os.homedir
}
```

### Settings Hierarchy Resolution (0264.js, line 181)
```js
function CjL(H) {
  let A = [...H].reverse(), L = {};
  for (let $ of A)
    L = {
      general: GjL($.general, L.general),
      mcp: QjL($.mcp, L.mcp),
      droids: XjL($.droids, L.droids),
      skills: DVH($.skills, L.skills),
      hooks: FjL($.hooks, L.hooks),
      commands: TjL($.commands, L.commands),
    };
  return L;
}
```

### Config File Discovery (0260.js, line 48)
```js
class by {
  resolver = new DIH({
    homeDirProvider: () => C$(),
    userDirName: sL,           // ".opendroid"
    projectDirName: ".opendroid",
    folderDirName: ".opendroid",
  });
}
```

### Model Settings Path (2201.js, line 574)
```js
get modelSettingsPath() {
  return P4.join(this.missionDir, "model-settings.json");
}
```

### Config File Names (0258.js, ErrTracker init section)
```js
var BIH = "settings.json",
    KhH = "config.json",
    eTH = "mcp.json",
    WIH = "droids",
    HVH = "skills",
    AVH = "commands",
    TIH = "hooks",
    ChH = "hooks.json";
```

## Integration Points (cross-system)

- **CLI Entry (05-infrastructure/infra-cli-entry)**: Commander.js (module 0209) parses `--model`, `--provider`, and other global flags that feed into the config system. CLI flags have the highest precedence in the config chain.
- **Session Manager (05-infrastructure/infra-session-manager)**: Session restore reads model settings from the session's stored configuration, which feeds into the config resolution chain.
- **Chat Session (05-infrastructure/infra-chat-session)**: Uses `F$()` to resolve model configuration for each conversation turn, applying the full config precedence chain.
- **Plugin Manager (05-infrastructure/infra-plugin-manager)**: Plugin settings are loaded at each config level (org, user, project) as a separate settings layer within the hierarchy.
- **IPC/Daemon (05-infrastructure/infra-ipc-daemon)**: JSON-RPC session controller (module 3708) uses config for droid mode (`Qf().setDroidMode()`), model settings, and session dispatch routing.
- **Mission System (02-orchestration)**: `MissionFileService` manages per-mission model-settings.json with orchestrator/worker/validation model overrides.
- **Tool System (03-tool-agent-system)**: Tools access settings through `vA()` singleton for permission policies, command allowlists/denylists, and hook configurations.

## Implementation Notes

1. **Replace the DIH resolver**: The path discovery system (`by` class, module 0260) uses a built-in resolver. Replace with a portable config directory convention (XDG on Linux, AppData on Windows, ~/Library on macOS).
2. **Preserve the merge algorithm**: The domain-specific merge functions in module 0264 are well-designed. Port them directly: they handle edge cases like array dedup, custom model id-based merging, and skill enabled-state resolution.
3. **Replace dotenv**: Module 0005 is a vendored dotenv. Use the npm `dotenv` package directly or switch to a more modern config loading approach.
4. **Separate model registry**: Module 0257 is a large hardcoded model registry. Extract it into a configurable data file or external API to avoid code changes when models are added/updated.
5. **Settings schema validation**: The system uses `Rn.safeParse()` (Zod) for org settings validation. Extend this pattern to all config levels for consistency.

## Module Reference

| Module | Lines | Key Functions/Classes |
|--------|-------|-----------------------|
| 00-runtime.js | 2 | `sL = ".opendroid"`: global opendroid directory name constant |
| 0201.js | 8 | `C$()`: home directory with OPENDROID_HOME_OVERRIDE support |
| 0209.js | 1-999 | `BPA` (Command class): CLI flag/subcommand parsing |
| 0257.js | 1-710 | `_X`: model registry with provider routing, apiProviders |
| 0258.js | 25-28 | Config file name constants: settings.json, config.json, mcp.json |
| 0260.js | 48-63 | `by` class: path discovery, `VIH` class: folder source with watching |
| 0264.js | 1-198 | `CjL`: deep merge, `KjL`: persist merge, `GjL`: general merge, `QjL`: MCP merge |
| 0267.js | 15-435 | `MD` class: SettingsManager singleton, `getResolvedSettings()`, `loadAllLevelsWithAttribution()` |
| 0934.js | 5-53 | `F$()`: model config resolver, `n3()`: default reasoning effort |
| 0939.js | 1-930 | `p4$`: SettingsService, model policy, session defaults, hooks management |
| 0940.js | 19 | `_mE = path.join(C$(), sL, "settings.json")`: user settings path |
| 1181.js | 40-79 | `kG()`: config dir resolver, `ZgH()`: opendroid dir discovery |
| 2201.js | 37-626 | `PMH`: MissionFileService, `modelSettingsPath`, `readModelSettings()`, `writeModelSettings()` |
| 3134.js | 1-1078 | `wWD()`: Anthropic message prep, `F$()` usage, provider routing |
| 3708.js | 1-929 | `VBL`: JSON-RPC session controller, `routeRequest()`, droid mode config |
| 0005.js | 1-212 | `GL0`: dotenv configDotenv, .env and .env.local loading |
