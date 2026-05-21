# tui-composer-input: TUI Architecture Notes

## Overview

The OpenDroid TUI does not contain a dedicated multi-line "Composer" widget. Instead, text entry is handled by a single-line `TextInput` component (`C8` in module 3861.js) that is used across dialog patterns (AskUser, Proposal Confirmation, Plan Confirmation). The component supports cursor management, keyboard navigation, paste sanitization, and password masking. Multi-line terminal rendering capabilities exist at the Ink vendor layer (2273.js) via diff-based renderers (`GDf`, `FDf`), but the application-level input widget is fundamentally single-line: typed newlines are stripped, though pasted content may retain them.

## Module Map

| Module   | Size | Role                              | Vendor/App |
|---------|-------|----------------------------------|------------|
| 3861.js | ~350 lines | `C8` TextInput component: core input widget | App |
| 3863.js | 304 lines | `CfA` AskUser controller: uses C8 in text-input mode | App |
| 3860.js | ~60 lines | `JfA` / `wfA` AskUser UI helpers (breadcrumb + options) | App |
| 3864.js | ~80 lines | `PiD` AskUser read-only renderer | App |
| 2272.js | ~60 lines | Ink cursor ANSI utilities (`zMH`, `ea`, `rdA`) | Vendor |
| 2273.js | ~200 lines | Ink text renderers (`GDf`, `FDf`, `QDf`) with multi-line support | Vendor |

## Architecture

```
┌─────────────────────────────────────────┐
│  App Layer (Dialog Controllers)         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ 3863.js │ │ 3869.js │ │ 3871.js │   │
│  │ AskUser │ │Proposal │ │  Plan   │   │
│  │Controller│ │Confirm │ │Confirm │   │
│  └────┬────┘ └────┬────┘ └────┬────┘   │
│       │           │           │         │
│       └───────────┼───────────┘         │
│                   ▼                     │
│            ┌─────────────┐              │
│            │   3861.js   │              │
│            │  C8 TextInput│              │
│            └──────┬──────┘              │
│                   │                     │
│            ┌──────┴──────┐              │
│            │ 3860.js     │              │
│            │ JfA / wfA   │              │
│            │ UI helpers  │              │
│            └─────────────┘              │
└─────────────────────────────────────────┘
                   │
┌──────────────────┼──────────────────────┐
│  Ink Vendor Layer│                      │
│  ┌───────────────┴───────────────┐      │
│  │      2272.js (cursor)         │      │
│  │  zMH.show/hide/toggle         │      │
│  │  ea(cursorPosition)           │      │
│  └───────────────┬───────────────┘      │
│                  │                      │
│  ┌───────────────┴───────────────┐      │
│  │      2273.js (renderer)       │      │
│  │  GDf: full rewrite           │      │
│  │  FDf: diff-based incremental │      │
│  │  QDf: mode selector          │      │
│  └───────────────────────────────┘      │
└─────────────────────────────────────────┘
```

## Key Components

### 1. C8 TextInput (3861.js, line 70–340)

The core text-input React component. Props:

```js
function C8({
  value: H,           // current text value
  onChange: A,        // callback(value)
  onSubmit: L,        // callback(value) on Return
  placeholder: $ = "",
  focus: I = true,    // whether input is active
  showCursor: D = true,
  mask: E,            // optional mask char (password mode)
}) {
  let [f, M] = g4.useState(H.length),  // cursor offset state
    U = g4.useRef(f),                  // cursor ref
    P = g4.useRef(H),                  // value ref
    B = g4.useRef(A),                  // onChange ref
    W = g4.useRef(L),                  // onSubmit ref
    V = Math.min(f, H.length);         // clamped cursor
```

**Cursor rendering** uses ANSI inverse video:

```js
let N = E ? E.repeat(H.length) : H,   // apply mask if provided
    x = N,
    v = $ ? q$.grey($) : void 0;

if (D && I) {
  // Placeholder with inverse first char
  v = $.length > 0 ? q$.inverse($[0]) + q$.grey($.slice(1)) : q$.inverse(" ");
  if (N.length > 0) {
    x = "";
    for (let g = 0; g < N.length; g++)
      x += g === V ? q$.inverse(N[g]) : N[g];
    if (V === N.length) x += q$.inverse(" ");  // cursor at end
  } else {
    x = q$.inverse(" ");
  }
}
return _iD.jsxDEV(u, { children: N.length > 0 ? x : v }, ...);
```

