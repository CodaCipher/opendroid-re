# Observed Protocol Samples Analysis: OpenDroid Real Protocol Schemas

## Overview

Analysis of OpenDroid's actual observed protocol samples from `~/.opendroid/` (copied to `opendroid-re/protocol-samples/`). The data contains **1 mission** (Sample Application), **11 handoff files**, **8 session directories** with multiple session files, configuration files, plugin data, and 3 droid definitions. This provides exact, validated schemas for the mission protocol, session format, configuration system, and droid definitions.

### Directory Structure Found

```
~/.opendroid/
├── missions/                          # Mission data (1 mission found)
│   └── {uuid}/                        # Mission UUID directory
│       ├── features.json              # Feature definitions (the plan)
│       ├── validation-state.json      # Assertion tracking
│       ├── state.json                 # Mission state (paused/running/completed)
│       ├── mission.md                 # Mission description (Markdown)
│       ├── agent-guidelines.md                  # Agent guidance / coding conventions
│       ├── validation-contract.md     # Full validation contract (Markdown)
│       ├── model-settings.json        # Per-mission model config
│       ├── runtime-custom-models.json # Custom model definitions
│       ├── working_directory.txt      # Working directory for the mission
│       ├── progress_log.jsonl         # Timeline of all events (JSONL)
│       ├── worker-transcripts.jsonl   # Full worker transcripts (JSONL)
│       ├── handoffs/                  # Worker handoff JSON files
│       ├── contract-work/             # Contract surface descriptions
│       └── evidence/                  # Test evidence (screenshots, etc.)
├── sessions/                          # Session data (per working directory)
│   └── {cwd-slug}/                    # Encoded working directory path
│       ├── {session-uuid}.jsonl       # Session messages (JSONL)
│       └── {session-uuid}.settings.json  # Session settings
├── plugins/
│   ├── installed_plugins.json         # Installed plugin registry
│   ├── known_marketplaces.json        # Marketplace sources
│   ├── cache/                         # Cached plugin files
│   └── marketplaces/                  # Marketplace source repos
├── droids/                            # Custom droid (subagent) definitions
│   └── {name}.md                      # YAML frontmatter + Markdown prompt
├── config.json                        # Global config (custom models, etc.)
├── settings.json                      # User settings (plugins, models, etc.)
├── computer.json                      # Machine registration
├── feature-flags.json                 # Org feature flags & configs
├── background-processes.json          # Running background processes
├── changelog.json                     # CLI version/changelog tracking
├── history.json                       # Chat history (array of messages)
└── sessions-index.json                # Session index (all sessions metadata)
```

---

## Mission Protocol (EXACT SCHEMA)

### state.json

```json
{
  "missionId": "mis_0fe1958c",
  "state": "paused",
  "workingDirectory": "<workspace>\\SampleProject",
  "createdAt": "2026-04-20T05:09:49.607Z",
  "updatedAt": "2026-04-20T22:14:09.898Z",
  "lastReviewedHandoffCount": 11
}
```

Field documentation:
- `missionId` (string): Short mission ID with `mis_` prefix, NOT the UUID directory name. Derived server-side.
- `state` (string enum): `"paused"` observed. Possible values: `awaiting_input`, `initializing`, `running`, `paused`, `orchestrator_turn`, `completed` (from prior RE).
- `workingDirectory` (string): Absolute path to the project directory.
- `createdAt` (ISO 8601 string): Mission creation timestamp.
- `updatedAt` (ISO 8601 string): Last state update timestamp.
- `lastReviewedHandoffCount` (integer): Number of handoffs the orchestrator has processed. Used to track progress through the handoff directory.

### features.json

```json
{
  "features": [
    {
      "id": "feature-alpha",
      "description": "Implement item tracking: fetch external dataset from external API via backend proxy...",
      "skillName": "frontend-worker",
      "preconditions": ["interactive canvas renders", "backend proxy for /api/items returns external dataset data"],
      "expectedBehavior": ["180+ items visible on canvas...", "Click item shows cyan orbit track polyline..."],
      "verificationSteps": ["npm test: --grep 'item'", "Manual: Click sample item, verify orbit track + info panel", "curl http://localhost:3001/api/items returns 180+ records"],
      "fulfills": ["VAL-SAT-001", "VAL-SAT-002", "VAL-SAT-003", ...],
      "milestone": "data-layers",
      "status": "pending",
      "workerSessionIds": [],
      "currentWorkerSessionId": null,
      "completedWorkerSessionId": null
    }
  ]
}
```

