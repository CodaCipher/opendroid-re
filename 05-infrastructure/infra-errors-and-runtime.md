# infra-errors-and-runtime: Architecture Notes

## Overview

OpenDroid implements a layered error class hierarchy rooted in a custom `MetaError` (aliased as `vH`) base class that extends native `Error` with metadata support, cause chaining, and prototype fixup. On top of this base, domain-specific error classes like `FetchError`, `RipgrepError`, `SessionNotFoundError`, `ResponseError`, `InterceptorConfigurationError`, `McpError`, `EmptyLLMResponseError`, and `ToolAbortError` provide structured error handling across subsystems. Runtime helper utilities provide safe error extraction (`getErrorMessage`), error stack formatting (`ERH`, `fR`), sensitive data redaction (`Pf`), and error-aware logging functions (`uH`, `gH`, `BH`). The system also includes polyfill-style compatibility patterns for older Node.js environments via `Object.setPrototypeOf` fixup and `Error.captureStackTrace`.

## Module Map

| Module | Size (lines) | Role | Vendor/App |
|--------|-------------|------|------------|
| 0008.js | ~38 | Core error class hierarchy: MetaError, FetchError, NetworkError, ToolAbortError + type guards | App |
| 0011.js | ~24 | ResponseError class (HTTP status + metadata) | App |
| 0093.js | ~119 | Error logging/reporting: `uH` (error), `gH` (warn), `BH` (info) + ErrTracker integration | App |
| 0948.js | ~80 | Domain errors: EmptyLLMResponseError, VSCodeCliNotAvailableError, RipgrepError, WindowsPowerShellNotFoundError, SessionNotFoundError | App |
| 0992.js | ~460+ | McpError (MCP protocol errors with error codes) | App |
| 2606.js | ~361 | gRPC interceptor errors: InterceptorConfigurationError + InterceptingCall + ListenerBuilder/RequesterBuilder | Vendor (gRPC) |
| 2769.js | ~293 | Google API retry/settings: CallSettings, RetryOptions, backoff configuration, validation errors | Vendor (google-gax) |
| 3686.js | ~252 | Zip archive module (archiver): compression method errors, CRC32 streams | Vendor (archiver) |
| 3882.js | ~330 | BlogPostService: error state management, `instanceof Error` normalization, generation error handling | App |
| 3849.js | ~210 | CLI status command: error-safe status display, command execution with error handling | App |
| 4017.js | ~648 | Session browser UI (React): Ink/TUI component for session selection/renaming | App |
| 1883.js | ~220 | OpenTelemetry PG instrumentation: `getErrorMessage` utility for span error reporting | Vendor (@opentelemetry/instrumentation-pg) |
| 0018.js | ~84 | Sensitive data redaction: `Pf()` redacts secrets/tokens/keys from strings | App |
| 0019.js | ~30 | Runtime sanitization: `WNH()` normalizes values for safe logging, secret detection (`PNH`) | App |
| 0089.js | ~5079 | Runtime helpers: `ERH` (extract stack), `fRH` (extract cause), `fR` (format error summary) | App |

## Architecture

### Error Class Hierarchy

```
Error (native)
├── MetaError (vH)                          [0008.js]
│   ├── .metadata (arbitrary key-value)
│   ├── .name = "MetaError"
│   ├── constructor(msg, { cause, ...metadata })
│   ├── Object.setPrototypeOf fixup
│   │
│   ├── FetchError                          [0008.js]
│   │   ├── .response (Response object)
│   │   ├── .name = "FetchError"
│   │   └── constructor(msg, response)
│   │
│   ├── RipgrepError                        [0948.js]
│   │   ├── .exitCode, .stderr
│   │   ├── .name = "RipgrepError"
│   │   └── constructor(msg, exitCode, stderr, cause)
│   │
│   └── SessionNotFoundError (Wc)           [0948.js]
│       ├── .name = "SessionNotFoundError"
│       └── constructor(metadata)
│
├── NetworkError                            [0008.js]
│   └── .name = "NetworkError"
│
├── ToolAbortError (II)                     [0008.js]
│   └── .name = "ToolAbortError"
│   └── default msg: "Tool execution timed out or was cancelled"
│
├── ResponseError (AK)                      [0011.js]
│   ├── .statusCode, .metadata
│   ├── .name = "ResponseError"
│   ├── .toPublicObject() → { detail, status, title }
│   └── .toDebugObject() → { ...public, cause, ...metadata }
│
├── InterceptorConfigurationError           [2606.js]
│   ├── .name = "InterceptorConfigurationError"
│   └── Error.captureStackTrace
│
├── McpError                                [0992.js]
│   ├── .code, .data
│   ├── .name = "McpError"
│   └── constructor(code, message, data)
│
├── EmptyLLMResponseError                   [0948.js]
│   └── .name = "EmptyLLMResponseError"
│
├── VSCodeCliNotAvailableError              [0948.js]
│   └── .name = "VSCodeCliNotAvailableError"
│
├── WindowsPowerShellNotFoundError          [0948.js]
│   └── .name = "WindowsPowerShellNotFoundError"
│
└── (various vendor error classes)
    ├── CallSettings validation errors      [2769.js]
    └── Zip compression errors             [3686.js]
```

