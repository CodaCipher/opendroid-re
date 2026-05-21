# mission-prompts-and-templates: Mission System RE

## Overview

OpenDroid's prompt template system is a multi-layered architecture that assembles system prompts for three distinct agent roles: the base Droid identity, the orchestrator (mission planner/delegator), and workers (feature implementers). Prompts are composed from static template strings, SKILL.md files, agent-guidelines.md context, mission-specific JSON data, and dynamic `<system-reminder>` injections. The system uses a provider-aware message builder (`QrI` in 2857.js) that concatenates base identity + role-specific prompt + provider-specific additions into Anthropic/OpenAI-compatible message arrays.

## Module Map

| Module | Size | Role | Notes |
|--------|------|------|-------|
| 2175.js | 135.8 KB | Prompt template library | Contains ALL orchestrator and worker skill prompts as embedded template strings (`s6I`, `mAf`, `cAf`, `pAf`, `lAf`, `iAf`, `nAf`, `uAf`, `gAf`) |
| 2541.js | 4.3 KB | Worker bootstrap prompt assembler | `GNI()` assembles the worker's initial system message; `VNI()`/`XNI()` inject mission context |
| 2526.js | 5.6 KB | Skill injection engine | `WKH` class loads skill droids and injects their `systemPrompt` into the conversation |
| 2517.js | 5.6 KB | SKILL.md parser | `km` class parses YAML frontmatter + markdown body from SKILL.md files |
| 2519.js | 5.8 KB | Droid registry | `d2` class manages droid CRUD with `systemPrompt` field |
| 2857.js | 7.3 KB | System message builder | `QrI()` assembles final message array from base prompt + override + provider additions |
| 2522.js | 5.8 KB | Subagent prompt assembler | `cs.buildCommandArgs()` constructs Task tool subagent prompts |
| 2312.js | 8.0 KB | Default system prompt | `hMH`: the default interactive CLI system prompt |
| 0060.js | 96 B | Tag constants | `<system-reminder>`, `</system-reminder>`, `<system-notification>` constants |
| 3399.js | 28.6 KB | Mission context injector | Injects orchestrator reminders and mission directory context into user messages |
| 1181.js | 4.2 KB | agent-guidelines.md discovery | `eO$()` searches for agent-guidelines.md variants |
| 4065.js | 4.6 KB | Prompt display filter | `JA0()` strips `<system-reminder>` tags from UI display |
| 00-runtime.js |: | Base identity | `h7` = "You are Droid, an AI software engineering agent built by OpenDroid." |
| 2536.js |: | Droid generator | `loA.buildPrompt()` generates droid configs via LLM |

## Architecture / Flow

