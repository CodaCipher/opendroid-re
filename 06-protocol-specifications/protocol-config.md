# Protocol: Configuration System
## Status: DEFINITIVE (cross-validated)

Cross-referenced observed protocol samples (`protocol-samples/runtime-findings.md`: Configuration Schema section) with architecture notes reports (`infra-config-loader.md`, `infra-plugin-manager.md`).

---

## 1. Overview

OpenDroid's configuration system uses a **hierarchical, multi-level settings manager** that resolves configuration from multiple sources in a defined precedence order. Configuration files are stored as JSON within `.opendroid/` directories at various levels (org, folder, project, user). The system supports two coexisting config formats with different naming conventions: `config.json` (snake_case) and `settings.json` (camelCase).

Additionally, the system manages feature flags (remote/org-controlled), plugin registry, marketplace sources, droid definitions, and per-mission model settings.

---

## 2. Configuration Precedence Chain

The architecture notes (`infra-config-loader.md`, module 0267) documents this precedence order, confirmed by observed protocol samples:

```
 1. Builtin defaults        (hardcoded in wjL() → pJ)
 2. Org settings            (remote API or local file)
 3. Folder settings         (.opendroid/ in ancestor directories, closest-first)
 4. Project settings        (.opendroid/ in git project root)
 5. Project plugin settings (plugin-specific at project level)
 6. User settings           (~/.opendroid/settings.json)
 7. User agent settings     (droid-specific at user level)
 8. User plugin settings    (plugin-specific at user level)
 9. Dynamic config          (runtime server-managed)
10. Environment variables   (OPENDROID_HOME_OVERRIDE, OPENDROID_ORG_MANAGED_SETTINGS_URL, etc.)
11. CLI flags               (--provider, --model, etc. via Commander.js)
```

Higher numbers override lower. CLI flags have ultimate precedence.

---

## 3. Config File Inventory

| File | Location | Format | Purpose |
|------|----------|--------|---------|
| `settings.json` | `~/.opendroid/settings.json` or `.opendroid/settings.json` | camelCase JSON | Primary user/project settings |
| `config.json` | `~/.opendroid/config.json` or `.opendroid/config.json` | snake_case JSON | Legacy/alternate custom model config |
| `mcp.json` | `.opendroid/mcp.json` | camelCase JSON | MCP server definitions |
| `hooks.json` | `.opendroid/hooks.json` | camelCase JSON | Hook definitions |
| `feature-flags.json` | `~/.opendroid/feature-flags.json` | camelCase JSON | Org feature flags & configs |
| `installed_plugins.json` | `~/.opendroid/plugins/installed_plugins.json` | camelCase JSON | Installed plugin registry |
| `known_marketplaces.json` | `~/.opendroid/plugins/known_marketplaces.json` | camelCase JSON | Marketplace sources |
| `model-settings.json` | `{mission-dir}/model-settings.json` | camelCase JSON | Per-mission model overrides |
| `runtime-custom-models.json` | `{mission-dir}/runtime-custom-models.json` | camelCase JSON | Per-mission model snapshot |
| `computer.json` | `~/.opendroid/computer.json` | camelCase JSON | Machine registration |
| `background-processes.json` | `~/.opendroid/background-processes.json` | camelCase JSON | Running background processes |
| `changelog.json` | `~/.opendroid/changelog.json` | camelCase JSON | CLI version/changelog tracking |

---

## 4. settings.json Schema (camelCase)

### 4.1 JSON Schema

```json
{
  "enabledPlugins": { "{scope}@{plugin-name}": true|false },
  "logoAnimation": "always"|"never"|"once",
  "customModels": [
    {
      "model": "string (model identifier for API)",
      "id": "string (custom:{displayName}-{index})",
      "index": 0,
      "baseUrl": "string (API endpoint URL)",
      "apiKey": "string (SENSITIVE)",
      "displayName": "string (human-readable name)",
      "maxOutputTokens": 131072,
      "reasoningEffort": "none"|"low"|"medium"|"high",
      "noImageSupport": true|false,
      "provider": "generic-chat-completion-api"|"anthropic"|"openai"
    }
  ],
  "overrideTerminalColors": true|false,
  "hasSeenMissionOnboarding": true|false
}
```

