# infra-crypto: Architecture Notes

## Overview

OpenDroid's crypto subsystem is built on Node.js `crypto` module and two high-level JWT/JWS libraries: **jsonwebtoken** (module 2735) and **jose v5.9.6** (modules 0831, 0833, 0834, 0847, 0849, 0850). The system provides JWT signing/verification with full algorithm support (HS256, RS256, PS256, ES256, EdDSA, and variants), AES-256-GCM encryption for credential storage, system certificate extraction for TLS trust, and various hashing utilities. A Google Cloud `NodeCrypto` adapter (2718) provides SHA-256, HMAC-SHA256, and RSA-SHA256 sign/verify operations. The `object-hash` library (2764) offers deterministic hashing of arbitrary JavaScript objects. Encryption at rest is handled by two key management patterns: keyring-based (0866) and scrypt-derived (1131), both using AES-256-GCM with authentication tags.

## Module Map

| Module | Size | Lines | Role | Category |
|--------|------|-------|------|----------|
| 2735.js | 159L | JWT crypto core | jsonwebtoken sign/verify | App |
| 0831.js | 214L | JOSE error classes + CryptoKey validation | jose errors | App |
| 0833.js | 115L | JWK import key mapping (RSA/EC/OKP) | jose JWK | App |
| 0834.js | ~120L | KeyObject/JWK to CryptoKey conversion | jose keys | App |
| 0847.js | 100L | JWKS (JSON Web Key Set) local resolver | jose JWKS | App |
| 0849.js | 100L | JWKS remote fetching with caching | jose JWKS | App |
| 0850.js | ~5L | jose entry point (user-agent, Symbol cache) | jose config | App |
| 2718.js | ~40L | Google Cloud NodeCrypto adapter | GCP crypto | App |
| 2764.js | 302L | object-hash (deterministic object hashing) | utility | App |
| 0445.js | ~20L | AWS SSO token filepath (SHA-1 hash) | AWS credential | App |
| 0866.js | 180L | Encrypted credential store (keyring-based AES-256-GCM) | security | App |
| 1131.js | 135L | MCP OAuth encrypted storage (scrypt + AES-256-GCM) | security | App |
| 0371.js | ~10L | crypto.randomUUID wrapper | utility | App |
| 0960.js | ~20L | SHA-1 digest (git-style hashing) | utility | App |
| 4071.js | 1674L | System certificate extraction + TLS trust store | TLS/certs | App |
| 3059.js | 532L | AWS Signin service client (Smithy SDK auth) | AWS auth | App |

**Note on seed modules:** The originally assigned seed modules (2689, 3837, 3344, 2594, 3902, 3337) were determined to be non-crypto (gRPC load balancing, shell utilities, Vim/TypeScript syntax highlighters, Unicode utilities). Actual crypto modules were discovered via targeted grep and analyzed instead.

## Architecture

### Crypto Layer Stack

```
┌──────────────────────────────────────────────┐
│          Application Layer                    │
│  (Auth, MCP OAuth, AWS creds, TLS trust)     │
├──────────────────────────────────────────────┤
│        High-level Crypto Libraries            │
│  ┌─────────────────┐ ┌──────────────────────┐│
│  │  jsonwebtoken    │ │  jose v5.9.6         ││
│  │  (2735.js)       │ │  (0831-0850)         ││
│  │  JWT sign/verify │ │  JWS/JWE/JWK/JWKS    ││
│  └─────────────────┘ └──────────────────────┘│
├──────────────────────────────────────────────┤
│       Encryption at Rest Layer                │
│  ┌─────────────────┐ ┌──────────────────────┐│
│  │ Keyring-based   │ │  Scrypt-derived      ││
│  │ (0866.js)       │ │  (1131.js)           ││
│  │ AES-256-GCM     │ │  AES-256-GCM         ││
│  └─────────────────┘ └──────────────────────┘│
├──────────────────────────────────────────────┤
│       Utility Layer                           │
│  object-hash (2764) │ SHA-1 (0960) │ UUID   │
│  AWS SSO hash (0445)│ GCP NodeCrypto (2718) │
├──────────────────────────────────────────────┤
│     System Certificate Extraction (4071)      │
│  Windows/macOS/Linux cert store → PEM cache   │
├──────────────────────────────────────────────┤
│        Node.js `crypto` module (native)       │
└──────────────────────────────────────────────┘
```

