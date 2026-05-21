# infra-gcp-client: Architecture Notes

## Overview

OpenDroid integrates with Google Cloud Platform (GCP) through a layered client stack built on `google-auth-library`, `google-gax` (gRPC), and `gaxios` (HTTP). The GCP integration serves multiple purposes: authentication via Google OAuth2 / Service Accounts / Application Default Credentials (ADC), CloudDB/Firestore for data persistence (agent readiness reports), Cloud KMS for key management, IAM policy operations, and Google Vertex AI as an LLM provider. The system also includes GCP serverless detection for Cloud Run and Cloud Functions environments, and a comprehensive CSP whitelist for CloudDB and Google Auth endpoints.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 2759.js | ~483 lines | GoogleAuth: main authentication class, ADC resolution, credential caching, signing | Vendor (google-auth-library) |
| 2729.js | ~482 lines | OAuth2Client: OAuth2 token flow (auth URL, token exchange, refresh, verify) | Vendor (google-auth-library) |
| 2747.js | ~269 lines | BaseExternalAccountClient: STS token exchange, workforce pool, service account impersonation | Vendor (google-auth-library) |
| 2771.js | ~263 lines | GrpcClient: gRPC client construction, proto loading, SSL/mTLS credential setup | Vendor (google-gax) |
| 2714.js | ~60 lines | GCP residency detection (GCE, Cloud Run, Cloud Functions) | Vendor (google-auth-library) |
| 2699.js | ~94 lines | gaxios: Google HTTP client package metadata (v6.7.1) | Vendor (gaxios) |
| 2835.js | ~146 lines | IamClient: Cloud KMS IAM policy operations (getIamPolicy, setIamPolicy, testIamPermissions) | Vendor (google-gax / KMS) |
| 2834.js | ~623 lines | IAM + Service Account proto definitions (google.iam.v1, google.type.Expr) | Vendor (proto definitions) |
| 2661.js | ~455 lines | @grpc/grpc-js channelz: gRPC channel diagnostics | Vendor (@grpc/grpc-js) |
| 0894.js | ~103 lines | CSP whitelist: CloudDB, Google Auth, Google Analytics allowed domains | App (security config) |
| 1208.js | ~60 lines | store_agent_readiness_report tool: Firestore persistence for agent readiness data | App (tool) |
| 3122.js | ~77 lines | Secret type constants: GOOGLE_SECRETS, GOOGLE_VERTEXAI_CREDENTIALS, GCP_BIGQUERY_SECRETS | App (config) |
| 3922.js | ~886 lines | Droid management UI component (seed module) | App (GUI) |
| 3127.js | ~418 lines | Schema/tool definition conversion: Anthropic to OpenAI format (seed module) | App (provider adapter) |
| 2339.js | ~461 lines | Utility functions: command split, truncate, throttle (seed module) | App (utils) |
| 3752.js | ~472 lines | Automation scheduler: create/run/pause automations (seed module) | App (scheduler) |
| 3623.js | ~386 lines | Archiver: ZIP file creation Transform stream (seed module) | Vendor (archiver) |

## Architecture

The GCP client layer follows a hierarchical architecture:

```
┌─────────────────────────────────────────────────────┐
│ Application Layer                                    │
│  1208.js (Firestore tool)                            │
│  3122.js (Secret config: GOOGLE_SECRETS, VERTEX_AI)  │
│  0894.js (CSP whitelist for CloudDB/Auth endpoints) │
├─────────────────────────────────────────────────────┤
│ gRPC Client Layer                                    │
│  2771.js (GrpcClient: proto loading, stub creation)  │
│  2835.js (IamClient: Cloud KMS IAM ops)              │
│  2834.js (Proto definitions: IAM, ServiceAccount)    │
│  2661.js (@grpc/grpc-js: transport, channelz)        │
├─────────────────────────────────────────────────────┤
│ Auth Layer                                           │
│  2759.js (GoogleAuth: ADC, credential resolution)    │
│  2729.js (OAuth2Client: token lifecycle)             │
│  2747.js (BaseExternalAccountClient: STS exchange)   │
│  2714.js (GCP residency: GCE/serverless detection)   │
├─────────────────────────────────────────────────────┤
│ HTTP Transport                                       │
│  2699.js (gaxios v6.7.1: Google HTTP client)         │
└─────────────────────────────────────────────────────┘
```

