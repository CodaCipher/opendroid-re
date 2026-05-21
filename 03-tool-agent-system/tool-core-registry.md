# Tool Core Registry: Tool System Architecture Notes

## Overview

OpenDroid's tool system is built around a centralized registry pattern implemented in module **1176.js**. The registry (`qgH` class) is a `Map`-backed singleton keyed by tool ID, providing lookup by both internal ID and LLM-facing ID. Tools are defined via the `eI()` factory function (module **1201.js**), which wraps a Zod-validated schema with LLM-appropriate JSON Schema conversion. The main registration site is module **2548.js**, which registers 19 built-in tools at application startup. Module **2519.js** exports the canonical `d1` enum mapping tool names to their LLM-facing IDs. Module **2518.js** provides validation and tool category grouping (read-only, edit, execute, web, mcp) used for permission resolution and droid configuration.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 1176.js | ~1KB | Tool Registry class (`qgH`): Map-backed keyed store | app-tools |
| 1177.js | ~2KB | Singleton factory (`k0()`, `X8`) via `og()` lazy initializer | app-tools |
| 1201.js | ~10KB | Tool definition factory `eI()`, Zod schema definitions | app-tools |
| 2548.js | ~7KB | Built-in tool registration (19 tools registered at startup) | app-tools |
| 2518.js | ~11KB | Tool validation, categorization (`O9f`, `Z9f`, `vzI`), manifest helpers | app-tools |
| 2519.js | ~8KB | `d1` tool name enum, `xzI` user-selectable list, `hoA` hidden set | app-tools |
| 0966.js | ~2KB | Tool ID string constants (`GKA="Read"`, `QKA="Edit"`, etc.) | app-tools |
| 1029.js | ~1KB | Zod→JSON Schema converter (`f7$ = cqA`) | app-tools |
| 1175.js | ~5KB | MCP tool wrapper class: adapts MCP tools to registry format | app-tools |
| 1178.js | ~15KB | MCP service: manages MCP server tool registration/unregistration | app-tools |
| 2547.js | ~3KB | Task tool manager: lazy registration/unregistration of `task-cli` | app-tools |

## Architecture

### Tool Definition Schema (`eI()` factory)

Each tool is created by the `eI()` function in **1201.js** (line 233):

```js
function eI(H) {
  let A = H.inputSchema,
    L = H.llmInputSchema ?? H.inputSchema,
    $ = H.outputSchemas;
  if (!$) $ = { result: a3H, updates: void 0 };
  let I = f7$(L);
  return { ...H, inputSchema: I, inputZodSchema: A, outputSchemas: $, isMcpTool: false };
}
```

The input object `H` has the following shape:
- `id`: Internal tool identifier (e.g., `"read-cli"`, `"edit-cli"`)
- `llmId`: LLM-facing name (e.g., `"Read"`, `"Edit"`, `"Execute"`)
- `uiGroupId`: UI grouping key (e.g., `"view_file"`, `"edit_file"`, `"planning"`)
- `displayName`: Human-readable name shown in UI
- `description`: Full description text (often multi-paragraph)
- `executionLocation`: Where tool runs (`"client"` or `"server"`)
- `inputSchema`: Zod schema (kept as `inputZodSchema`)
- `llmInputSchema`: Optional separate schema for LLM (defaults to `inputSchema`)
- `outputSchemas`: Map of output keys to Zod schemas
- `outputTransform`: Optional post-processing function for results
- `isVisibleToUser`: Whether tool appears in user-facing UI
- `isTopLevelTool`: Whether tool is a top-level tool
- `requiresConfirmation`: Whether tool needs user confirmation
- `toolkit`: Toolkit group (`"Base"`)
- `isToolEnabled`: Boolean or function `({modelProvider}) => boolean`

The factory:
1. Extracts `inputSchema` (Zod) and `llmInputSchema` (Zod, optional override)
2. Converts the LLM schema to JSON Schema via `f7$()` (= `cqA`, module **1029.js**)
3. Stores original Zod schema as `inputZodSchema`, converted as `inputSchema`
4. Sets `isMcpTool: false` for all built-in tools

### Registry Class (`qgH`)