Field documentation:
- `features` (array): Top-level wrapper. Each feature:
  - `id` (string): Unique feature identifier, kebab-case (e.g. `"feature-alpha"`, `"feature-beta-fix"`).
  - `description` (string): Detailed description of what to implement.
  - `skillName` (string): Which skill/droid to use for this feature. Observed values: `"frontend-worker"`, `"backend-worker"`, `"scrutiny-validator"`, `"user-testing-validator"`.
  - `preconditions` (string[]): List of preconditions that must be met before starting.
  - `expectedBehavior` (string[]): List of expected outcomes after completion.
  - `verificationSteps` (string[]): Commands or manual steps to verify. Can be empty `[]`.
  - `fulfills` (string[]): Array of validation assertion IDs (e.g. `"VAL-SAT-001"`). Maps to validation-state.json keys. Can be empty `[]`.
  - `milestone` (string): Milestone this feature belongs to (e.g. `"foundation"`, `"data-layers"`, `"shaders"`, `"hud"`, `"ui-panels"`, `"backend-llm"`, `"polish"`).
  - `status` (string enum): `"pending"`, `"in_progress"`, `"completed"`. Not observed: likely `"failed"`.
  - `workerSessionIds` (string[]): Array of ALL worker session IDs that have worked on this feature (multiple for retries/iterations).
  - `currentWorkerSessionId` (string | null): Currently active worker, or null.
  - `completedWorkerSessionId` (string | null): Not observed as non-null in this dataset. Possibly unused or set on final completion.

**Key observations:**
- The orchestrator can CREATE new features dynamically (e.g., `feature-beta-fix` was created after scrutiny found bugs)
- `scrutiny-validator-foundation` and `user-testing-validator-foundation` are special validator features with their own entries
- Validator features have `verificationSteps: []` but use their skill logic instead

### validation-state.json

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

Field documentation:
- `assertions` (object): Map of assertion ID → status object.
  - Key (string): Assertion ID following pattern `VAL-{CATEGORY}-{NUMBER}`. Categories: `GLOBE`, `SAT`, `AIR`, `SHADE`, `HUD`, `UI`, `LLM`, `WE` (external-data), `ANIM`, `SFX`, `LOAD`, `CROSS`.
  - `status` (string enum): Only `"pending"` observed in this dataset (no tests passed due to browser automation issues). Other possible values from handoff reports: `"pass"`, `"fail"`, `"blocked"`, `"skipped"`.

**Total assertions in this mission: 118** (matching the validation contract count)

### Handoff JSON Schema

Each handoff file is named: `{ISO-timestamp}__{featureId}__{workerSessionId}.json`

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

Field documentation (top-level):
- `timestamp` (ISO 8601 string): When the handoff was created.
- `workerSessionId` (UUID string): The worker's session ID.
- `featureId` (string): Feature this worker was assigned.
- `milestone` (string): Milestone the feature belongs to.
- `commitId` (string): Git commit hash (can be full SHA or short). The worker's final commit.
- `successState` (string enum): `"success"` or `"failure"`.
- `returnToOrchestrator` (boolean): Whether the orchestrator should take back control. `true` for validator features and when issues are found.

Field documentation (handoff object):
- `salientSummary` (string): One-paragraph summary of what happened.
- `whatWasImplemented` (string): Detailed description of implementation.
- `whatWasLeftUndone` (string): What wasn't finished. Empty string `""` if complete.
- `verification.commandsRun` (array): Each entry:
  - `command` (string): The command that was run.
  - `exitCode` (integer): Process exit code.
  - `observation` (string): What was observed.
- `verification.interactiveChecks` (array): Empty `[]` in all observed data. Reserved for manual checks.
- `tests.added` (string[]): List of test files added. Empty `[]` if none.
- `tests.updated` (string[]): (Optional, seen in scrutiny validator) List of test files updated.
- `tests.coverage` (string): Human-readable coverage description.
- `discoveredIssues` (array): Each entry:
  - `severity` (string enum): `"blocking"`, `"non_blocking"`, `"suggestion"`.
  - `description` (string): Issue description.
  - `suggestedFix` (string): (Optional) How to fix the issue.