### JWT Signing Architecture (jsonwebtoken: 2735.js)

The `jsonwebtoken` library provides a unified `jwsSign(algorithm)` factory. Given an algorithm string like `"RS256"`, it parses the family prefix (`RS`, `hs`, `ps`, `es`, `none`) and bit length (256/384/512), then returns `{ sign, verify }` functions using the appropriate Node.js crypto primitive:
- **HS*** → `crypto.createHmac("sha*", secret)`
- **RS*** → `crypto.createSign("RSA-SHA*", key)` / `crypto.createVerify("RSA-SHA*", key)`
- **PS*** → `crypto.createSign("RSA-SHA*", { padding: RSA_PKCS1_PSS_PADDING, saltLength: RSA_PSS_SALexternal datasetN_DIGEST })`
- **ES*** → Same as RS, with DER↔JOSE conversion via `ecdsa-sig-formatter` (ieA dependency)

### JOSE Library Architecture (0831-0850)

The `jose` v5.9.6 library provides a Web Crypto API-compatible implementation with Node.js KeyObject bridging:
1. **Error hierarchy** (0831): `JOSEError` → `JWTClaimValidationFailed`, `JWTExpired`, `JOSEAlgNotAllowed`, `JOSENotSupported`, `JWEDecryptionFailed`, `JWSInvalid`, `JWTInvalid`, `JWKInvalid`, `JWKSInvalid`, `JWKSNoMatchingKey`, `SignatureVerificationFailed`
2. **JWK import** (0833): Maps JWK `kty`+`alg` to Web Crypto `algorithm` parameters. Supports RSA (PKCS1, PSS, OAEP), EC (P-256/384/521, ECDH), OKP (Ed25519/Ed448). Detects private vs public key via `d` field presence.
3. **Key bridging** (0834): Converts Node.js `KeyObject` → CryptoKey via `subtle.importKey("jwk", ...)`. Uses WeakMap caches for performance.
4. **JWKS resolver** (0847): Local JWKS key set with `getKey(alg, kid)` matching. Handles `kty`, `kid`, `alg`, `use`, `key_ops`, and `crv` filtering.
5. **JWKS fetcher** (0849): Remote JWKS endpoint with HTTP fetch, cooldown (30s), cache max-age (600s), and timeout (5s). Supports Cloudflare/Vercel edge runtimes.

### Encryption at Rest

Two patterns for encrypting sensitive data on disk:

**Pattern 1: Keyring-based (0866.js: `y2A` class)**
- Uses OS keychain (via `keyring` npm package) to store a 256-bit encryption key
- Encryption: `randomBytes(12)` IV → `createCipheriv("aes-256-gcm", key, iv)` → `iv:authTag:ciphertext` (base64)
- Key lifecycle: check keyring → if valid key found, use it → otherwise generate new `randomBytes(32)` → store in keyring
- Used for credential/token file encryption

**Pattern 2: Scrypt-derived (1131.js: `FOA` class)**
- Stores a 32-byte random secret in `<file>.secret` alongside the encrypted file
- Derives encryption key via `scryptSync(secret, salt, 32)` where salt = `hostname-username-opendroid`
- Same AES-256-GCM format but uses hex encoding: `iv:authTag:ciphertext` (hex)
- Used specifically for MCP OAuth credential storage
- Atomic writes: writes to `.tmp` file then renames

### System Certificate Extraction (4071.js)

Windows-specific certificate extraction for TLS trust:
1. Counts certificates via PowerShell (Cert:\ stores + HKLM registry)
2. Caches extracted PEM certificates in `~/.opendroid/certs/` with versioned JSON format
3. Cache invalidation: platform check (win32 only), version check, age check, count comparison
4. PEM validation: checks `BEGIN/END CERTIFICATE` markers, base64 validity, DER ASN.1 tag (byte 0x30), minimum 200 bytes
5. Extracts from: `Cert:\LocalMachine\Root`, `Cert:\LocalMachine\CA`, HKLM registry stores (Root, CA, AuthRoot)