**Authentication Flow:**
1. `GoogleAuth` (2759.js) resolves credentials via Application Default Credentials (ADC):
   - `GOOGLE_APPLICATION_CREDENTIALS` env var → file path
   - Well-known file: `~/.config/gcloud/application_default_credentials.json` (or `%APPDATA%/gcloud/` on Windows)
   - GCE metadata server (if running on Google Compute Engine)
   - External account client (workload identity federation)
2. `OAuth2Client` (2729.js) manages token lifecycle: generate auth URL, exchange code for tokens, refresh tokens, verify ID tokens
3. `BaseExternalAccountClient` (2747.js) handles external identity federation via STS (Security Token Service) token exchange

**GCP Service Usage:**
- **CloudDB/Firestore:** Agent readiness report storage (1208.js), CSP whitelisted endpoints (0894.js)
- **Cloud KMS:** IAM policy management via IamClient (2835.js), scopes: `cloud-platform` + `cloudkms`
- **Google Vertex AI:** Referenced as secret type `GOOGLE_VERTEXAI_CREDENTIALS` (3122.js): LLM provider
- **GCP BigQuery:** Referenced as secret type `GCP_BIGQUERY_SECRETS` (3122.js): analytics
- **Google OAuth2/Auth:** User authentication via IdProvider integration, CSP whitelisted domains (0894.js)

**gRPC Layer:**
- `GrpcClient` (2771.js) loads proto definitions and creates typed gRPC stubs
- Supports mTLS via client certificate detection (`GOOGLE_API_USE_CLIENT_CERTIFICATE=true`)
- Universe domain validation (`googleapis.com` default)
- Proto cache for repeated loading

## Key Findings

### 1. Complete Google Auth Library Integration
The application bundles the full `google-auth-library-nodejs` including `GoogleAuth`, `OAuth2Client`, `BaseExternalAccountClient`, `Compute` (GCE), and `JWT` credential types. The ADC (Application Default Credentials) resolution chain is fully implemented (2759.js, lines 126-164).

### 2. CloudDB/Firestore for Agent Readiness
Module 1208.js defines a `store_agent_readiness_report` tool that writes evaluation results directly to Firestore. This is an internal tool visible only to the `agent-readiness-droid` or when `ENABLE_READINESS_REPORT=true`.

### 3. GCP Secret Management
Module 3122.js defines three GCP-related secret types:
- `GOOGLE_SECRETS`: General Google API credentials
- `GOOGLE_VERTEXAI_CREDENTIALS`: Vertex AI service account credentials (LLM provider)
- `GCP_BIGQUERY_SECRETS`: BigQuery analytics credentials

### 4. GCP Residency Detection
Module 2714.js provides automatic detection of Google Cloud environments:
- Cloud Run (`CLOUD_RUN_JOB`), Cloud Functions (`FUNCTION_NAME`, `K_SERVICE`)
- GCE Linux (BIOS vendor check: `/sys/class/dmi/id/bios_vendor` contains "Google")
- GCE MAC address detection (prefix `42:01`)
- Used by `GoogleAuth._checkIsGCE()` to decide between metadata server auth and file-based auth

### 5. gRPC Transport Layer
The `@grpc/grpc-js` library (2661.js) provides the transport for all Google Cloud gRPC services. The `GrpcClient` (2771.js) wraps this with proto loading, SSL credential management, and mTLS support.

### 6. Cloud KMS IAM Operations
Module 2835.js implements an `IamClient` for Cloud KMS with methods: `getIamPolicy`, `setIamPolicy`, `testIamPermissions`. Service endpoint: `cloudkms.googleapis.com:443`.

### 7. CSP Whitelist for GCP Endpoints
Module 0894.js configures Content Security Policy to allow connections to:
- `identitytoolkit.googleapis.com` (CloudDB Auth)
- `securetoken.googleapis.com` (CloudDB token verification)
- `firestore.googleapis.com` (Firestore database)
- `*.clouddbio.com`, `*.clouddbapp.com` (CloudDB hosting/realtime DB)
- `accounts.google.com`, `apis.google.com` (Google Auth)

## Code Examples