### 4.2 Field Documentation

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `enabledPlugins` | object | No | `{}` | Map of plugin ID → boolean. Format: `"{scope}@{plugin-name}": boolean` |
| `logoAnimation` | string | No |: | Logo animation preference. Values: `"always"`, `"never"`, `"once"` |
| `customModels` | array | No | `[]` | User-defined BYOK models. Merged by `id` field with dedup (RE: GjL function) |
| `customModels[].model` | string | Yes |: | Model identifier sent to the API (e.g., `"glm-5.1"`) |
| `customModels[].id` | string | Yes |: | OpenDroid internal ID. Format: `"custom:{displayName}-{index}"` |
| `customModels[].index` | integer | Yes | `0` | Always `0` observed |
| `customModels[].baseUrl` | string | Yes |: | API endpoint URL |
| `customModels[].apiKey` | string | Yes |: | API key (**SENSITIVE**: stored in plaintext) |
| `customModels[].displayName` | string | Yes |: | Human-readable name shown in UI |
| `customModels[].maxOutputTokens` | integer | No |: | Max output tokens. Observed: `131072`, `64000` |
| `customModels[].reasoningEffort` | string | No |: | Default reasoning effort: `"none"`, `"low"`, `"medium"`, `"high"` |
| `customModels[].noImageSupport` | boolean | No | `false` | Whether the model supports image input |
| `customModels[].provider` | string | Yes |: | Provider type. Observed: `"generic-chat-completion-api"`, `"anthropic"` |
| `overrideTerminalColors` | boolean | No | `false` | Override terminal color scheme |
| `hasSeenMissionOnboarding` | boolean | No | `false` | Whether user has seen mission onboarding flow |

### 4.3 Example (from observed protocol samples)

```json
{
  "enabledPlugins": {
    "core@opendroid-plugins": true
  },
  "logoAnimation": "always",
  "customModels": [
    {
      "model": "glm-5.1",
      "id": "custom:built-in model-[Z.AI]-0",
      "index": 0,
      "baseUrl": "https://api.z.ai/api/coding/paas/v4",
      "apiKey": "...",
      "displayName": "built-in model [Z.AI]",
      "maxOutputTokens": 131072,
      "noImageSupport": true,
      "provider": "generic-chat-completion-api"
    }
  ],
  "overrideTerminalColors": true,
  "hasSeenMissionOnboarding": true
}
```

---

## 5. config.json Schema (snake_case) ⚠️

### 5.1 JSON Schema

```json
{
  "custom_models": [
    {
      "model_display_name": "string",
      "model": "string",
      "base_url": "string",
      "api_key": "string (SENSITIVE)",
      "provider": "string",
      "max_tokens": 64000,
      "reasoning_effort": "none"|"low"|"medium"|"high"
    }
  ]
}
```

### 5.2 Field Documentation

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `custom_models` | array | No | Alternative custom model definitions (snake_case) |
| `custom_models[].model_display_name` | string | Yes | Display name (vs `displayName` in settings.json) |
| `custom_models[].model` | string | Yes | Model identifier |
| `custom_models[].base_url` | string | Yes | API endpoint (vs `baseUrl` in settings.json) |
| `custom_models[].api_key` | string | Yes | API key (SENSITIVE) |
| `custom_models[].provider` | string | Yes | Provider type |
| `custom_models[].max_tokens` | integer | No | Max output tokens (vs `maxOutputTokens` in settings.json) |
| `custom_models[].reasoning_effort` | string | No | Reasoning effort (vs `reasoningEffort` in settings.json) |

### 5.3 Example (from observed protocol samples)

```json
{
  "custom_models": [
    {
      "model_display_name": "MiniMax-M2.7",
      "model": "MiniMax-M2.7",
      "base_url": "https://api.minimax.io/anthropic",
      "api_key": "...",
      "provider": "anthropic",
      "max_tokens": 64000,
      "reasoning_effort": "high"
    }
  ]
}
```

### 5.4 ⚠️ Key Discrepancy: Two Config Formats Coexist

| Aspect | config.json | settings.json |
|--------|-------------|---------------|
| **Naming convention** | snake_case | camelCase |
| **Array field name** | `custom_models` | `customModels` |
| **Display name field** | `model_display_name` | `displayName` |
| **URL field** | `base_url` | `baseUrl` |
| **API key field** | `api_key` | `apiKey` |
| **Max tokens field** | `max_tokens` | `maxOutputTokens` |
| **Reasoning effort field** | `reasoning_effort` | `reasoningEffort` |
| **ID field** | ❌ Absent | ✅ `id` (format: `custom:{name}-{index}`) |
| **Index field** | ❌ Absent | ✅ `index` (always `0`) |
| **Image support field** | ❌ Absent | ✅ `noImageSupport` |
| **Additional fields** | None | `enabledPlugins`, `logoAnimation`, etc. |
| **Purpose** | Legacy/alternate custom model definitions only | Primary user settings (models + preferences) |