### Object Hashing (2764.js)

The `object-hash` library provides deterministic hashing of any JavaScript value:
- Default: SHA-1 hex digest
- Options: algorithm (all Node.js hashes), encoding (buffer/hex/binary/base64), excludeValues, respectType, respectFunctionNames, unorderedArrays/Sets/Objects
- Used for content-addressing, cache keys, and comparison

## Key Findings

1. **Dual JWT library usage**: OpenDroid uses both `jsonwebtoken` (module 2735) and `jose v5.9.6` (modules 0831-0850). The `jsonwebtoken` library is the classic Node.js approach using `crypto.createSign/createVerify` directly. The `jose` library is Web Crypto API compatible and supports JWKS remote key sets. Both coexist for different use cases: `jsonwebtoken` for local JWT operations, `jose` for standards-compliant OIDC/JWKS verification.

2. **Full algorithm coverage**: The system supports HS256/384/512 (HMAC), RS256/384/512 (RSA-PKCS1), PS256/384/512 (RSA-PSS), ES256/384/512 (ECDSA P-256/384/521), EdDSA (Ed25519/Ed448), and `none` (unsigned). This covers all mainstream JWT/JWS algorithms.

3. **Two encryption-at-rest patterns**: Keyring-based (0866) stores the encryption key in the OS keychain, while scrypt-derived (1131) stores a secret file and derives the key using scrypt KDF. Both use AES-256-GCM with authentication tags, providing confidentiality and integrity.

4. **System certificate extraction is Windows-optimized**: Module 4071.js has a sophisticated PowerShell-based certificate extraction pipeline with caching, PEM validation (DER structure check), and atomic file operations. This feeds into TLS trust for outgoing HTTPS connections.

5. **GCP NodeCrypto adapter provides sign/verify**: Module 2718.js wraps Node.js crypto for Google Cloud auth needs: `sha256DigestBase64`, `randomBytesBase64`, `verify` (RSA-SHA256), `sign` (RSA-SHA256), `signWithHmacSha256`, and base64 encode/decode.

6. **Seed module mismatch**: The originally assigned seed modules (2689, 3837, 3344, 2594, 3902, 3337) are not crypto-related. They are gRPC, shell utilities, syntax highlighters, and Unicode utilities. This report analyzes the actual crypto modules discovered via grep.

## Code Examples

### JWT Sign/Verify OpenDroid (2735.js, lines 20-159)
```js
// Algorithm dispatch factory
AcI.exports = function (A) {
  var L = { hs: sgI, rs: egI, ps: HJf, es: LJf, none: IJf },
    $ = { hs: e5f, rs: HcI, ps: AJf, es: $Jf, none: DJf },
    I = A.match(/^(RS|PS|ES|HS)(256|384|512)$|^(none)$/);
  if (!I) throw _z(o5f, A);
  var D = (I[1] || I[3]).toLowerCase(), E = I[2];
  return { sign: L[D](E), verify: $[D](E) };
};
```

### RSA-PSS Signing with PSS Padding (2735.js, lines 83-97)
```js
function HJf(H) {
  return function (L, $) {
    (tgI($), (L = ZCH(L)));
    var I = Wj.createSign("RSA-SHA" + H),
      D = (I.update(L), I.sign({
        key: $,
        padding: Wj.constants.RSA_PKCS1_PSS_PADDING,
        saltLength: Wj.constants.RSA_PSS_SALexternal datasetN_DIGEST,
      }, "base64"));
    return DHL(D);
  };
}
```

### AES-256-GCM Encryption (0866.js, lines 64-71)
```js
encrypt(H, A) {
  let L = jxH.randomBytes(AX$),  // 12-byte IV
    $ = jxH.createCipheriv("aes-256-gcm", A, L),
    I = Buffer.concat([$.update(H, "utf8"), $.final()]),
    D = $.getAuthTag();
  return `${L.toString("base64")}:${D.toString("base64")}:${I.toString("base64")}`;
}
```

### Scrypt Key Derivation (1131.js, lines 52-58)
```js
async deriveEncryptionKey() {
  let H = await this.loadOrGenerateSecret(),
    A = `${$Y$.hostname()}-${$Y$.userInfo().username}-opendroid`,
    L = FGH.scryptSync(H, A, 32);
  return ((this.encryptionKeyCache = L), L);
}
```

