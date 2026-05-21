# infra-semver-resolve: Architecture Notes

## Overview

OpenDroid bundles two major semver resolution implementations and a comprehensive glob/pattern-matching toolkit. The primary semver library is **node-semver v7.7.2** (declared in `package.json` at `0969.js`), spanning ~10 modules that implement full SemVer specification parsing, range resolution, comparator evaluation, and subset/intersects checks. A secondary, lighter semver `satisfies()` is bundled within the OpenTelemetry SDK for internal version compatibility checks. The glob implementation is **glob v10.5.0** built on **picomatch v4.0.3**, providing async/sync/stream/iterate filesystem pattern matching with support for globstars (`**`), brace expansion, extglobs, and ignore patterns. These utilities underpin the CLI's version checking, package dependency resolution, config file discovery, and plugin/skill filesystem scanning.

## Module Map

| Module | Size (lines) | Role | Vendor/App |
|--------|-------------|------|------------|
| 0287.js | ~15 | Semver debug logger (`SEMVER` env flag) | Vendor (node-semver) |
| 0291.js | 173 | SemVer class: parse, compare, increment, format | Vendor (node-semver) |
| 0316.js | 288 | Range class: parse range expressions (`^1.2.3`, `>=1.0.0 <2.0.0`) | Vendor (node-semver) |
| 0317.js | 93 | Comparator class: single version comparison (`>=1.0.0`) | Vendor (node-semver) |
| 0322.js | ~40 | minVersion: find minimum satisfying version for a range | Vendor (node-semver) |
| 0324.js | ~42 | outside: test if version falls outside a range (gtr/ltr) | Vendor (node-semver) |
| 0329.js | 112 | subset: test if one range is a subset of another | Vendor (node-semver) |
| 0330.js | 95 | Semver index: re-exports all semver functions (parse, satisfies, coerce, etc.) | Vendor (node-semver) |
| 1486.js | 242 | OTel semver satisfies: lightweight semver constraint checker | Vendor (@opentelemetry/core) |
| 1584.js | ~80 | OTel semver satisfies (duplicate instance) | Vendor (@opentelemetry/core) |
| 3588.js | 124 | Pattern class: parsed glob pattern analyticssvcs with GLOBSTAR/regex support | Vendor (glob) |
| 3591.js | 277 | GlobWalker/GlobUtil/GlobStream: async/sync filesystem traversal | Vendor (glob) |
| 3592.js | 185 | Glob class: main glob API with pattern compilation and walk dispatch | Vendor (glob) |
| 3594.js | 100 | glob index: exports glob, globSync, globStream, iterate, Ignore | Vendor (glob) |
| 2501.js | ~50 | glob entry point + ripgrep binary extraction | App (OpenDroid wrapper) |
| 3895.js | ~15 | picomatch loader: dynamic require with fallback | Vendor (picomatch) |
| 3898.js | 234 | picomatch scanner: token-level glob parsing (brace, extglob, globstar) | Vendor (picomatch) |
| 3900.js | 99 | picomatch main: compile, makeRe, isMatch, parse, scan API | Vendor (picomatch) |
| 0969.js | 183 | package.json manifest: declares semver ^7.7.2, glob ^10.5.0, picomatch ^4.0.3 | App |

**Seed modules analyzed but determined unrelated:**
- 1150.js: JPEG encoder (jpeg-js), not semver/glob
- 2602.js: AVL/sorted tree data structure, not semver/glob
- 3845.js: UI model name display formatter, not semver/glob
- 0194.js: OpenTelemetry core utilities, not semver/glob
- 2402.js: OpenAI streaming chat completions, not semver/glob
- 2592.js: gRPC ChannelCredentials (TLS), not semver/glob

## Architecture

The semver and glob subsystems form the foundation for version and path resolution across OpenDroid:

### Semver Resolution Pipeline
1. **Input**: A version string (e.g., `"1.2.3"`) and optional range constraint (e.g., `"^1.0.0"`)
2. **Parsing**: `SemVer` class (0291.js) parses the version string using regex patterns, extracting major/minor/patch/prerelease/build analyticssvcs
3. **Range Construction**: `Range` class (0316.js) splits range expressions on `||` (union), then parses each sub-range into `Comparator` instances (0317.js)
4. **Evaluation**: `satisfies()` checks if a version matches a range; `minVersion()` (0322.js) finds the lowest version satisfying a range; `outside()` (0324.js) checks if a version is outside bounds
5. **Advanced**: `subset()` (0329.js) determines range containment; `intersects()` checks if two ranges overlap

