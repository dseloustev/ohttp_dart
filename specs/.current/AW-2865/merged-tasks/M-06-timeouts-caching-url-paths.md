# Add HTTP timeouts, KeyConfig caching, and normalize URL paths

**Estimate:** 2d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

`OhttpClient.send()` has three independent network-reliability gaps that compound into a poor user experience for a wallet:

1. Neither the KeyConfig GET (`_httpClient.get(...)` at `lib/src/ohttp_client.dart:81-83`) nor the gateway POST (`_httpClient.post(...)` at lines 115–119) has a `.timeout(...)` chain. A non-responsive gateway can stall an `await` indefinitely. For an interactive wallet UI, this surfaces as a frozen screen with no progress indication and no escape hatch — the user must kill the app.
2. There is no KeyConfig cache. Every `send()` fires a fresh discovery GET (lines 81–89), doubling the round trips for every RPC call (balance refresh, transaction list, fee estimation). For a wallet that fires many calls in quick succession, this doubles latency and traffic for no benefit — the KeyConfig is long-lived material the gateway intentionally rotates infrequently. The library is also a poor citizen toward the relay operator, which sees two requests per logical operation.
3. URL composition is done by string concatenation at three call sites (`lib/src/ohttp_client.dart:82, 116, 154`) without normalizing slashes. A caller who supplies `gatewayBaseUrl: 'https://gw.example/'` and `configPath: '/key'` produces `https://gw.example//key` — a double slash that most servers tolerate but a strict reverse proxy may reject. The mirror case (no trailing slash on the base, no leading slash on the path) produces a path that misses the separator entirely.

## Technical Details

**1. Configurable timeouts** (`lib/src/ohttp_client.dart:81-83, 115-119`)

Add `.timeout(...)` chains to both HTTP calls in `OhttpClient.send()`. Allow the caller to override via `OhttpGatewayConfig` (or a new `OhttpClientOptions` shape) with sensible defaults. Document the defaults. On timeout, throw a library-owned typed exception.

Decide separately on retry semantics — at minimum, document the absence of automatic retries and the caller's responsibility.

**2. KeyConfig cache with TTL** (`lib/src/ohttp_client.dart:69-89`)

`OhttpClient` today has only `_httpClient` and `gateway` fields. Add a `KeyConfig?` cache field with a configurable TTL. Cache hits short-circuit the discovery GET. On a request failure that suggests a stale config (specific gateway error code or AEAD auth failure on response), invalidate the cache and retry the discovery once.

- TTL is configurable via `OhttpGatewayConfig` (or `OhttpClientOptions`); the default must reflect realistic key rotation cadence and be documented.
- Concurrency: Dart is single-isolate; a simple guarded `Future<KeyConfig>` is sufficient to ensure overlapping `send()` calls share one in-flight fetch.

**3. URL path normalization** (`lib/src/ohttp_client.dart:82, 116, 154`)

Three call sites currently concatenate strings:
- Line 82: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}')`
- Line 116: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.requestPath}')`
- Line 154: `Uri.parse('${gateway.effectiveDirectBaseUrl}$path')` (or its private replacement if the sendDirect privacy task has landed first)

Introduce a single helper that normalizes slash handling (handles all four cases: base with/without trailing slash × path with/without leading slash) and validates that the constructed `Uri` is well-formed before returning. The three call sites delegate to the helper.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- Both `_httpClient.get` and `_httpClient.post` calls in `OhttpClient.send` chain `.timeout(...)` with a configurable duration.
- Default timeout is documented in the `OhttpGatewayConfig` (or `OhttpClientOptions`) doc comment.
- Timeout produces a typed library exception (the same hierarchy used elsewhere in the client).
- `OhttpClient` exposes a cache field and TTL configuration for the parsed `KeyConfig`.
- First `send()` populates the cache; subsequent `send()` calls within TTL skip the discovery GET.
- TTL is configurable and the default reflects realistic key rotation cadence; the cache invalidation policy is documented.
- A single helper handles URL composition for KeyConfig GET, gateway POST, and direct-send paths; it handles all four slash combinations and validates the result.
- Unit tests cover: timeout fires on a slow mock client; cache miss → populate; cache hit within TTL → no GET; cache expiry → re-fetch; invalidation-on-failure → re-fetch; URL composition for each of the four slash combinations.

## Additional

- Tests: unit tests using a mock `http.Client` for the timeout and caching paths; pure unit tests for the URL helper.
- Performance: cache hit must be observable in tests as "zero HTTP calls to the configPath endpoint."
