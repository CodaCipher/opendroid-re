# infra-cache: Architecture Notes

## Overview

OpenDroid employs a **multi-layer caching architecture** spanning vendor libraries and application-specific implementations. The caching subsystem is not a single monolithic module but rather a collection of cache patterns used across different subsystems. At its core, OpenDroid uses two LRU cache implementations: a **custom lightweight LRUCache** (module 2726, used by GCP auth) and the **full-featured npm `lru-cache` library** (module 2492, used broadly). Application-level caching includes a **file system crawler cache** with TTL-based invalidation (module 3901), an **encrypted file-based credential cache** with mtime-based invalidation (module 0866), and **path resolution caches** for glob operations (module 3587). The system favors in-memory caching with TTL-based eviction; there is no explicit disk serialization format for the LRU caches themselves:persistence is handled at the application layer through file I/O.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 2726.js | ~66 lines | Custom lightweight LRUCache class (capacity + maxAge eviction) | App (GCP auth) |
| 2492.js | ~935 lines | Full npm `lru-cache` library (LRU + TTL + size-based eviction, fetch, memo) | Vendor (npm lru-cache) |
| 3585.js | ~893 lines | Duplicate of lru-cache (re-export) | Vendor (npm lru-cache) |
| 3587.js | ~1082 lines | PathScurry path resolution with ResolveCache + ChildrenCache (LRU-based) | Vendor (glob) |
| 2741.js | ~85 lines | JWTAccess: JWT token caching using custom LRUCache (500 capacity, 1h TTL) | App (GCP auth) |
| 3901.js | ~153 lines | FileCache: file crawler with Map-based TTL cache + in-flight dedup | App (file discovery) |
| 0866.js | ~180 lines | Encrypted credential storage with in-memory cache + mtime-based invalidation | App (auth/keyring) |
| 1461.js | ~197 lines | AI telemetry span processor with cached input tokens tracking | App (telemetry) |

## Architecture

### Cache Layer Diagram

```
┌─────────────────────────────────────────────────────┐
│                   Application Layer                  │
├──────────────┬──────────────┬────────────────────────┤
│  JWTAccess   │  FileCache   │  EncryptedFileStore    │
│ (2741.js)    │ (3901.js)    │  (0866.js)             │
│ capacity:500 │ Map + TTL    │  cache + mtimeMs       │
│ maxAge:1h    │ ttlMs:30s    │  AES-256-GCM encrypted │
├──────────────┼──────────────┼────────────────────────┤
│ ResolveCache │ ChildrenCache│  PathScurry            │
│ (3587.js)    │ (3587.js)    │  (3587.js)             │
│ max:256      │ maxSize:16KB │  glob path resolution  │
├──────────────┴──────────────┴────────────────────────┤
│              LRU Cache Implementations                │
├───────────────────────┬─────────────────────────────┤
│  Custom LRUCache      │   npm lru-cache             │
│  (2726.js, ~66 loc)   │   (2492.js, ~935 loc)       │
│  capacity + maxAge    │   max + maxSize + TTL        │
│  Map-based LRU        │   Array-indexed LRU          │
│  Simple eviction      │   fetch(), memo(), dispose   │
└───────────────────────┴─────────────────────────────┘
```

### Cache Key Structure

Each cache consumer uses a domain-specific key scheme:

1. **JWTAccess (2741.js)**: Key = `url` or `url_scope1_scope2` (scopes joined with `_`). Method `getCachedKey(url, scopes)` constructs a composite string key.

2. **FileCache (3901.js)**: Key = `path\0maxFiles\0maxDepth\0showHidden\0respectGitignore\0excludePatterns`: a null-byte delimited string encoding all crawl options. Constructed by static method `getCacheKey()`.

3. **EncryptedFileStore (0866.js)**: Single-slot cache (not a map): `this.cache` holds the last loaded content as a string, with `this.cacheMtimeMs` tracking the file modification time for invalidation.