### Prompt Layer Stack

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Dynamic Context Injection                         │
│  - <system-reminder> tags (mission context, agent-guidelines.md)      │
│  - System notifications (spec mode, approvals)              │
│  - Injected per-message by 3399.js / 2541.js                │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Role-Specific Prompt                              │
│  - Orchestrator: s6I (2175.js)                              │
│  - Worker base: mAf (2175.js)                               │
│  - Worker bootstrap: GNI() (2541.js)                        │
│  - Skills: SKILL.md content (2526.js)                       │
│  - Subagent: cs.buildCommandArgs() (2522.js)                │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Base System Prompt                                │
│  - hMH (2312.js): default interactive CLI behavior         │
│  - h7 + systemPromptOverride (2857.js)                      │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Identity Anchor                                   │
│  - h7 (00-runtime.js): "You are Droid..."                   │
└─────────────────────────────────────────────────────────────┘
```

### System Message Assembly Flow

1. **Base identity** (`h7` from `00-runtime.js`) is always first: `"You are Droid, an AI software engineering agent built by OpenDroid."`
2. **Role prompt** is selected based on session type:
   - Normal mode: `hMH` (default CLI behavior from 2312.js)
   - Orchestrator mode: `s6I` (orchestrator prompt from 2175.js)
   - Worker mode: `GNI()` output (worker bootstrap from 2541.js) + skill injection
3. **Provider-specific additions** are appended by `QrI()` in 2857.js:
   - OpenAI: markdown spec, CLI preference spec, solution persistence
   - Google: spec mode guidelines, tool usage rules, todo tool guidelines
4. **Dynamic context** is injected as `<system-reminder>` blocks into the user message content by 3399.js
5. **Skills** are injected via the Skill tool (2526.js) which wraps the skill's `systemPrompt` in `<skill>` XML tags
6. **agent-guidelines.md** content is discovered by 1181.js and provided as context files in the worker bootstrap

## Key Findings

### 1. Prompt Template Structure and Variable Interpolation

Prompts are static template strings with embedded JavaScript variable interpolation. The primary patterns are:

**Template string with interpolation (2541.js:69-108):**
```js
// Module 2541.js, line 69
function GNI(H, A, L) {
  let $ = HMH.includes(A.skillName),
    I = L ? L.replace(/[^a-zA-Z0-9]/g, "") : "",
    D = I ? I.slice(0, 12) : "<workerSessionId>",
    E = $
      ? `## Your Task\n\n1. Invoke the '${A.skillName}' skill...`
      : `## Your Task\n\n1. First, invoke the '${nnH}' skill...`;
  return `<system-reminder>
You are a worker assigned to implement feature "${A.id}".
...
## Your Assigned Feature
\`\`\`json
${JSON.stringify({ id: A.id, description: A.description, ... }, null, 2)}
\`\`\``;
}
```

The `GNI()` function takes three parameters: `H` (missionDir path), `A` (feature object), and `L` (worker session ID). It interpolates:
- Feature ID, description, skillName, milestone
- Mission file paths
- Worker session ID (sanitized to 12 chars)
- Conditional task instructions based on whether it's a validator skill

**JSON serialization for feature data:** The entire feature object is JSON-stringified with 2-space indentation and embedded inside a markdown code block.

### 2. Skill Injection into Prompts

Skills are stored as droids with a `systemPrompt` field. The Skill tool (2526.js) loads the skill and injects it into the conversation:

```js
// Module 2526.js, lines 25-35
yield {
  type: "result",
  isError: false,
  value: `Skill "${D.metadata.name}" is now active.

<skill name="${D.metadata.name}" filePath="${D.filePath}">
${D.systemPrompt}
</skill>`,
};
```

**Key observations:**
- Skills are loaded from droid registry (2519.js) by name
- The skill's `systemPrompt` is wrapped in `<skill name="..." filePath="...">` XML tags
- The skill content is returned as a tool result, which becomes part of the conversation history
- The `filePath` attribute references the SKILL.md location (e.g., `.opendroid/skills/mission-analyzer/SKILL.md`)
- Skills have validation: invalid or disabled skills are rejected

**SKILL.md parsing (2517.js:8-90):**
```js
// Module 2517.js, line 8
static parse(H, A) {
  let { frontmatter: L, body: $ } = this.extractSections(H),
    I = this.parseFrontmatter(L),
    D = $.trim();
  if (!D) throw new vH("System prompt cannot be empty");
  return { metadata: this.extractMetadata(I, A), systemPrompt: D, rawMetadata: I };
}
```

The `km` class parses YAML frontmatter (between `---` markers) and a markdown body. The body becomes the `systemPrompt`. Metadata fields include: `name`, `description`, `model`, `reasoningEffort`, `tools`, `color`, `version`.

### 3. Worker System Prompt Assembly

The worker's system prompt is assembled in three stages:

**Stage 1: Worker bootstrap (2541.js:40-108)**
The `GNI()` function creates the initial worker message containing:
- Worker session ID and agent-browser rules
- Mission file listing (mission.md, validation-contract.md, features.json, agent-guidelines.md)
- `.opendroid/` directory listing (services.yaml, init.sh, library/)
- The assigned feature as a JSON code block
- Instructions to read `fulfills` assertions from validation-contract.md
- Instructions to parallelize startup by reading context files + invoking skills simultaneously

**Stage 2: Skill invocation**
The worker invokes `mission-worker-base` skill (embedded in 2175.js as `mAf`), which provides:
- Phase 1: Startup (read context, init environment, baseline validation)
- Phase 2: Work (invoke feature-specific skill)
- Phase 3: Cleanup (stop services, commit, EndFeatureRun)

**Stage 3: Feature-specific skill**
The worker then invokes the skill named in `feature.skillName` (e.g., `mission-analyzer`), which provides the actual work procedure.

**Worker vs Validator distinction:**
```js
// Module 2541.js, line 47-53
let $ = HMH.includes(A.skillName);  // HMH = [sfH, efH] = scrutiny/user-testing validators
let E = $
  ? `## Your Task\n\n1. Invoke the '${A.skillName}' skill...\n2. Call EndFeatureRun when done`
  : `## Your Task\n\n1. First, invoke the '${nnH}' skill...\n2. Then, invoke the '${A.skillName}' skill...\n3. Call EndFeatureRun when done`;
