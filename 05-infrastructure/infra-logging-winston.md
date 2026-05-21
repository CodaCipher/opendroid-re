# infra-logging-winston: Architecture Notes

## Overview

OpenDroid employs a **multi-layered logging architecture** composed of three distinct logging subsystems rather than a single winston instance: (1) **OpenTelemetry DiagAPI**: the primary diagnostic/structured logging framework used across the application, providing namespaced component loggers with configurable log levels (VERBOSE, DEBUG, INFO, WARN, ERROR); (2) **gRPC internal logging**: a dedicated logging layer for gRPC transport with `LogVerbosity` levels (DEBUG, INFO, ERROR, NONE) controlled via `GRPC_NODE_VERBOSITY`/`GRPC_VERBOSITY` environment variables; and (3) **ErrTracker Winston Transport**: a custom winston transport that bridges winston log entries into ErrTracker error tracking with level mapping and structured metadata. The system does NOT use a traditional standalone winston logger with file rotating transports; instead, winston appears only as a ErrTracker integration bridge. Structured logging is achieved through OTel's component logger pattern, and log output goes to console (via `DiagConsoleLogger`).

## Module Map

| Module | Lines | Role | Vendor/App |
|--------|-------|------|------------|
| 0121.js | ~30 | DiagComponentLogger: OTel component logger with namespace support | Vendor (OTel API) |
| 0124.js | ~60 | DiagAPI: Core OTel diagnostic API (setLogger, createComponentLogger, log methods) | Vendor (OTel API) |
| 0129.js | ~27 | DiagConsoleLogger: Console-backed diagnostic logger (error/warn/info/debug/verbose) | Vendor (OTel API) |
| 0151.js | ~9 | diag singleton: exports `DiagAPI.instance()` | Vendor (OTel API) |
| 0192.js | ~28 | diagLogLevelFromString: String-to-DiagLogLevel mapping (ALL/VERBOSE/DEBUG/INFO/WARN/ERROR/NONE) | Vendor (OTel API) |
| 0800.js | ~90 | OTLPExportDelegate: OTLP exporter with _diagLogger for structured export diagnostics | Vendor (OTel Exporter) |
| 1547.js | ~48 | ErrTracker Winston Transport: bridges winston log entries to ErrTracker capture (ak function) | Vendor (ErrTracker) |
| 1655.js | ~200 | OTel Fastify Instrumentation: uses _logger (diag.createComponentLogger) for HTTP request tracing | Vendor (OTel Instrumentation) |
| 1570.js | ~100 | ErrTracker SDK init: debug mode toggle, console.warn for warnings | Vendor (ErrTracker) |
| 2583.js | ~50 | gRPC LogVerbosity enum: Status codes + LogVerbosity (DEBUG/INFO/NONE) | Vendor (gRPC) |
| 2585.js | ~100 | gRPC logging module: trace(), log(), getLogger(), setLogger(), GRPC_NODE_VERBOSITY/TRACE env vars | Vendor (gRPC) |
| 2592.js | ~333 | gRPC ChannelCredentials: uses LogVerbosity.ERROR for TLS error reporting | Vendor (gRPC) |
| 2685.js | ~623 | gRPC ServerInterceptingCall: uses LogVerbosity.DEBUG for server call tracing | Vendor (gRPC) |
| 0104.js | ~542 | OTel Semantic Resource Attributes: defines AWS log group/stream ARN constants | Vendor (OTel SDK) |
| 2561.js | ~343 | Evaluation report template: references winston/pino/bunyan as detection criteria | App |
| 1194.js | ~large | Repository evaluation criteria: checks for winston/pino in dependencies | App |

## Architecture

### Layer 1: OpenTelemetry DiagAPI (Primary Logging)

The primary logging system uses OpenTelemetry's `DiagAPI` singleton (module 0124.js, 0151.js). The architecture follows this pattern:

1. **Global Registration**: `DiagAPI.instance()` creates a singleton with methods `setLogger()`, `createComponentLogger()`, `verbose()`, `debug()`, `info()`, `warn()`, `error()`
2. **Component Logger**: `createComponentLogger({ namespace })` creates a `DiagComponentLogger` (0121.js) that prefixes all messages with the namespace and delegates to the global `diag` instance
3. **Console Backend**: By default, `DiagConsoleLogger` (0129.js) maps log levels to `console.error`, `console.warn`, `console.info`, `console.debug`, `console.trace`
4. **Level Filtering**: `diagLogLevelFromString` (0192.js) converts string levels → numeric `DiagLogLevel` enum (ALL/VERBOSE/DEBUG/INFO/WARN/ERROR/NONE)

### Layer 2: gRPC Internal Logging

gRPC has its own isolated logging system (2585.js):
- **Levels**: `LogVerbosity` enum (DEBUG=0, INFO=1, ERROR=2, NONE=3) defined in 2583.js
- **Control**: `GRPC_NODE_VERBOSITY` or `GRPC_VERBOSITY` env vars
- **Trace**: `GRPC_NODE_TRACE` or `GRPC_TRACE` env vars for per-component trace filtering
- **Output**: Console-based with format `"E/I/D {message}"` prefix
- **API**: `getLogger()`, `setLogger()`, `setLoggerVerbosity()`, `trace()`, `log()`