**Keyboard handler** (`C` callback, lines 120–220) covers:
- **Paste**: `fiD(UiD(DH))` sanitizes pasted text (allows tabs/newlines); `MiD` strips control chars for normal typing
- **Return**: fires `onSubmit`
- **Escape / Tab / Up / Down**: ignored at this level (handled by parent)
- **Ctrl+W**: delete word backward (`KfA`)
- **Ctrl+U**: delete to start (`DiD`)
- **Ctrl+V** (`\v`): delete to end (`EiD`)
- **Ctrl+A**: cursor to start (`w(0)`)
- **Ctrl+E**: cursor to end (`w(e.length)`)
- **Ctrl+←/→**: word jump (`JTL` / `wTL`)
- **Ctrl+Backspace / Ctrl+Delete**: word delete
- **Meta+←/→/Backspace/Delete**: same as Ctrl variants
- **Home / End** (`PyM = "\x1B[H"`, `ByM = "\x1B[F"`): cursor to start/end
- **← / →**: single char move (only when `showCursor`)
- **Backspace / Delete**: single char delete
- **Normal chars**: inserted via `MiD` sanitizer (filters out `\t`, `\n`, `\r`)

**Text manipulation helpers** (lines 10–65):

```js
function KfA(H, A) {   // delete word backward
  let L = A - 1;
  while (L > 0 && /\s/.test(H[L - 1])) L--;
  while (L > 0 && !/\s/.test(H[L - 1])) L--;
  return { value: H.slice(0, L) + H.slice(A), cursorOffset: L };
}

function QTL(H, A) {   // delete word forward
  let L = A;
  while (L < H.length && /\s/.test(H[L])) L++;
  while (L < H.length && !/\s/.test(H[L])) L++;
  return { value: H.slice(0, A) + H.slice(L), cursorOffset: A };
}

function fiD(H) {      // paste sanitizer: allows tabs & newlines
  let A = "";
  for (let L of H) {
    let $ = L.charCodeAt(0);
    if ($ === 9 || $ === 10 || $ === 13 || ($ >= 32 && $ !== 127)) A += L;
  }
  return A;
}

function MiD(H) {      // type sanitizer: strips control chars
  let A = "";
  for (let L of H) {
    let $ = L.charCodeAt(0);
    if ($ >= 32 && $ !== 127) A += L;
  }
  return A;
}

function UiD(H) {      // normalize line endings to \n
  return H.replace(/\r\n/g, "\n").replace(/\r/g, "\n")
          .replace(/\u2028/g, "\n").replace(/\u2029/g, "\n");
}
```

### 2. AskUser Controller (3863.js, lines 1–304)

Manages a questionnaire state machine where each question can be answered by selecting an option or typing free text.

```js
function CfA({
  questions: H,
  parseError: A,
  onComplete: L,
  onCancel: $,
  isFocused: I = true,
  width: D = 95,
  questionIndex: E,
  onQuestionIndexChange: f,
  answerStates: M,
  onAnswerStatesChange: U,
}) {
  // Per-question state: { selectedIndex, ownAnswer, isTextInputMode, submittedAnswer }
  let N = IJ.useMemo(() =>
    C[Q] || { selectedIndex: 0, ownAnswer: "", isTextInputMode: false }, ...);
```

**Mode switching logic** (lines 150–175):

```js
let VH = IJ.useCallback((HH, PH) => {
  if (!q) return;
  let EH = q.options.length - 1;
  if (PH.upArrow) {
    e({ selectedIndex: x > 0 ? x - 1 : EH });
    return;
  }
  if (PH.downArrow) {
    if (x === EH) e({ isTextInputMode: true });   // last option → text mode
    else e({ selectedIndex: x + 1 });
    return;
  }
  if (PH.return) {
    l(q.options[x]);                               // submit selected option
    return;
  }
  if (HH && !PH.upArrow && !PH.downArrow && !PH.return)
    e({ isTextInputMode: true, ownAnswer: HH });   // any char → text mode
}, ...);
```

**C8 integration** (lines 235–260):

```js
CF.jsxDEV(C8, {
  focus: g,                    // g = isTextInputMode
  showCursor: true,
  value: v,                    // v = ownAnswer
  onChange: (HH) => e({ ownAnswer: HH }),
  onSubmit: IH,                // IH = submit handler for free text
  placeholder: "Or type your own answer...",
}, ...);
```

### 3. AskUser UI Helpers (3860.js)