- `skillFeedback` (object, optional): Only seen in scrutiny validators:
  - `followedProcedure` (boolean): Whether the skill was followed.
  - `deviations` (string[]): List of deviations.
  - `suggestedChanges` (string[]): Suggestions for improving the skill.

### progress_log.jsonl

Each line is a JSON object representing a timeline event:

```
{"timestamp":"2026-04-20T04:31:51.567Z","type":"mission_accepted","title":"Sample Mission"}
{"timestamp":"2026-04-20T05:09:49.613Z","type":"mission_run_started","message":"Starting sample mission execution..."}
{"timestamp":"2026-04-20T05:09:52.238Z","type":"worker_selected_feature","workerSessionId":"4a74ea1d-...","featureId":"project-scaffold"}
{"timestamp":"2026-04-20T05:09:52.254Z","type":"worker_started","workerSessionId":"4a74ea1d-...","spawnId":"worker_32eaaf3d","featureId":"project-scaffold"}
{"timestamp":"2026-04-20T05:22:02.013Z","type":"worker_completed","workerSessionId":"4a74ea1d-...","featureId":"project-scaffold","successState":"success","returnToOrchestrator":false,"commitId":"6692b644...","exitCode":0,"validatorsPassed":true,"handoff":{...full handoff object...}}
{"timestamp":"2026-04-20T14:52:33.197Z","type":"worker_paused","workerSessionId":"7328fc5c-...","featureId":"zustand-stores"}
{"timestamp":"2026-04-20T14:52:33.225Z","type":"mission_paused"}
{"timestamp":"2026-04-20T14:52:58.649Z","type":"mission_resumed","resumeWorkerSessionId":"7328fc5c-..."}
{"timestamp":"2026-04-20T14:58:46.438Z","type":"milestone_validation_triggered","milestone":"foundation","featureId":"scrutiny-validator-foundation"}
```

Event types observed:
- `mission_accepted`: Mission started by user. Fields: `title`.
- `mission_run_started`: Orchestrator begins execution. Fields: `message`.
- `mission_paused`: User paused the mission.
- `mission_resumed`: User resumed. Fields: `resumeWorkerSessionId`.
- `worker_selected_feature`: Orchestrator picked a feature. Fields: `workerSessionId`, `featureId`.
- `worker_started`: Worker session begins. Fields: `workerSessionId`, `spawnId`, `featureId`.
- `worker_paused`: Worker was paused (e.g., user interrupted). Fields: `workerSessionId`, `featureId`.
- `worker_completed`: Worker finished. Fields: `workerSessionId`, `featureId`, `successState`, `returnToOrchestrator`, `commitId`, `exitCode`, `validatorsPassed`, `handoff` (full embedded handoff object).
- `milestone_validation_triggered`: Milestone validation begins. Fields: `milestone`, `featureId`.

### worker-transcripts.jsonl

Each line is a JSON object with a complete transcript skeleton:

```json
{
  "workerSessionId": "<run-id>",
  "featureId": "project-scaffold",
  "milestone": "foundation",
  "skeleton": "## Tool: Skill\n{\"skill\":\"mission-worker-base\"}\n\n## Tool: Read\n{...}\n\n## Tool Result\n...\n\n## Tool: EndFeatureRun\n{...}",
  "timestamp": "2026-04-20T05:22:02.043Z"
}
```

Field documentation:
- `workerSessionId` (UUID string): Worker's session ID.
- `featureId` (string): Feature being worked on.
- `milestone` (string): Milestone name.
- `skeleton` (string): **Full text transcript** of all tool calls and results in a readable format. NOT JSON: it's a structured text format with `## Tool: ToolName` headers, JSON inputs, `## Tool Result` blocks, and `## Assistant` blocks.
- `timestamp` (ISO 8601 string): When the transcript was recorded.

**Critical finding:** The skeleton contains the COMPLETE tool call history including `EndFeatureRun` calls with the full handoff JSON. This is the raw audit trail.

