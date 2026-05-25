# Hardened transport: HTTP timeouts, OHTTP-layer response cap + BHTTP parser bounds/caps

**Estimate:** 2.5d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

Two paired robustness workstreams against a misbehaving or malicious gateway. Both emit new typed throws into the `OhttpException` hierarchy introduced in L-03; doing them together keeps the configuration surface (timeouts, caps) consolidated in one place rather than churned twice.

**Two concerns from the original L-03 are resolved by the L-01 restructure and are dropped from this task:**
- **KeyConfig caching** is provided by `KeyConfigCache` (TTL + single-flight de-dup + manual invalidation + injectable clock), introduced by L-01 alongside `OhttpSession`'s invalidate-on-`OhttpHttpException` policy. No further work needed here.
- **URL path normalization** is eliminated by L-01 — `HttpClientTransport` takes `Uri keysUrl` and `Uri gatewayUrl` directly, no string concatenation of `gatewayBaseUrl + configPath`. The four-slash-combinations problem is gone.

**Network-side hardening — `HttpClientTransport` has one gap:**

1. Neither `_client.get(_keysUrl)` (in `HttpClientTransport.fetchKeyConfig`) nor `_client.post(_gatewayUrl, ...)` (in `HttpClientTransport.postToGateway`) has a `.timeout(...)` chain. A non-responsive gateway can stall an `await` indefinitely. For an interactive wallet UI, this surfaces as a frozen screen with no progress indication and no escape hatch.

**Orchestration-side hardening — `OhttpSession.send` has one gap:**

2. **No size cap on gateway response bytes** before they are passed to `ohttpDecapsulate`. An adversarial gateway can return a multi-hundred-MiB body; on a mobile wallet, even a sub-OOM allocation in the hundreds of MiB is enough to terminate the app.

**Parser-side hardening — `lib/src/bhttp.dart` has four overlapping defects:**

3. **`RangeError` instead of `FormatException`**: `decodeVarint` (lines 44–67) and the `sublist` calls at lines 173, 178, and 187 throw `RangeError` on truncated input. `RangeError` is an `Error`, not an `Exception`, so callers wrapping BHTTP parsing in `try { ... } on FormatException { ... }` will not catch it and the program will crash.
4. **No size caps on response headers / body**: `parseResponse` (lines 164–187) reads `headersLen` and `contentLen` as raw varints. An adversarial gateway can declare `headersLen = 2^30`, forcing a ~1 GiB allocation in `data.sublist(...)` at line 173 or 178.
5. **No size cap on `serializeRequest` inputs**: lines 74–111 apply no upper bound on `method`, `scheme`, `authority`, `path`, `headers`, or `body` before they are varint-encoded.
6. **No status-code range validation**: lines 148–162 read `statusCode` as a raw varint and only skip informational `1xx` responses. A malformed or malicious gateway response could carry a status of 999, 0, or `2^62 - 1` and `parseResponse` would happily return it.

RFC references: RFC 9292 §3.3 (framing), §3.5.3 (response framing), §3.5.4 (varint via RFC 9000 §16); RFC 9110 §15 (status code range); RFC 9458 §1 (transport).

## Technical Details

### Part A — Network-side hardening (`lib/src/adapters/http/http_transport.dart`)

**1. Configurable timeouts**

Add `.timeout(...)` chains to both `http.Client` calls in `HttpClientTransport`. Extend the constructor:

```dart
HttpClientTransport({
  required http.Client client,
  required Uri keysUrl,
  required Uri gatewayUrl,
  Duration keysTimeout = const Duration(seconds: 10),
  Duration gatewayTimeout = const Duration(seconds: 30),
});
```