```js
// Module 1176.js
class qgH {
  registry = new Map();
  register(H) {
    this.registry.set(H.tool.id, H);
  }
  getExecutor(H) {
    let A = this.registry.get(H);
    if (!A) return;
    if (!A.executorInstance) A.executorInstance = A.executorFactory() ?? void 0;
    return A.executorInstance;
  }
  getTool(H) {
    return this.registry.get(H)?.tool;
  }
  getAllTools() {
    return Array.from(this.registry.values()).map((H) => H.tool);
  }
  getToolByLlmId(H) {
    for (let A of this.registry.values()) if (A.tool.llmId === H) return A.tool;
    return;
  }
  unregisterTool(H) {
    this.registry.delete(H);
  }
}
```

The registry stores `{ tool, executorFactory }` entries keyed by `tool.id`. Lazy executor instantiation via `executorFactory()` pattern.

### Singleton Access (`k0()`)

Two instances are created via `og()` (lazy singleton):

```js
// Module 1177.js, line 4
((X8 = og(() => new qgH())), (k0 = og(() => new qgH())));
```

`k0()` is the primary tool registry. `X8` appears to be a secondary/alternate instance.

### Built-in Tool Registration

**Module 2548.js** registers 19 built-in tools at startup:

```js
k0().register({ tool: tR, executorFactory: () => new LKH() });  // Read
k0().register({ tool: oc, executorFactory: () => new swH() });  // LS
k0().register({ tool: tnH, executorFactory: () => new Nh() });  // ? 
k0().register({ tool: sR, executorFactory: () => new R7H() });  // Edit
k0().register({ tool: Hh, executorFactory: () => new A7H() });  // ApplyPatch (ExitSpecMode)
k0().register({ tool: tc, executorFactory: () => new awH() });  // Grep
k0().register({ tool: ac, executorFactory: () => new twH() });  // Glob
k0().register({ tool: eR, executorFactory: () => new N7H() });  // Create
k0().register({ tool: M2, executorFactory: () => new doA() });  // ExitSpecMode
k0().register({ tool: sc, executorFactory: () => new Z7H() });  // AskUser
k0().register({ tool: kO, executorFactory: () => new _KH() });  // WebSearch
k0().register({ tool: bO, executorFactory: () => new UKH() });  // TodoWrite
k0().register({ tool: rc, executorFactory: () => new u7H() });  // FetchUrl
k0().register({ tool: qQ, executorFactory: () => new $KH() });  // slackPostMessage
k0().register({ tool: hX, executorFactory: () => new IKH() });  // ?
k0().register({ tool: ec, executorFactory: () => new loA() });  // GenerateDroid
// After async init:
k0().register({ tool: U2, executorFactory: () => new WKH() });  // Skill
k0().register({ tool: T1H, executorFactory: () => new ooA() });  // StartMissionRun
k0().register({ tool: V1H, executorFactory: () => new HtA() });  // EndFeatureRun
k0().register({ tool: G1H, executorFactory: () => new roA() });  // ?
k0().register({ tool: X1H, executorFactory: () => new ioA() });  // ?
```

### Tool ID Constants

**Module 0966.js** defines string constants:
```js
XKA = "LS", GKA = "Read", FKA = "Create", QKA = "Edit",
JKA = "ApplyPatch", wKA = "Glob", KKA = "Grep", C4H = "Execute",
CKA = "TodoWrite", q4H = "AskUser", qKA = "WebSearch", YKA = "FetchUrl",
OKA = "ExitSpecMode", ZKA = "ProposeMission", Y4H = "StartMissionRun",
zKA = "EndFeatureRun", NKA = "DismissHandoffItems"
```

### Tool Name Enum (`d1`)

**Module 2519.js** maps tool variable references to their LLM IDs:
```js
d1 = {
  Read: aG(tR),        // "Read"
  Grep: aG(tc),        // "Grep"
  Glob: aG(ac),        // "Glob"
  Ls: aG(oc),          // "LS"
  Create: aG(eR),      // "Create"
  Edit: aG(sR),        // "Edit"
  Execute: aG(aR),     // "Execute"
  AskUser: aG(sc),     // "AskUser"
  WebSearch: aG(kO),   // "WebSearch"
  FetchUrl: aG(rc),    // "FetchUrl"
  ApplyPatch: aG(Hh),  // "ApplyPatch"
  TodoWrite: aG(bO),   // "TodoWrite"
  Skill: aG(U2),       // "Skill"
  ExitSpecMode: aG(M2),// "ExitSpecMode"
  GenerateDroid: aG(ec),// "GenerateDroid"
  slackPostMessage: aG(qQ), // "slackPostMessage"
}
```

Where `aG = (H) => H.llmId || H.id` (module **2518.js**, ErrTracker init section6).

### User-Selectable and Hidden Tools

