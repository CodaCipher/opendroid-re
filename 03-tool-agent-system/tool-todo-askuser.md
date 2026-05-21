# TodoWrite & AskUser Tools: Tool System Architecture Notes

## Overview

OpenDroid implements two interactive planning/communication tools: **TodoWrite** and **AskUser**. TodoWrite provides structured task tracking with a text-based status marker format (`[completed]`, `[in_progress]`, `[pending]`), parsed from a multi-line string input into structured objects. AskUser enables the agent to present structured multiple-choice questionnaires (1–10 questions, 2–10 options each) to gather user clarification during execution. Both tools are client-side, integrated into the conversation state manager, and have dedicated UI rendering paths. AskUser is conditionally enabled only in interactive terminal modes, while TodoWrite is always available and includes a staleness detection mechanism that reminds the LLM to update its task list.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 1246.js | ~6.3 KB | TodoWrite tool definition (`eI()` call with full description, input schema `bZA`, display name "Plan") | App |
| 1225.js | ~1.5 KB | AskUser tool definition (`eI()` call with questionnaire schema, conditional enablement) | App |
| 1217.js | ~4.5 KB | Shared input schemas for all tools including AskUser (`eZ$` questionnaire schema) | App |
| 2317.js | ~4.5 KB | Questionnaire parser (`gCI()`): validates [question], [topic], [option] format | App |
| 2318.js | ~2.0 KB | AskUser execution handler class (`Z7H`): parses questionnaire, collects answers via `SCI()` | App |
| 2316.js | ~3.0 KB | AskUser answer collection (`SCI()`): JSON-RPC and TUI modes, event bus routing | App |
| 3361.js | ~2.0 KB | Todo item parser (`Yz()`, `R1L()`): parses status markers from string to structured objects | App |
| 2572.js | ~28.7 KB | Conversation state manager (`class as`): holds `currentTodos`, `updateTodoState()`, persistence | App |
| 3411.js | ~5.1 KB | Tool UI formatter: TodoWrite display formatting (`kIA`, `xYH`, `yIA`) | App |
| 3399.js | ~18.7 KB | LLM tool loop: TodoWrite staleness detection, reminder injection | App |
| 2519.js | ~1.0 KB | Tool registry map: `TodoWrite: aG(bO)`, `AskUser: aG(sc)` entries | App |
| 0966.js | ~1.5 KB | Tool ID constants: `CKA = "TodoWrite"`, `q4H = "AskUser"` | App |
| 0060.js | ~1.0 KB | Tool action enum: `U.AskUser = "ask_user"` | App |
| 0724.js | ~1.5 KB | Settings store: `askUserToolEnabled` flag, `setAskUserToolEnabled()` | App |
| 3417.js | ~6.0 KB | ACP adapter: TodoWrite plan event forwarding via `sessionUpdate("plan")` | App |
| 3420.js | ~22.1 KB | Session adapter: TodoWrite tool result handling during streaming | App |
| 0084.js | ~6.5 KB | AskUser request schema (`YM0` type with questionnaire, parsed, parseError) | App |
| 3428.js | ~12.0 KB | JSON-RPC adapter: `pendingAskUserRequests` map, ask-user request/response forwarding | App |
| 3384.js | ~8.0 KB | TUI renderer: TodoWrite "updating"/"updated"/"failed" status display in terminal | App |
| 0226.js | ~0.5 KB | External tool name mapping: `TodoWrite: "TodoWrite"` (Claude compatibility) | App |

## Architecture

### TodoWrite Tool Architecture

```
LLM generates todos string (numbered lines with [status] markers)
    ↓
Tool input: bZA schema (todos: string)
    ↓
Parsing: R1L() in 3361.js: regex: /^(?:\d+[.)]\s*)?\[(completed|in_progress|pending)\]\s*(.+)$/
    ↓
Structured: { id, status, content, priority }[]
    ↓
ConversationStateManager.updateTodoState() in 2572.js
    ├── currentTodos = input
    ├── currentTodoWriteId = toolCallId
    ├── todoUpdateMessageIndex = user message count
    ├── BA().appendTodoState(): session persistence
    └── scheduleReactUpdate("updateTodoState")
    ↓
UI: "✓ completed item" / "○ pending item" formatting via xYH() in 3411.js
    ↓
ACP: sessionUpdate("plan") with entries (content, priority, status)
    ↓
Staleness detection: KIM() checks for pending/in_progress items
    → wIM() counts tool calls since last TodoWrite
    → If ≥ QCI (threshold) tool calls passed → inject reminder into next LLM prompt
```