### model-settings.json (per-mission)

```json
{
  "workerModel": "custom:MiniMax-M2.7-0",
  "workerReasoningEffort": "high",
  "validationWorkerModel": "custom:built-in model-[Z.AI]-0",
  "validationWorkerReasoningEffort": "none"
}
```

Field documentation:
- `workerModel` (string): Model ID for implementation workers. Uses custom model ID format `custom:{displayName}-{index}`.
- `workerReasoningEffort` (string): Reasoning effort for workers. Values: `"none"`, `"low"`, `"medium"`, `"high"`.
- `validationWorkerModel` (string): Model ID for validation workers.
- `validationWorkerReasoningEffort` (string): Reasoning effort for validation workers.

### runtime-custom-models.json (per-mission)

```json
{
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
  ]
}
```

Same schema as `settings.json` → `customModels` array. Snapshotted into the mission directory.

### working_directory.txt

```
<workspace>\SampleProject
```

Simple text file with the project's absolute path.

### agent-guidelines.md & mission.md

Both are Markdown files:
- `mission.md`: The full mission plan (description, milestones, environment setup, testing strategy, etc.)
- `agent-guidelines.md`: Agent guidance / coding conventions (boundaries, styling, testing notes, milestones)
- `validation-contract.md`: Full validation contract with all assertion IDs and their criteria

### contract-work/ directory

Contains Markdown "surface" descriptions for each feature area:
- `primary-surface.md`, `secondary-surface.md`, `tertiary-surface.md`, etc.
- These describe the testing surfaces for validation

### evidence/ directory

```
evidence/
└── foundation/
    ├── primary-navigation/
    │   └── initial-view.png
    └── user-testing/
        └── landing-view.png
```

Screenshots and artifacts from validation runs, organized by milestone and test group.

---

## Session Format

### Directory Structure

Sessions are stored in directories named by encoding the working directory path:
- `<workspace>\OpenDroid` → `-workspace-OpenDroid`
- `C:\Users\<user>` → `-C-Users-user`

Each directory contains pairs of files:
- `{session-uuid}.jsonl`: The session messages
- `{session-uuid}.settings.json`: Session configuration

### Session JSONL Format

Each line is a JSON object. Two event types observed:

**1. session_start:**
```json
{
  "type": "session_start",
  "id": "<session-id-1>",
  "title": "<session-title>",
  "sessionTitle": "New Session",
  "owner": "<user>",
  "version": 2,
  "cwd": "C:\\Users\\<user>"
}
```

**2. message:**
```json
{
  "type": "message",
  "id": "<session-id-2>",
  "timestamp": "2026-04-20T03:25:46.750Z",
  "message": {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "<system-reminder>...</system-reminder>"
      },
      {
        "type": "text",
        "text": "actual user message"
      }
    ]
  },
  "parentId": "<session-id-2>"
}
```

For assistant messages with tool use:
```json
{
  "type": "message",
  "id": "...",
  "timestamp": "...",
  "message": {
    "role": "assistant",
    "content": [
      {
        "type": "thinking",
        "signature": "<thinking-signature>",
        "signatureProvider": "anthropic",
        "thinking": "the thinking content..."
      },
      {
        "type": "tool_use",
        "id": "call_function_oplxzp4kozqm_1",
        "name": "FetchUrl",
        "input": {"url": "https://docs.opendroid.dev/llms.txt"}
      }
    ]
  },
  "parentId": "..."
}
```

For tool results:
```json
{
  "type": "message",
  "message": {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "call_function_oplxzp4kozqm_1",
        "is_error": false,
        "content": "URL Content from: ..."
      }
    ]
  },
  "parentId": "..."
}
```

**For OpenAI-compatible models, assistant messages include:**
```json
{
  "openaiMessageId": "<message-id>",
  "chatCompletionReasoningField": "reasoning_content",
  "chatCompletionReasoningContent": "the reasoning text..."
}
```

**For Anthropic models, thinking uses:**
```json
{
  "type": "thinking",
  "signature": "hex_signature",
  "signatureProvider": "anthropic",
  "thinking": "thinking content"
}
```