```js
// Module 2519.js
xzI = [d1.Read, d1.Ls, d1.Grep, d1.Glob, d1.Create, d1.Edit,
       d1.Execute, d1.AskUser, d1.WebSearch, d1.FetchUrl];
hoA = new Set([d1.ExitSpecMode, d1.GenerateDroid]); // Hidden from user
Y9f = new Set([d1.TodoWrite, d1.Skill]);             // Special category
```

### LLM Function-Calling Manifest Generation

The flow for generating LLM tool manifests:
1. `k0().getAllTools()` returns all registered tool objects
2. `O9f()` (module **2518.js**) filters out MCP tools and hidden tools, collecting `llmId || id` for each
3. `joA()` collects MCP-specific tool IDs (those with `isMcpTool: true`)
4. `vzI()` combines both: `[...O9f(), ...joA()]`
5. `Z9f()` creates category groupings: `{ "read-only", "edit", "execute", "web", "mcp" }`
6. Each tool's `inputSchema` (JSON Schema, converted from Zod via `f7$`) is used directly as the LLM function-calling parameter definition
7. The `d1` enum provides the function name for LLM APIs

For OpenAI models specifically, Edit tool is replaced with ApplyPatch (see `normalizeTools` in **2518.js**).

### Zod → JSON Schema Conversion

The conversion pipeline uses `cqA` (aliased as `f7$` in module **1030.js**):
- Zod schemas are defined per tool using `p.object({...})` syntax
- Each field uses `.describe()` for LLM-facing descriptions
- The `f7$` function converts Zod schemas to JSON Schema format
- Tools can have separate `llmInputSchema` for LLM-specific schema differences

### MCP Tool Registration (Dynamic)

MCP tools are registered dynamically by the MCP service (**1178.js**):
```js
// Module 1178.js, line 592
let A = k0();
for (let [L, $] of Object.entries(H))
  let I = NO.createMcpToolImplementations(L, $);
  // Registers each MCP tool with mcp__server__tool namespace
```

MCP tools set `isMcpTool: true` and are wrapped by class in **1175.js**.

### Task Tool Lazy Registration

The Task tool (`task-cli`) is managed by `TaskToolManager` (**2547.js**) with lazy registration:
```js
k0().register({ tool: RZA(H), executorFactory: () => new cs() });
// Unregistration: k0().unregisterTool("task-cli")
```

## Key Findings

### Finding 1: Tool Definition OpenDroid Pattern

```js
// Module 1201.js, line 233
function eI(H) {
  let A = H.inputSchema,
    L = H.llmInputSchema ?? H.inputSchema,
    $ = H.outputSchemas;
  if (!$) $ = { result: a3H, updates: void 0 };
  let I = f7$(L);
  return { ...H, inputSchema: I, inputZodSchema: A, outputSchemas: $, isMcpTool: false };
}
```

This factory converts Zod schemas to JSON Schema for LLM function-calling while preserving the original Zod schema for runtime validation. The `isMcpTool: false` flag distinguishes built-in tools from dynamically registered MCP tools.

### Finding 2: Dual-ID System (internal vs LLM)

```js
// Module 0966.js: LLM-facing IDs
GKA = "Read", QKA = "Edit", C4H = "Execute"

// Module 1221.js: Internal tool registration
sR = eI({
  id: "edit-cli",        // Internal ID
  llmId: QKA,            // = "Edit" (LLM-facing)
  ...
});
```

The registry is keyed by internal `id` (e.g., `"edit-cli"`), but lookups for LLM tool calls use `getToolByLlmId()` with the `llmId` (e.g., `"Edit"`). The `aG` helper resolves the display name: `aG = (H) => H.llmId || H.id`.

### Finding 3: Tool Categories for Permission Resolution

```js
// Module 2518.js, lines 12-20
function Z9f() {
  let A = [d1.Read, d1.Grep, d1.Glob, d1.Ls],     // read-only
    L = [d1.Edit, d1.Create, d1.ApplyPatch],         // edit
    $ = [d1.Execute],                                  // execute/execution
    I = [d1.WebSearch, d1.FetchUrl];                  // web
  return {
    "read-only": ..., "edit": ..., "execute": ...,
    "execution": ..., "web": ..., "mcp": []
  };
}
```

Tools are grouped into permission categories that map to sandbox/permission levels.

### Finding 4: OpenAI Edit→ApplyPatch Substitution

