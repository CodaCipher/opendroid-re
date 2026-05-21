# infra-aws-sdk: Architecture Notes

## Overview

OpenDroid integrates AWS SDK v3 (version 3.926.0) primarily for two purposes: **S3-based software update distribution** and **SSO OIDC authentication**. The system uses a thin S3 client wrapper (`M3H` class in 0697.js) that exposes `fetchFileText` and `fetchFileBuffer` methods on top of the AWS SDK S3 `GetObjectCommand`. The SSO OIDC client (3074.js) handles token-based authentication with Sigv4 signing. The credential resolution chain follows the standard AWS SDK provider hierarchy: environment variables → INI profile (with ECS/Ec2/Environment credential sources) → container metadata → instance metadata. Smithy runtime v4 provides the middleware stack, serialization/deserialization, and HTTP signing.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 0701.js | 462 lines | Update manager: uses S3 client to check/download updates | App |
| 0697.js | ~45 lines | S3 client wrapper (`M3H`): fetchFileText/fetchFileBuffer | App |
| 0582.js | ~80 lines | S3 Client class (`e7`): extends Smithy Client, full middleware stack | Vendor (@aws-sdk/client-s3) |
| 0635.js | ~80 lines | GetObjectCommand (`y0H`): S3 GetObject operation with checksum validation | Vendor (@aws-sdk/client-s3) |
| 3074.js | 437 lines | SSO OIDC Client (`wEL`): token management, Sigv4 auth, error classes | Vendor (@aws-sdk/client-sso-oidc) |
| 0469.js | 146 lines | S3 package metadata + env credential provider (`AGA`) | Vendor (@aws-sdk/client-s3 + credential-provider-env) |
| 0532.js | ~50 lines | INI credential source resolver: EcsContainer/Ec2InstanceMetadata/Environment | Vendor (@aws-sdk/credential-provider-ini) |
| 0534.js | ~60 lines | STS auth scheme: AssumeRoleWithWebIdentity, Sigv4 config | Vendor (@aws-sdk/client-sts) |
| 0410.js | ~80 lines | S3 flexible checksums response validation middleware | Vendor (@aws-sdk/middleware-flexible-checksums) |

## Architecture

### Update Distribution via S3

The update system (0701.js, class `DO`) checks for new versions by reading a `LATEST` file from an S3 bucket. It constructs the S3 key using a configurable prefix (`s3KeyPrefix`) and downloads the update binary and its SHA-256 checksum via S3. The S3 client wrapper (`M3H` in 0697.js) lazy-initializes a single `e7` (S3Client) instance with region `us-west-1` and provides two methods:

1. **`fetchFileText(bucket, key)`**: returns file content as string (for LATEST version file + checksums)
2. **`fetchFileBuffer(bucket, key)`**: returns file content as Buffer (for binary update downloads)

The update manager falls back to HTTP `fetch()` when S3 is not configured (no `s3Bucket` in remote config).

### S3 Client Architecture (Smithy-based)

The S3 Client (`e7` in 0582.js) extends Smithy's `Client` base class and configures a full middleware stack:

1. `getUserAgentPlugin`: AWS SDK user-agent injection
2. `getRetryPlugin`: automatic retry with configurable policy
3. `getContentLengthPlugin`: content-length header
4. `getHostHeaderPlugin`: host header resolution
5. `getLoggerPlugin`: request/response logging
6. `getRecursionDetectionPlugin`: recursion detection header
7. `getHttpAuthSchemeEndpointRuleSetPlugin`: Sigv4/Sigv4a authentication
8. `getHttpSigningPlugin`: HTTP request signing
9. S3-specific plugins: bucket-endpoint, expect-continue, flexible-checksums, SSE-C, location-constraint, session endpoints

The `GetObjectCommand` (0635.js) uses `ep()` (endpoint parameters) with `Bucket` and `Key` as context params, flexible checksum validation (CRC64NVME, CRC32, CRC32C, SHA256, SHA1), and full serialization/deserialization.

### SSO OIDC Authentication

The SSO OIDC Client (`wEL` in 3074.js) handles token-based authentication for AWS SSO:

- **Token lifecycle**: `CreateTokenCommand` with `refresh_token` grant type
- **Error classes**: `AccessDeniedException`, `AuthorizationPendingException`, `ExpiredTokenException`
- **Credential config**: Sigv4 signing with `sso-oauth` as default signing name
- **Region/endpoint resolution**: Standard AWS SDK region config chain with FIPS and dual-stack support
- **Token refresh**: Automatic via `D_D` function that creates a new SSO OIDC client and calls `CreateTokenCommand`