### Error Handling Patterns

1. **Error normalization**: Every catch block normalizes non-Error values: `U instanceof Error ? U : Error(String(U))` (3882.js)
2. **Metadata propagation**: `MetaError` carries structured metadata through the error chain, enabling machine-readable error contexts
3. **Cause chaining**: Uses `{ cause }` option in `super()` to preserve causal chains per ES2022 spec
4. **Prototype fixup**: `Object.setPrototypeOf(this, ClassName.prototype)` ensures `instanceof` works correctly in transpiled ES5 classes
5. **Sensitive data redaction**: All error messages/stacks pass through `Pf()` (0018.js) which redacts API keys, tokens, and secrets using regex patterns

### Runtime Helper Functions

| Function | Module | Purpose |
|----------|--------|---------|
| `ERH(H)` | 0089.js:5026 | Extract stack trace from Error or Error-like object |
| `fRH(H)` | 0089.js:5031 | Extract `.cause` from Error (for chain walking) |
| `fR(H)` | 0089.js:5035 | Format error as `"Name: message"` string |
| `Pf(H)` | 0018.js:44 | Redact secrets/tokens/keys from strings (regex-based) |
| `WNH(H)` | 0019.js:6 | Normalize values for safe logging (numbers→strings, arrays→mapped, objects→JSON) |
| `getErrorMessage(Xv1)` | 1883.js:214 | Extract `.message` from Error-like objects safely |
| `Xn(H)` | 0008.js:35 | Type guard: `isFetchError` (Error with `.response instanceof Response`) |
| `AQ(H)` | 0008.js:39 | Type guard: `isNetworkError` (TypeError with network messages or NetworkError) |
| `uH(H, A, L, $)` | 0093.js:38 | Error logger: extracts cause chain, redacts, reports to ErrTracker |
| `gH(H, A, L)` | 0093.js:67 | Warning logger with ErrTracker breadcrumb |
| `BH(H, A, L)` | 0093.js:63 | Info logger with ErrTracker breadcrumb |
| `_RH(H)` | 0093.js:14 | Sanitize error context object for logging (redact body/error/messages) |
| `PNH(H)` | 0019.js:16 | Detect if a string looks like a secret (entropy check + pattern match) |

### Polyfill / Compatibility Patterns

| Pattern | Module | Purpose |
|---------|--------|---------|
| `Object.setPrototypeOf(this, vH.prototype)` | 0008.js | Fix `instanceof` for transpiled class hierarchy |
| `Error.captureStackTrace(this, ClassName)` | 2606.js | V8 stack trace capture (Chrome/Node specific) |
| `Object.defineProperty(exports, "__esModule", { value: true })` | 2606.js, 2769.js, 1883.js | CJS/ESM interop marker |
| `util.inherits($3, IvD)` | 3686.js | Node.js prototypal inheritance (archiver) |

## Key Findings

1. **MetaError (`vH`) is the central error base class.** It extends native Error with metadata, cause chaining, and prototype fixup. At least 3 domain errors subclass it (FetchError, RipgrepError, SessionNotFoundError). The obfuscated name `vH` is the internal reference throughout the codebase.