### Glob Matching Pipeline
1. **Input**: A glob pattern (e.g., `"src/**/*.ts"`) and options
2. **Scanning**: picomatch scanner (3898.js) tokenizes the pattern, detecting braces, extglobs, globstars, character classes, and negation
3. **Compilation**: picomatch main (3900.js) compiles scanned tokens into a RegExp via `makeRe()`, with caching
4. **Pattern Object**: `Pattern` class (3588.js) wraps parsed pattern analyticssvcs with GLOBSTAR/regex identification and path handling
5. **Filesystem Walk**: `Glob` class (3592.js) creates `GlobWalker` (3591.js) which traverses the filesystem, applying pattern matching at each directory level
6. **Output**: Results returned as string paths, PathScurry objects, or streams

### OpenDroid Integration
- **Ripgrep extraction** (2501.js): The glob module entry point also contains logic to extract the ripgrep binary from a bundled path, with SHA-256 verification for integrity
- **Config discovery** (0259.js): Uses glob for scanning `.opendroid/` directories to find settings, hooks, droids, skills, and MCP configurations
- **Version checking** (3756.js): CLI version detection uses regex pattern matching to parse `droid --version` output

## Key Findings

1. **Full node-semver v7.7.2 bundled**: Complete SemVer spec implementation with all functions: parse, valid, clean, satisfies, compare, gt/lt/gte/lte, coerce, Range, Comparator, minVersion, maxSatisfying, minSatisfying, outside, gtr, ltr, intersects, subset, simplifyRange. Exported as a single index module (0330.js) with ~40 named exports.

2. **Duplicate semver in OpenTelemetry**: The OTel SDK bundles its own lightweight `satisfies()` implementation (1486.js, 1584.js) rather than importing node-semver, reducing dependency footprint. This OTel version handles `~`, `^`, `~>`, comparators, hyphen ranges, and prerelease filtering.

3. **Glob v10 with picomatch v4**: Modern glob implementation using picomatch for pattern parsing. Supports globstar (`**`), brace expansion, extglobs (`!(...)`, `?(...)`), character classes, and ignore patterns. Provides async `glob()`, sync `globSync()`, streaming `globStream()`, and iterator `globIterate()` APIs.

4. **Pattern matching with GLOBSTAR**: The `Pattern` class (3588.js) handles UNC paths (`\\server\share`), drive letters, and absolute paths, splitting them into analyticssvcs for matching. Uses `GLOBSTAR` sentinel for `**` patterns.

5. **Ripgrep binary management**: Module 2501.js contains OpenDroid-specific code to extract a bundled ripgrep binary (`rg.exe`) from a Bun resource, verify SHA-256 checksum, and write to `~/.opendroid/bin/rg.exe`. This integrates with the glob/search system for fast file content searching.

6. **Config filesystem watching**: The glob infrastructure supports `chokidar`/`fsevents`-based file watching for the `.opendroid/` config directory, enabling hot-reload of settings, droids, skills, and hooks (visible in 0259.js).

7. **Seed modules misassigned**: The 6 seed modules (1150, 2602, 3845, 0194, 2402, 2592) do not contain semver/glob code. They are: JPEG encoding (jpeg-js), sorted tree data structure, UI model name formatter, OpenTelemetry core, OpenAI streaming, and gRPC TLS credentials respectively.

## Code Examples

### SemVer class construction and comparison (0291.js, lines 14-50)
```js
class nY {
  constructor(H, A) {
    // Parse version string via regex (loose or strict)
    let L = H.trim().match(A.loose ? $jH[IjH.LOOSE] : $jH[IjH.FULL]);
    this.major = +L[1];
    this.minor = +L[2];
    this.patch = +L[3];
    this.prerelease = L[4] ? L[4].split(".").map(...) : [];
    this.build = L[5] ? L[5].split(".") : [];
  }
  compareMain(H) {
    if (this.major < H.major) return -1;
    // ... cascading minor/patch comparison
  }
}
```

### Semver index exports (0330.js, lines 50-90)
```js
YxL.exports = {
  parse, valid, clean, inc, diff,
  major, minor, patch, prerelease,
  compare, rcompare, compareLoose, compareBuild,
  sort, rsort, gt, lt, eq, neq, gte, lte, cmp, coerce,
  Comparator, Range, satisfies, toComparators,
  maxSatisfying, minSatisfying, minVersion, validRange,
  outside, gtr, ltr, intersects, simplifyRange, subset,
  SemVer, re, src, tokens,
  SEMVER_SPEC_VERSION, RELEASE_TYPES,
};
```