### TodoWrite State Transitions

```
pending → in_progress → completed
   ↑                       |
   +---- (if blocked) -----+
   
Rules:
- Exactly ONE [in_progress] at any time
- [completed] only after verified work
- Staleness: QCI+ tool calls without update → LLM reminder injected
- No [pending] items without exactly one [in_progress] (CRITICAL constraint)
```

### AskUser Tool Architecture

```
LLM generates questionnaire text (with [question], [topic], [option] markers)
    ↓
Tool input: eZ$ schema (questionnaire: string)
    ↓
Preprocessing: J0f(): splits inline [topic]/[option] to separate lines
    ↓
Parsing: gCI() in 2317.js
    ├── Validates 1-10 questions (FoH=10)
    ├── Validates 2-10 options per question (bCI=2, QoH=10)
    ├── Validates unique options (case-insensitive)
    ├── Validates [topic] and [option] ordering
    └── Returns: { questions: [{ index, topic, question, options }] }
    ↓
Answer collection: SCI() in 2316.js
    ├── JSON-RPC mode:
    │   ├── Generates requestId (U1())
    │   ├── Emits "ask-user-request" on event bus
    │   └── Waits for "ask-user-response" (with abort support)
    └── TUI mode:
        ├── Stores in global Map (Es) keyed by toolCallId
        ├── Triggers listeners (VoH) for UI update
        └── Plays "awaiting input" sound
    ↓
JSON-RPC routing (3428.js):
    ├── DroidRequestHandler stores in pendingAskUserRequests
    ├── Broadcasts to client via "droid.ask_user" method
    └── Client resolves → "ask-user-response" emitted → answer returned
    ↓
Result formatting (2318.js):
    "1. [question] Question text
     [answer] User's answer"
```

### AskUser Enablement Gate

```
isToolEnabled: ({ cliDroidMode, askUserToolEnabled }) =>
    askUserToolEnabled === true && 
    (cliDroidMode === "terminal-ui" || cliDroidMode === "interactive-cli")
```

Controlled by settings store in 0724.js. Feature flag: `EnableAskUserTool` (0216.js, flag key: `<feature-flag-key>`).

## Key Findings

### 1. TodoWrite Input Schema: Text-Based Status Markers

Module 3361.js, `R1L()` function parses the todo text format:

```js
// Module 3361.js, line 9
function R1L(H) {
  let A = H.trim()
    .split(`\n`)
    .filter(($) => $.trim());
  if (A.length === 0) return [];
  let L = [];
  return (
    A.forEach(($, I) => {
      let D = $.match(/^(?:\d+[.)]\s*)?\[(completed|in_progress|pending)\]\s*(.+)$/);
      if (D) L.push({ id: String(I + 1), status: D[1], content: D[2].trim(), priority: "high" });
    }),
    L
  );
}
```

The `Yz()` wrapper handles multiple input formats (string, array of strings, JSON array):

```js
// Module 3361.js, line 24
function Yz(H) {
  let { todos: A } = H;
  if (Array.isArray(A)) {
    if (A.length > 0 && typeof A[0] === "string")
      return R1L(A.join(`\n`));
    return A;
  }
  if (typeof A !== "string") return [];
  let L = A.trim();
  if (L.startsWith("["))
    try {
      let $ = JSON.parse(L);
      if (Array.isArray($)) {
        if ($.length > 0 && typeof $[0] === "string")
          return R1L($.join(`\n`));
        return $;
      }
    } catch {}
  return R1L(L);
}
```

### 2. AskUser Questionnaire Validation with Strict Format Enforcement

Module 2317.js, `gCI()` function implements a state-machine parser:

```js
// Module 2317.js, line 44 (constants)
var FoH = 10,   // max questions
    bCI = 2,    // min options per question
    QoH = 10;   // max options per question

// Module 2317.js (regex patterns from 2317.js)
uCI = /^(?:(?:\d+[.)]|[-*\u2022])\s*)?\[question\]\s*(.+?)\s*(?:\(multi\))?\s*$/i;
xCI = /^\s*\[topic\]\s*(.+?)\s*$/i;
vCI = /^\s*\[option\]\s*(.+?)\s*$/i;
F0f = /```(?:\w+)?\s*([\s\S]*?)```/m;  // extract from code fences
```