Document the defaults in the doc comment. On timeout, throw `OhttpTimeoutException` (from L-03's hierarchy).

Decide separately on retry semantics — at minimum, document the absence of automatic retries and the caller's responsibility.

### Part B — Orchestration-side hardening (`lib/src/ohttp_session.dart`)

**2. Response body size cap**

In `OhttpSession.send`, after `_transport.postToGateway(...)` returns and before calling `ohttpDecapsulate`, verify `gatewayResponseBytes.length <= _maxResponseBytes`. Reject with `OhttpParseException` (from L-03's hierarchy).

Make the cap configurable via the `OhttpSession` constructor (and `OhttpSession.withTransport`), with a sensible wallet-RPC default in the single-digit MiB range. Document the default.

### Part C — BHTTP parser hardening (`lib/src/bhttp.dart`)

**3. Bounds-check helper** (lines 44–67, 173, 178, 187)

Introduce a private helper (e.g., `_requireBytes(data, offset, n)`) that raises a `FormatException` with a clear message when `offset + n > data.length`. Apply it before every indexed read (`data[offset]`, `data[offset + i]`) and `sublist(...)` call in `decodeVarint` and in the response/request parsing paths. The change is mechanical and contained; no API change beyond replacing `RangeError` with `FormatException` for these specific call sites.

**4. Response size caps in `parseResponse`** (lines 164–187)

Apply configurable upper bounds on `headersLen` and `contentLen` declarations in `parseResponse`, before allocating. Reject oversized declarations with `FormatException` (using the helper from step 3). Defaults are conservatively low (single-digit MiB) and documented; the same option surface should be reachable through `OhttpSession` so callers configure caps in one place (BHTTP-level defaults exist for direct `bhttp.parseResponse` users; `OhttpSession` passes through its configured cap when invoking the parser).

**5. Request size caps in `serializeRequest`** (lines 74–111)

Apply configurable upper bounds on `serializeRequest` inputs (`method`, `scheme`, `authority`, `path`, header name/value counts and lengths, `body` length) before serialization begins. Reject oversized inputs with `OhttpParseException` (or — if a stricter typed name is preferred — `OhttpRequestTooLargeException`; coordinate naming with the L-03 hierarchy). Defaults are conservative and documented; wallet RPC payloads rarely exceed a few hundred kilobytes.

**6. Status-code range validation** (lines 148–162)

After reading the final (non-informational) status code, validate it falls in 100–599 inclusive (RFC 9110 §15). Raise `FormatException` on out-of-range values. Document the constraint in the `parseResponse` doc comment.

### Part D — Tests

**7. Network tests** (`test/adapters/http_adapter_test.dart`):
- `MockClient` with `Future.delayed(Duration(seconds: 5))` exceeding the configured `keysTimeout` → `HttpClientTransport.fetchKeyConfig` throws `OhttpTimeoutException`.
- Same for `postToGateway` against the configured `gatewayTimeout`.
- The timeout exception is catchable as `on OhttpException`.

**8. Session tests** (`test/ohttp_session_test.dart`):
- Gateway response one byte above the configured cap → `OhttpSession.send` throws `OhttpParseException` before calling `ohttpDecapsulate`.
- Gateway response at exactly the cap → passes the cap check (decap may fail downstream; assert only the cap check passed).

**9. Parser tests** (`test/bhttp_test.dart`):
- Truncated input at each cited offset: varint single-byte boundary, varint 2/4/8-byte boundary, header name, header value, content. Assert `FormatException` (not `RangeError`).
- Oversized `headersLen` declaration → `FormatException`.
- Oversized `contentLen` declaration → `FormatException`.
- Status code: 100 (accept after informational skip), 200 (accept), 599 (accept), 600 (reject), 99 (reject), `2^62 - 1` (reject).
- `serializeRequest` rejects each over-cap input class (method/scheme/authority/path/header-name/header-value/body) with the typed library exception.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

**Timeouts:**
- `HttpClientTransport` constructor accepts configurable `keysTimeout` and `gatewayTimeout` durations; both `http.Client` calls chain `.timeout(...)`.
- Defaults are documented in the constructor doc comment; timeout produces `OhttpTimeoutException` (from L-03).

**Response cap (OHTTP layer):**
- `OhttpSession` constructor (and `OhttpSession.withTransport`) accept a configurable `maxResponseBytes` cap with a documented wallet-RPC-appropriate default.
- `OhttpSession.send` rejects gateway response bodies above the cap with `OhttpParseException` before calling `ohttpDecapsulate`.

**BHTTP parser:**
- All `data[offset]`, `data[offset + i]`, and `data.sublist(...)` operations in the BHTTP parser go through a shared bounds-checking helper or have an equivalent guard, and raise `FormatException` (never `RangeError`) on truncation.
- `try { ... } on FormatException { ... }` correctly catches every truncation scenario.
- `parseResponse` rejects header sections and content larger than the configured caps with `FormatException`; defaults are conservatively low and documented.
- `serializeRequest` rejects inputs that exceed configured upper bounds for method/scheme/authority/path/headers/body length, with a typed library exception.
- `parseResponse` rejects status codes outside 100–599 with `FormatException`; doc comment cites RFC 9110 §15.

**Tests:**
- Unit tests cover every scenario enumerated under "Tests" above.

## Additional

- The size-cap configuration surface for BHTTP-layer (`parseResponse`, `serializeRequest`) defaults exists at the BHTTP API level; `OhttpSession` exposes its own `maxResponseBytes` cap for the OHTTP-layer check. These two layers protect against different attack surfaces (oversized declaration vs. oversized actual payload) and should remain independent.
- Depends on L-01 (the `HttpClientTransport`, `OhttpSession`, and `KeyConfigCache` types do not exist before the restructure lands; `bhttp.dart` is unchanged by L-01 but the OHTTP-layer cap belongs to the new `OhttpSession`).
- Depends on L-03 (uses `OhttpTimeoutException`, `OhttpParseException` from the typed hierarchy).
- L-06's BHTTP test additions (truncation cases asserting `FormatException`) assume the parser-hardening work in this task has landed.
- Dropped from the original L-03 (now handled by L-01): KeyConfig caching with TTL (now `KeyConfigCache`), URL path normalization for KeyConfig GET / gateway POST / direct-send (the direct-send path no longer exists; the other two now accept `Uri` directly).
- Source merged tasks (in `../merged-tasks/`): M-06 (timeouts/caching/URLs) — the caching and URL halves are absorbed by L-01; M-07 (BHTTP parser hardening) — fully preserved here, plus the OHTTP-layer response cap that previously lived on `OhttpClient.send`.
