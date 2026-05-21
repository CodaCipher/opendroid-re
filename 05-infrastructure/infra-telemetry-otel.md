# infra-telemetry-otel: Architecture Notes

## Overview

OpenDroid uses the OpenTelemetry (OTel) JS SDK for both distributed tracing and metrics collection. The telemetry stack is built on `@opentelemetry/api` for the cross-cutting API, `@opentelemetry/sdk-trace-base` for trace production, and `@opentelemetry/sdk-metrics` for metric instruments. An application-specific `CustomerOtelClient` (module 1187) configures an OTLP metric exporter pointing to a Google Cloud Run collector endpoint, with opt-in behavior controlled via environment variables. The codebase contains approximately 80+ vendor modules from the `@opentelemetry` ecosystem, making it one of the larger vendor dependencies in 05-infrastructure.

## Module Map

| Module | Size | Role | Vendor/App |
|-------|-------|-----|------------|
| 0161.js | 223 lines | Main OTel API entry point: exports `trace`, `metrics`, `propagation`, `context`, `diag`, `SpanKind`, `SpanStatusCode`, `ProxyTracerProvider` | @opentelemetry/api |
| 0159.js | ~50 lines | `TraceAPI` singleton: `setGlobalTracerProvider`, `getTracerProvider`, `getTracer` | @opentelemetry/api |
| 0153.js | ~20 lines | `MetricsAPI` singleton: `setGlobalMeterProvider`, `getMeterProvider`, `getMeter` | @opentelemetry/api |
| 1520.js | ~80 lines | `BasicTracerProvider`: tracer registry, `spanProcessors`, `forceFlush`, `shutdown` | @opentelemetry/sdk-trace-base |
| 1514.js | ~15 lines | `BatchSpanProcessor`: batches spans before export | @opentelemetry/sdk-trace-base |
| 1521.js | ~40 lines | `ConsoleSpanExporter`: development/debug exporter printing spans to console | @opentelemetry/sdk-trace-base |
| 1306.js | ~20 lines | `registerInstrumentations`: registers auto-instrumentations with providers | @opentelemetry/instrumentation |
| 1309.js | ~60 lines | `InstrumentationAbstract`: base class for all auto-instrumentations | @opentelemetry/instrumentation |
| 1348.js | 526 lines | HTTP instrumentation utilities: span status, HTTP attributes, request/response metrics | @opentelemetry/instrumentation-http |
| 0750.js | 130 lines | `Resource` class: async attribute resolution, merge semantics | @opentelemetry/resources |
| 0770.js | 138 lines | Resource detectors: env, host, OS, process detectors | @opentelemetry/resources |
| 0772.js | 83 lines | Metric instruments factory: `Counter`, `Histogram`, `Gauge`, `UpDownCounter`, `ObservableGauge` | @opentelemetry/sdk-metrics |
| 1187.js | ~100 lines | `CustomerOtelClient`: app-specific OTLP metrics client, env-configured | App code |
| 0140.js | ~30 lines | `NoopTracer`: fallback tracer when no provider is registered | @opentelemetry/api |
| 0152.js | ~10 lines | `NoopMeterProvider`: fallback meter provider | @opentelemetry/api |

> **Note on seed modules:** Of the 6 assigned seeds (1348, 0499, 2681, 3964, 2679, 0992), only **1348.js** is directly telemetry-related. The others (0499=AWS SSO OAuth, 2681=gRPC channel, 3964=React MCP UI, 2679=gRPC retry, 0992=MCP schema) are not telemetry modules. The actual OTel implementation is spread across ~80+ vendor modules in the 0100–1800 range.