### Credential Resolution Chain

The credential provider chain follows AWS SDK standard hierarchy:

1. **Environment variables** (0469.js): `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_CREDENTIAL_EXPIRATION`, `AWS_CREDENTIAL_SCOPE`, `AWS_ACCOUNT_ID`
2. **INI profile** (0532.js): Reads `~/.aws/credentials` and `~/.aws/config`, with configurable credential sources:
   - `EcsContainer`: from ECS task metadata (HTTP + container metadata chain)
   - `Ec2InstanceMetadata`: from EC2 IMDS
   - `Environment`: from env vars
3. **STS Assume Role** (0534.js): `AssumeRoleWithWebIdentity` operation (no Sigv4 auth required for web identity), standard Sigv4 for other STS operations

### Smithy Runtime

Smithy runtime (v4) provides the foundational infrastructure:

- **Middleware Stack** (`@smithy/middleware-stack`): Ordered plugin system for request/response processing
- **Serialization/Deserialization** (`@smithy/middleware-serde`): XML/JSON protocol handling for AWS services
- **HTTP Handler** (`@smithy/node-http-handler` / `@smithy/fetch-http-handler`): Node.js HTTP/HTTPS transport
- **Protocol HTTP** (`@smithy/protocol-http`): HTTP request/response model
- **Auth Schemes** (`@smithy/core`): Sigv4/Sigv4a signing integration
- **Checksum validation** (`@aws-sdk/middleware-flexible-checksums`, 0410.js): Response payload integrity verification

## Key Findings

1. **S3 is update-only**: S3 is used exclusively for software update distribution (checking versions, downloading binaries, verifying checksums). No general-purpose S3 storage operations exist.

2. **Hardcoded region**: The S3 client wrapper initializes with `region: "us-west-1"`: this is not configurable at runtime and assumes all update buckets are in us-west-1.

3. **Dual download strategy**: The update manager (0701.js) supports both S3 and HTTP-based update distribution, switching based on whether `s3Bucket` is configured in remote config.

4. **SSO OIDC for authentication**: The system uses AWS SSO OIDC for identity-based authentication rather than static access keys, indicating an enterprise SSO-integrated deployment.

5. **Sigv4a support**: The S3 client supports both Sigv4 and Sigv4a (multi-region) signing, suggesting S3 multi-region access point capability.

6. **Full checksum validation**: S3 responses are validated with flexible checksums (CRC64NVME, CRC32, CRC32C, SHA256, SHA1), and update binaries are verified against SHA-256 checksums before staging.

7. **Standard credential chain**: The application uses the standard AWS SDK credential provider chain with environment variables, INI profiles, container metadata, and instance metadata support.

## Code Examples

### S3 Client Wrapper (0697.js)
```js
class M3H {
  s3Client;
  constructor() {
    this.s3Client = new e7({ apiVersion: "latest", region: "us-west-1" });
  }
  async fetchFileText(H, A) {
    let L = new y0H({ Bucket: H, Key: A }),
      { Body: $ } = await this.s3Client.send(L);
    if (!$) throw new vH("Failed to download file as string", { bucket: H, key: A });
    return await $.transformToString();
  }
  async fetchFileBuffer(H, A) {
    let L = new y0H({ Bucket: H, Key: A }),
      { Body: $ } = await this.s3Client.send(L);
    if (!$) throw new vH("Failed to download file as buffer", { bucket: H, key: A });
    let I = $;
    return await new Promise((E, f) => {
      let M = [];
      I.on("data", (U) => M.push(Buffer.from(U)));
      I.on("error", (U) => f(U));
      I.on("end", () => E(Buffer.concat(M)));
    });
  }
  destroy() { this.s3Client.destroy(); }
}
```

### Environment Credential Provider (0469.js, lines 120-135)
```js
var _kH = "AWS_ACCESS_KEY_ID",
  PkH = "AWS_SECRET_ACCESS_KEY",
  HH$ = "AWS_SESSION_TOKEN",
  AH$ = "AWS_CREDENTIAL_EXPIRATION";
var AGA = (H) => async () => {
  H?.logger?.debug("@aws-sdk/credential-provider-env - fromEnv");
  let A = process.env[_kH],
    L = process.env[PkH],
    $ = process.env[HH$],
    I = process.env[AH$];
  // ... validates and returns credentials
};
```