```js
// Module 2518.js, normalizeTools()
if (B.includes(d1.Edit)) {
  let Q = this.getModelProvider(A);
  if (Q === "openai") {
    B = B.filter((w) => w !== d1.Edit);
    if (!B.includes(d1.ApplyPatch)) B.push(d1.ApplyPatch);
  }
}
```

When using OpenAI models, the Edit tool is automatically replaced with ApplyPatch because OpenAI's function calling doesn't handle the old_str/new_str replacement pattern well.

### Finding 5: Edit Tool with Conditional Enablement

```js
// Module 1221.js
sR = eI({
  id: "edit-cli",
  ...
  isToolEnabled: ({ modelProvider: H }) => H !== "openai",
});
```

Individual tools can declare conditional enablement based on the model provider, checked at runtime.

## Integration Points

- **Tool Permissions (tool-permissions)**: The `Z9f()` category function and `xzI` selectable list feed directly into the permission resolver. The `d1` enum is used by permission checks.
- **Tool Sandbox (tool-sandbox)**: Sandbox uses `d1.Execute` and `d1.Edit` category checks. The `executionLocation` property determines client vs server execution.
- **MCP Client (tool-mcp-client)**: Module **1178.js** dynamically registers MCP tools into the same `k0()` registry. Module **1175.js** wraps MCP tools with `isMcpTool: true`.
- **Task Subagent (tool-task-subagent)**: Module **2547.js** lazily registers `task-cli` tool when a task is initiated.
- **LLM Adapters (tool-llm-*)**: The `inputSchema` (JSON Schema) from each tool is used directly as function-calling parameter definitions for Anthropic/OpenAI APIs.
- **Streaming (tool-streaming)**: Tool calls detected mid-stream reference tools by `llmId`, resolved via `getToolByLlmId()`.
- **Agent State (tool-agent-state)**: The tool execution loop iterates over registered tools and dispatches to executors.
- **TUI/01-terminal-ui**: `uiGroupId` property determines how tools are displayed in the TUI.
- **02-orchestration**: `requiresConfirmation` flag triggers the mission system's ask-user flow.

## Implementation Notes

### Tool Registry API
```typescript
// Singleton access
const registry = k0();  // lazy singleton

// Registration
registry.register({ tool: toolDefinition, executorFactory: () => new Executor() });

// Lookup
registry.getTool("edit-cli");        // by internal ID
registry.getToolByLlmId("Edit");     // by LLM-facing name
registry.getAllTools();               // all registered tools
registry.getExecutor("edit-cli");    // lazy-instantiated executor

// Unregistration
registry.unregisterTool("edit-cli");
```

### Tool Definition Template
```typescript
const myTool = eI({
  id: "my-tool",
  llmId: "MyTool",
  uiGroupId: "my_group",
  displayName: "My Tool",
  description: "...",
  executionLocation: "client",
  inputSchema: z.object({ ... }),
  outputSchemas: { result: z.string() },
  outputTransform: (result) => formattedResult,
  isVisibleToUser: true,
  isTopLevelTool: true,
  requiresConfirmation: false,
  toolkit: "Base",
  isToolEnabled: true,  // or ({ modelProvider }) => boolean
});
```

### Permission Categories
- `read-only`: Read, Grep, Glob, LS
- `edit`: Edit, Create, ApplyPatch
- `execute`: Execute
- `web`: WebSearch, FetchUrl
- `mcp`: dynamically populated

## Open Questions / Left Undone

- **Executor class hierarchy**: Each tool has a paired executor class (e.g., `LKH` for Read, `R7H` for Edit). The full executor interface (methods, lifecycle) was not traced in this analysis: deferred to individual tool feature analyses.
- **Schema conversion details**: The `cqA` Zod→JSON Schema converter was found but its full implementation (handling of `.describe()`, nested objects, optional fields) was not traced in detail.
- **Tool dispatch pipeline**: How the agent loop selects which tool to call from LLM output → dispatches to executor → collects result was not fully traced. This spans agent-state and streaming features.
- **Module 3264.js** was initially flagged as the core module (121KB) but turned out to be a Mathematica/Wolfram symbol catalog: not tool-related. The actual tool registry is distributed across smaller modules.
- **Tool cache invalidation**: `k5.invalidateToolCaches()` in module **2518.js** clears `EKH` and `fKH` caches, but the trigger conditions were not traced.
- **`tnH` and `aR` tool variables**: Some tool variable names (tnH, aR, hX, T1H, V1H, G1H, X1H) were found in registration but their exact tool IDs were not fully resolved: they map to ProposeMission, Execute, and other mission/skill tools.
