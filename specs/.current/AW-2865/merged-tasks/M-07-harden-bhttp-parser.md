# Harden BHTTP parser (bounds, size caps, status code, request cap)

**Estimate:** 2d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

`lib/src/bhttp.dart` has four overlapping parser-robustness gaps that together expose the library to crashes, DoS, and silent acceptance of malformed input from a malicious or compromised gateway. A wallet that trusts the gateway only to the extent of its cryptographic guarantees still needs the BHTTP layer to fail gracefully on anything outside the spec:

1. **`RangeError` instead of `FormatException`**: `decodeVarint` (lines 44–67) and the `sublist` calls at lines 173, 178, and 187 throw `RangeError` on truncated input. `RangeError` is an `Error`, not an `Exception`, so callers wrapping BHTTP parsing in `try { ... } on FormatException { ... }` — the natural Dart idiom — will not catch it and the program will crash.
2. **No size caps on response headers / body**: `parseResponse` (lines 164–187) reads `headersLen` and `contentLen` as raw varints. An adversarial gateway can declare `headersLen = 2^30`, forcing a ~1 GiB allocation in `data.sublist(...)` at line 173 or 178. Similarly, the client-side `gatewayResponse.bodyBytes` is passed to `ohttpDecapsulate` at `lib/src/ohttp_client.dart:129-133` with no length check. On a mobile wallet, even a sub-OOM allocation in the hundreds of MiB is enough to terminate the app.
3. **No size cap on `serializeRequest` inputs**: lines 74–111 apply no upper bound on `method`, `scheme`, `authority`, `path`, `headers`, or `body` before they are varint-encoded. An accidental wallet bug that passes a multi-megabyte body produces a giant BHTTP request, ciphertext, and POST to the gateway before failing.
4. **No status-code range validation**: lines 148–162 read `statusCode` as a raw varint and only skip informational `1xx` responses. A malformed or malicious gateway response could carry a status of 999, 0, or `2^62 - 1` and `parseResponse` would happily return it. Downstream wallet code that switches on the status code may behave unpredictably.

RFC references: RFC 9292 §3.3 (framing), §3.5.3 (response framing), §3.5.4 (varint via RFC 9000 §16); RFC 9110 §15 (status code range).

## Technical Details

**1. Bounds-check helper** (`lib/src/bhttp.dart:44-67, 173, 178, 187`)

Introduce a private helper (e.g., `_requireBytes(data, offset, n)`) that raises a `FormatException` with a clear message when `offset + n > data.length`. Apply it before every indexed read (`data[offset]`, `data[offset + i]`) and `sublist(...)` call in `decodeVarint` and in the response/request parsing paths. The change is mechanical and contained; no API change beyond replacing `RangeError` with `FormatException` for these specific call sites.

**2. Response size caps** (`lib/src/bhttp.dart:164-187`, `lib/src/ohttp_client.dart:129-133`)

Apply two layered caps with sensible defaults appropriate for wallet RPC payloads (single-digit MiB at most), both configurable:

- BHTTP layer: in `parseResponse`, check `headersLen` and `contentLen` against the configured upper bounds before allocating. Reject oversized declarations with `FormatException` (using the helper from step 1).
- OHTTP client layer: in `OhttpClient.send` (lines 129–133), check `gatewayResponse.bodyBytes.length` against the configured upper bound before handing to `ohttpDecapsulate`. Reject with the typed library exception.

**3. Request size caps** (`lib/src/bhttp.dart:74-111`)

Apply configurable upper bounds on `serializeRequest` inputs (`method`, `scheme`, `authority`, `path`, header name/value counts and lengths, `body` length) before serialization begins. Reject oversized inputs with a typed library exception. Defaults are conservative and documented; wallet RPC payloads rarely exceed a few hundred kilobytes.

**4. Status-code range validation** (`lib/src/bhttp.dart:148-162`)

After reading the final (non-informational) status code, validate it falls in 100–599 inclusive (RFC 9110 §15). Raise `FormatException` on out-of-range values. Document the constraint in the `parseResponse` doc comment.

**5. Tests** (`test/bhttp_test.dart`)

Add tests in the response-parsing group:
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

- All `data[offset]`, `data[offset + i]`, and `data.sublist(...)` operations in the BHTTP parser go through a shared bounds-checking helper or have an equivalent guard, and raise `FormatException` (never `RangeError`) on truncation.
- `try { ... } on FormatException { ... }` correctly catches every truncation scenario.
- `parseResponse` rejects header sections and content larger than the configured caps with `FormatException`; defaults are conservatively low and documented.
- `OhttpClient.send` rejects gateway responses larger than the configured cap before parsing.
- `serializeRequest` rejects inputs that exceed configured upper bounds for method/scheme/authority/path/headers/body length, with a typed library exception.
- `parseResponse` rejects status codes outside 100–599 with `FormatException`; doc comment cites RFC 9110 §15.
- Unit tests cover every scenario enumerated under "Tests" above.

## Additional

- Tests: comprehensive negative-input unit tests for the parser; positive tests for the boundary cases (status 100/200/599; cap-1/cap exactly/cap+1).
- The size-cap configuration surface should align with whatever options structure other client tasks introduce (consolidate in one place).