Validation throws `vH("Invalid AskUser questionnaire format", { line, message })` for:
- Missing [question] text
- [topic]/[option] without preceding [question]
- Duplicate options (case-insensitive)
- Exceeding max questions (10) or options (10)
- Unexpected lines (no headers or code fences allowed)

### 3. AskUser Answer Collection: Dual-Mode (JSON-RPC / TUI)

Module 2316.js, `SCI()` function:

```js
// Module 2316.js, line 18
function SCI(H, A, L, $) {
  if (Qf().isJsonRpcMode()) {
    // JSON-RPC: route via event bus
    let I = U1(); // generate unique ID
    return new Promise((D, E) => {
      // Listen for "ask-user-response" event
      v$.on("ask-user-response", (P) => {
        if (P.requestId !== I) return;
        if (P.result.cancelled === true) {
          E(Error("User cancelled AskUser"));
          return;
        }
        D(P.result.answers);
      });
      // Emit request event
      v$.emit("ask-user-request", { 
        sessionId: $, requestId: I, 
        toolCallId: H, questions: A 
      });
    });
  }
  // TUI mode: store in global Map
  return new Promise((I, D) => {
    let P = {
      toolCallId: H, questions: A,
      resolve: (W) => { I(W); },
      reject: (W) => { D(W); },
    };
    Es.set(H, P);
    // Play "awaiting input" sound
    VoH(); // trigger UI listeners
  });
}
```

### 4. TodoWrite Staleness Detection & Reminder Injection

Module 3399.js detects stale todo lists and injects reminders:

```js
// Module 3399.js, line 398-415
let hM = N$.getLatestTodoState();
if (!hM || !hM?.todos?.todos?.length)
  // No todos at all → inject initial reminder
  CE.push({
    type: "text",
    text: "IMPORTANT: TodoWrite was not called yet. You must call it for any non-trivial task..."
  });
else if (KIM(hM.todos)) {
  // Has pending items → check staleness
  let oA = L(),  // conversation history
      $L = wIM(oA);  // count tool calls since last TodoWrite
  if ($L >= QCI)  // threshold exceeded
    CE.push({
      type: "text",
      text: `IMPORTANT: Your todo list has pending items but hasn't been updated in the last ${$L} tool calls...`
    });
}
```

### 5. AskUser Schema Definition

Module 1217.js defines the AskUser input schema:

```js
// Module 1217.js, line 234
eZ$ = p.object({
  questionnaire: p.string().describe(
    `A plain-text questionnaire to ask the user. Use this format:
1. [question] Which features do you want to enable?
[topic] Features
[option] Auth handling
[option] Login Page
...
Notes:
- 1-4 questions
- 2-4 options per question
- [topic] is a short label for UI navigation bar
- Do NOT include an 'Own answer' option; UI provides it automatically
- Keep option labels short and mutually exclusive`
  ),
});
```

### 6. Tool Registry Entries

Module 2519.js registers both tools:

```js
// Module 2519.js, line 17-21
AskUser: aG(sc),    // from 1225.js
TodoWrite: aG(bO),   // from 1246.js

// Special sets
Y9f = new Set([d1.TodoWrite, d1.Skill]),  // tools with special handling
```

Module 0966.js defines LLM-facing IDs:
```js
// Module 0966.js, line 22-23
CKA = "TodoWrite",   // TodoWrite LLM ID
q4H = "AskUser",     // AskUser LLM ID
```

### 7. AskUser Result Formatting

Module 2318.js formats the collected answers back to the LLM:

```js
// Module 2318.js, line 52-56
let D = [];
for (let E of I)
  (D.push(`${E.index}. [question] ${E.question}`),
   D.push(`[answer] ${E.answer}`),
   D.push(""));