**`JfA`**: breadcrumb trail showing question topics with completion status:

```js
function JfA({ questions, questionIndex, answerStates }) {
  return <Box flexDirection="row">
    {questions.map((q, i) => {
      let done = !!answerStates[i]?.submittedAnswer;
      let active = i === questionIndex;
      let color = active ? n8.primary : (done ? n8.success : n8.text.muted);
      return <Text color={color} bold={active}>{q.topic}</Text>;
    })}
  </Box>;
}
```

**`wfA`**: options list with `isTextInputMode` flag:

```js
function wfA({ options, selectedIndex, isTextInputMode, children }) {
  return <Box marginTop={1} flexDirection="column">
    {options.map((opt, i) => {
      let active = !isTextInputMode && i === selectedIndex;
      return <Text color={active ? n8.primary : n8.text.muted}>
        {active ? "> " : "  "}{opt}
      </Text>;
    })}
    {children}   // C8 TextInput slot
  </Box>;
}
```

### 4. Ink Cursor Utilities (2272.js)

Low-level ANSI cursor control used by Ink's renderer:

```js
zMH.show = (H = LKI.stderr) => { if (H.isTTY) { trH = false; H.write("\x1B[?25h"); } };
zMH.hide = (H = LKI.stderr) => { if (H.isTTY) { HKI(); trH = true; H.write("\x1B[?25l"); } };
zMH.toggle = (H, A) => { if (H !== void 0) trH = H; trH ? zMH.show(A) : zMH.hide(A); };

// Position cursor at (x, y) relative to current output block
ea = (H, A) => {
  let L = H - A.y;
  return (L > 0 ? dB.cursorUp(L) : "") + dB.cursorTo(A.x) + "\x1B[?25h";
};

// Render diff prefix: hide old cursor, move to start of previous block
rdA = (H) => {
  let A = H.cursorWasShown ? "\x1B[?25l" : "",
      L = $KI(H.previousLineCount, H.previousCursorPosition),
      $ = ea(H.visibleLineCount, H.cursorPosition);
  return A + L + $;
};
```

### 5. Ink Text Renderers (2273.js)

Three renderer strategies selected by `QDf`:

```js
QDf = (H, { showCursor, incremental } = {}) => {
  if (incremental) return FDf(H, { showCursor });   // diff-based
  return GDf(H, { showCursor });                     // full rewrite
}
```

**`GDf`** (full rewrite, lines 10–70):
- Tracks `previousLineCount`, `previousCursorPosition`, `visibleLineCount`, `cursorPosition`
- On every render: hides cursor → erases previous lines → writes new text → positions cursor
- Uses `w7H` to hide cursor and move to bottom of previous output, then `dB.eraseLines(L)` to clear

**`FDf`** (diff-based incremental, lines 75–150):
- Compares line-by-line between previous (`L`) and current (`Q`) output arrays
- Reuses identical lines (just moves cursor down)
- Replaces changed lines with `cursorTo(0) + newLine + eraseEndLine`
- Efficient for large multi-line outputs where only a few lines change

Both renderers support `showCursor` flag and track cursor via `{x, y}` objects.

## Theme / Style Tokens

- `n8.primary`: active selection / cursor highlight color
- `n8.text.muted`: inactive / placeholder text
- `n8.success`: completed question indicator
- `n8.error`: parse error messages
- `q$.inverse(char)`: cursor rendering (ANSI inverse video)
- `q$.grey(text)`: placeholder styling

## Keyboard / Input Handling

| Key / Sequence          | Action in C8 (`showCursor=true`)     | Action in AskUser (option mode) |
|-------------------------|---------------------------------------|---------------------------------|
| `↑`                     | ignored                               | previous option                 |
| `↓`                     | ignored                               | next option / enter text mode   |
| `Return`                | `onSubmit(value)`                     | submit selected option          |
| `Tab`                   | ignored                               | next question                   |
| `Shift+Tab`             | ignored                               | previous question               |
| `Escape`                | ignored                               | cancel dialog                   |
| `←` / `→`               | cursor -1 / +1                        |:                               |
| `Ctrl+←` / `Ctrl+→`     | word jump                             |:                               |
| `Backspace`             | delete char before cursor             |:                               |
| `Delete`                | delete char at cursor                 |:                               |
| `Ctrl+W`                | delete word backward                  |:                               |
| `Ctrl+U`                | delete to start                       |:                               |
| `Ctrl+V` (`\v`)         | delete to end                         |:                               |
| `Ctrl+A`                | cursor to start                       |:                               |
| `Ctrl+E`                | cursor to end                         |:                               |
| `Home` (`\x1B[H`)       | cursor to start                       |:                               |
| `End` (`\x1B[F`)        | cursor to end                         |:                               |
| `Paste`                 | insert sanitized text (allows `\n`)   |:                               |
| Any printable char      | insert char (strips `\t`, `\n`)       | switch to text mode + append    |