### GCP NodeCrypto Adapter (2718.js, lines 8-38)
```js
class euI {
  async sha256DigestBase64(H) {
    return OUH.createHash("sha256").update(H).digest("base64");
  }
  async verify(H, A, L) {
    let $ = OUH.createVerify("RSA-SHA256");
    return ($.update(A), $.end(), $.verify(H, L, "base64"));
  }
  async signWithHmacSha256(H, A) {
    let L = typeof H === "string" ? H : sQf(H);
    return aQf(OUH.createHmac("sha256", L).update(A).digest());
  }
}
```

### Certificate PEM Validation (4071.js, lines 770-788)
```js
function OpA(H) {
  try {
    if (!H.includes("-----BEGIN CERTIFICATE-----")) return false;
    let A = H.match(/-----BEGIN CERTIFICATE-----\s*([\s\S]+?)\s*-----END CERTIFICATE-----/);
    if (!A || !A[1]) return false;
    let L = A[1].replace(/\s/g, "");
    if (!/^[A-Za-z0-9+/]+=*$/.test(L)) return false;
    let $ = Buffer.from(L, "base64");
    if ($.length < 4 || $[0] !== 48) return false;  // ASN.1 SEQUENCE tag
    if ($.length < 200) return false;
    return true;
  } catch { return false; }
}
```

## Integration Points

### Cross-System Dependencies

- **infra-auth (VAL-INF-010)**: The JWT crypto layer (2735.js) is the foundational signing/verification engine used by the auth subsystem. Token creation, validation, and refresh all depend on these crypto primitives.
- **infra-aws-sdk (VAL-INF-012)**: Module 0445.js provides AWS SSO token file path hashing (SHA-1 of start URL → filename). Module 3059.js provides the AWS Signin service client using Smithy SDK auth with `defaultSigningName: "signin"`.
- **infra-gcp-client (VAL-INF-013)**: Module 2718.js provides `NodeCrypto` adapter used by Google Cloud SDK for request signing and HMAC authentication.
- **infra-http-client (VAL-INF-011)**: Module 4071.js's system certificate extraction feeds TLS trust roots into the HTTP client layer for secure outbound connections.
- **infra-config-loader (VAL-INF-004)**: Encrypted credential stores (0866.js, 1131.js) persist sensitive configuration (API keys, OAuth tokens) that the config loader reads.
- **Plugin system (M2/Mission)**: The `randomUUID()` wrapper (0371.js) may be used for generating unique session/task IDs in the mission orchestration layer.

### External Library Dependencies

- `jsonwebtoken`: JWT creation and verification
- `jose` v5.9.6: JOSE framework (JWS, JWE, JWK, JWKS)
- `ecdsa-sig-formatter`: DER↔JOSE signature format conversion
- `object-hash`: Deterministic JavaScript object hashing
- Node.js `crypto`: Core cryptographic primitives

## Implementation Notes

1. **JWT handling**: Replace `jsonwebtoken` + `jose` with equivalent Rust crates (`jsonwebtoken`, `jose-jwk`). The algorithm dispatch pattern in 2735.js maps directly to enum-based signing in Rust.

2. **Encryption at rest**: Use `ring` or `aes-gcm` crate for AES-256-GCM. The keyring pattern (0866.js) maps to OS keychain access via `keyring-rs`. The scrypt pattern (1131.js) maps to `scrypt` crate from RustCrypto.

3. **Certificate extraction**: On Windows, use `rustls` with `webpki-roots` or `rustls-native-certs` crate. Module 4071.js's PowerShell approach should be replaced with native WinAPI cert store access via `windows-sys` crate.

4. **Object hashing**: Replace `object-hash` with a deterministic serialization format (e.g., `serde_json` with sorted keys) + SHA-256 digest via `sha2` crate.

5. **Key management**: The current dual-pattern (keyring vs scrypt) should be unified in the port. Recommend using a single key derivation + OS keychain storage approach.

## Module Reference

