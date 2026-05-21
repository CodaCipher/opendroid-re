# infra-fs-helpers: Architecture Notes

## Overview

OpenDroid implements a robust filesystem helper layer providing atomic write operations, secure file permissions, temp file management, and path resolution utilities. The core pattern is **write-to-temp → fsync → atomic-rename**, which guarantees data integrity even during crashes. This pattern is used across OAuth credential storage (1131.js), encrypted auth file storage (0866.js), and the general-purpose atomic write utility (0861.js). Secure file permissions (mode 0o600 for files, 0o700 for directories) are consistently enforced. Temp file cleanup is managed through a centralized `m3H` temp manager with automatic purge on process exit. Path resolution follows a simple convention: `getOpenDroidHome()` returns `OPENDROID_HOME_OVERRIDE || os.homedir()`, and all data is stored under `~/.opendroid/`.

## Module Map

| Module | Size | Rol | Category |
|--------|------|-----|----------|
| 1131.js | 135 lines | OAuth credential store with atomic write | Core |
| 0866.js | 180 lines | Encrypted auth file store with keyring | Core |
| 0861.js | 112 lines | General-purpose atomic write utility (`j2A`) | Core |
| 0865.js | 98 lines | Secure file/directory permissions (`rK`, `lo`, `Ic`, `M6`) | Core |
| 0201.js | 85 lines | Path resolution: `getOpenDroidHome()`, terminal detection | Core |
| 3836.js | 133 lines | Image temp file manager with cleanup | App |
| 0199.js | 174 lines | Telemetry log file with `ensureDirectoryExists` | App |
| 0203.js | 174 lines | CLI telemetry client initialization, log dir creation | App |
| 1620.js | 90 lines | Node.js fs function categorization (promise/callback/sync) | Vendor |
| 1624.js | 120 lines | OpenTelemetry fs instrumentation (rename, copy, mkdir) | Vendor |

## Architecture

### Atomic Write Pattern

The system implements a three-tier atomic write architecture:

1. **General-purpose atomic write** (`j2A` in 0861.js): Full-featured with temp file scheduling, fsync, retry logic, chown/chmod preservation, and long-path truncation. Uses a `m3H` temp file manager that creates files with pattern `{path}.tmp-{timestamp}{random}` and purges on process exit via `ThH()` (process exit handler).

2. **Simplified atomic write** (1131.js, 0866.js): Used in credential/auth storage. Pattern: `writeFile(path.tmp, data, {mode: 0o600})` → `rename(path.tmp, path)`. Simpler but effective for small files.

3. **Secure permissions layer** (0865.js): `setSecureFilePermissions` (0o600) and `setSecureDirectoryPermissions` (0o700) applied after writes. `ensureAllSecurePermissions()` runs on startup to fix all `.opendroid/` subdirectory permissions.

### Temp File Management

The `m3H` temp manager (0861.js) handles:
- **Creation**: Generates unique filenames with `.tmp-{timestamp}{random6}` suffix
- **Tracking**: In-memory store tracks active temp files
- **Purge**: `purgeSync()` deletes individual files; `purgeSyncAll()` cleans all on process exit
- **Truncation**: Handles ENAMETOOLONG by truncating the base filename while preserving the suffix pattern

### Path Resolution

- `getOpenDroidHome()` (0201.js): Returns `process.env.OPENDROID_HOME_OVERRIDE || os.homedir()`
- `sL` (OPENDROID_DIR_NAME constant): `.opendroid` directory name
- Standard layout: `~/.opendroid/{sessions,updates,logs,droids,config.json,auth.json,mcp.json,settings.json,history.json}`
- Temp dirs: `~/.opendroid/temp/extensions/`, `~/.opendroid/temp/images/{sessionId}/`

## Key Findings

### 1. Robust Atomic Write with Retry and Fallback (0861.js)
The `j2A` function implements industrial-grade atomic writes: creates temp file → writes data → fsyncs → renames to target. If rename fails with ENAMETOOLONG, it truncates the filename and retries. Uses `YX.retry.*` wrappers for all fs operations, providing automatic retry on transient failures.

### 2. Consistent Secure Permission Model (0865.js)
File mode 0o600 (owner read/write only) and directory mode 0o700 (owner full access) are used consistently. The `ensureAllSecurePermissions()` function runs at startup, walking `~/.opendroid/` and securing all known config files, session files, and droid configs. This is defense-in-depth against misconfigured permissions.

### 3. Encryption-First Credential Storage (1131.js, 0866.js)
Both credential stores use AES-256-GCM encryption with scrypt-derived keys. The atomic write pattern ensures credentials are never in a partially-written state. 1131.js uses machine-bound keys (hostname + username), while 0866.js optionally integrates with OS keyring (keytar/safeStorage).

### 4. Temp File Lifecycle Management (0861.js, 3836.js)
The `m3H` manager registers `purgeSyncAll` on process exit. Image manager (3836.js) maintains its own temp dir under `~/.opendroid/temp/images/{id}/` with explicit `cleanup()` that removes individual files then rmdirs the directory.

### 5. Directory Bootstrap Pattern
Multiple modules implement `ensureDirectoryExists()`: mkdir with `{recursive: true}` before any write operation. This is idempotent and handles both first-run and existing-directory cases.

## Code Examples

### Atomic Write Pattern (simplified: 1131.js, line 34-36)
```js
async saveSecret(H) {
    await zk.mkdir(IY$.dirname(this.secretFilePath), { recursive: true });
    let A = `${this.secretFilePath}.tmp`;
    await zk.writeFile(A, H, { mode: 384 });  // 0o600
    await zk.rename(A, this.secretFilePath);
}
```

