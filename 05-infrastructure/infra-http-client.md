# infra-http-client: Architecture Notes

## Overview

OpenDroid's HTTP client layer is built on **gaxios** (Google's HTTP client library, v6.7.1), not Axios. Gaxios provides a fetch-like API with built-in retry logic, request/response interceptors, and transparent proxy support. The HTTP client is primarily used by the Google Auth Library (v9.15.1) for OAuth token exchange, STS credential exchange, and GCP API calls. The `DefaultTransporter` wraps gaxios to add Google-specific headers (User-Agent, x-goog-api-client). A separate OAuth client module (0996.js) uses native `fetch` directly for MCP OAuth flows. The system also includes gRPC load balancing (3174.js) with outlier detection, and AWS SDK's Smithy middleware stack for non-Google HTTP interactions.

## Module Map

| Module | Size | Rol | Vendor/App |
|--------|------|-----|------------|
| 2708.js | ~366 lines | **Core Gaxios class**: HTTP client with interceptors, retry, proxy, streaming | Vendor (gaxios) |
| 2702.js | ~87 lines | **Retry configuration**: getRetryConfig, exponential backoff, status code filtering | Vendor (gaxios) |
| 2701.js | ~126 lines | **GaxiosError**: error class with response data redaction | Vendor (gaxios) |
| 2703.js | ~10 lines | **GaxiosInterceptorManager**: Set-based interceptor chain | Vendor (gaxios) |
| 2709.js | ~35 lines | **Gaxios module index**: exports Gaxios, GaxiosError, request(), singleton instance | Vendor (gaxios) |
| 2722.js | ~61 lines | **DefaultTransporter**: wraps gaxios with Google headers and error processing | Vendor (google-auth-library) |
| 2727.js | ~61 lines | **AuthClient**: base auth class with gaxios integration, retry config | Vendor (google-auth-library) |
| 2721.js | ~87 lines | **google-auth-library package.json**: dependencies, gaxios ^6.1.1 | Vendor |
| 2699.js | ~94 lines | **gaxios package.json**: metadata, version 6.7.1 | Vendor |
| 2763.js | ~209 lines | **Google Auth re-exports**: barrel module for all auth classes | Vendor (google-auth-library) |
| 2759.js | ~483 lines | **GoogleAuth**: ADC credential detection, gcloud CLI integration | Vendor (google-auth-library) |
| 2746.js | ~100 lines | **StsCredentials**: STS token exchange via DefaultTransporter | Vendor (google-auth-library) |
| 0996.js | ~460 lines | **MCP OAuth client**: fetch-based OAuth 2.0 flows (authorize, token, register) | App (OpenDroid) |
| 2695.js | ~213 lines | **OTel semantic attributes**: HTTP request/response attribute constants | Vendor (opentelemetry) |
| 3174.js | ~490 lines | **gRPC OutlierDetection LB**: request volume tracking, success rate ejection | Vendor (grpc) |
| 1747.js | ~484 lines | **MongoDB instrumentation**: OTel auto-instrumentation for MongoDB | Vendor (opentelemetry) |
| 0383.js | ~480 lines | **Smithy Client**: AWS SDK middleware stack, request handler | Vendor (aws-sdk) |
| 0106.js | ~93 lines | **AutoIt highlighter**: syntax highlighting for AutoIt language | Vendor (highlight.js) |
| 3803.js | ~271 lines | **OpenDroid skill/cost command**: token usage display, skill creation workflow | App (OpenDroid) |

## Architecture

### HTTP Client Hierarchy