**Special message with visibility:**
```json
{
  "type": "message",
  "message": {
    "role": "user",
    "content": [{"type": "text", "text": "Error: Connection error."}],
    "visibility": "user_only"
  }
}
```

### Session Settings Format

```json
{
  "assistantActiveTimeMs": 242854,
  "model": "custom:built-in model-[Z.AI]-0",
  "reasoningEffort": "none",
  "interactionMode": "agi",
  "autonomyLevel": "off",
  "autonomyMode": "normal",
  "tags": [
    {
      "name": "mission-session",
      "metadata": {
        "role": "orchestrator",
        "missionId": "<mission-id>"
      }
    }
  ],
  "providerLock": "generic-chat-completion-api",
  "providerLockTimestamp": "2026-04-20T23:31:29.449Z",
  "tokenUsage": {
    "inputTokens": 86496,
    "outputTokens": 7416,
    "cacheCreationTokens": 0,
    "cacheReadTokens": 1595840,
    "thinkingTokens": 0
  }
}
```

Field documentation:
- `assistantActiveTimeMs` (integer): Total time the assistant was active in milliseconds.
- `model` (string): Model ID used. Format: `"custom:{displayName}-{index}"` for BYOK, or `"claude-opus-4-7"` for built-in.
- `reasoningEffort` (string enum): `"none"`, `"low"`, `"medium"`, `"high"`.
- `interactionMode` (string enum): `"auto"`, `"agi"`. `"auto"` = user approves actions. `"agi"` = autonomous.
- `autonomyLevel` (string): `"off"` observed.
- `autonomyMode` (string): `"normal"` observed.
- `tags` (array): Session tags. Each:
  - `name` (string): Tag name. Observed: `"mission-session"`, `"exec"`, `"subagent"`.
  - `metadata` (object, optional): Additional metadata. For mission sessions: `{"role": "orchestrator"|"worker", "missionId": "..."}`.
- `callingSessionId` (string, optional in sessions-index): Parent session that spawned this one (for subagents/workers).
- `callingToolUseId` (string, optional in sessions-index): The tool call that spawned this session.
- `providerLock` (string): The provider type locked for this session. Observed: `"generic-chat-completion-api"`, `"anthropic"`.
- `providerLockTimestamp` (ISO 8601 string): When the provider was locked.
- `tokenUsage` (object, optional): Token usage statistics:
  - `inputTokens` (integer)
  - `outputTokens` (integer)
  - `cacheCreationTokens` (integer)
  - `cacheReadTokens` (integer)
  - `thinkingTokens` (integer)

### Session Index (sessions-index.json)

```json
{
  "version": 1,
  "entries": [
    {
      "sessionId": "<session-id-3>",
      "mtime": <timestamp-ms-1>,
      "settingsMtime": <timestamp-ms-2>,
      "title": "LLM Competency Assessment for Concept Change Analysis",
      "cwd": "<workspace>\\SampleWorkspace\\SampleApp",
      "messagesCount": 730,
      "callingSessionId": "...(optional)",
      "callingToolUseId": "...(optional)",
      "tags": [
        {"name": "exec"},
        {"name": "subagent"}
      ]
    }
  ]
}
```

Field documentation:
- `version` (integer): Schema version. Currently `1`.
- `entries` (array): All sessions. Each:
  - `sessionId` (UUID string): Unique session identifier.
  - `mtime` (integer): Last modified time as Unix epoch milliseconds.
  - `settingsMtime` (integer): Settings file modification time.
  - `title` (string): Session title (first user message or auto-generated).
  - `cwd` (string): Working directory for the session.
  - `messagesCount` (integer): Total message count in the session.
  - `callingSessionId` (string, optional): Parent session that spawned this (for workers/subagents).
  - `callingToolUseId` (string, optional): Tool call ID that spawned this session.
  - `tags` (array, optional): Tags array (same format as settings.json tags).

**Total sessions in index: 52** (across all working directories)

### History (history.json)

```json
[
  {
    "command": "user message text...",
    "timestamp": "2026-04-15T02:00:14.278Z",
    "type": "message",
    "mode": "chat"
  }
]
```

A flat array of user messages across all sessions. Field documentation:
- `command` (string): The user's message text.
- `timestamp` (ISO 8601 string): When the message was sent.
- `type` (string): Observed: `"message"`.
- `mode` (string): Observed: `"chat"`.