### Range parsing with OR/AND logic (0316.js, lines 35-80)
```js
class qVH { // Range
  constructor(H, A) {
    this.raw = H.trim().replace(h9E, " ");
    // Split on || for union (OR)
    this.set = this.raw.split("||")
      .map(L => this.parseRange(L.trim()))
      .filter(L => L.length);
  }
  parseRange(H) {
    // Transform: hyphen -> tilde -> caret -> comparator trim
    H = H.replace(DQ[VG.HYPHENRANGE], t9E(...));
    H = H.replace(DQ[VG.TILDETRIM], b9E);
    H = H.replace(DQ[VG.CARETTRIM], x9E);
    // Split on spaces for intersection (AND)
    let E = H.split(" ").map(P => c9E(P, this.options))
      .map(P => new gWA(P, this.options));
    return E;
  }
}
```

### Picomatch scanner token detection (3898.js, lines 30-80)
```js
// Detects: braces {}, brackets [], parens (), globstar **, wildcards *?
// Handles: escape sequences, negation (!), extglobs (?(), *(), +(), @())
while (U < $) {
  e = VH(); // next char code
  if (e === fzH) { /* backslash escape */ }
  if (e === eTL) { /* left brace {: start brace expansion */ v++; }
  if (e === HbM) { /* left bracket [: start char class */ }
  if (e === tTL && DH() === tTL) { /* **: globstar */ }
}
```

### Glob class with walk dispatch (3592.js, lines 130-145)
```js
class zzD { // Glob
  async walk() {
    return [...(await new G0A.GlobWalker(
      this.patterns, this.scurry.cwd, {
        ...this.opts,
        maxDepth: this.maxDepth !== Infinity ? this.maxDepth + this.scurry.cwd.depth() : Infinity,
        platform: this.platform,
      }
    ).walk())];
  }
  walkSync() { /* synchronous variant */ }
  stream() { return new G0A.GlobWalker(...).stream(); }
}
```

### Ripgrep binary extraction (2501.js, lines 28-45)
```js
function wzI() {
  let H = VoA.join(C$(), sL, "bin"),
    L = VoA.join(H, "rg.exe"),
    I = E9f(ToA); // SHA-256 of bundled binary
  if (!QzI(L) || /* checksum mismatch */) {
    I9f(H, { recursive: true });
    JzI(L, XoA(ToA), { mode: 493 }); // extract binary
    JzI($, I); // write checksum
  }
  return L;
}
```

## Integration Points (cross-system)

- **Config Loader (infra-config-loader)**: Glob is used by config filesystem resolver (0259.js) to discover `.opendroid/` settings files, MCP configs, droid definitions, skills, hooks, and custom models. File watching uses chokidar which depends on glob patterns.

- **Update Checker (infra-update-checker)**: Semver is used for version comparison during update checks. The CLI version detection (3756.js) parses version strings and compares against minimum required versions.

- **Plugin Manager (infra-plugin-manager)**: Glob patterns are used for plugin discovery, scanning filesystem directories for plugin entry points.

- **Cache (infra-cache)**: Glob-based cache key patterns could be used for cache invalidation of file-based caches.

- **CLI Entry (infra-cli-entry)**: The CLI bootstrapping process uses glob for config discovery and semver for version validation.

- **Telemetry (infra-telemetry-otel)**: OpenTelemetry bundles its own semver satisfies (1486.js) for SDK version compatibility checks, independent of the main node-semver.

- **Tool System (03-tool-agent-system)**: File search tools use glob patterns for file discovery and the ripgrep binary for content search.

- **GUI (04-desktop-gui)**: Glob patterns are used for workspace file tree rendering and file type filtering.

## Implementation Notes

1. **Semver**: Replace with any standard semver library (e.g., Python's `packaging.version`, Rust's `semver` crate). The API surface is well-defined: parse, compare, satisfies, Range, coerce. The OTel-internal semver can be left as-is since it's part of the OTel SDK bundle.

2. **Glob**: Replace with platform-native glob implementations (Python's `pathlib.glob`, Rust's `glob` crate). Key features to replicate: `**` recursive, brace expansion, negation (`!pattern`), ignore lists, streaming results.

3. **Picomatch**: This is an internal dependency of the glob library. When replacing glob, picomatch comes along. If custom pattern matching is needed, implement a minimal glob-to-regex converter handling `*`, `?`, `**`, `[...]`, and `{...}`.

4. **Ripgrep extraction**: The bundled ripgrep binary pattern (2501.js) is Bun-specific. For OpenDroid, use system-installed ripgrep or bundle platform-specific binaries with integrity verification (SHA-256 checksum pattern is reusable).

5. **Config discovery**: The filesystem scanning pattern (0259.js) using glob + file watching is reusable. Ensure the target platform has equivalent filesystem watch APIs (e.g., `inotify` on Linux, `FSEvents` on macOS, `ReadDirectoryChangesW` on Windows).