2. **ResponseError (`AK`) is the HTTP-layer error class.** It carries HTTP status codes and structured metadata, with `toPublicObject()` and `toDebugObject()` for safe error serialization in API responses.

3. **Error-aware logging trio (`uH`/`gH`/`BH`) provides unified error reporting.** The `uH` function (error logger) walks the cause chain up to 3 levels, extracts and redacts metadata, and reports to ErrTracker with scope tags. This is the primary error reporting path.

4. **Sensitive data redaction (`Pf`) is deeply integrated.** Every error message and stack trace passes through the redaction system before logging. It uses entropy detection (`PNH`) to identify base64 strings, hex tokens, and credential patterns.

5. **Error normalization is a codebase-wide pattern.** Every catch block wraps non-Error values with `instanceof Error ? X : Error(String(X))`, ensuring downstream error handlers always deal with proper Error instances.

6. **gRPC interceptor errors use `Error.captureStackTrace`.** The `InterceptorConfigurationError` class (2606.js) captures V8-specific stack traces, indicating the system targets Node.js runtime environments.

7. **ToolAbortError signals tool execution cancellation.** It represents a specific cancellation case distinct from network or response errors, used when tools are aborted during execution.

## Code Examples

### MetaError base class with metadata support (0008.js, line 3-13)
```js
vH = class vH extends Error {
  metadata;
  constructor(H, A) {
    let { cause: L, ...$ } = A || {};
    super(H, { cause: L });
    Object.setPrototypeOf(this, vH.prototype);
    this.name = "MetaError";
    this.metadata = Object.keys($).length > 0 ? $ : void 0;
  }
};
```

### ResponseError with HTTP status (0011.js, line 5-22)
```js
AK = class AK extends Error {
  statusCode; metadata;
  constructor(H, A, L) {
    super(H);
    this.name = "ResponseError";
    this.metadata = L;
    this.statusCode = A;
    Object.setPrototypeOf(this, AK.prototype);
  }
  toPublicObject() {
    return { detail: this.message, status: this.statusCode, title: J3L[this.statusCode] };
  }
};
```

### Error-aware logging with cause chain walking (0093.js, line 38-62)
```js
function uH(H, A, L, $) {
  let I = Pf(A), D = L || {};
  if ((H instanceof vH || H instanceof AK) && H.metadata) D = { ...H.metadata, ...D };
  let E = `${I}\n${Pf(ERH(H))}`, f = fRH(H) || L?.cause;
  if (f) D.cause = f;
  let M = 0;
  while (f && M < 3) {
    E += `\nCaused by:\n${Pf(ERH(f))}`;
    if ((f instanceof vH || f instanceof AK) && f.metadata)
      D = { ...f.metadata, ...D };
    f = fRH(f); M++;
  }
  // ... ErrTracker reporting ...
}
```

### InterceptorConfigurationError (2606.js, line 14-20)
```js
class InterceptorConfigurationError extends Error {
  constructor(H) {
    super(H);
    this.name = "InterceptorConfigurationError";
    Error.captureStackTrace(this, InterceptorConfigurationError);
  }
}
```

### Error normalization pattern (3882.js, line 71, 103, 175, 285)
```js
// Consistent pattern throughout:
let P = U instanceof Error ? U : Error(String(U));
```

### Type guards for error classification (0008.js, line 35-48)
```js
function Xn(H) {
  return H instanceof Error && "response" in H && H.response instanceof Response;
}
function AQ(H) {
  return (H instanceof TypeError &&
    (H.message.includes("network error") || H.message.includes("Failed to fetch"))) ||
    H instanceof NetworkError;
}
```

### getErrorMessage utility (1883.js, line 211-213)
```js
function Xv1(H) {
  return typeof H === "object" && H !== null && "message" in H ? String(H.message) : void 0;
}
```

## Integration Points (cross-system)