**Total entries: 440**

---

## Configuration Schema

### settings.json

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

Field documentation:
- `enabledPlugins` (object): Map of plugin ID → boolean. Format: `"{scope}@{plugin-name}": boolean`.
- `logoAnimation` (string): Logo animation preference. Values: `"always"`, `"never"`, `"once"`.
- `customModels` (array): User-defined BYOK models. Each:
  - `model` (string): The model identifier sent to the API.
  - `id` (string): OpenDroid internal ID. Format: `"custom:{displayName}-{index}"`.
  - `index` (integer): Always `0` observed.
  - `baseUrl` (string): API endpoint URL.
  - `apiKey` (string): API key (SENSITIVE: stored in plaintext).
  - `displayName` (string): Human-readable name shown in UI.
  - `maxOutputTokens` (integer): Maximum output tokens. Observed: `131072`, `64000`.
  - `reasoningEffort` (string, optional): Default reasoning effort. `"high"` observed.
  - `noImageSupport` (boolean): Whether the model supports image input.
  - `provider` (string): Provider type. Observed: `"generic-chat-completion-api"`, `"anthropic"`.
- `overrideTerminalColors` (boolean): Whether to override terminal color scheme.
- `hasSeenMissionOnboarding` (boolean): Whether user has seen the mission onboarding flow.

### config.json

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

**NOTE:** This uses snake_case while settings.json uses camelCase. This appears to be a legacy/alternative config format. Field documentation:
- `custom_models` (array): Alternative custom model definitions.
  - `model_display_name` (string): Display name.
  - `model` (string): Model identifier.
  - `base_url` (string): API endpoint.
  - `api_key` (string): API key (SENSITIVE).
  - `provider` (string): Provider type.
  - `max_tokens` (integer): Max output tokens.
  - `reasoning_effort` (string): Reasoning effort level.

### computer.json

```json
{
  "computerId": "<computer-id>",
  "registeredAt": <timestamp-ms-3>
}
```

Field documentation:
- `computerId` (UUID string): Unique machine identifier.
- `registeredAt` (integer): Unix epoch milliseconds when this machine was registered with OpenDroid.

### feature-flags.json

```json
{
  "orgId": "cOOVMcwv4k5RvFpqF136",
  "flags": {
    "internal_app": false,
    "analytics_dashboards": false,
    "gpt_5_4_fast": true,
    "gpt_5_4_mini": true,
    "claude_opus_4_7": true,
    "glm_5_1": true,
    "minimax_m2_7": true,
    "plugins_command": true,
    "enable_structured_diffs": true,
    "droid_computers": true,
    "managed_computers": true,
    "automations": false,
    "wiki": false,
    "install_qa": true,
    "cli_reskin_v1": true,
    "sub_agents_v2": false,
    "spec_new_session_handoff": false,
    "...": "..."
  },
  "configs": {
    "opendroid_psa_banner": {
      "message": "",
      "severity": "info",
      "link": "",
      "linkText": ""
    },
    "cli_default_settings": {
      "enabledPlugins": {"core@opendroid-plugins": true},
      "extraKnownMarketplaces": {
        "opendroid-plugins": {
          "source": {"source": "github", "repo": "OpenDroid/opendroid-plugins"}
        }
      }
    },
    "provider_routing": {
      "version": 1,
      "defaults": {"anthropic": ["anthropic"], "openai": ["openai"]},
      "models": {
        "claude-sonnet-4-6": ["anthropic"],
        "claude-opus-4-6": ["anthropic"],
        "gpt-5-2025-08-07": ["openai"],
        "gpt-5-codex": ["openai"],
        "gpt-5.1": ["openai"],
        "gpt-5.1-codex": ["openai"]
      }
    },
    "workspace_rebuild_cutoff_utc_milliseconds": {"cutoffMilliseconds": <timestamp-ms-4>},
    "cloud_sessions_cutoff_utc_milliseconds": {"cutoffMilliseconds": <max-safe-placeholder>}
  }
}
```

