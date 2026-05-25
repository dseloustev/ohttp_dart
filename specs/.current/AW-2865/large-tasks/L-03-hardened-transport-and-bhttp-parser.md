# Hardened transport: HTTP timeouts, KeyConfig caching, URL normalization + BHTTP parser bounds/caps

**Estimate:** 4d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

Two paired robustness workstreams against a misbehaving or malicious gateway. Both emit new typed throws into the `OhttpException` hierarchy introduced in L-02; doing them together keeps the configuration surface (timeouts, caps) consolidated in one options structure rather than churned twice.

**Network-side hardening — `OhttpClient.send()` has three independent gaps:**

1. Neither the KeyConfig GET (`_httpClient.get(...)` at `lib/src/ohttp_client.dart:81-83`) nor the gateway POST (`_httpClient.post(...)` at lines 115–119) has a `.timeout(...)` chain. A non-responsive gateway can stall an `await` indefinitely. For an interactive wallet UI, this surfaces as a frozen screen with no progress indication and no escape hatch.
2. There is no KeyConfig cache. Every `send()` fires a fresh discovery GET (lines 81–89), doubling the round trips for every RPC call (balance refresh, transaction list, fee estimation). For a wallet that fires many calls in quick succession, this doubles latency and traffic for no benefit — the KeyConfig is long-lived material the gateway intentionally rotates infrequently.
3. URL composition is done by string concatenation at three call sites (`lib/src/ohttp_client.dart:82, 116, 154`) without normalizing slashes. `gatewayBaseUrl: 'https://gw.example/'` plus `configPath: '/key'` produces `https://gw.example//key` — a double slash that a strict reverse proxy may reject. The mirror case (no trailing slash on the base, no leading slash on the path) produces a path that misses the separator entirely.

**Parser-side hardening — `lib/src/bhttp.dart` has four overlapping defects:**

1. **`RangeError` instead of `FormatException`**: `decodeVarint` (lines 44–67) and the `sublist` calls at lines 173, 178, and 187 throw `RangeError` on truncated input. `RangeError` is an `Error`, not an `Exception`, so callers wrapping BHTTP parsing in `try { ... } on FormatException { ... }` will not catch it and the program will crash.
2. **No size caps on response headers / body**: `parseResponse` (lines 164–187) reads `headersLen` and `contentLen` as raw varints. An adversarial gateway can declare `headersLen = 2^30`, forcing a ~1 GiB allocation in `data.sublist(...)` at line 173 or 178. Similarly, the client-side `gatewayResponse.bodyBytes` is passed to `ohttpDecapsulate` at `lib/src/ohttp_client.dart:129-133` with no length check. On a mobile wallet, even a sub-OOM allocation in the hundreds of MiB is enough to terminate the app.
3. **No size cap on `serializeRequest` inputs**: lines 74–111 apply no upper bound on `method`, `scheme`, `authority`, `path`, `headers`, or `body` before they are varint-encoded.
4. **No status-code range validation**: lines 148–162 read `statusCode` as a raw varint and only skip informational `1xx` responses. A malformed or malicious gateway response could carry a status of 999, 0, or `2^62 - 1` and `parseResponse` would happily return it.

RFC references: RFC 9292 §3.3 (framing), §3.5.3 (response framing), §3.5.4 (varint via RFC 9000 §16); RFC 9110 §15 (status code range); RFC 9458 §1 (transport).

## Technical Details

### Part A — Network-side hardening (`lib/src/ohttp_client.dart`)

**1. Configurable timeouts** (`lib/src/ohttp_client.dart:81-83, 115-119`)

Add `.timeout(...)` chains to both HTTP calls in `OhttpClient.send()`. Allow the caller to override via `OhttpGatewayConfig` (or a new `OhttpClientOptions` shape) with sensible defaults. Document the defaults. On timeout, throw `OhttpTimeoutException` from the L-02 hierarchy.

Decide separately on retry semantics — at minimum, document the absence of automatic retries and the caller's responsibility.

**2. KeyConfig cache with TTL** (`lib/src/ohttp_client.dart:69-89`)

`OhttpClient` today has only `_httpClient` and `gateway` fields. Add a `KeyConfig?` cache field with a configurable TTL. Cache hits short-circuit the discovery GET. On a request failure that suggests a stale config (specific gateway error code or AEAD auth failure on response), invalidate the cache and retry the discovery once.

- TTL is configurable via the options structure; the default must reflect realistic key rotation cadence and be documented.
- Concurrency: Dart is single-isolate; a simple guarded `Future<KeyConfig>` is sufficient to ensure overlapping `send()` calls share one in-flight fetch.

**3. URL path normalization** (`lib/src/ohttp_client.dart:82, 116, 154`)

Three call sites currently concatenate strings:
- Line 82: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}')`
- Line 116: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.requestPath}')`
- Line 154: `Uri.parse('${gateway.effectiveDirectBaseUrl}$path')` (or its private replacement, depending on L-01 outcome)

Introduce a single helper that normalizes slash handling (handles all four cases: base with/without trailing slash × path with/without leading slash) and validates that the constructed `Uri` is well-formed before returning. All three call sites delegate to the helper.

### Part B — BHTTP parser hardening (`lib/src/bhttp.dart`)

