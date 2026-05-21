# infra-auth: Architecture Notes

## Overview

OpenDroid's authentication system is a multi-layered, protocol-rich infrastructure spanning OAuth 2.0 device flows, AWS STS role assumption, DPoP (Demonstrating Proof-of-Possession) JWT signing, and MCP (Model Context Protocol) token management. The system integrates IdProvider for user identity, AWS SDK v3 for cloud credential chains, and a custom OAuth token persistence layer backed by OS keyring services. Auth observability is provided via OpenTelemetry HTTP instrumentation with configurable header capture.

## Module Map

| Module | Size | Role | Vendor/App |
|-------|-------|-----|------------|
| 3059.js | 532 lines | AWS Sign-in Service OAuth2 + DPoP JWT | AWS SDK (smithy) |
| 3035.js | 463 lines | AWS STS credential provider + AssumeRole | AWS SDK (STS) |
| 1349.js | 463 lines | OpenTelemetry HTTP instrumentation (header capture) | @opentelemetry |
| 1107.js | 531 lines | MCP client protocol (initialize, capabilities) | Model Context Protocol |
| 0870.js | ~65 lines | IdProvider OAuth 2.0 device flow | IdProvider API |
| 1135.js | 105 lines | MCP OAuth token storage (keyring-backed) | OpenDroid |
| 0996.js | 460 lines | OAuth2 error classes + client auth methods | OpenDroid |
| 3843.js | 601 lines | Achievements/badges (NOT auth-related) | OpenDroid |
| 3261.js | 65 lines | LSL syntax highlighter (NOT auth-related) | highlight.js |

## Architecture

The auth architecture consists of four parallel subsystems:

1. **User Identity (IdProvider OAuth 2.0 Device Flow)**: 0870.js implements RFC 8628 device authorization grant against `auth.idprovider.com`. It polls for token completion with `authorization_pending`/`slow_down`/`expired_token` handling and supports refresh token rotation.

2. **Cloud Credentials (AWS STS + SDK Auth)**: 3035.js provides the AWS credential provider chain with `AssumeRole` and `AssumeRoleWithWebIdentity`, including MFA (`SerialNumber`/`TokenCode`) and `ExternalId` support. 3059.js adds OAuth2 token creation (`CreateOAuth2Token`) and DPoP JWT proof generation for the AWS Sign-in Service.

3. **Protocol Auth (MCP OAuth)**: 1135.js persists MCP server OAuth tokens (`mcpOAuth` namespace) with `accessToken`, `refreshToken`, `tokenType: "Bearer"`, and `expiresAt`. Storage is backed by a keyring service (`JOA`) with fallback file storage (`QOA`). 1107.js handles the MCP `initialize` handshake where capabilities (including auth) are negotiated.

4. **Auth Observability (OpenTelemetry)**: 1349.js instruments HTTP requests and can capture auth-related headers via `captureRequestHeaders`/`captureResponseHeaders` configuration.

## Key Findings

### 1. DPoP JWT Proof Generation (3059.js)
The AWS Sign-in Service client generates DPoP JWTs using ES256 with EC P-256 keys. The `createDPoPInterceptor` middleware injects a `DPoP` header into outgoing requests. This is a modern, security-hardened OAuth2 extension that cryptographically binds access tokens to the client that requested them.

### 2. IdProvider Device Flow with Polling (0870.js)
User authentication uses IdProvider's device flow: `POST /user_management/authorize/device` returns a device code, then a polling loop calls `POST /user_management/authenticate` with `grant_type=urn:ietf:params:oauth:grant-type:device_code`. The loop handles `authorization_pending`, `slow_down`, `access_denied`, and `expired_token` errors with adaptive interval backoff.

### 3. MCP OAuth Token Lifecycle (1135.js)
MCP server connections store OAuth tokens per-server (keyed by `serverName` + `serverUrl`). Tokens include `accessToken`, `refreshToken`, `tokenType`, `expiresAt`, and `scope`. The storage layer uses OS keyring when available (`JOA` with `keyringService`/`keyringAccount`) and falls back to file-based storage (`QOA`). Token expiry is checked at load time (`expiresAt - Date.now()`).