Key flag values (50+ flags total):
- Model availability: `gpt_5_4_fast`, `gpt_5_4_mini`, `claude_opus_4_7`, `glm_5_1`, `minimax_m2_7`
- Features: `plugins_command`, `droid_computers`, `managed_computers`, `install_qa`, `sub_agents_v2`
- Internal: `internal_app`, `analytics_dashboards`, `automations`, `wiki`

### background-processes.json

```json
{
  "processes": []
}
```

Empty when no background processes are running.

### changelog.json

```json
{
  "cliVersion": "0.104.1",
  "entry": {
    "version": "v0.104.0",
    "date": "April 17",
    "features": ["Custom ripgrep path...", "BYOK config in bug reports..."]
  },
  "viewCount": 1
}
```

---

## Plugin/MCP Format

### installed_plugins.json

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

Field documentation:
- `schemaVersion` (integer): Currently `1`.
- `plugins` (object): Map of plugin ID → array of installations. Each installation:
  - `scope` (string): `"user"` observed.
  - `installPath` (string): Absolute path to cached plugin files.
  - `version` (string): Git commit hash (short).
  - `installedAt` (ISO 8601 string): Installation timestamp.
  - `lastUpdated` (ISO 8601 string): Last update timestamp.
  - `source` (string): Source marketplace name.

### known_marketplaces.json

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

Field documentation:
- Key (string): Marketplace name.
  - `source.source` (string): Source type. `"github"` observed.
  - `source.repo` (string): GitHub repo in `owner/repo` format.
  - `installLocation` (string): Local path where marketplace is cloned.
  - `lastUpdated` (ISO 8601 string): Last sync timestamp.
  - `autoUpdate` (boolean): Whether to auto-update.

---

## Droid Definitions

### Format

Droids are Markdown files with YAML frontmatter:

```markdown
---
name: scrutiny-feature-reviewer
description: >-
  Code review for a single feature during mission validation. Used only within missions.
model: inherit
---

# Scrutiny Feature Reviewer

You are a code reviewer spawned as a subagent...
```

### Frontmatter Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier. Lowercase, digits, `-`, `_`. |
| `description` | string | No | Description shown in UI. Keep ≤500 chars. |
| `model` | string | No | Model to use. `"inherit"` for parent session's model, or specific model ID. |
| `reasoningEffort` | string | No | For models that support it: `"low"`, `"medium"`, `"high"`. |
| `tools` | string/array | No | Tool access. Omit for all tools. Categories: `"read-only"`, `"edit"`, `"execute"`, `"web"`, `"mcp"`. Or array of tool IDs. |

### Observed Droids

**1. worker.md** (499 bytes):
```markdown
---
name: worker
description: >-
  General-purpose worker droid for delegating tasks. Use for non-trivial tasks
  that benefit from parallel execution, such as code exploration, Q&A, research,
  analysis.
model: inherit
---
# Worker Droid
Complete the requested task and report back concisely...
```

**2. scrutiny-feature-reviewer.md** (6173 bytes):
- Purpose: Code review for a single feature during mission validation
- Model: `inherit`
- No tools restriction (all tools available)
- Outputs a JSON review report to a specific file path
- Review structure: `featureId`, `reviewedAt`, `commitId`, `status` (pass/fail), `codeReview`, `sharedStateObservations`

**3. user-testing-flow-validator.md** (6875 bytes):
- Purpose: Test validation contract assertions through real user surface
- Model: `inherit`
- No tools restriction
- Outputs a JSON test report with assertion results, evidence, frictions, blockers
- Assertion statuses: `"pass"`, `"fail"`, `"blocked"`, `"skipped"`

### Mission Workers vs Custom Droids

Mission workers (like `frontend-worker`, `backend-worker`, `scrutiny-validator`, `user-testing-validator`) are **SKILLS**, not droids. They are defined in the project's `.opendroid/skills/` directory and invoked by the mission orchestrator. The droids in `~/.opendroid/droids/` are general-purpose subagents for use outside missions.

The key distinction:
- **Droids** → Custom subagents defined in `~/.opendroid/droids/*.md` or `.opendroid/droids/*.md`
- **Skills** → Mission-specific capabilities defined in `.opendroid/skills/{name}/SKILL.md`

---

## Key Findings