| Module | Line(s) | Function/Class | Description |
|--------|---------|----------------|-------------|
| 2735.js | 5 | `Wj = xH("crypto")` | Node.js crypto import |
| 2735.js | 14-15 | `sgI(H)` | HMAC-SHA sign function factory |
| 2735.js | 20-25 | `egI(H)` | RSA-SHA sign function factory |
| 2735.js | 81-95 | `HJf(H)` | RSA-PSS sign function factory |
| 2735.js | 97-109 | `LJf(H)` | ECDSA sign with DER→JOSE conversion |
| 2735.js | 139-155 | `module.exports` | Main JWS algorithm dispatch |
| 0831.js | 3-30 | `WT` (JOSEError) | Base error class for JOSE |
| 0831.js | 32-40 | `D6` (JWTClaimValidationFailed) | JWT claim validation error |
| 0831.js | 42-50 | `r0H` (JWTExpired) | JWT expiration error |
| 0831.js | 113-120 | `vT$(H, A, ...L)` | CryptoKey algorithm validation |
| 0833.js | 22-60 | `auE(H)` | JWK → Web Crypto algorithm mapping |
| 0833.js | 97-103 | `suE` | JWK → CryptoKey import function |
| 0834.js | 10-20 | `euE(H, A)` | KeyObject/JWK → CryptoKey (public) |
| 0834.js | 22-32 | `HgE(H, A)` | KeyObject/JWK → CryptoKey (private) |
| 0847.js | 18-25 | `MgE(H)` | Algorithm → JWK kty mapping |
| 0847.js | 35-75 | `GV$` class | JWKS local key resolver |
| 0849.js | 30-80 | `wV$` class | JWKS remote fetcher with cache |
| 0850.js | 4-5 | `F2A, NxH` | jose user-agent string + cache Symbol |
| 2718.js | 8-35 | `euI` (NodeCrypto) | GCP crypto adapter class |
| 2718.js | 12-14 | `sha256DigestBase64` | SHA-256 → base64 |
| 2718.js | 16-19 | `verify(H, A, L)` | RSA-SHA256 signature verify |
| 2718.js | 20-23 | `sign(H, A)` | RSA-SHA256 signature creation |
| 2718.js | 30-33 | `signWithHmacSha256` | HMAC-SHA256 signing |
| 2764.js | 7-8 | `yCH(H, A)` | object-hash main function |
| 2764.js | 19-24 | `EmI(H, A)` | Options normalization |
| 2764.js | 65-70 | `R7f(H, A)` | Hash dispatch (write to stream) |
| 0445.js | 5-9 | `IJE(H)` | SSO token filepath (SHA-1 hash) |
| 0866.js | 8-12 | `y2A` class | Keyring-based encrypted store |
| 0866.js | 35-50 | `loadOrCreateEncryptionKey` | Keyring key management |
| 0866.js | 64-71 | `encrypt(H, A)` | AES-256-GCM encrypt |
| 0866.js | 73-80 | `decrypt(H, A)` | AES-256-GCM decrypt |
| 1131.js | 8-12 | `FOA` class | MCP OAuth encrypted storage |
| 1131.js | 23-30 | `loadOrGenerateSecret` | 32-byte random secret management |
| 1131.js | 52-58 | `deriveEncryptionKey` | scrypt KDF key derivation |
| 1131.js | 59-68 | `encrypt(H)` | AES-256-GCM encrypt (hex format) |
| 1131.js | 69-80 | `decrypt(H)` | AES-256-GCM decrypt (hex format) |
| 4071.js | 741-745 | `eGI()` | Windows cert count (PowerShell) |
| 4071.js | 770-788 | `OpA(H)` | PEM certificate validation |
| 4071.js | 811-820 | `HFI(H, A)` | Certificate cache validation |
| 4071.js | 833-845 | `LFI(H)` | PEM text normalization |
| 4071.js | 847-860 | `FnH(H)` | Extract individual PEM certs from bundle |
| 4071.js | 868-880 | `IFI(H, A)` | Save cert cache to disk |
| 4071.js | 901-960 | `La1()` | Windows cert extraction (PowerShell) |
| 3059.js | 18-20 | `olf(H)` | AWS Signin client config |
| 3059.js | 40-55 | `alf(H)` | HTTP auth scheme configuration |