**Resolution:** config.json is a **legacy format** for custom model definitions only. settings.json is the **primary format** supporting all configuration domains. The RE merge algorithm (`GjL` in module 0264) merges `customModels` by `id` field with deduplication: this logic only applies to settings.json format. config.json's `custom_models` lacks the `id` field required for this merge strategy.

---

## 6. Merge Algorithm

The architecture notes (module 0264, `CjL` function) documents domain-specific merge strategies:

| Domain | Merge Function | Strategy |
|--------|---------------|----------|
| `general` | `GjL` | Deep merge with array union; customModels merged by `id` with dedup |
| `mcp` | `QjL` | MCP server merge (project overrides user) |
| `droids` | `XjL` | Custom droid dedup by name |
| `skills` | `DVH` | Skills merge (enabled wins) |
| `hooks` | `FjL` | Hook merge (per-event-type) |
| `commands` | `TjL` | Command merge (by name dedup) |

The merge algorithm reverses the hierarchy array and applies level-specific strategies per domain. For `general` settings (which includes customModels), primitives are overridden, arrays are unioned (append unique values), and objects are recursively merged.

**Runtime confirmation:** The observed protocol samples shows both `config.json` and `settings.json` with different model definitions, confirming that the system reads both and must reconcile them. The `runtime-custom-models.json` in the mission directory is a snapshotted copy of the resolved custom models at mission creation time.

---

## 7. Feature Flags (feature-flags.json)

### 7.1 JSON Schema

```json
{
  "orgId": "string (CloudDB-style org ID)",
  "flags": {
    "{flag_name}": true|false
  },
  "configs": {
    "{config_name}": { ... arbitrary config object ... }
  }
}
```

### 7.2 Field Documentation

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `orgId` | string | Yes | Organization identifier (e.g., `"cOOVMcwv4k5RvFpqF136"`) |
| `flags` | object | Yes | Map of flag name → boolean. 50+ flags observed |
| `configs` | object | Yes | Nested config blocks with arbitrary structure |

### 7.3 Flag Categories (50+ flags observed)

| Category | Example Flags | Purpose |
|----------|--------------|---------|
| **Model availability** | `gpt_5_4_fast`, `gpt_5_4_mini`, `claude_opus_4_7`, `glm_5_1`, `minimax_m2_7` | Control which models appear in UI |
| **Core features** | `plugins_command`, `droid_computers`, `managed_computers`, `install_qa`, `sub_agents_v2` | Feature toggles for major capabilities |
| **Internal** | `internal_app`, `analytics_dashboards`, `automations`, `wiki` | Internal/enterprise features |
| **UI/UX** | `cli_reskin_v1`, `enable_structured_diffs`, `spec_new_session_handoff` | UI and UX experiments |

### 7.4 Config Blocks

| Config Name | Structure | Purpose |
|-------------|-----------|---------|
| `opendroid_psa_banner` | `{ message, severity, link, linkText }` | PSA banner shown to users |
| `cli_default_settings` | `{ enabledPlugins, extraKnownMarketplaces }` | Default CLI settings for new users |
| `provider_routing` | `{ version, defaults, models }` | Provider routing map for model→provider resolution |
| `workspace_rebuild_cutoff_utc_milliseconds` | `{ cutoffMilliseconds }` | Timestamp for workspace rebuild eligibility |
| `cloud_sessions_cutoff_utc_milliseconds` | `{ cutoffMilliseconds }` | Timestamp for cloud sessions migration |

### 7.5 Example: provider_routing config

```json
{
  "version": 1,
  "defaults": {
    "anthropic": ["anthropic"],
    "openai": ["openai"]
  },
  "models": {
    "claude-sonnet-4-6": ["anthropic"],
    "claude-opus-4-6": ["anthropic"],
    "gpt-5-2025-08-07": ["openai"],
    "gpt-5-codex": ["openai"],
    "gpt-5.1": ["openai"],
    "gpt-5.1-codex": ["openai"]
  }
}
```

---