### Validated from RE
1. **Mission state machine** confirmed: `paused`, `running` states observed. State file structure matches prior RE.
2. **Handoff schema** fully validated: All fields match our architecture notes. The handoff is the core communication mechanism between workers and orchestrator.
3. **Feature lifecycle** confirmed: `pending` → `in_progress` → `completed`. Worker sessions tracked in `workerSessionIds` array.
4. **Two-phase validation** confirmed: Scrutiny (code review + lint/typecheck/test) followed by User Testing (browser automation).
5. **Orchestrator never writes code** confirmed: Only plans, delegates, creates fix features when validation finds issues.

### New Fields Not Previously Discovered
1. **`lastReviewedHandoffCount`** in state.json: tracks orchestrator progress through handoffs.
2. **`skillFeedback`** in handoffs: structured feedback about skill procedure compliance (only from scrutiny validators).
3. **`spawnId`** in progress_log.jsonl: internal worker spawn identifier (e.g., `"worker_32eaaf3d"`).
4. **`validatorsPassed`** in worker_completed events: boolean flag whether validators passed.
5. **`providerLock`** and **`providerLockTimestamp`** in session settings: provider locking mechanism.
6. **`tokenUsage`** in session settings: detailed token tracking with cache metrics.
7. **`visibility: "user_only"`** on messages: marks system-generated error messages.
8. **`chatCompletionReasoningField`** and **`chatCompletionReasoningContent`**: OpenAI-compatible reasoning fields.
9. **`runtime-custom-models.json`**: Per-mission model configuration snapshot.
10. **`contract-work/`** directory: Contains surface descriptions for each feature area's validation.
11. **`working_directory.txt`**: Simple text file with the project path.

### Discrepancies Between RE and Runtime
1. **`missionId` format**: Runtime shows `mis_` prefix with hex string (`mis_0fe1958c`), not UUID. The directory name is a UUID but the internal ID is different.
2. **Feature creation**: The orchestrator dynamically creates fix features (e.g., `feature-beta-fix`) during execution. These aren't in the original plan.
3. **Multiple worker sessions**: Features accumulate `workerSessionIds` across retries. The `currentWorkerSessionId` is separate from the completed list.
4. **config.json uses snake_case** while settings.json uses camelCase: two different config formats coexist.
5. **No compaction observed** in session data: sessions appear to store full message history without summarization.

---

## OpenDroid Implementation Notes

### Required Schemas (Use Directly)
1. **state.json**: Exact schema for mission state tracking
2. **features.json**: Feature definition format with all observed fields
3. **validation-state.json**: Simple assertion status map
4. **Handoff JSON**: Full schema with all nested objects
5. **progress_log.jsonl**: Event types and their field structures
6. **sessions-index.json**: Session metadata index format
7. **Session JSONL**: Message format with all content types

### Required vs Optional Fields
- **state.json**: All fields required except `lastReviewedHandoffCount`
- **features.json**: `verificationSteps` and `fulfills` can be empty arrays
- **Handoffs**: `tests.updated`, `skillFeedback`, `discoveredIssues[].suggestedFix` are optional
- **Session settings**: `tags`, `tokenUsage`, `providerLock*` are optional

### Default Values Observed
- `status`: `"pending"` for new features
- `workerSessionIds`: `[]` (empty array)
- `currentWorkerSessionId`: `null`
- `completedWorkerSessionId`: `null`
- `whatWasLeftUndone`: `""` (empty string, not null)
- `discoveredIssues`: `[]` (empty array)
- `verification.interactiveChecks`: `[]` (empty array)
- `tests.added`: `[]` (empty array)
- `interactionMode`: `"auto"`
- `autonomyLevel`: `"off"`
- `autonomyMode`: `"normal"`

### Architecture Implications
1. **Mission orchestrator** needs to read handoffs in timestamp order, track `lastReviewedHandoffCount`
2. **Worker spawning** creates a new session with `callingSessionId` pointing to the orchestrator
3. **Validation flow**: After all milestone features complete → scrutiny validator runs → if fail → create fix features → re-validate → user testing validator
4. **Evidence storage**: Organized as `evidence/{milestone}/{test-group}/` with descriptive filenames
5. **Plugin system**: GitHub-based marketplace, local cache, version tracked by git commit hash