## Architecture

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Code                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ CLI entry   │  │ HTTP client │  │  CustomerOtelClient │  │
│  │ (sessions)  │  │ (requests)  │  │  (1187.js)          │  │
│  └──────┬──────┘  └──────┬──────┘  └─────────────────────┘  │
│         │                │                                   │
│         ▼                ▼                                   │
│  ┌──────────────────────────────────────────────┐           │
│  │  OpenTelemetry API (0161.js / @opentelemetry/api) │       │
│  │  • trace.getTracer()   • metrics.getMeter()      │       │
│  │  • context             • propagation             │       │
│  │  • diag (logging)                                  │       │
│  └──────────────────────────────────────────────┘           │
│         │                          │                         │
│         ▼                          ▼                         │
│  ┌──────────────┐         ┌──────────────┐                  │
│  │ Trace SDK    │         │ Metrics SDK  │                  │
│  │ (1520.js)    │         │ (0772.js)    │                  │
│  │• BasicTracer │         │• Counter     │                  │
│  │  Provider    │         │• Histogram   │                  │
│  │• BatchSpan   │         │• Gauge       │                  │
│  │  Processor   │         │• Observable* │                  │
│  │• ConsoleSpan │         └──────┬───────┘                  │
│  │  Exporter    │                │                          │
│  └──────────────┘                ▼                          │
│                            ┌─────────────┐                  │
│                            │ OTLP Export │                  │
│                            │ (1187.js)   │                  │
│                            │• DELTA temp │                  │
│                            │• 60s interval│                 │
│                            └──────┬──────┘                  │
│                                   │                         │
│                                   ▼                         │
│                         ┌──────────────────┐               │
│                         │ Collector URL    │               │
│                         │ (GCP Cloud Run)  │               │
│                         └──────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Trace Data Flow

1. **Instrumentation** (1309.js / 1348.js) creates spans via `trace.getTracer(name, version).startSpan(...)`
2. **Tracer** delegates to `BasicTracerProvider` (1520.js) which maintains a `MultiSpanProcessor`
3. **Span Processor** (`BatchSpanProcessor` in 1514.js) buffers spans and exports batches
4. **Exporter** (`ConsoleSpanExporter` in 1521.js for dev; OTLP for production) serializes and sends spans
5. **Resource** attributes (0750.js / 0770.js) are attached to every span for service identity

### Metrics Data Flow

1. **CustomerOtelClient** (1187.js) initializes a `MeterProvider` with a `PeriodicExportingMetricReader`
2. **Meter** (0772.js) creates instruments: `Counter`, `Histogram`, `Gauge`, `UpDownCounter`
3. **Instruments** record values which are aggregated by the SDK
4. **PeriodicExportingMetricReader** flushes aggregated metrics every 60 seconds via OTLP
5. **OTLP Metric Exporter** sends DELTA temporality metrics to the configured endpoint

## Key Findings

### 1. App-Level Telemetry Client (`CustomerOtelClient`, 1187.js)

The most significant discovery is the application-specific `CustomerOtelClient` class in module **1187.js**. This is not generic OTel SDK code: it is OpenDroid's own telemetry initialization:

- **Service identity:** `serviceName: "cli"`, `serviceVersion: "0.64.0"`
- **Deployment environment:** `"production"`
- **Default collector endpoint:** `https://telemetry-collector.example.com`
- **Opt-in via environment:** `OTEL_CUSTOMER_ENABLED="true"`
- **Configurable endpoint:** `OTEL_CUSTOMER_ENDPOINT` overrides the default
- **Configurable headers:** `OTEL_CUSTOMER_HEADERS` parsed as comma-separated `key=value` pairs
- **Timeout:** 10 seconds (`nD1 = 10000`)
- **Export interval:** 60 seconds (`iD1 = 60000`)

The client only initializes **metrics** (not traces) in its `initializeMetrics()` method, using an OTLP metric exporter with `DELTA` temporality preference.

### 2. Global API Singletons (0159.js, 0153.js, 0161.js)

The OTel API is exposed through singleton globals:

- `TraceAPI.getInstance()`: manages the global tracer provider via `setGlobalTracerProvider()` / `getTracerProvider()`
- `MetricsAPI.getInstance()`: manages the global meter provider via `setGlobalMeterProvider()` / `getMeterProvider()`
- If no provider is registered, fallbacks are `NoopTracerProvider` (0143.js) and `NoopMeterProvider` (0152.js)

This pattern means telemetry is **opt-in by default**: if the app doesn't call `setGlobalTracerProvider`, all tracing calls are no-ops with minimal overhead.