### STS Auth Scheme (0534.js, lines 36-46)
```js
var kjE = (H) => {
  let A = [];
  switch (H.operation) {
    case "AssumeRoleWithWebIdentity": { A.push(yjE(H)); break; }
    default: A.push(SjE(H));  // Sigv4 with "sts" signing name
  }
  return A;
};
```

## Integration Points (cross-system)

- **infra-update-checker**: S3 client is used by the update manager (0701.js) for version checking and binary downloads
- **infra-auth**: SSO OIDC client provides token-based authentication; STS AssumeRole supports web identity federation
- **infra-http-client**: Smithy HTTP handler layer sits below the AWS SDK; Node.js HTTP handler used for SDK transport
- **infra-telemetry-otel**: AWS SDK logger plugin (`getLoggerPlugin`) can integrate with the OpenTelemetry tracing subsystem
- **infra-config-loader**: S3 bucket and key prefix come from remote config (`config.remoteConfig.s3Bucket`, `config.remoteConfig.s3KeyPrefix`)
- **infra-errors-and-runtime**: SSO OIDC error classes (AccessDeniedException, AuthorizationPendingException, ExpiredTokenException) extend Smithy's ServiceException

## Implementation Notes

1. **Replace S3 client wrapper**: The `M3H` class (0697.js) is the main integration point. For OpenDroid, either keep AWS SDK v3 as a dependency or replace with an abstract storage interface (e.g., GCS, Azure Blob, or local filesystem).

2. **Credential management**: If not using AWS, the entire credential chain (0469.js, 0532.js, 0534.js) can be replaced with the target cloud provider's auth. The Smithy credential provider interface is pluggable.

3. **Update distribution**: The `shouldUseS3Client()` check in 0701.js already provides an HTTP fallback path. OpenDroid can use HTTP-only updates or integrate with the target platform's update mechanism.

4. **SSO OIDC replacement**: The SSO OIDC client (3074.js) is vendor-specific. Replace with the target identity provider's OIDC client (e.g., Google Identity, Azure AD).

5. **Smithy runtime**: If keeping AWS SDK, Smithy runtime modules (0582.js, 0635.js, 0410.js) work as-is. If replacing AWS entirely, these can be removed.

## Module Reference (file + line + function)

| Module | Line | Function/Class | Description |
|--------|------|---------------|-------------|
| 0701.js | 15 | `DO.awsS3Client` | S3 client instance field |
| 0701.js | 42 | `DO.shouldUseS3Client()` | Checks if S3 is configured |
| 0701.js | 45 | `DO.getS3Key(H)` | Builds S3 key from prefix |
| 0701.js | 59-68 | `DO.checkForUpdates()` | Fetches LATEST version via S3 |
| 0701.js | 117-123 | `DO.downloadAndStageUpdate()` | Downloads binary via S3 |
| 0701.js | 141-146 | `DO.downloadAndStageUpdate()` | Downloads checksum via S3 |
| 0697.js | 3-9 | `M3H.constructor()` | Initializes S3Client with us-west-1 |
| 0697.js | 10-15 | `M3H.fetchFileText(bucket, key)` | GetObject → string |
| 0697.js | 16-26 | `M3H.fetchFileBuffer(bucket, key)` | GetObject → Buffer via stream |
| 0697.js | 27 | `M3H.destroy()` | Cleanup S3 client |
| 0582.js | 14-65 | `e7.constructor()` | S3Client with full middleware stack |
| 0635.js | 9-30 | `y0H` | GetObjectCommand with checksum validation |
| 3074.js | 27-29 | `znf` | Built-in params (UseFIPS, Endpoint, Region, UseDualStack) |
| 3074.js | 32-63 | `Nnf(H)` | Auth scheme configuration (credentials, httpAuthSchemes) |
| 3074.js | 84-112 | `wEL.constructor()` | SSO OIDC Client with middleware stack |
| 3074.js | 126-139 | `B$A` | AccessDeniedException error class |
| 3074.js | 408-416 | `I_D` | SSO OIDC client factory function |
| 3074.js | 417-426 | `D_D` | Token refresh via CreateTokenCommand |
| 0469.js | 1-80 | `aeL` | @aws-sdk/client-s3 package metadata (v3.926.0) |
| 0469.js | 122-135 | `AGA` | Environment credential provider (fromEnv) |
| 0532.js | 8-38 | `e$$` | INI credential source resolver (ECS/EC2/Env) |
| 0534.js | 36-46 | `kjE` | STS auth scheme provider (Sigv4 + noAuth for WebIdentity) |
| 0534.js | 46 | `bjE` | STS client constructor injection |
| 0410.js | 8-35 | `hpL` | S3 flexible checksums response validation middleware |
