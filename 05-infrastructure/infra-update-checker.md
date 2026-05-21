# infra-update-checker: Architecture Notes

## Overview

OpenDroid's auto-update/version-check system is a full-featured binary update mechanism built around the `Updater` class (0701.js). The system fetches `LATEST` version information from S3 or HTTP-based remote repositories, performs semver comparison, and if a new version exists, downloads the binary and stages it with SHA-256 checksum verification. On Windows, updates are managed via a pending manifest file and applied on the next restart. Two distinct update flows exist: an auto-update running in the background at CLI startup and the manual `droid update` command. Monitored via telemetry metrics (update latency, outcome), the system includes production-grade features such as rollback support and cross-device rename fallback.

## Module Map

| Module | Size | Role | Vendor/App |
|-------|-------|-----|------------|
| 0701.js | ~462 lines | **Updater core class**: checkForUpdates, downloadAndStageUpdate, applyUpdate, performUpdate, runAutoUpdate, restartProcess, applyPendingWindowsUpdate | App |
| 2099.js | ~100 lines | **Auto-update startup**: background update check at CLI startup, sGI() function, telemetry metrics | App |
| 3789.js | ~154 lines | **`droid update` CLI command**: manual update command, check-only mode, UI/progress display | App |
| 0331.js | ~30 lines | **Update state enum**: UpdateState and UpdateResult constants | App |
| 2468.js | ~435 lines | OpenAI SDK client constructor: PACKAGE_VERSION information, version context | Vendor (OpenAI) |
| 1706.js | ~408 lines | KafkaJS OTel instrumentation: PACKAGE_VERSION pattern, version consistency reference | Vendor (OTel) |
| 2072.js | ~382 lines | amqplib OTel instrumentation: PACKAGE_NAME/VERSION pattern, moduleVersion tracking | Vendor (OTel) |
| 1191.js | ~449 lines | Patch application system: file update mechanism, file diff/patch rather than semver comparison | App |
| 0044.js | ~280 lines | SonicBoom file writer: process.versions.node version check, file writing utility (winston dependency) | Vendor |
| 1406.js | ~448 lines | ErrTracker client: _updateSessionFromEvent, captureSession, error reporting (update error reporting) | Vendor (ErrTracker) |

## Architecture

### Update Flow Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  CLI Startup     │────▶│  sGI()          │────▶│  Updater(DO)     │
│  (2099.js)       │     │  auto-update    │     │  (0701.js)       │
└─────────────────┘     └─────────────────┘     └──────────────────┘
                                                         │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  droid update   │────▶│  createUpdate   │────▶│  checkForUpdates │
│  command(3789)  │     │  Command()      │     │  (S3 or HTTP)    │
└─────────────────┘     └─────────────────┘     └──────────────────┘
                                                         │
                                                         ▼
                                                ┌──────────────────┐
                                                │ SemVer compare   │
                                                │ (current vs      │
                                                │  LATEST)         │
                                                └──────────────────┘
                                                         │
                              ┌──────────┬───────────────┤
                              ▼          ▼               ▼
                         no-update   isRollback    new version
                              │          │               │
                              ▼          ▼               ▼
                           return    skip or      downloadAndStage
                           null      force update      Update()
                                                         │
                                                  ┌──────┤──────┐
                                                  ▼      ▼      ▼
                                              download verify install
                                                  │      │      │
                                                  ▼      ▼      ▼
                                              S3/HTTP  SHA256  rename/
                                              fetch    check   copyFile
```

### 1. Version Check Mechanism (`Updater.checkForUpdates()`)

The system can perform update checks from two different remote sources:

- **S3 Mode**: Reads the `LATEST` file from an S3 bucket via `awsS3Client.fetchFileText(bucket, getS3Key("LATEST"))`
- **HTTP Mode**: Fetches the version string from an HTTP endpoint via `fetch(getDownloadUrl("LATEST"))`

Version comparison is performed using the `x6A.SemVer` class:
```js
let L = new x6A.SemVer(A),  // A = remote LATEST content
    $ = L.compare(this.currentVersion);
if ($ === 0) return null;   // Already up to date
let M = $ < 0;              // isRollback flag
```

### 2. Download & Stage Update (`downloadAndStageUpdate()`)

3-phase process:
1. **Download**: The binary file is downloaded from S3 or HTTP and written to a staging directory
2. **Verify**: The SHA-256 checksum file is downloaded and compared against the binary's hash
3. **Stage**: Staging is completed while preserving file permissions (chmod)

```js
// Checksum verification (0701.js, line ~170)
let E = D.trim().split(" ")[0],     // Expected checksum
    f = UkE("sha256").update($).digest("hex");  // Actual checksum