### 3. `BasicTracerProvider` Architecture (1520.js)

The `BasicTracerProvider` class reveals the SDK's internal structure:

- **Tracer registry:** Maps `name@version:schemaUrl` to `Tracer` instances
- **Resource:** Attached to all spans; defaults to `defaultResource()` from detectors
- **Span processors:** Array of processors passed to constructor; wrapped in `MultiSpanProcessor`
- **Configuration merge:** `loadDefaultConfig()` → `reconfigureLimits()` → final config
- **Lifecycle:** `forceFlush()` (with timeout) and `shutdown()` delegates to all span processors

### 4. Auto-Instrumentation Framework (1306.js, 1309.js)

OpenDroid includes the full OTel auto-instrumentation framework:

- `registerInstrumentations({ tracerProvider, meterProvider, loggerProvider, instrumentations })` enables all instrumentations
- `InstrumentationAbstract` is the base class for every auto-instrumentation
- Each instrumentation gets its own named tracer/meter/logger: `diag.createComponentLogger({ namespace: H })`
- Instrumentations can be enabled/disabled via config: `{ enabled: true, ...config }`

### 5. HTTP Instrumentation Semantic Conventions (1348.js)

Module 1348.js is part of `@opentelemetry/instrumentation-http`. It implements:

- **Span status mapping:** HTTP 4xx for CLIENT spans → ERROR; 5xx for SERVER spans → ERROR
- **Request attribute capture:** `http.request.header.*`, `http.response.header.*`
- **Content length tracking:** Compressed vs uncompressed content length
- **URL redaction:** Query string parameters in `DEFAULT_QUERY_STRINGS_TO_REDACT` are replaced with `REDACTED`
- **Error handling:** `setSpanWithError()` sets span status to ERROR and calls `recordException()`

### 6. Metric Instruments Available (0772.js)

The `Meter` class in 0772.js creates 7 instrument types:

| Instrument | Method | Recording Behavior |
|------------|--------|-------------------|
| Counter | `createCounter()` | `add(value)`: monotonic, rejects negative |
| UpDownCounter | `createUpDownCounter()` | `add(value)`: allows negative |
| Histogram | `createHistogram()` | `record(value)`: bucketed distribution, rejects negative |
| Gauge | `createGauge()` | `record(value)`: last-value wins |
| ObservableCounter | `createObservableCounter()` | Callback-based, monotonic |
| ObservableUpDownCounter | `createObservableUpDownCounter()` | Callback-based |
| ObservableGauge | `createObservableGauge()` | Callback-based, last-value wins |

### 7. Resource Detectors (0770.js)

The `detectResources()` function automatically discovers:
- **envDetector**: `OTEL_RESOURCE_ATTRIBUTES` environment variable
- **hostDetector**: hostname
- **osDetector**: OS type, OS version
- **processDetector**: process PID, executable name, command line arguments
- **serviceInstanceIdDetector**: unique service instance ID

## Code Examples

### CustomerOtelClient initialization (1187.js, lines 15–45)

```js
class ygH {
  config;
  meterProvider = null;
  constructor() {
    this.config = {
      serviceName: "cli",
      serviceVersion: "0.64.0",
      deploymentEnv: "production",
      endpoint: process.env.OTEL_CUSTOMER_ENDPOINT || "https://telemetry-collector.example.com",
      headers: ygH.parseHeadersFromEnv(),
      enabled: process.env.OTEL_CUSTOMER_ENABLED === "true",
    };
    if (this.config.enabled) this.init();
  }
  initializeMetrics(H) {
    let A = new d0H({
        url: `${this.config.endpoint}/v1/metrics`,
        headers: this.config.headers || {},
        timeoutMillis: 10000,
        temporalityPreference: Ac.DELTA,
      }),
      L = new zGH(A),
      $ = new IxH({ exporter: L, exportIntervalMillis: 60000 });
    this.meterProvider = new WxH({ resource: H, readers: [$] });
  }
}
```

### TraceAPI singleton (0159.js, lines 13–50)