**4. Bounds-check helper** (`lib/src/bhttp.dart:44-67, 173, 178, 187`)

Introduce a private helper (e.g., `_requireBytes(data, offset, n)`) that raises a `FormatException` with a clear message when `offset + n > data.length`. Apply it before every indexed read (`data[offset]`, `data[offset + i]`) and `sublist(...)` call in `decodeVarint` and in the response/request parsing paths. The change is mechanical and contained; no API change beyond replacing `RangeError` with `FormatException` for these specific call sites.

**5. Response size caps** (`lib/src/bhttp.dart:164-187`, `lib/src/ohttp_client.dart:129-133`)

Apply two layered caps with sensible defaults appropriate for wallet RPC payloads (single-digit MiB at most), both configurable through the same options structure as Part A:

- BHTTP layer: in `parseResponse`, check `headersLen` and `contentLen` against the configured upper bounds before allocating. Reject oversized declarations with `FormatException` (using the helper from step 4).
- OHTTP client layer: in `OhttpClient.send` (lines 129–133), check `gatewayResponse.bodyBytes.length` against the configured upper bound before handing to `ohttpDecapsulate`. Reject with `OhttpParseException` (from the L-02 hierarchy).

**6. Request size caps** (`lib/src/bhttp.dart:74-111`)

Apply configurable upper bounds on `serializeRequest` inputs (`method`, `scheme`, `authority`, `path`, header name/value counts and lengths, `body` length) before serialization begins. Reject oversized inputs with a typed library exception. Defaults are conservative and documented; wallet RPC payloads rarely exceed a few hundred kilobytes.

**7. Status-code range validation** (`lib/src/bhttp.dart:148-162`)

After reading the final (non-informational) status code, validate it falls in 100–599 inclusive (RFC 9110 §15). Raise `FormatException` on out-of-range values. Document the constraint in the `parseResponse` doc comment.

### Part C — Tests

**8. Network tests** — using a mock `http.Client`:
- Timeout fires on a slow mock client; throws `OhttpTimeoutException`.
- Cache miss → populate; cache hit within TTL → no GET.
- Cache expiry → re-fetch; invalidation-on-failure → re-fetch.
- URL composition: cover each of the four slash combinations (base with/without trailing slash × path with/without leading slash) for KeyConfig GET, gateway POST, and direct-send paths.

**9. Parser tests** (`test/bhttp_test.dart`):
- Truncated input at each cited offset: varint single-byte boundary, varint 2/4/8-byte boundary, header name, header value, content. Assert `FormatException` (not `RangeError`).
- Oversized `headersLen` declaration → `FormatException`.
- Oversized `contentLen` declaration → `FormatException`.
- Oversized client-layer response body → typed library exception.
- Status code: 100 (accept after informational skip), 200 (accept), 599 (accept), 600 (reject), 99 (reject), `2^62 - 1` (reject).
- `serializeRequest` rejects each over-cap input class.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

**Timeouts / caching / URLs:**
- Both `_httpClient.get` and `_httpClient.post` calls in `OhttpClient.send` chain `.timeout(...)` with a configurable duration.
- Default timeout is documented in the `OhttpGatewayConfig` (or `OhttpClientOptions`) doc comment; timeout produces `OhttpTimeoutException`.
- `OhttpClient` exposes a cache field and TTL configuration for the parsed `KeyConfig`; first `send()` populates the cache; subsequent `send()` calls within TTL skip the discovery GET.
- TTL is configurable and the default reflects realistic key rotation cadence; the cache invalidation policy is documented.
- A single helper handles URL composition for KeyConfig GET, gateway POST, and direct-send paths; it handles all four slash combinations and validates the result.

**BHTTP parser:**
- All `data[offset]`, `data[offset + i]`, and `data.sublist(...)` operations in the BHTTP parser go through a shared bounds-checking helper or have an equivalent guard, and raise `FormatException` (never `RangeError`) on truncation.
- `try { ... } on FormatException { ... }` correctly catches every truncation scenario.
- `parseResponse` rejects header sections and content larger than the configured caps with `FormatException`; defaults are conservatively low and documented.
- `OhttpClient.send` rejects gateway responses larger than the configured cap before parsing.
- `serializeRequest` rejects inputs that exceed configured upper bounds for method/scheme/authority/path/headers/body length, with a typed library exception.
- `parseResponse` rejects status codes outside 100–599 with `FormatException`; doc comment cites RFC 9110 §15.

**Tests:**
- Unit tests cover every scenario enumerated under "Tests" above.
- Cache hit must be observable in tests as "zero HTTP calls to the configPath endpoint."

## Additional

- The size-cap and timeout configuration surface should be consolidated in one place (`OhttpGatewayConfig` or a new `OhttpClientOptions`) so this task and future tasks share a single options structure.
- This task depends on L-02 (uses `OhttpTimeoutException`, `OhttpParseException`, and the parser-hardened size caps may coordinate with the typed exception surface for `OhttpClient.send`).
- L-05's BHTTP test additions (truncation cases asserting `FormatException`) assume the parser-hardening work in this task has landed.
- Source merged tasks (in `../merged-tasks/`): M-06 (timeouts/caching/URLs), M-07 (BHTTP parser hardening).