4. **ResolveCache / ChildrenCache (3587.js)**: Keys are path strings (absolute). ResolveCache has max 256 entries; ChildrenCache has maxSize 16384 with `sizeCalculation: (val) => val.length + 1`.

### TTL Handling

| Cache | TTL Configuration | Mechanism |
|-------|-------------------|-----------|
| Custom LRUCache (2726.js) | `maxAge` in ms | Checked on every `set()`: iterates from oldest, evicts if `Date.now() - lastAccessed > maxAge` |
| npm lru-cache (2492.js) | `ttl` in ms, `ttlResolution`, `ttlAutopurge` | Dedicated TTL arrays (`ttls[]`, `starts[]`); stale check: `now - starts[i] > ttls[i]`; optional auto-purge via `setTimeout` |
| FileCache (3901.js) | `ttlMs` (default 30000ms) | Lazy TTL check: `Date.now() - entry.timestamp < ttlMs` on `getFiles()` |
| EncryptedFileStore (0866.js) | File mtime-based | Checks `mtimeMs` on load; if file changed externally, cache invalidated |

### LRU Eviction Algorithm

**Custom LRUCache (2726.js)**: Simple Map-based:
- Uses JavaScript `Map` which preserves insertion order
- On `set(key, value)`: deletes and re-inserts key to move to end (most recent)
- Eviction (`reA` private method): iterates from oldest entry; evicts while `size > capacity` OR `lastAccessed < (now - maxAge)`
- No size-based eviction, only count-based + TTL

**npm lru-cache (2492.js)**: Array-indexed doubly-linked list:
- Maintains `next[]` and `prev[]` arrays for LRU ordering
- `head` = most recently used, `tail` = least recently used
- `moveToTail(index)` relinks pointers to move accessed item to tail
- Eviction (`#x` method): evicts from head (LRU position), calls `dispose()` callback
- Supports both `max` (count) and `maxSize` (byte-like) eviction
- `sizeCalculation` function allows custom size metrics per entry

### Disk Persistence

The LRU caches themselves are **purely in-memory**: no built-in disk serialization. Persistence is handled at the application layer:

1. **JWTAccess**: Tokens cached in memory only; re-generated on expiry. JWT cache entry = `{ expiration: number, headers: { Authorization: string } }`.

2. **FileCache**: Results stored in memory Map. On miss, triggers filesystem crawl. No disk serialization of crawl results.

3. **EncryptedFileStore (0866.js)**: The most sophisticated persistence model:
   - Data stored on disk as encrypted files using **AES-256-GCM** encryption
   - Encryption key stored in OS keyring via `keyringStorage` (e.g., `keytar`)
   - Disk format: `base64(IV):base64(authTag):base64(ciphertext)`
   - Automatic migration from plaintext to encrypted format
   - Atomic write via `writeFile` with mode `0600` (owner-only permissions)
   - Cache invalidation: compares file `mtimeMs` on load

## Key Findings

1. **Two LRU implementations coexist**: The custom LRUCache (2726.js, ~66 loc) is a minimal Map-based implementation used only by GCP auth. The full npm `lru-cache` (2492.js) is a comprehensive solution used by the glob subsystem and potentially other areas. Both implement LRU + TTL eviction.

2. **No unified cache abstraction**: OpenDroid does not have a single CacheManager or CacheProvider interface. Each subsystem implements its own caching strategy appropriate to its needs. This is a pragmatic approach but could lead to inconsistent cache behavior.

3. **File-based persistence uses encryption**: The EncryptedFileStore (0866.js) represents the most sophisticated caching pattern in the system: it combines in-memory caching with AES-256-GCM encrypted disk storage and OS keyring integration. This is used for credential/auth token persistence.

4. **FileCache uses in-flight request deduplication**: Module 3901.js maintains an `inFlightCrawls` Map that prevents duplicate filesystem crawls for the same path, returning a shared Promise to concurrent callers. This is a significant performance optimization.