```
┌─────────────────────────────────────────────────────┐
│  Application Layer                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ GoogleAuth   │  │ StsCredentials│  │ MCP OAuth  │ │
│  │ (2759.js)    │  │ (2746.js)     │  │ (0996.js)  │ │
│  └──────┬───────┘  └──────┬────────┘  └─────┬──────┘ │
│         │                 │                  │ fetch   │
│  ┌──────▼─────────────────▼──────────┐       │        │
│  │     DefaultTransporter (2722.js)  │       │        │
│  │  - User-Agent headers             │       │        │
│  │  - x-goog-api-client header       │       │        │
│  │  - Error normalization            │       │        │
│  └──────────────┬────────────────────┘       │        │
│                 │                             │        │
│  ┌──────────────▼────────────────────┐       │        │
│  │     Gaxios (2708.js)              │       │        │
│  │  ┌──────────────────────────────┐ │       │        │
│  │  │ Request Interceptors         │ │       │        │
│  │  │ (GaxiosInterceptorManager)   │ │       │        │
│  │  └──────────────────────────────┘ │       │        │
│  │  ┌──────────────────────────────┐ │       │        │
│  │  │ Config Merge + URL Build     │ │       │        │
│  │  │ - defaults + request opts    │ │       │        │
│  │  │ - proxy detection            │ │       │        │
│  │  │ - body serialization         │ │       │        │
│  │  └──────────────────────────────┘ │       │        │
│  │  ┌──────────────────────────────┐ │       │        │
│  │  │ _request() with Retry Loop   │ │       │        │
│  │  │ - validateStatus check       │ │       │        │
│  │  │ - getRetryConfig()           │ │       │        │
│  │  │ - exponential backoff        │ │       │        │
│  │  └──────────────────────────────┘ │       │        │
│  │  ┌──────────────────────────────┐ │       │        │
│  │  │ Response Interceptors        │ │       │        │
│  │  │ (GaxiosInterceptorManager)   │ │       │        │
│  │  └──────────────────────────────┘ │       │        │
│  └──────────────┬────────────────────┘       │        │
│                 │ node-fetch / window.fetch   │        │
│  ┌──────────────▼────────────────────┐       │        │
│  │     Fetch Adapter (_defaultAdapter)│       │        │
│  └───────────────────────────────────┘       │        │
│                                               │        │
│  ┌───────────────────────────────────────────▼───┐    │
│  │  Smithy Client (0383.js): AWS SDK middleware │    │
│  │  - middlewareStack (constructStack)            │    │
│  │  - retryStrategy (standard mode)              │    │
│  └───────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### Request Flow

1. **AuthClient** (2727.js) creates a `DefaultTransporter` or accepts a custom `transporter`
2. **DefaultTransporter** (2722.js) adds User-Agent and x-goog-api-client headers
3. **Gaxios.request()** (2708.js) is called:
   - **Phase 1: Merge config** (`quI` private method): merges defaults + per-request opts, resolves URL, serializes body, sets up proxy agent
   - **Phase 2: Request interceptors** (`KuI` private method): runs each request interceptor's resolved/rejected handler
   - **Phase 3: Execute** (`_request`): calls adapter (fetch by default), validates status, on failure calls `getRetryConfig()` for retry decision
   - **Phase 4: Response interceptors** (`CuI` private method): runs each response interceptor
4. **Retry loop**: if `shouldRetry=true`, increments `currentRetryAttempt` and recurses `_request()`

## Key Findings

### 1. Gaxios is the primary HTTP client (not Axios)

The codebase uses **gaxios v6.7.1** ("A simple common HTTP client specifically for Google APIs and services") rather than Axios. Gaxios wraps `node-fetch` in Node.js and `window.fetch` in browsers. A singleton instance is exported at `Mz.instance = new Gaxios()` (2709.js, ErrTracker init section).

### 2. Interceptor chain is Set-based

`GaxiosInterceptorManager` (2703.js) extends `Set`: interceptors are stored as an ordered set of `{resolved, rejected}` objects. The chain processes request interceptors before execution and response interceptors after, similar to Axios but using Set iteration.

### 3. Retry with exponential backoff

The retry system (2702.js) is sophisticated:
- **Default max retries**: 3
- **Retryable status codes**: 100-199, 408, 429, 500-599
- **Retryable methods**: GET, HEAD, PUT, OPTIONS, DELETE (POST excluded by default)
- **No-response retries**: 2 (when server returns no response)
- **Backoff formula**: `initialDelay + ((multiplier^attempt - 1) / 2) * 1000` with configurable multiplier (default 2)
- **Total timeout**: `Number.MAX_SAFE_INTEGER` (effectively unlimited)
- **Abort handling**: AbortError explicitly excluded from retry

### 4. AuthClient provides default retry configuration

`AuthClient.RETRY_CONFIG` (2727.js, line 58-62) overrides default retry to include more HTTP methods: `["GET", "PUT", "POST", "HEAD", "OPTIONS", "DELETE"]`: notably including POST.

### 5. Error redaction for sensitive data

`GaxiosError` (2701.js) applies `defaultErrorRedactor` which redacts:
- `authentication`, `authorization` headers
- Fields matching `/secret/i`
- `grant_type`, `assertion`, `client_secret` in body/data
- `token`, `client_secret` URL query parameters

### 6. MCP OAuth uses native fetch (0996.js)

The MCP (Model Context Protocol) OAuth module uses native `fetch` directly, not gaxios. It implements the full OAuth 2.0 Authorization Code flow with PKCE, dynamic client registration, and protected resource metadata discovery. Functions like `WqA`, `TqA`, `TuH`, `LrE` accept a `fetchFn` parameter (defaults to `fetch`).

### 7. Dual HTTP client architecture

The system has two distinct HTTP client paths:
- **gaxios path**: Used for all Google Cloud API calls (auth, storage, compute): wrapped in DefaultTransporter
- **native fetch path**: Used for MCP OAuth flows: direct fetch calls with `fetchFn` dependency injection

### 8. AWS SDK uses Smithy middleware (0383.js)

For AWS services, the Smithy `Client` class provides its own middleware stack (`middlewareStack`) with `send()` method, `retryStrategy` support, and `requestHandler`. This is a completely separate HTTP client pipeline from gaxios.

## Code Examples

### Gaxios class construction and interceptors (2708.js, lines 116-130)

```js
class yeA {
  constructor(H) {
    this.agentCache = new Map();
    this.defaults = H || {};
    this.interceptors = {
      request: new JuI.GaxiosInterceptorManager(),
      response: new JuI.GaxiosInterceptorManager(),
    };
  }
  async request(H = {}) {
    H = await Je(this, CUH, "m", quI).call(this, H);  // merge config
    H = await Je(this, CUH, "m", KuI).call(this, H);  // request interceptors
    return Je(this, CUH, "m", CuI).call(this, this._request(H)); // execute + response interceptors
  }
}
```

### Retry logic with exponential backoff (2702.js, lines 6-35)

```js
async function ZFf(H) {
  let A = ovI(H);
  // ... defaults setup ...
  A.retry = A.retry === void 0 || A.retry === null ? 3 : A.retry;
  A.retryDelayMultiplier = A.retryDelayMultiplier ? A.retryDelayMultiplier : 2;
  A.noResponseRetries = A.noResponseRetries === void 0 ? 2 : A.noResponseRetries;
  
  let I = NFf(A); // compute backoff delay
  H.config.retryConfig.currentRetryAttempt += 1;
  let D = A.retryBackoff
    ? A.retryBackoff(H, I)
    : new Promise((E) => { setTimeout(E, I); });
  if (A.onRetryAttempt) A.onRetryAttempt(H);
  await D;
  return { shouldRetry: true, config: H.config };
}
```

### DefaultTransporter wrapping gaxios (2722.js, lines 12-34)

```js
class wCH {
  constructor() { this.instance = new U5f.Gaxios(); }
  configure(H = {}) {
    H.headers = H.headers || {};
    if (typeof window === "undefined") {
      H.headers["User-Agent"] = wCH.USER_AGENT; // "google-api-nodejs-client/9.15.1"
      H.headers["x-goog-api-client"] = `gl-node/${process.version.replace(/^v/, "")}`;
    }
    return H;
  }
  request(H) {
    H = this.configure(H);
    return this.instance.request(H).catch((A) => { throw this.processError(A); });
  }
}
```

### AuthClient retry config (2727.js, lines 58-62)

```js
static get RETRY_CONFIG() {
  return {
    retry: true,
    retryConfig: { httpMethodsToRetry: ["GET", "PUT", "POST", "HEAD", "OPTIONS", "DELETE"] },
  };
}
```

### MCP OAuth fetch-based token exchange (0996.js, lines 396-399)

```js
let U = await (E ?? fetch)(f, { method: "POST", headers: M, body: L });
// ... response handling
```

## Integration Points (cross-system)

- **infra-auth (VAL-INF-010)**: AuthClient (2727.js) is the bridge: provides gaxios instance to all auth credential classes (JWT, Compute, OAuth2). GoogleAuth (2759.js) creates credential instances that use DefaultTransporter.
- **infra-gcp-client (VAL-INF-013)**: All GCP API calls (Storage, Compute) go through gaxios via DefaultTransporter. The transporter adds GCP-specific headers.
- **infra-aws-sdk (VAL-INF-012)**: Uses separate Smithy middleware stack (0383.js): not gaxios. Has its own retry strategy (`retryMode: "standard"`).
- **infra-telemetry-otel (VAL-INF-009)**: OpenTelemetry semantic attributes (2695.js) define HTTP request/response span attributes. The gRPC load balancer (3174.js) tracks request volumes for outlier detection.
- **infra-plugin-manager (VAL-INF-005)**: Plugins may need HTTP client for external API calls: the gaxios singleton (2709.js) is available.
- **Tool & Agent section (Tool System)**: MCP OAuth flow (0996.js) is used by tool system for authenticating MCP tool servers. The `fetchFn` parameter enables test injection.

## Implementation Notes

1. **Replace gaxios with a generic HTTP client**: If not using Google APIs, replace gaxios with Axios or native fetch. The interceptor pattern is nearly identical to Axios, so migration is straightforward.
2. **Preserve retry logic**: The retry system (2702.js) is well-designed: extract it as a standalone module with the same exponential backoff formula and status code filtering.
3. **Maintain error redaction**: The `defaultErrorRedactor` pattern (2701.js) should be kept for security: it sanitizes auth tokens, secrets, and credentials in error objects before logging.
4. **Unify HTTP clients**: The dual-path (gaxios + native fetch) adds complexity. For OpenDroid, consider a single HTTP client abstraction that supports both authenticated (GCP-style) and simple fetch patterns.
5. **Interceptor manager**: The Set-based interceptor approach (2703.js) is simple and effective: port it directly or use Axios's built-in interceptor system.
6. **Proxy support**: Gaxios auto-detects proxy from HTTPS_PROXY/HTTP_PROXY env vars with NO_PROXY exclusion: ensure the replacement supports the same.

## Module Reference

| Module | Lines Referenced | Key Functions/Classes |
|--------|-----------------|----------------------|
| 2708.js | L80-130 (constructor, request), L133-175 (_request with retry), L200-320 (config merge, proxy, body serialization) | `yeA` (Gaxios class), `_request()`, `_defaultAdapter()`, `request()` |
| 2702.js | L6-87 (full file) | `ZFf` (getRetryConfig), `zFf` (shouldRetry), `NFf` (backoff calculation), `ovI` (getRetryConfigFromError) |
| 2701.js | L15-126 (full file) | `ReA` (GaxiosError class), `rvI` (defaultErrorRedactor) |
| 2703.js | L1-10 (full file) | `svI` (GaxiosInterceptorManager extends Set) |
| 2709.js | L1-35 (full file) | `Mz` module: exports `Gaxios`, `GaxiosError`, `request()`, `instance` |
| 2722.js | L12-61 (full file) | `wCH` (DefaultTransporter): `configure()`, `request()`, `processError()` |
| 2727.js | L20-63 (AuthClient class) | `qgI` (AuthClient): `gaxios` getter, `RETRY_CONFIG` static |
| 2746.js | L15-60 (StsCredentials) | `JHL` (StsCredentials): `exchangeToken()` |
| 2759.js | L1-80 (GoogleAuth constructor) | `sHL` (GoogleAuth): ADC credential detection |
| 2763.js | L1-80 (barrel exports) | Re-exports all auth classes + DefaultTransporter + gaxios |
| 2699.js | L1-30 (package metadata) | gaxios package.json: version 6.7.1 |
| 2721.js | L1-30 (package metadata) | google-auth-library package.json: version 9.15.1 |
| 0996.js | L1-40 (OAuth errors), L140-260 (OAuth client), L320-460 (token exchange) | OAuth error classes, `WqA` (fetchProtectedResourceMetadata), `TuH` (fetchAuthorizationServerMetadata), `LrE` (exchangeAuthorizationCode) |
| 2695.js | L153-206 (HTTP attributes) | `ATTR_HTTP_REQUEST_METHOD`, `ATTR_HTTP_RESPONSE_STATUS_CODE`, HTTP method values |
| 3174.js | L1-80 (outlier detection config), L80-160 (config validation) | `XCH` (OutlierDetectionLoadBalancingConfig), `oX` (trace) |
| 0383.js | L1-80 (Smithy Client) | `ZcL` (Client): `send()`, `middlewareStack`, `destroy()` |