```

Validators skip the `mission-worker-base` skill and go directly to their validation skill.

### 4. Orchestrator Prompt Template

The orchestrator prompt (`s6I` in 2175.js:431-540) is a large static template defining:

```markdown
# Role & Mindset
You are the architect and manager of a multi-agent mission. You plan the work, design the system of workers that will build it, and ensure quality through that system.

## CRITICAL: You Do NOT Implement
You are an architect. You NEVER write implementation code or do hands-on work yourself.

## Delegation Model
Your context window is finite. Preserve it for orchestration by delegating hands-on work to subagents using the Task tool.

## Workflow Overview
1. Mission Planning
2. Worker Design
3. Creating Mission Artifacts
4. Managing Execution
```

**Orchestrator vs Worker key differences:**
| Aspect | Orchestrator | Worker |
|--------|-------------|--------|
| Role | Architect/manager | Implementer |
| Writes code? | NEVER | Yes |
| Uses Task tool? | Yes, for delegation | Rarely |
| Manages missionDir? | Yes (creates agent-guidelines.md, features.json) | Reads only |
| Calls StartMissionRun? | Yes | No |
| Context focus | High-level planning, sequencing | Feature implementation details |

### 5. agent-guidelines.md Content Inclusion

agent-guidelines.md is discovered and read via 1181.js:

```js
// Module 1181.js, line 3
tO$ = ["CLAUDE.md", "Claude.md", "agent-guidelines.md", "agent-guidelines.md", "agent-guidelines.md"];
```

The `eO$()` function searches for these files by walking up from the current working directory to the git root, plus the user's home directory. It returns an array of `{ filePath, fileName, content, isPersonal }` objects.

**How agent-guidelines.md is included in prompts:**
1. For workers: The worker bootstrap (2541.js) lists agent-guidelines.md as one of the mission files to read during startup
2. For orchestrator: The orchestrator prompt (2175.js) references agent-guidelines.md extensively as the source of mission boundaries and guidance
3. For subagents: The Task tool prompt (2522.js) includes the subagent's system prompt but does NOT automatically include agent-guidelines.md: the parent agent must pass relevant context in the task description
4. agent-guidelines.md content is NOT directly embedded in system prompts; workers are instructed to read it via the Read tool

### 6. Mission Context Injection

Mission context is injected in multiple ways:

**A. `<system-reminder>` blocks (2541.js:22-40)**
```js
// Module 2541.js, line 22
function VNI(H) {
  return `${d0}
The mission has been approved. You must now author mission artifacts in ${H}:
- ${H}/validation-contract.md
- ${H}/validation-state.json
- ${H}/features.json
- ${H}/agent-guidelines.md
...
${aE}`;
}
```

**B. Orchestrator reminder (3399.js:421-432)**
```js
// Module 3399.js, line 421
CE.push({
  type: "text",
  text: Dl(
    "REMINDER: You are the orchestrator. Your role is to plan, design worker systems, and steer execution. Do NOT implement code yourself..."
  ),
});
```

**C. Mission directory context (3399.js:432)**
```js
// Module 3399.js, line 432
CE.push({
  type: "text",
  text: Dl(
    `Msample itemION CONTEXT: The current mission directory is ${kL}. All mission artifacts (features.json, agent-guidelines.md, etc.) are located there.`
  ),
});
```

**D. Paused mission reminder (2541.js:33-40)**
```js
// Module 2541.js, line 33
function XNI(H) {
  return `${d0}
The mission was paused while worker session "${H}" was in progress.
When the user asks you to continue or resume:
- Call start_mission_run with resumeWorkerSessionId="${H}"...
${aE}`;
}
```

### 7. System Message Construction (2857.js)

The `QrI()` function assembles the final system message array:

```js
// Module 2857.js, line 109
function QrI({ modelId: H, modelProvider: A, tools: L, systemPromptOverride: $ }) {
  let I = [
    { type: "text", text: h7 },           // Base identity
    { type: "text", text: $ ?? hMH },     // Role prompt or default
  ];
  if (A === "openai") {
    I.push({ type: "text", text: XrI() });  // Markdown spec
    let E = GrI(L);                          // CLI preference spec
    if (E) I.push({ type: "text", text: E });
    if (H && v7({ modelId: H }).systemPromptAdditions?.persistence)
      I.push({ type: "text", text: FrI() }); // Solution persistence
  }
  if (A === "google") {
    I.push({ type: "text", text: BrI() });  // Execute-cli rules
    I.push({ type: "text", text: TrI() });  // Tool usage rules
    I.push({ type: "text", text: WrI() });  // Spec mode guidelines
    I.push({ type: "text", text: VrI() });  // Todo tool guidelines
  }
  let D = I.length - 1;
  if (D >= 0) I[D] = { ...I[D], cache_control: { type: "ephemeral" } };
  return I;
}
```

**Key design:** The last block gets `cache_control: { type: "ephemeral" }` for Anthropic prompt caching. The `systemPromptOverride` parameter allows overriding the default `hMH` prompt: this is how orchestrator and worker modes inject their role-specific prompts.

### 8. Subagent Prompt Assembly (2522.js)

The Task tool constructs subagent prompts by combining:
1. The subagent droid's `systemPrompt`
2. A standardized mission context block
3. The parent-provided task description

```js
// Module 2522.js, lines 45-75 (excerpt)
P = [
  "exec",
  [
    "# Task Tool Invocation",
    `Subagent type: ${L}`,
    "## Context",
    "You are a specialized subagent invoked by another agent...",
    "## Your Subagent Identity",
    "Your core identity and specialized capabilities are defined by the following system prompt:",
    "---BEGIN SUBAGENT SYSTEM PROMPT---",
    f.systemPrompt,        // <-- Injected skill/droid prompt
    "---END SUBAGENT SYSTEM PROMPT---",
    "## Mission",
    "Follow the instructions from your subagent system prompt and the task below.",
    "## Non-negotiable rules",
    "## Task",
    "---BEGIN TASK FROM PARENT AGENT---",
    I,                     // <-- Parent's task description
    "---END TASK FROM PARENT AGENT---",
  ].join("\n"),
];
```

### 9. `<system-reminder>` Tag System

```js
// Module 0060.js, line 90-93
var d0 = "<system-reminder>",
  aE = "</system-reminder>",
  gJ = "<system-notification>",
  S7 = "</system-notification>";