5. **npm lru-cache is feature-rich**: The vendor LRU cache (2492.js) supports background fetch (`fetch()` with AbortController), memoization (`memo()`), dispose/settle callbacks, size-based eviction, and TTL auto-purge: making it the most capable cache primitive in the system.

6. **Cache entry format varies by consumer**: JWTAccess stores `{ expiration, headers }`, FileCache stores `{ files, directories, timestamp, wasTruncated }`, EncryptedFileStore stores raw strings with mtime metadata. No unified CacheEntry type.

## Code Examples

### Custom LRUCache (2726.js): Core Implementation
```js
// Module 2726.js, line 43-65
class wgI {
  constructor(H) {
    _x.set(this, new Map());
    this.capacity = H.capacity;
    this.maxAge = H.maxAge;
  }
  set(H, A) {
    // neA: delete and re-insert to update position
    Uz(this, _x, "f").delete(H);
    Uz(this, _x, "f").set(H, { value: A, lastAccessed: Date.now() });
    // reA: evict oldest if over capacity or expired
    let A_cutoff = this.maxAge ? Date.now() - this.maxAge : 0;
    let L = Uz(this, _x, "f").entries().next();
    while (!L.done && (Uz(this, _x, "f").size > this.capacity || L.value[1].lastAccessed < A_cutoff))
      Uz(this, _x, "f").delete(L.value[0]);
  }
}
```

### FileCache Key Construction (3901.js)
```js
// Module 3901.js, line 132-136
static getCacheKey(H, A) {
  return `${H}\x00${A.maxFiles}\x00${A.maxDepth}\x00${A.showHidden}\x00${A.respectGitignore}\x00${A.excludePatterns.join(",")}`;
}
```

### JWTAccess Cache Usage (2741.js)
```js
// Module 2741.js, line 14-18
this.cache = new yJf.LRUCache({ capacity: 500, maxAge: 3600000 }); // 500 entries, 1 hour TTL
```

### EncryptedFileStore Cache Invalidation (0866.js)
```js
// Module 0866.js, line 111-116
async load() {
  if (this.cache !== void 0)
    if ((await this.getFileMtimeMs()) !== this.cacheMtimeMs)
      { this.cache = void 0; this.cacheMtimeMs = null; }  // Invalidate on external change
    else return this.cache;
  // ... proceed to read from disk
}
```

### npm lru-cache Eviction (2492.js)
```js
// Module 2492.js, line 553-567 (#x method: evict LRU entry)
#x(H) {
  let A = this.#B;  // head = LRU position
  let L = this.#M[A];  // key at head
  let $ = this.#$[A];  // value at head
  if (this.#K && this.#P($)) $.__abortController.abort(Error("evicted"));
  else if (this.#w || this.#G) {
    if (this.#w) this.#D?.($, L, "evict");      // dispose callback
    if (this.#G) this.#F?.push([$, L, "evict"]); // disposeAfter callback
  }
  this.#f.delete(L); this.#U--;  // remove from keyMap, decrement count
}
```

## Integration Points

- **infra-auth**: EncryptedFileStore (0866.js) persists auth credentials (API keys, tokens) to encrypted disk storage with in-memory cache. JWTAccess (2741.js) caches JWT bearer tokens in LRUCache.
- **infra-config-loader**: FileCache (3901.js) may be used during config file discovery and workspace scanning.
- **infra-session-manager**: Session files may benefit from EncryptedFileStore for secure persistence (same pattern).
- **infra-logging-winston**: Cache hit/miss metrics could be logged for cache performance monitoring.
- **infra-telemetry-otel**: Module 1461.js tracks `ai.usage.cachedInputTokens`: indicates model response caching integration with OpenTelemetry spans.
- **infra-fs-helpers**: PathScurry (3587.js) with its ResolveCache and ChildrenCache accelerates filesystem operations used throughout the tool system.
- **infra-plugin-manager**: FileCache (3901.js) file discovery likely used during plugin scanning phase.
- **03-tool-agent-system (Tool System)**: Tools that interact with the filesystem would benefit from the FileCache layer for file discovery operations.