## 8. Plugin Registry (installed_plugins.json)

### 8.1 JSON Schema

```json
{
  "schemaVersion": 1,
  "plugins": {
    "{scope}@{plugin-name}": [
      {
        "scope": "user",
        "installPath": "string (absolute path)",
        "version": "string (git commit hash)",
        "installedAt": "string (ISO 8601)",
        "lastUpdated": "string (ISO 8601)",
        "source": "string (marketplace name)"
      }
    ]
  }
}
```

### 8.2 Field Documentation

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `schemaVersion` | integer | Yes | Currently `1` |
| `plugins` | object | Yes | Map of plugin ID → array of installations |
| `plugins[].scope` | string | Yes | Installation scope. `"user"` observed |
| `plugins[].installPath` | string | Yes | Absolute path to cached plugin files |
| `plugins[].version` | string | Yes | Git commit hash (short, e.g., `"<commit-hash>"`) |
| `plugins[].installedAt` | string (ISO 8601) | Yes | Installation timestamp |
| `plugins[].lastUpdated` | string (ISO 8601) | Yes | Last update timestamp |
| `plugins[].source` | string | Yes | Source marketplace name |

### 8.3 Example (from observed protocol samples)

```json
{
  "schemaVersion": 1,
  "plugins": {
    "core@opendroid-plugins": [
      {
        "scope": "user",
        "installPath": "C:\\Users\\<user>\\.opendroid\\plugins\\cache\\opendroid-plugins\\core\\<commit-hash>",
        "version": "<commit-hash>",
        "installedAt": "<iso-timestamp>",
        "lastUpdated": "<iso-timestamp>",
        "source": "opendroid-plugins"
      }
    ]
  }
}
```

---

## 9. Marketplace Sources (known_marketplaces.json)

### 9.1 JSON Schema

```json
{
  "{marketplace-name}": {
    "source": {
      "source": "github",
      "repo": "string (owner/repo format)"
    },
    "installLocation": "string (absolute path)",
    "lastUpdated": "string (ISO 8601)",
    "autoUpdate": true|false
  }
}
```

### 9.2 Field Documentation

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Key (marketplace name) | string | Yes | e.g., `"opendroid-plugins"` |
| `source.source` | string | Yes | Source type. `"github"` observed |
| `source.repo` | string | Yes | GitHub repo in `owner/repo` format |
| `installLocation` | string | Yes | Local path where marketplace is cloned |
| `lastUpdated` | string (ISO 8601) | Yes | Last sync timestamp |
| `autoUpdate` | boolean | Yes | Whether to auto-update |

### 9.3 Example (from observed protocol samples)

```json
{
  "opendroid-plugins": {
    "source": {
      "source": "github",
      "repo": "OpenDroid/opendroid-plugins"
    },
    "installLocation": "C:\\Users\\<user>\\.opendroid\\plugins\\marketplaces\\opendroid-plugins",
    "lastUpdated": "2026-04-20T23:27:07.958Z",
    "autoUpdate": true
  }
}
```

---

## 10. MCP Plugin System

The architecture notes (`infra-plugin-manager.md`) documents the MCP-based plugin system. Key findings cross-referenced with observed protocol samples:

### 10.1 Architecture Summary

- **MCP = Plugin System**: OpenDroid's plugin system is MCP (Model Context Protocol) based. `McpService` (1178.js) serves as the plugin manager.
- **Discovery**: File-system-based via `mcpSettingsManager.getEnabledMcpServers()`
- **Server types**: `"http"` (remote) and `"stdio"` (local process)
- **Lifecycle**: `start() → registerAllMcpTools() → [tool registration] → cleanup()`
- **Hot reload**: Config changes trigger automatic server reload via `settings-changed` event

### 10.2 Tool Registration Pattern

MCP tools are registered in the global tool registry (`k0()`) with this naming convention:

| Field | Format | Example |
|-------|--------|---------|
| `id` | `mcp_{serverName}_{toolName}` | `mcp_opendroid_plugins_myTool` |
| `llmId` | `{serverName}___{toolName}` | `opendroid_plugins___myTool` |
| `displayName` | `[MCP] {serverName}:{toolName}` | `[MCP] opendroid-plugins:myTool` |
| `toolkit` | `MCP:{serverName}` | `MCP:opendroid-plugins` |

### 10.3 Event System