### GoogleAuth ADC Resolution (2759.js, lines 126-164)
```js
async getApplicationDefaultAsync(H = {}) {
  if (this.cachedCredential)
    return await Ud(this, _d, "m", uUH).call(this, this.cachedCredential);
  let A;
  // 1. Try GOOGLE_APPLICATION_CREDENTIALS env var
  if ((A = await this._tryGetApplicationCredentialsFromEnvironmentVariable(H)), A) {
    if (A instanceof bUH.JWT) A.scopes = this.scopes;
    return await Ud(this, _d, "m", uUH).call(this, A);
  }
  // 2. Try well-known file (~/.config/gcloud/)
  if ((A = await this._tryGetApplicationCredentialsFromWellKnownFile(H)), A) {
    if (A instanceof bUH.JWT) A.scopes = this.scopes;
    return await Ud(this, _d, "m", uUH).call(this, A);
  }
  // 3. Try GCE metadata server
  if (await this._checkIsGCE())
    return (H.scopes = this.getAnyScopes()),
      await Ud(this, _d, "m", uUH).call(this, new o6f.Compute(H));
  throw Error(X4.GoogleAuthExceptionMessages.NO_ADC_FOUND);
}
```

### GCP Residency Detection (2714.js, lines 8-21)
```js
function guI() {
  return !!(process.env.CLOUD_RUN_JOB || process.env.FUNCTION_NAME || process.env.K_SERVICE);
}
duI.isGoogleCloudServerless = guI;

function cuI() {
  if ((0, uuI.platform)() !== "linux") return false;
  try {
    (0, vuI.statSync)(duI.GCE_LINUX_BIOS_PATHS.BIOS_DATE);
    let H = (0, vuI.readFileSync)(duI.GCE_LINUX_BIOS_PATHS.BIOS_VENDOR, "utf8");
    return /Google/.test(H);
  } catch (H) { return false; }
}
```

### OAuth2 Token Refresh (2729.js, lines 101-140)
```js
async refreshTokenNoCache(H) {
  let L = this.endpoints.oauth2TokenUrl.toString(),
    $ = {
      refresh_token: H,
      client_id: this._clientId,
      client_secret: this._clientSecret,
      grant_type: "refresh_token",
    };
  I = await this.transporter.request({
    ...v5.RETRY_CONFIG,
    method: "POST",
    url: L,
    data: aeA.stringify($),
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
  });
  let D = I.data;
  if (I.data && I.data.expires_in)
    ((D.expiry_date = new Date().getTime() + I.data.expires_in * 1000), delete D.expires_in);
  return (this.emit("tokens", D), { tokens: D, res: I });
}
```

### Firestore Tool Registration (1208.js, lines 13-40)
```js
hX = eI({
  id: nZ$,
  llmId: "store_agent_readiness_report",
  displayName: "Store Agent Readiness Report",
  description: `Stores agent readiness evaluation results to Firestore...`,
  executionLocation: "client",
  isTopLevelTool: true,
  requiresConfirmation: false,
  inputSchema: uZ$,
  outputSchemas: { result: gZ$ },
  isVisibleToUser: false,
  isToolEnabled: ({ droidId: H, enabledToolIds: A }) =>
    A?.includes(nZ$) ||
    H === "agent-readiness-droid" ||
    process.env.ENABLE_READINESS_REPORT === "true",
});
```

### GrpcClient Credential Setup (2771.js, lines 52-68)
```js
async _getCredentials(H) {
  if (H.sslCreds) return H.sslCreds;
  let A = this.grpc,
    L = H.cert && H.key
      ? A.credentials.createSsl(null, Buffer.from(H.key), Buffer.from(H.cert))
      : A.credentials.createSsl(),
    $ = await this.auth.getClient();
  return A.credentials.combineChannelCredentials(
    L,
    A.credentials.createFromGoogleCredential($),
  );
}
```

## Integration Points (Cross-System)

- **03-tool-agent-system (Tool System):** `store_agent_readiness_report` (1208.js) is a registered tool that uses Firestore for persistence: bridges infra/cloud with the tool execution framework
- **04-desktop-gui (GUI):** CSP whitelist (0894.js) configures Electron security policy to allow Google/CloudDB endpoints: direct GUI security integration
- **infra-auth (05-infrastructure):** `GoogleAuth` and `OAuth2Client` (2759.js, 2729.js) provide the Google-specific auth layer that integrates with the broader auth system analyzed in infra-auth
- **infra-http-client (05-infrastructure):** `gaxios` (2699.js) is the Google-specific HTTP client layer that parallels the general axios/gaxios HTTP client analyzed in infra-http-client
- **infra-aws-sdk (05-infrastructure):** `BaseExternalAccountClient` (2747.js) includes `AwsClient` for AWS-to-GCP workload identity federation: cross-cloud integration point
- **Provider System:** `GOOGLE_VERTEXAI_CREDENTIALS` (3122.js) is a secret type for Vertex AI as an LLM provider, connecting to the model provider infrastructure
- **ErrTracker/OpenTelemetry:** CloudDB instrumentation (2092-2096.js, detected via grep) provides auto-instrumentation for Firestore and CloudDB Functions observability