## Module Reference (file + line + function)

### Semver (node-semver v7.7.2)
- `0287.js, line 7`: `CME()` debug logger (checks `NODE_DEBUG=semver`)
- `0291.js, line 14`: `class nY` (SemVer) constructor: version parsing
- `0291.js, line 51`: `nY.compare(H)`: version comparison
- `0291.js, line 56`: `nY.compareMain(H)`: major/minor/patch comparison
- `0291.js, line 69`: `nY.comparePre(H)`: prerelease analyticssvc comparison
- `0291.js, line 95`: `nY.inc(H, A, L)`: version increment (premajor/preminor/prepatch/prerelease/major/minor/patch)
- `0316.js, line 35`: `class qVH` (Range) constructor: range parsing with OR (`||`) splitting
- `0316.js, line 57`: `qVH.parseRange(H)`: transforms hyphen/tilde/caret into comparators
- `0316.js, line 100`: `qVH.test(H)`: test if version satisfies range
- `0317.js, line 7`: `class PjH` (Comparator) constructor: single comparator parsing
- `0317.js, line 35`: `PjH.test(H)`: test version against comparator
- `0317.js, line 43`: `PjH.intersects(H, A)`: comparator intersection check
- `0322.js, line 6`: `UUE(H, A)`: minVersion resolver
- `0324.js, line 15`: `FUE(H, A, L, $)`: outside checker (gtr/ltr)
- `0329.js, line 23`: `OUE(H, A, L)`: subset checker for ranges
- `0330.js, line 50`: exports index: all 40+ semver functions

### Semver (OpenTelemetry internal)
- `1486.js, line 7`: `Sg$` semver regex pattern (named groups: major/minor/patch/prerelease/build)
- `1486.js, line 11`: `wX1(H, A, L)`: satisfies(version, range, options)
- `1486.js, line 31`: `yg$(H, A, L, $)`: range expression parser (OR/AND/hyphen)
- `1584.js, line 7`: Duplicate OTel semver satisfies

### Glob (glob v10.5.0)
- `3588.js, line 18`: `class fUL` (Pattern) constructor: pattern/glob list pair
- `3588.js, line 44`: `fUL.pattern()`: current analyticssvc
- `3588.js, line 48`: `fUL.isGlobstar()`: check for `**`
- `3588.js, line 49`: `fUL.isRegExp()`: check for compiled regex
- `3588.js, line 53`: `fUL.rest()`: remaining pattern analyticssvcs
- `3591.js, line 18`: `class V0A` (GlobUtil): base walker with ignore/match logic
- `3591.js, line 68`: `V0A.matchCheck(H, A)`: async match with realpath/stat
- `3591.js, line 96`: `V0A.matchFinish(H, A)`: emit matched path
- `3592.js, line 24`: `class zzD` (Glob) constructor: pattern compilation with options
- `3592.js, line 130`: `zzD.walk()`: async walk using GlobWalker
- `3592.js, line 141`: `zzD.walkSync()`: synchronous walk
- `3594.js, line 16`: `SOH(H, A)`: globStreamSync
- `3594.js, line 18`: `XUL(H, A)`: globSync
- `3594.js, line 24`: `yzD(H, A)`: async glob

### Picomatch (v4.0.3)
- `3895.js, line 8`: Dynamic picomatch require with fallback
- `3898.js, line 15`: `IbM(H, A)`: scanner main function (token parsing)
- `3898.js, line 51`: Token detection loop (brace, extglob, globstar, negation)
- `3900.js, line 4`: `nW(H, A, L)`: picomatch main function (returns matcher)
- `3900.js, line 22`: `nW.test(H, A, L)`: test string against compiled regex
- `3900.js, line 56`: `nW.makeRe(H, A)`: compile pattern to RegExp
- `3900.js, line 63`: `nW.parse(H, A)`: full parse (AST)
- `3900.js, line 65`: `nW.scan(H, A)`: fast scan (token info)

### Application Integration
- `0969.js, line 100`: `glob: "^10.5.0"` dependency declaration
- `0969.js, line 111`: `picomatch: "^4.0.3"` dependency declaration
- `0969.js, line 114`: `semver: "^7.7.2"` dependency declaration
- `2501.js, line 28`: `wzI()`: ripgrep binary extraction with SHA-256 verify
- `0259.js, line 10`: Config loader using glob for `.opendroid/` scanning
- `0259.js, line 60`: `startWatching()`: filesystem watcher setup
- `3756.js, line 33`: `pcD(H)`: CLI version detection with regex matching