if (E !== f) throw new vH("Checksum verification failed", ...);
```

### 3. Update Apply Strategy

Different apply strategies depending on the platform:

**Windows (`applyUpdate`):**
- A pending manifest file is created (written via `N8$()`)
- `onStateChange({ type: "pending-install" })` is triggered
- The update is applied on the next restart via `applyPendingWindowsUpdate()`
- Backup: `${targetPath}.old` is created; if it fails, rollback is performed

**Unix (darwin/linux):**
- The binary is replaced directly via `fs.rename()`
- In case of a cross-device rename error, `copyFile + unlink` fallback is used
- After a successful apply, the new binary is spawned via `restartProcess()`

### 4. Auto-Update at Startup (`sGI()` in 2099.js)

Every time the CLI starts:
1. `extras.autoUpdateEnabled` check (disabled during npm installs)
2. A new Updater instance is created, `checkForUpdates()` is called
3. If a rollback is detected and `DROID_FORCE_BLOCKING_UPDATE=true`, a blocking update is performed
4. Normal updates run non-blocking (in the background): `performUpdate(f, { restartProcess: false })`
5. Telemetry metrics: `cli_startup_update_latency`, `cli_startup_update_check_latency`

### 5. Manual Update Command (`droid update` in 3789.js)

```bash
droid update           # Check and install
droid update --check   # Only check, don't install
droid update -v 1.2.3  # Update to specific version
```

### 6. Update State Machine (0331.js)

```
UpdateState:  checking → no-update | update-available → downloading → verifying → installing → complete | error | pending-install
UpdateResult: updated | no-update | skipped | error | background-in-progress | pending-restart
```

## Key Findings

1. **Dual Source Strategy**: Updates can come from an S3 bucket or HTTP CDN. Source selection is made via the `shouldUseS3Client()` method. S3 key prefix and bucket information are retrieved from `remoteConfig`.

2. **SemVer-Based Comparison**: Semantic version comparison is performed using the `x6A.SemVer` class. Rollback detection is present (`isRollback = L.compare(current) < 0`): if the remote version is lower than the current version, it is flagged as a rollback.

3. **Windows Pending Update Pattern**: Since binary overwrite is not possible at runtime on Windows, a pending manifest JSON file is created. On the next launch, the `applyPendingWindowsUpdate()` static method:
   - Reads the manifest (`R8$(updatesDir)`)
   - Checks staged binary existence (`h8$(stagedPath)`)
   - Creates a backup (`${target}.old`)
   - Performs copy + chmod + cleanup
   - Rolls back from backup on error

4. **Process Restart**: On Unix systems, the new binary is spawned after an update via `restartProcess()`. The child process is not detached; parent signals are forwarded to the child. When the child exits, the parent also exits.

5. **Auto-Update Guard**: Auto-update is disabled during npm installations via the `autoUpdateEnabled` flag. For manual updates, the PowerShell command `irm https://app.opendroid.dev/cli/windows | iex` is recommended.

6. **Telemetry Integration**: The update process is recorded as metrics via `iL.addToCounter()`: latency, outcome (updated/skipped/error/background-in-progress/pending-restart), and error messages.

7. **Security: SHA-256 Checksum Verification**: After each download, the checksum file is additionally downloaded and compared against the binary's SHA-256 hash. On mismatch, the update is cancelled.

## Code Examples (minimal)

### Version Check (0701.js, line ~90-115)
```js
async checkForUpdates() {
  let A;
  if (this.shouldUseS3Client()) {
    let U = this.getS3Key("LATEST"), P = this.config.remoteConfig;
    A = await this.awsS3Client.fetchFileText(P.s3Bucket, U));
  } else {
    let U = this.getDownloadUrl("LATEST");
    let P = await fetch(U);
    A = (await P.text()).trim();
  }
  let L = new x6A.SemVer(A),
      $ = L.compare(this.currentVersion);
  if ($ === 0) return null;
  // ... return { version, binPath, checksumPath, isRollback }
}
```

### Auto-Update Startup Guard (2099.js, line ~24-30)
```js
async function sGI() {
  let { extras: H } = OD();
  if (!H.autoUpdateEnabled)
    return BH("Auto-update disabled..."), "skipped";
  // ... proceed with update check
}
```