## Implementation Notes

1. **Google Auth Library:** Replace with a thin auth wrapper around `google-auth-library` npm package. The ADC pattern is standard and should be preserved. Map `GOOGLE_APPLICATION_CREDENTIALS` and `~/.config/gcloud/` paths to OpenDroid's config system.
2. **gRPC Layer:** The `@grpc/grpc-js` + `google-gax` stack is standard Google Cloud client infrastructure. Port by installing these as npm dependencies rather than bundling.
3. **Firestore Integration:** The `store_agent_readiness_report` tool (1208.js) should be reimplemented using the CloudDB Admin SDK (`clouddb-admin`) with proper initialization from config.
4. **Secret Management:** The `GOOGLE_SECRETS`, `GOOGLE_VERTEXAI_CREDENTIALS`, `GCP_BIGQUERY_SECRETS` secret types (3122.js) need a corresponding secret storage backend in OpenDroid (e.g., Vault, AWS Secrets Manager, or local encrypted store).
5. **CSP Configuration:** The CloudDB/Google endpoint whitelist (0894.js) should be extracted into a configurable CSP policy file that can be modified per deployment environment.

## Module Reference

| Module | Lines | Key Functions/Classes |
|--------|-------|----------------------|
| 2759.js | 1-483 | `GoogleAuth` class: constructor (line 80), `getProjectId` (line 95), `getApplicationDefaultAsync` (line 128), `_tryGetApplicationCredentialsFromEnvironmentVariable` (line 161), `_tryGetApplicationCredentialsFromWellKnownFile` (line 173), `sign` (line 408), `signBlob` (line 430) |
| 2729.js | 1-482 | `OAuth2Client` class: constructor (line 50), `generateAuthUrl` (line 76), `getTokenAsync` (line 101), `refreshToken` (line 118), `refreshTokenNoCache` (line 131), `getAccessTokenAsync` (line 167) |
| 2747.js | 1-269 | `BaseExternalAccountClient` class: constructor (line 42), STS credential exchange, Cloud Resource Manager URL (line 44), IAM workforce pool regex (line 45) |
| 2771.js | 1-263 | `GrpcClient` class: constructor (line 50), `_getCredentials` (line 62), `createStub` (line 140), `_detectClientCertificate` (line 187), `loadProto` (line 107), `loadProtoJSON` (line 116) |
| 2714.js | 1-60 | `isGoogleCloudServerless` (line 8), `isGoogleComputeEngineLinux` (line 18), `isGoogleComputeEngineMACAddress` (line 29), `detectGCPResidency` (line 38) |
| 2699.js | 1-94 | gaxios package.json metadata (v6.7.1, "A simple common HTTP client specifically for Google APIs") |
| 2835.js | 1-146 | `IamClient` class: constructor (line 10), `servicePath` = `cloudkms.googleapis.com` (line 75), `scopes` (line 85), `getIamPolicy` (line 100), `setIamPolicy`, `testIamPermissions` |
| 2834.js | 1-623 | Proto definitions: `google.iam.v1.IAMPolicy` (line 10), `google.iam.v1.Logging` (line 168), `google.api.ServiceAccount` (line 185), `google.type.Expr` (line 601) |
| 2661.js | 1-455 | `@grpc/grpc-js` channelz: `ChannelzTrace` (line 35), `ChannelzChildrenTracker` (line 65), `registerChannelzChannel` (line 1), `getChannelzServiceDefinition` (line 437) |
| 0894.js | 1-103 | CSP config: CloudDB endpoints (line 19-25), Google Auth endpoints (line 28-33), ErrTracker/Pylon endpoints (line 35-45) |
| 1208.js | 1-60 | `store_agent_readiness_report` tool: definition (line 13), Firestore storage, `isToolEnabled` gate (line 40) |
| 3122.js | 1-77 | Secret type constants: `GOOGLE_SECRETS` (line 23), `GOOGLE_VERTEXAI_CREDENTIALS` (line 24), `GCP_BIGQUERY_SECRETS` (line 37) |