yield {
  type: "result", isError: false,
  value: D.join(`\n`).trim(),
};
```

## Integration Points

### Cross-Mission Dependencies

- **Terminal UI section (TUI)**: Module 3384.js renders TodoWrite status in terminal UI ("↳ TodoWrite updating/updated/failed"). Module 3361.js contains `JIA()` which uses Ink's `jsxDEV` for React-based todo list rendering. AskUser questionnaire UI is TUI-rendered via the listener pattern (`VoH()`).
- **Orchestration section (Mission)**: TodoWrite is used within mission worker sessions for task tracking. The staleness detection and reminder system are part of the agent tool loop managed by mission orchestration.
- **Desktop GUI section (GUI)**: Module 3417.js (ACP adapter) forwards TodoWrite updates as `sessionUpdate("plan")` events to external clients. AskUser requests are routed via JSON-RPC (`droid.ask_user` method).
- **Infrastructure section (Infra/Session)**: TodoWrite state is persisted via `BA().appendTodoState()` and restored via `loadTodoStateFromSession()`. AskUser pending requests are stored in session state (`pendingAskUserRequests` in 0087.js schema). Module 0216.js uses feature flags for `EnableAskUserTool`.

### Tool-System Internal Dependencies

- **Tool Registry (3264.js)**: Both tools registered via `eI()` factory, looked up via `k0()` registry
- **Permission System**: AskUser has `requiresConfirmation: true`; TodoWrite has `requiresConfirmation: false`
- **Sandbox**: Both are `executionLocation: "client"` tools: no sandbox wrapping needed
- **LLM Adapters**: Tool IDs (`CKA`, `q4H`) used in function-calling manifest generation

## Implementation Notes

### TodoWrite Tool API

```typescript
interface TodoWriteInput {
  todos: string;  // Multi-line text with status markers
  // Format: "1. [completed] Task A\n2. [in_progress] Task B\n3. [pending] Task C"
}

interface TodoItem {
  id: string;      // Auto-generated "1", "2", etc.
  status: "completed" | "in_progress" | "pending";
  content: string;
  priority: "high";  // Currently always "high"
}

// Key constants
const MAX_TODO_ITEMS = F1H;  // defined in 1246.js
const MAX_CHARS_PER_ITEM = vt;  // defined in 1246.js
```

### AskUser Tool API

```typescript
interface AskUserInput {
  questionnaire: string;  // Structured text format
  // Format: 
  // 1. [question] Question text?
  // [topic] TopicLabel
  // [option] Option A
  // [option] Option B
}

interface ParsedQuestion {
  index: number;
  topic: string;     // Normalized (spaces → hyphens)
  question: string;
  options: string[];
}

// Validation limits
const MAX_QUESTIONS = 10;
const MIN_OPTIONS_PER_QUESTION = 2;
const MAX_OPTIONS_PER_QUESTION = 10;

// Enablement
// askUserToolEnabled must be true AND cliDroidMode must be "terminal-ui" or "interactive-cli"
```

### State Management Integration

```typescript
// ConversationStateManager (2572.js)
class ConversationStateManager {
  currentTodos: any;           // Latest todo state
  currentTodoWriteId: string;  // Last TodoWrite tool call ID
  todoUpdateMessageIndex: number;  // User message count at last update
  
  updateTodoState(input: any, toolCallId: string): void;
  loadTodoStateFromSession(): void;
  getTodoStaleMessageCount(): number;
}

// AskUserAnswerStore (2316.js): global Map
const Es: Map<string, {
  toolCallId: string;
  questions: ParsedQuestion[];
  resolve: (answers) => void;
  reject: (error) => void;
}>;
```

## Open Questions / Left Undone

1. **TodoWrite constants `F1H` and `vt`**: The max items count and max chars per item are referenced in 1246.js but defined in a dependency module that was not fully traced. These appear to be imported constants.
2. **TodoWrite `bZA` input schema**: The actual zod schema (`bZA`) for the `todos` field was not fully traced: it's defined as a dependency of 1246.js but in a separate module.
3. **Staleness threshold `QCI`**: The exact number of tool calls before the staleness reminder triggers was not traced to its definition.
4. **Todo persistence path**: `BA().appendTodoState()` persistence mechanism (likely session file) not fully traced.
5. **AskUser "Own answer" UI**: The description states "UI provides it automatically" but the client-side UI component for this was not analyzed (likely Desktop GUI section or Terminal UI section scope).
6. **AskUser JSON-RPC schema** (0084.js): The `YM0` schema for `ask_user` type includes `parsed` and `parseError` optional fields suggesting server-side pre-validation, but this flow was not fully traced.
7. **Module 2188.js, 3370.js, 1193.js, 3371.js** from the seed module list were not deeply analyzed: they may contain related UI or routing logic that would benefit from targeted follow-up.