### Windows Pending Apply (0701.js, line ~410-460)
```js
static async applyPendingWindowsUpdate() {
  let L = await R8$(A);  // Read pending manifest
  if (!L) return { applied: false };
  let I = `${L.targetPath}.old`;
  await b6A(L.targetPath, I, "windows");  // Backup
  await VQ.copyFile(L.stagedPath, L.targetPath);  // Apply
  await S6A(L.targetPath);  // Post-apply cleanup
  await VQ.unlink(L.stagedPath);
  await y6A(A);  // Clear manifest
  // ... error: rollback from .old
}
```

### CLI Update Command Definition (3789.js, line ~70-85)
```js
new mP("update")
  .description("Check for and install Droid updates")
  .option("-c, --check", "Only check for updates without installing")
  .option("-v, --version <version>", "Update to a specific version (e.g., 1.2.3)")
```

## Integration Points (cross-system)

- **Config System (infra-config-loader)**: Configuration is retrieved via the `OD()` function, where `downloadsBucket`, `downloadsPathPrefix`, `isProductionTier`, `deploymentEnv`, `autoUpdateEnabled` config values control update behavior
- **AWS SDK (infra-aws-sdk)**: The `M3H()` S3 client class is used to download the LATEST version file and binary/checksum from the S3 bucket (`awsS3Client.fetchFileText`, `awsS3Client.fetchFileBuffer`)
- **Telemetry (infra-telemetry-otel)**: Update metrics are collected via `iL.addToCounter()`: `cli_startup_update_latency`, `cli_startup_update_check_latency`
- **Error Reporting (1406.js/ErrTracker)**: Update errors are logged and reported to ErrTracker via the `uH()` and `gH()` functions
- **Logging (infra-logging-winston)**: Structured logging is performed via `BH()` and `gH()` functions
- **Filesystem Helpers (infra-fs-helpers)**: Atomic write, temp dir management, `mkdir({ recursive: true })`, `chmod` for update staging file operations

## Implementation Notes

1. **Updater Core**: The `Updater` class is portable across platforms. Platform and arch detection should be updated. Dependency on the SemVer library should be maintained.
2. **Source Strategy**: The S3/HTTP dual source mechanism should be preserved and extensible to new deployment environments (GCS, Azure Blob, etc.). The `remoteConfig` structure supports this.
3. **Windows Pending Update**: The pending manifest JSON format and `.old` backup pattern are directly portable. The `applyPendingWindowsUpdate()` static method should be called at app startup.
4. **CLI Command**: The `droid update` command is defined with yargs/mP and should be integrated into OpenDroid's CLI framework. The `--check` and `--version` options should be preserved.
5. **Telemetry**: `addToCounter` calls should be mapped to OpenDroid's telemetry system, preserving metric names and labels.

## Module Reference

| File | Line | Function / Structure |
|-------|-------|-----------------|
| 0701.js | 18-35 | `class DO` constructor: platform/arch detection, config init |
| 0701.js | 42-50 | `getCurrentVersion()`, `shouldUseS3Client()`, `getS3Key()` |
| 0701.js | 55-95 | `checkForUpdates()`: S3/HTTP LATEST fetch, SemVer compare, isRollback detection |
| 0701.js | 96-170 | `downloadAndStageUpdate()`: 3-phase download/verify/stage |
| 0701.js | 171-240 | `applyUpdate()`: platform-specific apply (Windows pending / Unix rename) |
| 0701.js | 241-290 | `performUpdate()`: full update orchestration with state callbacks |
| 0701.js | 291-370 | `runAutoUpdate()`: auto-update flow with dev mode guard |
| 0701.js | 382-410 | `static restartProcess()`: spawn new binary, signal forwarding |
| 0701.js | 410-462 | `static applyPendingWindowsUpdate()`: Windows startup pending apply |
| 2099.js | 24-95 | `async sGI()`: auto-update startup entry point |
| 2099.js | 18-22 | `y6H()`: update latency telemetry helper |
| 2099.js | 60-65 | Remote config selection (baseUrl vs s3Bucket) |
| 3789.js | 14-20 | `tpD()`: manual update fallback message |
| 3789.js | 22-35 | `UjM()`: CLI update state change handler (progress display) |
| 3789.js | 48-55 | `BjM()`: build update info for specific version |
| 3789.js | 57-85 | `WjM()`: `droid update` command definition |
| 3789.js | 86-154 | Command action: check/install logic, error handling |
| 0331.js | 1-15 | UpdateState enum: checking/no-update/update-available/downloading/verifying/installing/complete/error/pending-install |
| 0331.js | 16-23 | UpdateResult enum: updated/no-update/skipped/error/background-in-progress/pending-restart |
