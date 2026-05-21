# Task 09: Implement KeyConfig caching with TTL

**Severity:** HIGH
**Vector:** KeyConfig management
**Files:** `lib/src/ohttp_client.dart:69-89`

**Evidence:** Cross-ref C-5. `OhttpClient` has two fields only: `_httpClient` and `gateway`. There is no `OhttpKeyConfig?` cache field. Every call to `send()` invokes a fresh GET at lines 81–83 and parses the response at line 89.

## Description

Each `send()` produces two network round trips: one for KeyConfig discovery and one for the encapsulated request. For a wallet that fires many RPC calls in quick succession (balance refresh, transaction list, fee estimation), this doubles the latency and traffic for no benefit — the KeyConfig is long-lived material the gateway intentionally rotates infrequently. The library is also a poor citizen toward the relay operator, which sees two requests per logical operation.

## Proposed change

Add a `KeyConfig` cache on `OhttpClient` with a configurable TTL and a defined invalidation policy. Cache hits short-circuit the discovery GET. On a request failure that suggests a stale config (e.g., a specific gateway error code or AEAD auth failure on response), invalidate the cache and retry the discovery once. The TTL must default to a value that reflects realistic key rotation cadence; document this default. The cache must be thread-safe (Dart single-isolate, so a simple guarded `Future<KeyConfig>` is sufficient).

## Acceptance criteria

- `OhttpClient` exposes a cache field (and TTL configuration) for the parsed `KeyConfig`.
- First `send()` populates the cache; subsequent `send()` calls within TTL skip the discovery GET.
- TTL is configurable via `OhttpGatewayConfig` (or `OhttpClientOptions`) and documented.
- Cache invalidation occurs on a defined failure signal; behaviour is documented.
- Unit test covers: cache miss → populate, cache hit → no GET, cache expiry → re-fetch, invalidation-on-failure → re-fetch.