```

These tags are used to:
- Wrap dynamic context that should be visible to the LLM but distinguished from user input
- Allow UI layers to filter them out (4065.js `JA0()` strips them for display)
- Inject mission state changes (approvals, pauses, errors) without polluting the conversation

## Code Examples

### Base identity prompt definition
```js
// 00-runtime.js:2
var h7 = "You are Droid, an AI software engineering agent built by OpenDroid.";
```

### Orchestrator prompt (excerpt)
```js
// 2175.js:431
s6I = `# Role & Mindset

You are the architect and manager of a multi-agent mission. You plan the work, design the system of workers that will build it, and ensure quality through that system.

You don't build - you design systems that build, and steer them to success.

## Worker Capabilities & Limitations
Implementation workers are skilled and efficient and execute well-specified features well, but struggle with ambiguity and can be lazy.

## CRITICAL: You Do NOT Implement
You are an architect. You NEVER write implementation code or do hands-on work yourself.

## Delegation Model
Your context window is finite. Preserve it for orchestration by delegating hands-on work to subagents using the Task tool.`;
```

### Worker bootstrap with feature interpolation
```js
// 2541.js:40-108
function GNI(H, A, L) {
  let D = L ? L.replace(/[^a-zA-Z0-9]/g, "").slice(0, 12) : "<workerSessionId>";
  return `<system-reminder>
You are a worker assigned to implement feature "${A.id}".

## Worker Session
Your worker session id is: ${L ?? "(unknown)"}

## Mission Files
The following files are in ${H}:
- mission.md
- validation-contract.md
- validation-state.json
- features.json
- agent-guidelines.md

## Your Assigned Feature
\`\`\`json
${JSON.stringify({
  id: A.id,
  description: A.description,
  skillName: A.skillName,
  milestone: A.milestone,
  preconditions: A.preconditions,
  expectedBehavior: A.expectedBehavior,
  verificationSteps: A.verificationSteps
}, null, 2)}
\`\`\`
</system-reminder>`;
}
```

### Skill injection with XML wrapping
```js
// 2526.js:25-35
yield {
  type: "result",
  isError: false,
  value: `Skill "${D.metadata.name}" is now active.

<skill name="${D.metadata.name}" filePath="${D.filePath}">
${D.systemPrompt}
</skill>`,
};
```

### System message assembly with provider-specific additions
```js
// 2857.js:109-133
function QrI({ modelId: H, modelProvider: A, tools: L, systemPromptOverride: $ }) {
  let I = [
    { type: "text", text: h7 },
    { type: "text", text: $ ?? hMH },
  ];
  if (A === "openai") {
    I.push({ type: "text", text: XrI() });
    let E = GrI(L);
    if (E) I.push({ type: "text", text: E });
    if (v7({ modelId: H }).systemPromptAdditions?.persistence)
      I.push({ type: "text", text: FrI() });
  }
  if (A === "google") {
    I.push({ type: "text", text: BrI() });
    I.push({ type: "text", text: TrI() });
    I.push({ type: "text", text: WrI() });
    I.push({ type: "text", text: VrI() });
  }
  let D = I.length - 1;
  if (D >= 0) I[D] = { ...I[D], cache_control: { type: "ephemeral" } };
  return I;
}
```

## Integration Points

- **01-terminal-ui (TUI)**: 3399.js injects orchestrator reminders into the message stream; 3152.js renders hooks UI: TUI displays system notifications but filters `<system-reminder>` tags via 4065.js
- **03-tool-agent-system (Tool)**: 2522.js (Task tool) is the primary subagent invocation mechanism; 1238.js (EndFeatureRun) defines the handoff tool that workers must call
- **04-desktop-gui (GUI)**: agent-guidelines.md discovery (1181.js) and system message construction (2857.js) are used by the Electron renderer; 4065.js strips system reminders for UI display
- **05-infrastructure (Infra)**: Droid registry (2519.js) stores skills in `.opendroid/skills/` and personal `~/.opendroid/skills/`; 1181.js discovers agent-guidelines.md via filesystem walk

## Implementation Notes

### Core components to port

1. **Prompt template library** (2175.js): Extract all template strings (`s6I`, `mAf`, `cAf`, etc.) into markdown files under `.opendroid/skills/` and `.opendroid/prompts/`. Currently they are hardcoded JS strings: port to loadable templates.

2. **System message builder** (2857.js): Port `QrI()` to a configurable function that assembles message arrays from base identity + role prompt + provider additions. Support pluggable provider modules.

3. **SKILL.md parser** (2517.js): Port the `km` class for parsing YAML frontmatter + markdown body. This is a generic utility.

4. **Droid registry** (2519.js): Port the CRUD operations for droids/skills with `systemPrompt` storage. Store as `.md` files, not in a settings JSON.

5. **Worker bootstrap assembler** (2541.js): Port `GNI()` to a template engine that interpolates feature JSON into a worker prompt. Use a real template language (e.g., Handlebars, EJS) instead of JS template literals.

6. **agent-guidelines.md discovery** (1181.js): Port the filesystem walk to discover agent guideline files. Add caching.

7. **Subagent prompt assembler** (2522.js): Port the Task tool's prompt construction. Standardize the subagent identity/mission/task structure.

### Simplifications for OpenDroid
- Replace the massive 2175.js embedded prompt library with filesystem-based templates
- Add a `prompts/` directory with orchestrator.md, worker-base.md, mission-planning.md, etc.
- Make provider-specific additions (XrI, BrI, TrI, etc.) pluggable via provider config
- Support hot-reloading of prompts during development

## Known Gaps

- The exact mechanism by which `systemPromptOverride` is set for orchestrator vs worker sessions was not fully traced: it appears to be set via `BA().getDecompSessionType()` returning "orchestrator" or "worker" but the override injection point needs verification
- The `h7` base prompt is hardcoded in `00-runtime.js` which is part of the runtime bundle, not a user-editable file. For OpenDroid, this should be configurable
- The agent-guidelines.md content is read by workers via the Read tool, but the exact mechanism that injects agent-guidelines.md content into the orchestrator's context was not fully traced
- How the `systemPromptOverrideRef` (3134.js/3135.js) gets its value for different session types needs deeper tracing through the React state management
- The full list of provider-specific prompt additions (FrI, XrI, GrI, BrI, TrI, WrI, VrI) were referenced but their full content was not all read due to context budget