## Implementation Notes

1. **Replace custom LRUCache with a standard library**: The custom LRUCache (2726.js) is minimal and can be replaced by any standard LRU library or `Map` with TTL wrapper. Consider `lru-cache` npm package or a simple `Map` + `setTimeout` pattern.

2. **Extract EncryptedFileStore as a reusable module**: The EncryptedFileStore pattern (0866.js): in-memory cache + encrypted disk persistence + OS keyring: is highly reusable for any sensitive data persistence needs in OpenDroid.

3. **Unify cache configuration**: Consider a single cache configuration object that all subsystems reference, defining default TTLs, capacities, and eviction policies. This avoids the current scattered configuration.

4. **Add cache metrics/observability**: No cache hit/miss metrics are currently exposed. For OpenDroid, add Prometheus-style counters (`cache_hits_total`, `cache_misses_total`, `cache_evictions_total`) to each cache instance.

5. **Implement cache warming**: For startup performance, consider pre-populating caches (especially FileCache and ResolveCache) during initialization rather than lazy-loading on first access.

## Module Reference

| Module | Line | Function/Class | Description |
|--------|------|----------------|-------------|
| 2726.js | 43 | `class wgI` | Custom LRUCache constructor (capacity + maxAge) |
| 2726.js | 47 | `set(H, A)` | Store value with LRU re-insertion |
| 2726.js | 52 | `get(H)` | Retrieve value with LRU promotion |
| 2726.js | 56 | `neA (private)` | Internal: update entry timestamp and position |
| 2726.js | 61 | `reA (private)` | Internal: evict entries exceeding capacity or TTL |
| 2492.js | 130 | `class bwH` | npm lru-cache main class |
| 2492.js | 246 | `getRemainingTTL(H)` | Get remaining TTL for a key |
| 2492.js | 293 | `#Q (private)` | isStale check: `now - starts[i] > ttls[i]` |
| 2492.js | 415 | `purgeStale()` | Remove all stale entries |
| 2492.js | 467 | `set(H, A, L)` | Store with TTL, size, status tracking |
| 2492.js | 553 | `#x (private)` | Evict LRU entry from head position |
| 2492.js | 653 | `fetch(H, A)` | Async fetch with background refresh |
| 2492.js | 737 | `memo(H, A)` | Memoization via memoMethod callback |
| 2741.js | 14 | `constructor` | JWTAccess with LRUCache(500, 3600000ms) |
| 2741.js | 22 | `getCachedKey(H, A)` | Build composite cache key from url + scopes |
| 2741.js | 30 | `getRequestHeaders(H, A, L)` | Get cached JWT or generate new one |
| 3587.js | 131 | `class _0A` | ResolveCache extends LRUCache(max: 256) |
| 3587.js | 137 | `class DUL` | ChildrenCache extends LRUCache(maxSize: 16384) |
| 3901.js | 38 | `class ufA` | FileCache with Map + TTL + in-flight dedup |
| 3901.js | 49 | `getFiles(H, A)` | Cached file crawl with TTL check |
| 3901.js | 132 | `static getCacheKey(H, A)` | Null-byte delimited composite key builder |
| 3901.js | 100 | `invalidateCache(H)` | Selective or full cache invalidation |
| 0866.js | 13 | `class y2A` | EncryptedFileStore constructor |
| 0866.js | 58 | `encrypt(H, A)` | AES-256-GCM encryption |
| 0866.js | 69 | `decrypt(H, A)` | AES-256-GCM decryption |
| 0866.js | 111 | `load()` | Load with mtime-based cache invalidation |
| 0866.js | 141 | `save(H)` | Encrypt and persist to disk |