- **Mission system (M2):** `ToolAbortError` (0008.js) is the cancellation mechanism for tool execution during missions. Session errors (`SessionNotFoundError`, 0948.js) flow into mission state management.
- **Tool system (M3):** `McpError` (0992.js) wraps MCP protocol errors that originate from tool invocations. Error metadata flows into tool result reporting.
- **TUI (M1):** `uH`/`gH`/`BH` logging functions (0093.js) feed error display in the terminal UI. The status command (3849.js) uses error-safe patterns for CLI output.
- **HTTP/Network layer:** `FetchError` and `NetworkError` (0008.js) are used throughout the HTTP client subsystem. `ResponseError` (0011.js) wraps all API responses with error status codes.
- **ErrTracker integration:** Error logger `uH` (0093.js) sends structured exceptions with cause chains and metadata to ErrTracker. `gH`/`BH` add breadcrumbs for error context reconstruction.
- **OpenTelemetry:** `getErrorMessage` (1883.js) is used in OTEL span error status reporting, bridging error handling and telemetry.

## Implementation Notes

1. **Preserve the MetaError hierarchy.** The `vH` → `FetchError`/`RipgrepError`/`SessionNotFoundError` hierarchy should be ported as a proper `BaseError` abstract class with typed metadata generics.
2. **Port error normalization as a utility.** The `instanceof Error ? X : Error(String(X))` pattern should be encapsulated in a `normalizeError()` helper function.
3. **Preserve sensitive data redaction.** The `Pf()`/`WNH()`/`PNH()` redaction system should be ported as a dedicated `redact()` module with configurable patterns.
4. **Unify error logging.** The `uH`/`gH`/`BH` trio should become a structured logger with pluggable transports (console, ErrTracker, custom).
5. **Type guards should become proper type predicates.** `Xn()` → `isFetchError()`, `AQ()` → `isNetworkError()` with TypeScript type predicates.
6. **Cause chain walking** (the 3-level loop in `uH`) should be extracted as a reusable `walkCauseChain()` generator function.

## Module Reference (file + line + function)

| Module | Line | Function/Class | Description |
|--------|------|----------------|-------------|
| 0008.js | 3 | `vH` (MetaError) | Base error class with metadata and cause |
| 0008.js | 14 | `FetchError` | HTTP fetch error with response object |
| 0008.js | 21 | `NetworkError` | Network-level error |
| 0008.js | 27 | `II` (ToolAbortError) | Tool execution cancellation error |
| 0008.js | 35 | `Xn()` | Type guard: isFetchError |
| 0008.js | 39 | `AQ()` | Type guard: isNetworkError |
| 0011.js | 5 | `AK` (ResponseError) | HTTP response error with status code |
| 0018.js | 44 | `Pf()` | Sensitive data redaction |
| 0019.js | 6 | `WNH()` | Value normalization for logging |
| 0019.js | 16 | `PNH()` | Secret string detection (entropy check) |
| 0089.js | 5026 | `ERH()` | Extract stack trace from error |
| 0089.js | 5031 | `fRH()` | Extract cause from error |
| 0089.js | 5035 | `fR()` | Format error as "Name: message" |
| 0093.js | 14 | `_RH()` | Sanitize error context for logging |
| 0093.js | 38 | `uH()` | Error logger (with ErrTracker) |
| 0093.js | 63 | `BH()` | Info logger (with breadcrumbs) |
| 0093.js | 67 | `gH()` | Warning logger (with breadcrumbs) |
| 0948.js | 4 | `EmptyLLMResponseError` | Empty LLM response error |
| 0948.js | 10 | `VSCodeCliNotAvailableError` | VSCode CLI not found |
| 0948.js | 17 | `RipgrepError` | Ripgrep execution error |
| 0948.js | 29 | `WindowsPowerShellNotFoundError` | PowerShell not available |
| 0948.js | 46 | `Wc` (SessionNotFoundError) | Session lookup failure |
| 0992.js | 461 | `McpError` | MCP protocol error with code |
| 1883.js | 211 | `Xv1` (getErrorMessage) | Safe error message extraction |
| 2606.js | 14 | `InterceptorConfigurationError` | gRPC interceptor config error |
| 2769.js | 91 | validation | Retry/retryRequestOptions mutual exclusion |
| 3686.js | 53 | error callback | Compression method not implemented |
| 3882.js | 44 | `getError()` | BlogPostService error state getter |
| 3882.js | 71 | error normalization | `instanceof Error` normalization pattern |
| 3849.js | 95 | error handling | CLI status command error handling |