### 4. AWS STS Role Assumption with MFA (3035.js)
The credential provider supports cross-account role assumption with `RoleArn`, `RoleSessionName`, `DurationSeconds` (default 3600s), and `ExternalId`. When `mfa_serial` is present in the profile, it requires an `mfaCodeProvider` callback and injects `SerialNumber` + `TokenCode` into the `AssumeRole` request.

### 5. Full OAuth2 Error Hierarchy (0996.js)
The codebase implements a complete OAuth2 error class hierarchy aligned with RFC 6749: `invalid_request`, `invalid_client`, `invalid_grant`, `unauthorized_client`, `unsupported_grant_type`, `invalid_scope`, `access_denied`, `server_error`, `temporarily_unavailable`, plus token-specific errors `invalid_token` and `unsupported_token_type`. Each class has an `errorCode` static property and `toResponseObject()` method.

### 6. Client Authentication Method Negotiation (0996.js)
The OAuth2 client supports three `token_endpoint_auth_method` values: `client_secret_basic`, `client_secret_post`, and `none`. The `pnE()` function negotiates the method based on client capabilities and whether a `client_secret` is available.

## Code Examples

### DPoP JWT Generation (3059.js, line ~484)
```js
// createDPoPInterceptor middleware
// alg: "ES256", typ: "dpop+jwt"
// jwk: { kty: "EC", crv: "P-256", x: ..., y: ... }
// Payload: { jti: crypto.randomUUID(), htm: H, htu: A, iat: Math.floor(Date.now() / 1000) }
// Signature: sha256(Buffer.from(`${header}.${payload}`), privateKey)
```

### IdProvider Device Flow Polling (0870.js, line ~8)
```js
async function AEH({ deviceCode: H, expiresIn: A = 300, interval: L = 5 }) {
  let $ = AbortSignal.timeout(A * 1000), I = L;
  while (true) {
    let D = await fetch("https://auth.idprovider.com/user_management/authenticate", {
      method: "POST",
      body: new URLSearchParams({
        grant_type: "urn:ietf:params:oauth:grant-type:device_code",
        device_code: H,
        client_id: SxH,
      }),
      signal: $,
    }), E = await D.json();
    if (D.ok) return E;
    switch (E.error) {
      case "authorization_pending": await xR(I * 1000); break;
      case "slow_down": ((I += 1), await xR(I * 1000)); break;
      case "access_denied": case "expired_token": throw new vH("Authorization failed");
    }
  }
}
```

### MCP OAuth Token Storage (1135.js, line ~40)
```js
async saveTokens(H, A, L) {
  let $ = sEH(H, A), I = await this.load(), D = I.mcpOAuth[$];
  let E = L.expires_in ? Date.now() + L.expires_in * 1000 : void 0;
  let f = {
    serverName: H, serverUrl: A,
    accessToken: L.access_token, refreshToken: L.refresh_token,
    tokenType: L.token_type, expiresAt: E, scope: L.scope,
    updatedAt: Date.now(),
  };
  ((I.mcpOAuth[$] = f), await this.save(I));
}
```

### AWS STS AssumeRole with MFA (3035.js, line ~434)
```js
let P = {
  RoleArn: E.role_arn,
  RoleSessionName: E.role_session_name || `aws-sdk-js-${Date.now()}`,
  ExternalId: E.external_id,
  DurationSeconds: parseInt(E.duration_seconds || "3600", 10),
};
if (B) { // mfa_serial present
  if (!L.mfaCodeProvider)
    throw new P0L.CredentialsProviderError(`Profile ${H} requires multi-factor authentication...`);
  ((P.SerialNumber = B), (P.TokenCode = await L.mfaCodeProvider(B)));
}
```

### OAuth2 Error Class (0996.js, line ~1)
```js
XT = class XT extends Error {
  constructor(H, A) { super(H); ((this.errorUri = A), (this.name = this.constructor.name)); }
  toResponseObject() {
    let H = { error: this.errorCode, error_description: this.message };
    if (this.errorUri) H.error_uri = this.errorUri;
    return H;
  }
  get errorCode() { return this.constructor.errorCode; }
};
bEH = class bEH extends XT {}; bEH.errorCode = "invalid_grant";
UuH = class UuH extends XT {}; UuH.errorCode = "invalid_token";
```

## Integration Points (cross-system)