```js
class u_A {
  constructor() {
    this._proxyTracerProvider = new rYL.ProxyTracerProvider();
  }
  static getInstance() {
    if (!this._instance) this._instance = new u_A();
    return this._instance;
  }
  setGlobalTracerProvider(H) {
    let A = registerGlobal("trace", this._proxyTracerProvider, DiagAPI.instance());
    if (A) this._proxyTracerProvider.setDelegate(H);
    return A;
  }
  getTracer(H, A) {
    return this.getTracerProvider().getTracer(H, A);
  }
}
```

### BasicTracerProvider with span processors (1520.js, lines 25–55)

```js
class Sp$ {
  _tracers = new Map();
  _resource;
  _activeSpanProcessor;
  constructor(H = {}) {
    let A = merge({}, loadDefaultConfig(), reconfigureLimits(H));
    this._resource = A.resource ?? defaultResource();
    let L = [];
    if (H.spanProcessors?.length) L.push(...H.spanProcessors);
    this._activeSpanProcessor = new MultiSpanProcessor(L);
  }
  getTracer(H, A, L) {
    let $ = `${H}@${A || ""}:${L?.schemaUrl || ""}`;
    if (!this._tracers.has($))
      this._tracers.set($, new Tracer({ name: H, version: A, schemaUrl: L?.schemaUrl }, this._config, this._resource, this._activeSpanProcessor));
    return this._tracers.get($);
  }
}
```

### HTTP span status mapping (1348.js, lines 60–78)

```js
var $P1 = (H, A) => {
  let L = H === SpanKind.CLIENT ? 400 : 500;
  if (A && A >= 100 && A < L) return SpanStatusCode.UNSET;
  return SpanStatusCode.ERROR;
};
var DP1 = (H, A, L) => {
  let $ = A.message;
  if (L & SemconvStability.OLD)
    (H.setAttribute(AttributeNames.HTTP_ERROR_NAME, A.name),
      H.setAttribute(AttributeNames.HTTP_ERROR_MESSAGE, $));
  if (L & SemconvStability.STABLE) H.setAttribute(ATTR_ERROR_TYPE, A.name);
  (H.setStatus({ code: SpanStatusCode.ERROR, message: $ }), H.recordException(A));
};
```

### Metric instrument creation (0772.js, lines 35–65)

```js
class Y7A {
  createCounter(H, A) {
    let L = SR(H, UE.COUNTER, A),
      $ = this._meterSharedState.registerMetricStorage(L);
    return new Q7A($, L); // Q7A validates non-negative
  }
  createHistogram(H, A) {
    let L = SR(H, UE.HISTOGRAM, A),
      $ = this._meterSharedState.registerMetricStorage(L);
    return new w7A($, L); // w7A validates non-negative
  }
  createGauge(H, A) {
    let L = SR(H, UE.GAUGE, A),
      $ = this._meterSharedState.registerMetricStorage(L);
    return new J7A($, L);
  }
}
```

## Integration Points (cross-system)

### With CLI Entry (infra-cli-entry)
- The `CustomerOtelClient` service name is `"cli"`, indicating telemetry is tied to CLI execution
- Environment-based enablement (`OTEL_CUSTOMER_ENABLED`) suggests CLI flags or env setup controls telemetry

### With HTTP Client (infra-http-client)
- `@opentelemetry/instrumentation-http` (1348.js) automatically traces all HTTP requests made by axios/gaxios
- HTTP span attributes include request/response headers, content length, compression status
- Cross-reference: HTTP client module 3174.js would be wrapped by this instrumentation

### With Session Manager (infra-session-manager)
- Session lifecycle events (create, save, load, compact) could emit custom spans via `trace.getTracer("session").startSpan()`
- Session metrics (active session count, compact frequency) could be collected via `metrics.getMeter("session").createCounter()`

### With Plugin Manager (infra-plugin-manager)
- Auto-instrumentation framework (1306.js) mirrors the plugin registration pattern
- Plugin hooks (`onSessionStart`, `onToolCall`) could be instrumented with spans
- Instrumentation base class (1309.js) uses the same `setConfig` / `getConfig` pattern as plugins