**Important**: C8 is fundamentally single-line for typed input. Newlines are stripped by `MiD` during normal typing, but `fiD` allows them during paste. The Ink `<Text>` primitive (module 2273.js) handles multi-line *display* at the terminal level, but C8 itself does not implement per-line cursor coordinates.

## Integration Points

- **3863.js → 3861.js**: AskUser controller renders `C8` when `isTextInputMode === true`
- **3863.js → 3860.js**: Uses `JfA` (breadcrumb) and `wfA` (options list) as child components
- **3864.js → 3860.js**: Read-only AskUser renderer also uses `JfA` and `wfA`
- **3861.js → 2272.js / 2273.js**: C8 uses Ink's `useStdin` context (`fm`) and `<Text>` primitive, which internally relies on the cursor/renderer stack
- **3869.js / 3871.js → 3861.js**: Proposal and Plan confirmation dialogs also consume `C8` for comment input
- **Cross-feature**: `jL` (useInput hook) from `tui-ink-core` is used alongside C8's own stdin subscription for global key handling

## Implementation Notes

1. **TextInput Component** (`C8` pattern):
   - Implement as React functional component with `value`, `onChange`, `onSubmit`, `placeholder`, `focus`, `showCursor`, `mask` props
   - Maintain cursor offset in state + refs for latest callbacks
   - Render cursor with ANSI inverse video (`chalk.inverse` or `ansiStyles.inverse`)
   - Subscribe to stdin context in `useLayoutEffect`; unsubscribe on cleanup

2. **Sanitization**:
   - Paste: allow newlines/tabs but strip other control chars (`fiD` equivalent)
   - Typing: strip all control chars including newlines (`MiD` equivalent)
   - Normalize line endings to `\n` (`UiD` equivalent)

3. **Keyboard Map**:
   - Implement word-boundary helpers (`KfA`, `QTL`, `JTL`, `wTL`) using `/\s/` regex
   - Handle `\x1B[H` (Home) and `\x1B[F` (End) sequences
   - Distinguish `isPaste` flag from normal key events

4. **Masking**:
   - Simple: `maskChar.repeat(value.length)`; cursor offset clamped to masked string length

5. **Multi-line Composer (if needed)**:
   - C8 does NOT provide a true multi-line editor. If OpenDroid needs one, extend C8 with:
     - Per-line cursor `{line, column}` instead of flat offset
     - Line splitting on `\n`
     - Up/Down arrow handling for line navigation
   - Reuse Ink's `FDf` diff renderer (2273.js) for efficient multi-line terminal updates

6. **Dialog Integration**:
   - Follow AskUser pattern: maintain per-field state `{selectedIndex, ownAnswer, isTextInputMode, submittedAnswer}`
   - Down-arrow on last option enters text mode; any typed character also enters text mode
   - Tab cycles questions; Escape cancels

## Module Reference

| Module   | Lines   | Content                                    |
|---------|------------|-------------------------------------------|
| 3861.js | 10–65      | Text manipulation helpers (`KfA`, `QTL`, `fiD`, `MiD`, `UiD`) |
| 3861.js | 70–340     | `C8` TextInput component (props, state, keyboard, render) |
| 3863.js | 1–100      | `CfA` AskUser controller: state init, submission flow |
| 3863.js | 100–180    | Keyboard routing: Tab/Escape/Arrow/Return handlers |
| 3863.js | 180–260    | JSX tree: `JfA`, question text, `wfA`, `C8`, hint bar |
| 3860.js | 1–60       | `JfA` breadcrumb + `wfA` options list helpers |
| 3864.js | 1–80       | `PiD` read-only AskUser renderer |
| 2272.js | 1–60       | `zMH` cursor show/hide/toggle, `ea` position, `rdA` diff |
| 2273.js | 1–70       | `GDf` full-rewrite renderer |
| 2273.js | 75–150     | `FDf` incremental diff renderer |
| 2273.js | 192–198    | `QDf` renderer mode selector |