| Subsystem | Connection | Type |
|-----------|-----------|------|
| **MCP (03-tool-agent-system/Tool system)** | 1107.js + 1135.js provide OAuth-backed MCP client connections. Tool servers authenticate via OAuth2 before tool calls. | Protocol auth |
| **AWS SDK (infra-aws-sdk)** | 3035.js + 3059.js are the auth foundation for AWS service calls. `AssumeRole` outputs feed into S3/STS clients. | Credential chain |
| **Telemetry (infra-telemetry-otel)** | 1349.js captures auth headers in HTTP spans. Cross-cuts with infra-telemetry-otel for observability. | Observability |
| **HTTP Client (infra-http-client)** | 0870.js uses `fetch()` directly against IdProvider. 3059.js uses AWS SDK middleware stack. | Transport layer |
| **CLI Entry (infra-cli-entry)** | Device flow user prompts (authorization URL display) likely wired into CLI output. | User interaction |
| **Config Loader (infra-config-loader)** | AWS credential profiles (`~/.aws/credentials`, `~/.aws/config`) parsed by 3035.js. | Config source |

## Implementation Notes

1. **IdProvider Device Flow → Generic OAuth2 Device Flow**: Replace `auth.idprovider.com` endpoints with configurable `authorization_endpoint` and `token_endpoint`. The polling loop in 0870.js is standard RFC 8628 and portable.

2. **DPoP JWT → Reusable Module**: Extract the `createDPoPInterceptor` and JWT signing logic from 3059.js into a standalone `dpop.ts` module. It depends only on Node.js `crypto` and `Buffer`.

3. **MCP OAuth Storage → Pluggable Backend**: The `wOA` class in 1135.js already abstracts storage via `JOA` (keyring) + `QOA` (file). For OpenDroid, add a `keytar`-free fallback using OS credential APIs or encrypted flat files.

4. **AWS STS → Optional Dependency**: The AWS credential chain in 3035.js is AWS-specific. For OpenDroid, make it an optional plugin. The core interface (`credentials()`, `setCredentials()`) is generic enough to support other cloud providers.

5. **OAuth2 Errors → Standard Library**: The 0996.js error hierarchy is RFC 6749-compliant and should become a shared `@opendroid/auth-errors` package used by all OAuth flows.

## Module Reference

| File | Line | Function / Class | Description |
|-------|-------|-------------------|----------|
| 3059.js | 81 | `DEL extends cd.Client` | AWS Sign-in Service client |
| 3059.js | 484 | `createDPoPInterceptor(H)` | DPoP middleware injection |
| 3059.js | 512 | DPoP JWT header | `alg: "ES256"`, `typ: "dpop+jwt"` |
| 3035.js | 1 | `Fz extends ServiceException` | Base STS exception |
| 3035.js | 8 | `ExpiredTokenException` | Token expiry error |
| 3035.js | 434 | `bfD = (H) => !H.role_arn && !!H.credential_source` | Credential source profile check |
| 0870.js | 1 | `no()` | IdProvider device authorization request |
| 0870.js | 8 | `AEH({ deviceCode, expiresIn, interval })` | Token polling loop |
| 0870.js | 39 | `MX$(refresh_token, organization_id)` | Token refresh |
| 1135.js | 1 | `wOA` | MCP OAuth storage class |
| 1135.js | 40 | `saveTokens(serverName, serverUrl, tokens)` | Persist access/refresh tokens |
| 1135.js | 25 | `loadTokens(serverName, serverUrl)` | Load tokens with expiry check |
| 0996.js | 1 | `XT extends Error` | Base OAuth2 error |
| 0996.js | 40 | `bEH.errorCode = "invalid_grant"` | Invalid grant error |
| 0996.js | 78 | `cnE(token_endpoint_auth_method)` | Supported auth methods check |
| 0996.js | 82 | `pnE(client, supportedMethods)` | Auth method negotiation |
| 1107.js | 1 | `Zk extends iqA` | MCP client class |
| 1107.js | 115 | `connect(transport)` | Initialize handshake with capabilities |
| 1349.js | 1 | `hy$ extends InstrumentationBase` | HTTP instrumentation class |
| 1349.js | 434 | `_createHeaderCapture()` | Request/response header capture config |