| Event | Trigger | Payload |
|-------|---------|---------|
| `servers_reloading` | Before server reload | none |
| `servers_reloaded` | After reload completes | `{ startedServers, erroredServers, serverErrors }` |
| `server_started` | Single server started | `{ serverName, success, error? }` |
| `server_stopped` | Single server stopped | `{ serverName, success }` |
| `auth_required` | OAuth needed | Auth metadata |
| `auth_completed` | OAuth completed | `{ serverName, outcome, message }` |
| `error` | General error | Error object |

---

## 11. Per-Mission Model Settings

### 11.1 model-settings.json Schema

Located at `{mission-dir}/model-settings.json`:

```json
{
  "workerModel": "string (model ID)",
  "workerReasoningEffort": "none"|"low"|"medium"|"high",
  "validationWorkerModel": "string (model ID)",
  "validationWorkerReasoningEffort": "none"|"low"|"medium"|"high"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `workerModel` | string | Yes | Model ID for implementation workers. Format: `"custom:{displayName}-{index}"` for BYOK |
| `workerReasoningEffort` | string | Yes | Reasoning effort for workers |
| `validationWorkerModel` | string | Yes | Model ID for validation workers |
| `validationWorkerReasoningEffort` | string | Yes | Reasoning effort for validation workers |

### 11.2 Example (from observed protocol samples)

```json
{
  "workerModel": "custom:MiniMax-M2.7-0",
  "workerReasoningEffort": "high",
  "validationWorkerModel": "custom:built-in model-[Z.AI]-0",
  "validationWorkerReasoningEffort": "none"
}
```

### 11.3 runtime-custom-models.json Schema

Snapshotted copy of resolved custom models at mission creation time:

```json
{
  "customModels": [
    {
      "model": "string",
      "id": "string (custom:{displayName}-{index})",
      "index": 0,
      "baseUrl": "string",
      "apiKey": "string (SENSITIVE)",
      "displayName": "string",
      "maxOutputTokens": 131072,
      "noImageSupport": true|false,
      "provider": "string"
    }
  ]
}
```

This uses the **settings.json camelCase format** (not config.json snake_case), confirming that the resolved/active format is always camelCase.

---

## 12. Droid Definitions

### 12.1 Format

Droids are **Markdown files with YAML frontmatter** located at:
- User-level: `~/.opendroid/droids/{name}.md`
- Project-level: `.opendroid/droids/{name}.md`

### 12.2 YAML Frontmatter Schema

```yaml
---
name: string          # Required. Unique identifier (lowercase, digits, -, _)
description: string   # Optional. Description shown in UI (≤500 chars)
model: string         # Optional. "inherit" for parent model, or specific model ID
reasoningEffort: string  # Optional. "low"|"medium"|"high"
tools: string|array   # Optional. Tool access categories or specific tool IDs
---
```

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `name` | string | **Yes** |: | Unique identifier. Lowercase, digits, `-`, `_` |
| `description` | string | No |: | Description shown in UI. Keep ≤500 chars |
| `model` | string | No | `"inherit"` | Model to use. `"inherit"` = parent session's model |
| `reasoningEffort` | string | No |: | For models that support it: `"low"`, `"medium"`, `"high"` |
| `tools` | string or array | No | All tools | Tool access. Categories: `"read-only"`, `"edit"`, `"execute"`, `"web"`, `"mcp"`. Or array of specific tool IDs |

### 12.3 Body Format

After the YAML frontmatter, the rest of the Markdown file is the **system prompt** for the droid/subagent. This is a natural language instruction set that defines the droid's behavior, output format, and constraints.

### 12.4 Observed Droids (from observed protocol samples)

**1. worker.md** (~499 bytes)
```yaml
---
name: worker
description: >-
  General-purpose worker droid for delegating tasks. Use for non-trivial tasks
  that benefit from parallel execution, such as code exploration, Q&A, research,
  analysis.