### Layer 3: ErrTracker Winston Bridge

The ErrTracker SDK provides a winston transport (1547.js) that:
- Extends a winston Transport class
- Overrides `log()` method to capture winston log entries
- Extracts `level`, `message`, `timestamp` from winston info objects
- Maps numeric winston levels to ErrTracker levels (`info` default)
- Passes remaining metadata as ErrTracker context (`errtracker.origin: "auto.log.winston"`)

### Log Flow

```
Application Code
  ├── OTel Component Logger → DiagAPI → DiagConsoleLogger → console.error/warn/info/debug/trace
  ├── gRPC trace() → LogVerbosity filter → console.error("E/I/D" + message)
  └── winston Logger → ErrTracker Winston Transport → ErrTracker.captureMessage/captureException
```

## Key Findings

1. **No standalone winston logger**: Despite the feature name, the codebase does NOT contain a standalone winston logger with file transports, rotation, or structured JSON file output. Winston appears only as a ErrTracker integration transport (module 1547.js).

2. **OTel DiagAPI is the primary logger**: The application uses OpenTelemetry's DiagAPI as its primary diagnostic logging framework. Component loggers are created with `diag.createComponentLogger({ namespace })` and used throughout OTel instrumentation, exporters, and Fastify integration.

3. **gRPC has independent logging**: gRPC's logging is completely isolated, controlled by `GRPC_NODE_VERBOSITY`/`GRPC_TRACE` env vars, with its own level enum (`LogVerbosity`).

4. **DiagLogLevel hierarchy**: The OTel DiagLogLevel enum has 7 levels: ALL → VERBOSE → DEBUG → INFO → WARN → ERROR → NONE. The default is `INFO`. String-to-level conversion in module 0192.js provides safe fallback.

5. **ErrTracker Winston Transport level mapping**: The winston-to-ErrTracker bridge (1547.js) maps winston numeric levels to ErrTracker string levels with `"info"` as default fallback. It strips internal winston properties (`Symbol(level)`, `Symbol(message)`, `Symbol(splat)`) before passing to ErrTracker.

6. **Console is the sole output**: All three logging layers ultimately output to `console.error`/`console.warn`/`console.info`/`console.debug`. There is no file-based transport, no log rotation, and no structured JSON file logging.

7. **Seed modules are tangential**: The original seed modules (3327, 2685, 3198, 0104, 2561, 3224) are mostly vendor modules (CSS grammars, gRPC internals, OTel resource attributes, evaluation templates) rather than direct logging setup code.

## Code Examples

### DiagConsoleLogger: Console Backend (0129.js, lines 1-27)
```js
var e8A = [
  { n: "error", c: "error" },
  { n: "warn", c: "warn" },
  { n: "info", c: "info" },
  { n: "debug", c: "debug" },
  { n: "verbose", c: "trace" },
];
class tCL {
  constructor() {
    function H(A) {
      return function (...L) {
        if (console) {
          let $ = console[A];
          if (typeof $ !== "function") $ = console.log;
          if (typeof $ === "function") return $.apply(console, L);
        }
      };
    }
    for (let A = 0; A < e8A.length; A++) this[e8A[A].n] = H(e8A[A].c);
  }
}
```

### DiagAPI: Core API (0124.js, lines 23-62)
```js
class a8A {
  constructor() {
    let L = ($, I = { logLevel: DiagLogLevel.INFO }) => {
      // ... logger registration with global singleton
    };
    this.setLogger = L;
    this.createComponentLogger = ($) => {
      return new DiagComponentLogger($);
    };
    this.verbose = H("verbose");
    this.debug = H("debug");
    this.info = H("info");
    this.warn = H("warn");
    this.error = H("error");
  }
  static instance() {
    if (!this._instance) this._instance = new a8A();
    return this._instance;
  }
}
```

### ErrTracker Winston Transport (1547.js, lines 24-42)
```js
function hjA(H, A) {
  class L extends H {
    constructor($) {
      super($);
      this._levels = new Set(A?.levels ?? iF1);
    }
    log($, I) {
      // Extract level, message, timestamp from winston info object
      let D = $[Xd$],
        { level: E, message: f, timestamp: M, ...U } = $;
      let P = tF1[D] ?? "info";
      if (this._levels.has(P)) ak(P, f, { ...U, "errtracker.origin": "auto.log.winston" });
      if (I) I();
    }
  }
  return L;
}
```

### gRPC Logging (2585.js, lines 10-25)
```js
var z8f = {
  error: (H, ...A) => { console.error("E " + H, ...A); },
  info: (H, ...A) => { console.error("I " + H, ...A); },
  debug: (H, ...A) => { console.error("D " + H, ...A); },
};
// Controlled by: process.env.GRPC_NODE_VERBOSITY || process.env.GRPC_VERBOSITY
```

## Integration Points (Cross-System)