### With Logging (infra-logging-winston)
- `diag` API from OTel (0161.js) provides a `DiagConsoleLogger` that could bridge to winston
- `diag.error()`, `diag.warn()`, `diag.debug()` calls in 1187.js and 0772.js log telemetry lifecycle events

### With IPC Daemon (infra-ipc-daemon)
- WebSocket messages between daemon and GUI could be traced with custom spans
- IPC message types (request/response/event) map naturally to span kinds (CLIENT/SERVER/PRODUCER/CONSUMER)

## Implementation Notes

1. **Preserve opt-in behavior:** Keep `OTEL_CUSTOMER_ENABLED` environment variable check so telemetry is off by default
2. **Extract collector config:** Move hardcoded endpoint (`<collector-id>.us-central1.run.app`) to configuration file or env var
3. **Add trace initialization:** The current `CustomerOtelClient` only sets up metrics. Add `BasicTracerProvider` + `BatchSpanProcessor` + OTLP trace exporter for full tracing
4. **Instrument core operations:** Add manual spans for session lifecycle, tool execution, and model inference using `trace.getTracer("opendroid")`
5. **Add runtime metrics:** Create histograms for tool latency, model latency, and token count using the existing `Meter` API
6. **Vendor dependency note:** The ~80+ `@opentelemetry` vendor modules should be bundled as a single dependency group; consider using `@opentelemetry/auto-instrumentations-node` to reduce module count

## Module Reference

| File | Line | Function / Class | Description |
|-------|-------|-------------------|----------|
| 1187.js | 15 | `ygH.constructor()` | `CustomerOtelClient` config setup |
| 1187.js | 35 | `ygH.initializeMetrics()` | OTLP metric exporter + PeriodicExportingMetricReader init |
| 1187.js | 70 | `ygH.forceFlush()` | Async meter provider flush |
| 1187.js | 80 | `ygH.shutdown()` | Async meter provider shutdown |
| 0161.js | 1 | `YA` module | Main OTel API exports (`trace`, `metrics`, `diag`, etc.) |
| 0159.js | 13 | `u_A.constructor()` | `TraceAPI` singleton init with `ProxyTracerProvider` |
| 0159.js | 35 | `setGlobalTracerProvider()` | Registers global tracer provider |
| 0153.js | 8 | `j_A.constructor()` | `MetricsAPI` singleton init |
| 0153.js | 18 | `setGlobalMeterProvider()` | Registers global meter provider |
| 1520.js | 25 | `Sp$.constructor()` | `BasicTracerProvider` with `spanProcessors` array |
| 1520.js | 40 | `getTracer()` | Tracer registry lookup/create |
| 1520.js | 55 | `forceFlush()` | Timeout-based span processor flush |
| 1514.js | 5 | `Bp$.onShutdown()` | `BatchSpanProcessor` shutdown hook |
| 1521.js | 8 | `xp$.export()` | `ConsoleSpanExporter` export method |
| 1521.js | 25 | `_exportInfo()` | Span serialization for console output |
| 1306.js | 5 | `i91()` | `registerInstrumentations` entry point |
| 1309.js | 10 | `Yj$.constructor()` | `InstrumentationAbstract` base class |
| 1348.js | 60 | `$P1()` | HTTP status code → `SpanStatusCode` mapping |
| 1348.js | 70 | `DP1()` | `setSpanWithError`: sets status + records exception |
| 0750.js | 10 | `w3H.constructor()` | `Resource` class with async attribute support |
| 0770.js | 5 | `g0H` module | Resource detector exports (env, host, os, process) |
| 0772.js | 25 | `Y7A` class | `Meter` instrument factory (Counter, Histogram, Gauge) |
| 0140.js | 8 | `vqL.startSpan()` | `NoopTracer` fallback implementation |
| 0152.js | 5 | `N_A.getMeter()` | `NoopMeterProvider` fallback implementation |

---

*Report generated for feature `infra-telemetry-otel`: 05-infrastructure Infrastructure Architecture Notes + Cross-System Synthesis.*