### General-Purpose Atomic Write (0861.js, line 47-60)
```js
function j2A(H, A, L, $) {
  // H=target, A=data, L=options, $=callback
  let I = lV$(H, A, L);  // core async implementation
  if ($) I.then($, $);
  return I;
}
// Internally: open(tmp, 'w') → write → fsync → close → rename(tmp, target)
```

### Secure Permissions (0865.js, line 17-28)
```js
async function rK(H) {  // setSecureFilePermissions
  await tV$(H, aV$);    // chmod(path, 0o600)
}
async function lo(H) {  // setSecureDirectoryPermissions
  await tV$(H, sV$);    // chmod(path, 0o700)
}
```

### Temp File Manager (0861.js, line 9-23)
```js
A2 = {
  create: (H) => {
    let A = `000000${Math.floor(Math.random() * 16777215).toString(16)}`.slice(-6),
      L = Date.now().toString().slice(-10),
      I = `.tmp-${L}${A}`;
    return `${H}${I}`;
  },
  purgeSyncAll: () => {
    for (let H in A2.store) A2.purgeSync(H);
  },
};
ThH(A2.purgeSyncAll);  // register on process exit
```

### Path Resolution (0201.js, line 8-10)
```js
function C$() {  // getOpenDroidHome
  return process.env.OPENDROID_HOME_OVERRIDE || iHE();  // os.homedir()
}
```

## Integration Points (cross-system)

- **Config Loader** (infra-config-loader): Uses `getOpenDroidHome()` + path.join for `~/.opendroid/config.json` resolution
- **Session Manager** (infra-session-manager): Uses `mkdirSync({recursive:true})` for session dir creation, reads/writes session `.jsonl` files
- **Auth** (infra-auth): OAuth storage (1131.js) uses atomic write + encryption for credential persistence
- **Crypto** (infra-crypto): Key derivation uses `os.hostname()` + `os.userInfo().username` for machine-bound encryption keys
- **Telemetry** (infra-telemetry-otel): 1624.js provides OpenTelemetry instrumentation for all fs operations (rename, copy, mkdir, writeFile, etc.)
- **Update Checker** (infra-update-checker): Uses temp dirs for extension downloads (2530.js)
- **IPC Daemon** (infra-ipc-daemon): Log file initialization (0203.js) uses `ensureDirectoryExists` pattern
- **TUI** (01-terminal-ui): Terminal detection (0201.js) used by CLI telemetry client
- **Tool System** (03-tool-agent-system): File tools rely on atomic write patterns for safe file modifications

## Implementation Notes

1. **Atomic Write**: Implement `writeFileAtomic(path, data, options)` following the write-temp → fsync → rename pattern. The 0861.js implementation is the reference: include retry logic and ENAMETOOLONG fallback.
2. **Secure Permissions**: Always set 0o600 on sensitive files (auth, config, sessions) and 0o700 on directories. Implement a startup audit function similar to `ensureAllSecurePermissions()`.
3. **Temp File Manager**: Create a centralized temp file tracker with automatic cleanup on process exit. Use descriptive suffixes (`.tmp-{timestamp}{random}`) for debugging.
4. **Path Resolution**: Support `OPENDROID_HOME_OVERRIDE` env var for testing. Use `os.homedir()` as default. Keep all data under `{homeDir}/.opendroid/`.
5. **Directory Bootstrap**: Always use `mkdir({recursive:true})` before writes. Make it idempotent. Never assume directories exist.

## Module Reference

| Module | Line | Function | Description |
|--------|------|----------|-------------|
| 1131.js | 6-20 | `class FOA` | OAuth credential store with encryption |
| 1131.js | 34-36 | `saveSecret()` | Atomic write: writeFile.tmp → rename |
| 1131.js | 107-109 | `save()` | Atomic write with encryption for OAuth data |
| 0866.js | 21-30 | `class y2A` | Encrypted auth file store constructor |
| 0866.js | 79 | `ensureDirectoryExists()` | Mkdir recursive before writes |
| 0866.js | 104-106 | `writeFile()` | Calls `j2A` (atomic write) + `rK` (secure perms) |
| 0861.js | 9-23 | `m3H` / `A2` | Temp file manager: create, get, purge, truncate |
| 0861.js | 47-50 | `j2A()` | General-purpose atomic write entry point |
| 0861.js | 53-110 | `lV$()` | Atomic write implementation: open→write→fsync→close→rename |
| 0861.js | 93-99 | rename fallback | ENAMETOOLONG → truncate filename and retry |
| 0865.js | 17-21 | `rK()` | setSecureFilePermissions (0o600) |
| 0865.js | 24-28 | `lo()` | setSecureDirectoryPermissions (0o700) |
| 0865.js | 31-35 | `M6()` | setSecureFilePermissionsSync (0o600) |
| 0865.js | 38-42 | `Ic()` | setSecureDirectoryPermissionsSync (0o700) |
| 0865.js | 46-87 | `YgE()` | ensureAllSecurePermissions: startup audit |
| 0201.js | 8-10 | `C$()` | getOpenDroidHome: OPENDROID_HOME_OVERRIDE or homedir |
| 3836.js | 24-30 | `$S` constructor | Image temp dir creation under ~/.opendroid/temp/images/ |
| 3836.js | 93-95 | `cleanup()` | Remove images + rmdir temp directory |
| 0199.js | 28-30 | `ensureDirectoryExists()` | Telemetry log directory creation |
| 0203.js | 97-104 | `initializeSync()` | CLI telemetry: mkdir log dir before init |
| 1620.js | 4-87 | Module exports | Node.js fs function lists (promise/callback/sync) |
| 1624.js | 17-99 | `Xr$` | OpenTelemetry fs instrumentation setup |
| 3734.js | 73-77 | `installSshKey()` | SSH key write with mode 0o600 |