- **OTel DiagAPI ↔ Telemetry (infra-telemetry-otel)**: The DiagAPI is part of the `@opentelemetry/api` package. The telemetry subsystem's tracer/span creation uses the same `diag` singleton for internal diagnostics. Module 0159.js (TraceAPI) directly references `DiagAPI.instance()`.

- **ErrTracker Winston ↔ Error Tracking (infra-errors-and-runtime)**: The winston-to-ErrTracker transport bridges logging into error tracking. Custom error classes from the error hierarchy would flow through this transport if winston is configured as the logger.

- **gRPC Logging ↔ IPC Daemon (infra-ipc-daemon)**: The gRPC transport layer's logging (2585.js) provides diagnostic output for WebSocket/IPC connections. The IPC daemon likely uses gRPC underneath.

- **OTel Fastify Instrumentation (1655.js) ↔ HTTP Client (infra-http-client)**: Fastify HTTP request instrumentation uses `_logger = diag.createComponentLogger({ namespace })` to trace HTTP request/response cycles, connecting to the HTTP client layer.

- **Config (infra-config-loader)**: Log levels are controlled via environment variables (`GRPC_NODE_VERBOSITY`, `GRPC_TRACE`, `OTEL_LOG_LEVEL`) which would be part of the config precedence chain.

## Implementation Notes

1. **Replace OTel DiagAPI with a standard logger**: For OpenDroid, replace the OTel DiagAPI with a direct winston or pino logger. The component logger pattern (namespace-prefixed) should be preserved: implement as `logger.child({ component: namespace })`.

2. **Implement file transports**: Add winston file transports with rotation (`winston-daily-rotate-file`) for persistent log storage. Configure `maxsize`, `maxFiles`, and `zippedArchive` for production use.

3. **Unify logging layers**: Consolidate the three logging subsystems (OTel DiagAPI, gRPC, ErrTracker) into a single logger instance. Create adapters/transports for gRPC and ErrTracker that feed into the unified logger.

4. **Environment variable mapping**:
   - `GRPC_NODE_VERBOSITY` → unified `LOG_LEVEL`
   - `GRPC_TRACE` → component-level filter
   - `OTEL_LOG_LEVEL` → unified `LOG_LEVEL`

5. **Structured JSON logging**: Use `format.combine(format.timestamp(), format.json())` for machine-readable log output, with `format.printf()` for human-readable console output in development.

## Module Reference

| Module | Line(s) | Function/Export | Notes |
|--------|---------|-----------------|-------|
| 0121.js | 8-28 | `DiagComponentLogger` class | Component logger with namespace prefix |
| 0124.js | 11-62 | `DiagAPI` class | Core diagnostic API singleton |
| 0124.js | 30-36 | `setLogger()` | Register custom logger with logLevel |
| 0124.js | 37-39 | `createComponentLogger()` | OpenDroid for namespaced loggers |
| 0129.js | 5-27 | `DiagConsoleLogger` class | Console-backed diagnostic logger |
| 0129.js | 6-11 | `e8A` level mapping | Maps 5 OTel levels to console methods |
| 0151.js | 5 | `diag` export | Singleton DiagAPI instance |
| 0192.js | 6-13 | `VzL` level map | String→DiagLogLevel mapping |
| 0192.js | 15-26 | `diagLogLevelFromString()` | Safe string→level conversion |
| 0800.js | 12-20 | `$W$` constructor | OTLP delegate with _diagLogger |
| 0800.js | 24 | `export()` | Uses _diagLogger.debug for export tracing |
| 1547.js | 18-42 | `hjA()` | Creates ErrTracker Winston transport class |
| 1547.js | 28 | `_levels` Set | Winston level filtering |
| 1547.js | 37 | `ak()` call | ErrTracker capture with winston metadata |
| 1655.js | 37 | `_logger` | `diag.createComponentLogger({ namespace })` |
| 1655.js | 99, 148 | `_logger.debug()` | Route instrumentation debug messages |
| 1570.js | 32-39 | debug toggle | ErrTracker SDK debug mode with console.warn |
| 2583.js | 31-36 | `LogVerbosity` enum | DEBUG=0, INFO=1, ERROR=2, NONE=3 |
| 2585.js | 11-21 | `z8f` default logger | Console error/warn/info/debug with E/I/D prefix |
| 2585.js | 26-44 | Verbosity switch | GRPC_NODE_VERBOSITY env var parsing |
| 2585.js | 60-66 | `b8f trace()` | Timestamped trace with component filter |
| 2585.js | 67-69 | `nRI()` | Trace component filter (all/set membership) |
| 2592.js | 244-247 | TLS error logging | `log(LogVerbosity.ERROR, ...)` for TLS failures |
| 2685.js | 31 | `Xe()` wrapper | `JxI.trace(d6.LogVerbosity.DEBUG, wxI, H)` |
| 0104.js | 159-162 | AWS log attrs | `aws.log.group.names`, `aws.log.stream.arns` |
| 1194.js | 162 | Detection criteria | Checks for winston/pino/bunyan in package.json |