model: inherit
---
```
Purpose: General-purpose subagent for parallel task execution.

**2. scrutiny-feature-reviewer.md** (~6173 bytes)
- Purpose: Code review for a single feature during mission validation
- Model: `inherit`
- No tools restriction
- Outputs structured JSON review report

**3. user-testing-flow-validator.md** (~6875 bytes)
- Purpose: Test validation contract assertions through user surfaces
- Model: `inherit`
- No tools restriction
- Outputs JSON test report with assertion results

---

## 13. Droid vs Skill Distinction ⚠️

This is a critical architectural distinction confirmed by both observed protocol samples and architecture notes.

### 13.1 Definition

| Aspect | Droids | Skills |
|--------|--------|--------|
| **Location** | `~/.opendroid/droids/*.md` (user) or `.opendroid/droids/*.md` (project) | `.opendroid/skills/{name}/SKILL.md` (project only) |
| **Nature** | **Subagents**: autonomous workers spawned via the Task tool | **Mission capabilities**: procedure definitions invoked by workers |
| **Scope** | General-purpose, reusable across sessions and missions | Mission-specific, tied to a feature's `skillName` field |
| **Invocation** | `Task` tool with `subagent_type: "{droid-name}"` | `Skill` tool with `skill: "{skill-name}"` |
| **Examples** | `worker`, `scrutiny-feature-reviewer`, `user-testing-flow-validator` | `mission-worker-base`, `protocol-analyzer`, `frontend-worker`, `backend-worker` |
| **Model** | Configurable per-droid (`model` frontmatter field) | Inherits from mission's model-settings.json |
| **Tool access** | Configurable per-droid (`tools` frontmatter field) | Full access (no restriction mechanism in skills) |
| **Output** | Returns single final report to parent | Executes in-place, modifies project files directly |
| **Statefulness** | Stateless: each invocation is independent | Stateful: runs within the worker's session context |
| **Config merging** | Loaded via `droids` domain merge (`XjL` function) | Loaded via `skills` domain merge (`DVH` function) |

### 13.2 Relationship to Feature `skillName`

When a feature in `features.json` has `skillName: "frontend-worker"`, this refers to a **skill** defined in `.opendroid/skills/frontend-worker/SKILL.md`, NOT a droid. The skill defines the work procedure that the worker agent follows.

However, workers can also **spawn droids as subagents** during execution using the `Task` tool. For example, a worker executing the `scrutiny-validator` skill might spawn the `scrutiny-feature-reviewer` droid as a subagent to perform code review.

### 13.3 Naming Convention Overlap ⚠️

⚠️ **Discrepancy:** Some droids are named similarly to skills, causing potential confusion:
- Droid: `scrutiny-feature-reviewer` (in `~/.opendroid/droids/`): performs code review as subagent
- Skill: `scrutiny-validator` (in `.opendroid/skills/`): the skill that orchestrates validation including spawning the reviewer droid
- Droid: `user-testing-flow-validator` (in `~/.opendroid/droids/`): performs test validation as subagent
- Skill: `user-testing-validator` (in `.opendroid/skills/`): the skill that orchestrates user testing

The naming overlap is intentional: skills orchestrate, droids execute as subagents.

---

## 14. Config Discovery and Loading

### 14.1 Path Discovery

The architecture notes documents the `by` class (module 0260) for config discovery:

- Uses a resolver that walks from CWD upward to find `.opendroid/` directories
- Respects `OPENDROID_HOME_OVERRIDE` environment variable for custom home directory
- Supports org-level, folder-level, project-level, and user-level configs

### 14.2 Directory Convention

```
{any-dir}/.opendroid/          # Folder-level config (walked from CWD upward)
{git-root}/.opendroid/         # Project-level config
~/.opendroid/                  # User-level config
```

Each `.opendroid/` directory can contain:
```
.opendroid/
├── settings.json            # Primary settings
├── config.json              # Legacy/alternate config
├── mcp.json                 # MCP server definitions
├── hooks.json               # Hook definitions
├── plugins/                 # Plugin directory
├── droids/                  # Custom droid definitions
├── skills/                  # Custom skill definitions
└── commands/                # Custom command definitions
```

### 14.3 Environment Variable Overrides

| Variable | Purpose |
|----------|---------|
| `OPENDROID_HOME_OVERRIDE` | Override home directory (replaces `os.homedir()`) |
| `OPENDROID_ORG_MANAGED_SETTINGS_URL` | URL for org-level settings (remote) |
| `OPENDROID_ORG_MANAGED_SETTINGS_LOCAL_PATH` | Local file path for org settings |
| `OPENDROID_DISABLE_SETTINGS_PERSISTENCE` | Disable settings writes (for testing) |

---

## 15. Other Config Files

### 15.1 computer.json

```json
{
  "computerId": "UUID string",
  "registeredAt": <timestamp-ms-3>
}
```

| Field | Type | Notes |
|-------|------|-------|
| `computerId` | string (UUID) | Unique machine identifier |
| `registeredAt` | integer | Unix epoch milliseconds |

### 15.2 background-processes.json

```json
{
  "processes": []
}
```

Empty when no background processes are running.

### 15.3 changelog.json

```json
{
  "cliVersion": "0.104.1",
  "entry": {
    "version": "v0.104.0",
    "date": "April 17",
    "features": ["..."]
  },
  "viewCount": 1
}
```

---

## 16. Discrepancies (Inferred vs Observed)

| # | Field/Behavior | Architecture Notes Says | Observed Samples Show | Resolution |
|---|---------------|------------------|--------------------|------------|
| 1 | **Config format naming** | RE documents merge algorithm for `customModels` (camelCase) in settings.json domain | Runtime shows BOTH `config.json` (snake_case `custom_models`) and `settings.json` (camelCase `customModels`) coexisting | config.json is legacy format; settings.json is primary. The merge algorithm only processes camelCase format |
| 2 | **config.json field names** | RE doesn't explicitly document config.json's snake_case field names | Runtime shows: `model_display_name`, `base_url`, `api_key`, `max_tokens`, `reasoning_effort` | Documented as legacy format; OpenDroid should support both or normalize to camelCase |
| 3 | **Missing fields in config.json** | RE focuses on settings.json schema | config.json lacks `id`, `index`, `noImageSupport` fields present in settings.json | config.json is a simpler legacy format; settings.json is the complete schema |
| 4 | **Feature flags** | RE doesn't cover feature-flags.json structure | Runtime shows complete schema: `orgId`, 50+ `flags`, `configs` with nested blocks | Feature flags are org-level remote config, not covered by local config loader RE |
| 5 | **Plugin file formats** | RE covers MCP service lifecycle but not installed_plugins.json/known_marketplaces.json schemas | Runtime shows complete schemas for both files | Plugin file formats are managed by the plugin system, not the config loader |
| 6 | **Droid definition format** | RE mentions `droids` domain in merge algorithm (`XjL` function) | Runtime shows complete YAML frontmatter + Markdown format with 5 fields | RE was aware of droids as a config domain but didn't document the file format |
| 7 | **Droid vs Skill naming overlap** | RE doesn't address naming convention overlap | Runtime shows `scrutiny-feature-reviewer` (droid) vs `scrutiny-validator` (skill) | Skills orchestrate; droids execute as subagents. Naming overlap is intentional |
| 8 | **Settings domains** | RE documents 6 merge domains: general, mcp, droids, skills, hooks, commands | Runtime confirms settings.json uses `general` domain (customModels, enabledPlugins, etc.) | RE's domain analysis is accurate and complete |

---

## 17. Open Questions

1. **config.json origin**: Which part of the codebase writes/reads config.json? The architecture notes focuses on settings.json merge. Is config.json loaded separately and converted to camelCase?
2. **Feature flag source**: Feature-flags.json appears to be downloaded from the OpenDroid server (org-managed). The exact API endpoint and refresh mechanism is not documented.
3. **Plugin version tracking**: Plugin versions use git commit hashes (short). How does the system detect and fetch updates?
4. **Per-mission model snapshot timing**: When exactly is `runtime-custom-models.json` created during the mission lifecycle? Is it at mission creation or first worker spawn?
5. **Droid model resolution**: When a droid specifies `model: "inherit"`, does it inherit from the spawning session's current model or from the mission's model-settings.json?

---

## 18. OpenDroid Implementation Notes

1. **Unify to single format**: OpenDroid should use ONE config format (recommend camelCase/JSON) and normalize any legacy snake_case on load.
2. **Preserve merge algorithm**: The domain-specific merge functions (module 0264) are well-designed. Port them directly.
3. **Feature flags**: Implement as a remote config system with local cache. The `flags` object controls feature visibility; `configs` provides structured configuration.
4. **Plugin system**: Use MCP protocol for plugin architecture. Register tools in a global registry with namespaced IDs (`mcp_{server}_{tool}`).
5. **Droid definitions**: Support YAML frontmatter + Markdown format. Allow both user-level and project-level droids.
6. **Droid vs Skill separation**: Maintain the clear architectural boundary: droids are stateless subagents, skills are stateful worker procedures.
7. **Environment variable overrides**: Support `OPENDROID_HOME` (equivalent to `OPENDROID_HOME_OVERRIDE`) for custom installations.
