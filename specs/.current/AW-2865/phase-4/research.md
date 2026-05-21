# Phase 4 Research — OhttpClient Layer Audit
## AW-2865 · ohttp_dart investigation

**Ticket:** AW-2865
**Phase:** 4 of 7 (Iteration 4)
**Status:** RESEARCH_COMPLETE
**Date:** 2026-05-21
**Scope file:** `lib/src/ohttp_client.dart` (177 lines)
**Dependencies audited in prior phases:** `lib/src/hpke.dart` (Phase 1), `lib/src/bhttp.dart` (Phase 2), `lib/src/ohttp.dart` (Phase 3)

---

## Phase Scope

Phase 4 targets Layer 4 of the four-layer stack: `lib/src/ohttp_client.dart`. This is the only
layer a wallet integrator directly calls. It orchestrates the complete OHTTP round trip:

1. GET `KeyConfig` from gateway (`http.Client.get`)
2. BHTTP-serialize the inner request (`bhttp.serializeRequest`)
3. OHTTP-encapsulate (`ohttpEncapsulate`)
4. POST to gateway (`http.Client.post`)
5. OHTTP-decapsulate (`ohttpDecapsulate`)
6. BHTTP-parse inner response (`bhttp.parseResponse`)

It also exposes `sendDirect()`, a plain-HTTP escape hatch that bypasses OHTTP entirely.

This phase does not re-examine the lower-layer findings that are already recorded in Phases 1–3.
Cross-layer references appear where a client-layer gap compounds a lower-layer gap.

---

## Resolved Questions

### Q1: How does `SecretBoxAuthenticationError` surface to the wallet caller at the `OhttpClient.send()` level?

**Answer (user-directed):** Analyze fully at the client boundary.

**Finding:** `SecretBoxAuthenticationError` is defined in `package:cryptography/src/cryptography/mac.dart`
as `class SecretBoxAuthenticationError implements Exception`. It is thrown inside
`ohttpDecapsulate` at `ohttp.dart:219` when `aesGcm.decrypt(...)` detects a MAC mismatch. The
call chain at the client level is:

```
OhttpClient.send()
  → await ohttpDecapsulate(encResult.enc, encResult.exportedSecret, gatewayResponse.bodyBytes)
      → aesGcm.decrypt(secretBox, secretKey: ..., aad: [])
          throws SecretBoxAuthenticationError
```

The `await ohttpDecapsulate(...)` call is at `ohttp_client.dart:129-133` and has **no
surrounding try/catch**. `send()` itself has no try/catch block at all — the entire method body
from line 69 to line 145 is a bare sequence of `await` expressions and `if`-status-code checks
with `throw Exception(...)`. Therefore:

- `SecretBoxAuthenticationError` propagates unmodified through `ohttpDecapsulate` and through
  `OhttpClient.send()` to the wallet caller.
- The caller receives a `package:cryptography`-typed exception — a transitive internal dependency
  type — with no `ohttp_dart`-level wrapper.
- A wallet caller catching `Exception` will catch it (because `SecretBoxAuthenticationError
  implements Exception`). A caller catching only `OhttpClientException` (which does not exist
  yet) would miss it entirely.
- A caller that wants to distinguish "AEAD authentication failure (possible replay or active
  attacker)" from "network error" must currently import `package:cryptography` and catch
  `SecretBoxAuthenticationError` explicitly, coupling the wallet to an internal implementation
  dependency.

This is the same finding as Phase 3 O-9 (`ohttp.dart:219`), confirmed to propagate all the way
to `OhttpClient.send()` without any intervening catch or wrapping.

### Q2: What exception types does `package:http` actually throw?

**Answer (user-directed):** Inspect `package:http` source and cross-reference with `ohttp_client.dart`.

**Finding (from `package:http` 1.6.0 source):**

The `http` package defines exactly one exception class:

- `ClientException` (`lib/src/exception.dart`, line 6): `class ClientException implements Exception`
  — message string plus optional `Uri? uri`. This is the base type for all transport-level failures.

Two concrete subtypes:

- `_ClientSocketException extends ClientException implements SocketException`
  (`lib/src/io_client.dart`, line 27): thrown when `dart:io` `SocketException` occurs (DNS
  failure, connection refused, network unreachable). Implements both `ClientException` and
  `SocketException` so callers can catch either.
- `RequestAbortedException extends ClientException`
  (`lib/src/abortable.dart`, line 40): thrown when an `Abortable` request's `abortTrigger`
  completes. Not reachable from `ohttp_client.dart` because it uses plain `http.Client.get()` /
  `http.Client.post()` which do not pass an `Abortable` request.

`IOClient.send()` (`lib/src/io_client.dart`, lines 226–230) wraps:
- `SocketException` → `_ClientSocketException` (subtype of `ClientException`)
- `HttpException` → `ClientException`

All other `dart:io` errors (e.g., `HandshakeException` for TLS failures) propagate **unwrapped**
from `IOClient.send()` — they are **not** caught and re-wrapped by the `http` package. The
`on SocketException` and `on HttpException` handlers in `IOClient.send()` are the only two
catch clauses.

**Cross-reference with `ohttp_client.dart`:**

`ohttp_client.dart` imports `package:http` but never imports `package:http/src/exception.dart`
directly. The `send()` method makes two `http.Client` calls:

1. `_httpClient.get(...)` at line 81 — can throw `ClientException` (DNS, connection refused),
   `_ClientSocketException` (socket-level failure), `HandshakeException` (TLS failure — unwrapped
   by http package), or any other `dart:io` exception for TLS/SSL issues.
2. `_httpClient.post(...)` at line 115 — same set.

None of these are caught. They propagate through `send()` to the wallet caller as-is:
- `ClientException` (which `implements Exception`) — catchable with `on Exception`
- `_ClientSocketException` (a private type in the `http` package) — only catchable via
  `on ClientException` or `on SocketException` (since it implements both)
- `HandshakeException` — a `dart:io` type, catchable only with explicit `dart:io` import

The wallet caller receives a heterogeneous mix of exception types without any unifying
`ohttp_dart`-owned type. There is no documented contract in `OhttpClient` describing what
exceptions `send()` can throw.

### Q3: No other architectural preferences from user.

---

## Related Modules / Services

### Primary audit target

| File | Layer | Lines | Role |
|---|---|---|---|
| `lib/src/ohttp_client.dart` | 4 (top) | 177 | `OhttpClient`, `OhttpGatewayConfig`, `OhttpResponse`, `OhttpHeader` |

### Direct dependencies called by `ohttp_client.dart`

| Symbol | Source file | Call site in ohttp_client.dart |
|---|---|---|
| `bhttp.serializeRequest` | `lib/src/bhttp.dart` | line 97–104 |
| `bhttp.parseResponse` | `lib/src/bhttp.dart` | line 136 |
| `ohttpEncapsulate` | `lib/src/ohttp.dart` | line 108 |
| `ohttpDecapsulate` | `lib/src/ohttp.dart` | line 129–133 |
| `OhttpKeyConfig.parse` | `lib/src/ohttp.dart` | line 89 |
| `http.Client.get` | `package:http` 1.6.0 | line 81 |
| `http.Client.post` | `package:http` 1.6.0 | line 115 |
| `http.Client.send` | `package:http` 1.6.0 | line 163 (`sendDirect`) |

### Exception types that reach `OhttpClient.send()` callers

| Type | Origin | Reachable via |
|---|---|---|
| `Exception` (generic) | `ohttp_client.dart:85,121` | `if (statusCode != 200) throw Exception(...)` |
| `FormatException` | `lib/src/ohttp.dart` / `lib/src/bhttp.dart` | `OhttpKeyConfig.parse`, `bhttp.parseResponse` |
| `UnsupportedError` | `lib/src/ohttp.dart:82-85` | `OhttpKeyConfig.validate()` |
| `SecretBoxAuthenticationError` | `package:cryptography` | `ohttpDecapsulate` → `aesGcm.decrypt` |
| `RangeError` | `lib/src/bhttp.dart` | `bhttp.parseResponse` (Phase 2 R-3) |
| `StateError` | `lib/src/hpke.dart` | HPKE seq overflow (Phase 1 F-3, not reachable in single-seal OHTTP flow) |
| `ClientException` | `package:http` | `http.Client.get/post` — DNS failure, connection refused |
| `_ClientSocketException` | `package:http` (private) | socket-level failure; is-a `ClientException` AND `SocketException` |
| `HandshakeException` | `dart:io` | TLS failure — NOT wrapped by `package:http` |
| `ArgumentError` | `package:http` | invalid body type in `BaseClient._sendUnstreamed` (not reachable from `ohttp_client.dart` which passes `Uint8List`) |

---

## Current Endpoints & Contracts

### `OhttpGatewayConfig` (lines 35–53)

```
const OhttpGatewayConfig({
  required String gatewayBaseUrl,   // no https enforcement; accepts any scheme
  required String configPath,       // concatenated verbatim: gatewayBaseUrl + configPath
  required String requestPath,      // concatenated verbatim: gatewayBaseUrl + requestPath
  required String targetAuthority,  // embedded verbatim as BHTTP authority; no allow-list
  String targetScheme = 'https',    // default 'https'; passed to bhttp.serializeRequest
  String? directBaseUrl,            // non-OHTTP escape URL; co-located with OHTTP fields
})

String get effectiveDirectBaseUrl => directBaseUrl ?? gatewayBaseUrl;
```

### `OhttpClient` (lines 59–176)

**Constructor:** `OhttpClient({http.Client? httpClient, required OhttpGatewayConfig gateway})`
- `_httpClient` defaults to `http.Client()` (creates `IOClient` on VM)
- No timeout wrapper, no retry policy, no cancellation token field

**`send()` — line 69–145**

Step-by-step contract confirmed by source:

| Step | Call site | Contract gap |
|---|---|---|
| 1 — GET KeyConfig | `_httpClient.get(Uri.parse(...), )` at line 81 | No `.timeout(...)`, no retry, no caching |
| Status check | `if (configResponse.statusCode != 200)` at line 84 | Generic `Exception` — no typed hierarchy |
| Parse KeyConfig | `OhttpKeyConfig.parse(configResponse.bodyBytes)` at line 89 | Can throw `FormatException` or `UnsupportedError` |
| 2 — Serialize | `bhttp.serializeRequest(...)` at lines 97–104 | No body size cap before serialization |
| 3 — Encapsulate | `ohttpEncapsulate(config, binaryRequest)` at line 108 | No body size cap; `testKeyPair` parameter not passed (correct) |
| 4 — POST gateway | `_httpClient.post(Uri.parse(...), ...)` at lines 115–119 | No `.timeout(...)`, no retry, no cancellation |
| Status check | `if (gatewayResponse.statusCode != 200)` at line 120 | Generic `Exception` — no typed hierarchy |
| 5 — Decapsulate | `ohttpDecapsulate(enc, exportedSecret, gatewayResponse.bodyBytes)` at lines 129–133 | No size cap on `gatewayResponse.bodyBytes`; `SecretBoxAuthenticationError` propagates unwrapped |
| 6 — Parse BHTTP | `bhttp.parseResponse(binaryResponse)` at line 136 | No size cap on `binaryResponse`; `RangeError` can propagate (Phase 2 R-3) |
| Build response | `OhttpHeader(name: h.$1, value: h.$2)` at line 139 | `name` stored as-received (not lowercased) |

**`sendDirect()` — lines 148–171**

```
Future<OhttpResponse> sendDirect({
  required String method,
  required String path,
  Map<String, String>? headers,
  Uint8List? body,
})
```

- Constructs `Uri.parse('${gateway.effectiveDirectBaseUrl}$path')` at line 154 — string concatenation, not `Uri.resolve`.
- Sends via `_httpClient.send(request)` at line 163 — plain HTTP, no OHTTP.
- **No** `assert`, `@Deprecated`, doc comment, or `onLog` call warning about OHTTP bypass.
- Returns `OhttpResponse` with headers from `streamedResponse.headers.entries` — keys are already lowercased by `IOClient` (dart:io header normalization), so `sendDirect` response headers are lowercased while `send()` response headers are not.

**`dispose()` — line 173–175**

Calls `_httpClient.close()`. No additional cleanup.

---

## Patterns Used

1. **`http.Client` injection** — `_httpClient` accepted via constructor; defaults to `http.Client()`. Standard pattern; enables testing with `MockClient`. No timeout or retry wrapper applied at construction time.

2. **String URL construction** — All gateway URLs are built by `'${string}${string}'` concatenation. The pattern `Uri.parse(gatewayBaseUrl + configPath)` at line 82 and `Uri.parse(gatewayBaseUrl + requestPath)` at line 116 does not call `Uri.resolve` or `Uri.https` to normalize paths.

3. **`onLog` callback** — `send()` accepts an optional `void Function(String)? onLog` callback (line 74) and calls it at six informational points. Only the gateway URL and byte counts are logged — no key material. Pattern is sound for observability but `sendDirect()` has no equivalent callback.

4. **Single-shot OHTTP flow** — `ohttpEncapsulate` is called once per `send()` invocation; the `testKeyPair` optional parameter is **not** passed (confirmed at line 108). The production code path generates a fresh ephemeral key pair on every call. Correct.

5. **Bare `await` chain** — `send()` is a linear sequence of `await` + status-code checks with no `try/catch` block. Any thrown exception from any step propagates directly to the caller unchanged.

---

## Limitations & Risks

All findings below are confirmed by direct line-count inspection of `lib/src/ohttp_client.dart` (177 lines).

---

### Finding C-1 — `OhttpGatewayConfig.gatewayBaseUrl` has no `https`-scheme enforcement

| Attribute | Value |
|---|---|
| **ID** | C-1 |
| **File : lines** | `lib/src/ohttp_client.dart:35-53` (constructor), `81-82` (GET), `115-116` (POST) |
| **Concern category** | Scheme enforcement / Transport security |
| **Severity** | HIGH |

`OhttpGatewayConfig` accepts any string for `gatewayBaseUrl`. Neither the constructor, the
`const` factory, nor any validation method checks `Uri.parse(gatewayBaseUrl).scheme == 'https'`.
The calls at lines 81–82 and 115–116 execute `Uri.parse(gateway.gatewayBaseUrl + ...)` which
accepts `http://` silently.

A wallet misconfigured with an `http://` gateway URL transmits the outer channel — including the
encapsulated BHTTP ciphertext — without TLS. A network observer can then correlate request size
and timing. The inner content remains encrypted, but the outer channel OHTTP threat model
(RFC 9458 §1) requires TLS to prevent metadata leakage.

**Draft task title:** "Enforce `https`-scheme on `OhttpGatewayConfig.gatewayBaseUrl`"

---

### Finding C-2 — `configPath`/`requestPath` string-concatenated, not `Uri`-normalized

| Attribute | Value |
|---|---|
| **ID** | C-2 |
| **File : lines** | `lib/src/ohttp_client.dart:82` (GET), `lib/src/ohttp_client.dart:116` (POST) |
| **Concern category** | URL construction |
| **Severity** | IMPROVEMENT |

Both call sites construct the final URL as bare string concatenation:
- Line 82: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}')`
- Line 116: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.requestPath}')`

Double-slash risk: if `gatewayBaseUrl = "https://gw.example.com/"` and `configPath = "/ohttp-configs"`,
the resulting URL is `"https://gw.example.com//ohttp-configs"`. `Uri.parse` accepts this silently;
some gateway implementations are sensitive to path normalization.

Path-traversal risk: if `configPath` or `requestPath` contains `..` components (e.g., from
user-supplied configuration), string concatenation does not normalize them away. `Uri.resolve(path)`
normalizes `..` segments; string concatenation does not.

`sendDirect()` exhibits the same pattern at line 154:
`Uri.parse('${gateway.effectiveDirectBaseUrl}$path')`.

**Draft task title:** "Replace string path concatenation with `Uri.resolve` normalization in `OhttpClient`"

---

### Finding C-3 — `targetAuthority` embedded verbatim with no allow-list

| Attribute | Value |
|---|---|
| **ID** | C-3 |
| **File : lines** | `lib/src/ohttp_client.dart:98-101` (bhttp.serializeRequest call) |
| **Concern category** | Privacy / SSRF |
| **Severity** | HIGH |

`bhttp.serializeRequest` at lines 97–104 receives `authority: gateway.targetAuthority` verbatim.
`OhttpGatewayConfig.targetAuthority` is a plain string with no URI-scheme stripping, no format
validation, and no allow-list check.

A misconfigured or adversarially supplied `targetAuthority` could route the inner BHTTP request to
an unintended server. In the OHTTP threat model the outer OHTTP relay cannot see the inner request
destination; accordingly there is no relay-level guard against SSRF. The privacy implication is:
a `targetAuthority` set to a wallet-operator-controlled server that the wallet developer intended
to keep private becomes trivially discoverable if the string originates from any untrusted input.

**Draft task title:** "Add allow-list validation or scheme-stripping for `targetAuthority` in `OhttpGatewayConfig`"

---

### Finding C-4 — `sendDirect()` has no privacy-impact warning; co-located with OHTTP paths

| Attribute | Value |
|---|---|
| **ID** | C-4 |
| **File : lines** | `lib/src/ohttp_client.dart:148-171` (`sendDirect`), `41` (`directBaseUrl` field), `52` (`effectiveDirectBaseUrl`) |
| **Concern category** | Privacy / Documentation |
| **Severity** | HIGH |

`sendDirect()` issues a plain HTTP request to `gateway.effectiveDirectBaseUrl + path` without any
OHTTP encapsulation. The method:
- Has no doc comment warning that OHTTP privacy is bypassed.
- Has no `assert` or runtime check.
- Has no `@Deprecated` annotation.
- Has no `onLog` callback parameter (unlike `send()`).
- Lives on the same class as `send()`, sharing the same `OhttpGatewayConfig` object.

RFC 9458 §1 states that OHTTP's purpose is to prevent the target server from learning the client's
IP address. `sendDirect()` sends directly to the target (via `directBaseUrl ?? gatewayBaseUrl`),
fully exposing the client IP, method, path, headers, and body to the target server. A wallet
developer using `sendDirect()` as a connectivity fallback silently degrades the wallet's privacy
guarantee with no runtime signal.

`effectiveDirectBaseUrl` (line 52) returns `directBaseUrl ?? gatewayBaseUrl`: if `directBaseUrl`
is null, `sendDirect()` sends directly to the OHTTP gateway URL rather than the intended target,
which is a configuration footgun.

**Draft task title:** "Add mandatory privacy-impact doc comment (or `@Deprecated` annotation) to `sendDirect()`, and add `onLog` bypass signal"

---

### Finding C-5 — KeyConfig GET: no timeout, no retry, no caching

| Attribute | Value |
|---|---|
| **ID** | C-5 |
| **File : lines** | `lib/src/ohttp_client.dart:81-89` |
| **Concern category** | Network reliability / KeyConfig lifecycle |
| **Severity** | HIGH |

`_httpClient.get(Uri.parse(...))` at line 81 returns a bare `Future<Response>` with:
- No `.timeout(Duration(...))` chained on the future.
- No retry loop on failure (neither on `ClientException` nor on 5xx status codes).
- No reference to any stored `OhttpKeyConfig` from a previous invocation.

`OhttpKeyConfig` is re-fetched fresh on every call to `send()`, producing two network round trips
per inner request (GET config + POST gateway). Under poor network conditions the config GET may
intermittently fail even when the gateway POST would succeed. There is no configurable TTL or
forced-refresh policy; the class has no field for a cached config value.

The `OhttpClient` constructor (lines 63–67) accepts an optional `http.Client` but provides no
field for a pre-fetched or cached `OhttpKeyConfig`. The only instance field is `_httpClient` and
`gateway`.

**Draft task title:** "Add configurable TTL-based KeyConfig cache and timeout to KeyConfig GET in `OhttpClient`"

---

### Finding C-6 — Gateway POST: no timeout, no retry, no cancellation hook

| Attribute | Value |
|---|---|
| **ID** | C-6 |
| **File : lines** | `lib/src/ohttp_client.dart:115-122` |
| **Concern category** | Network reliability |
| **Severity** | HIGH |

`_httpClient.post(Uri.parse(...), headers: ..., body: ...)` at line 115 returns a bare
`Future<Response>` with:
- No `.timeout(Duration(...))` chained on the future.
- No retry logic for 5xx responses.
- No `CancelableOperation` or `Abortable` pattern.

The `package:http` 1.6.0 `Abortable` mixin is available (`lib/src/abortable.dart`) but not used.
A stalled gateway causes `send()` to hang indefinitely with no recovery path in the wallet. Since
`OhttpClient` holds a shared `_httpClient`, the wallet has no per-request cancellation mechanism.

**Draft task title:** "Add configurable timeout and optional cancellation to gateway POST in `OhttpClient`"

---

### Finding C-7 — Non-200 responses throw generic `Exception`; no typed hierarchy

| Attribute | Value |
|---|---|
| **ID** | C-7 |
| **File : lines** | `lib/src/ohttp_client.dart:84-87` (KeyConfig), `120-122` (gateway POST) |
| **Concern category** | Error handling |
| **Severity** | HIGH |

Both error paths throw untyped `Exception` with a message string:
- Line 85: `throw Exception('Failed to fetch KeyConfig: HTTP ${configResponse.statusCode}')`
- Line 121: `throw Exception('Gateway error: HTTP ${gatewayResponse.statusCode}')`

There is no `OhttpClientException` type, no status-code field on the thrown object, no distinction
between 4xx (permanent) and 5xx (transient) responses, and no structured error object. A wallet
caller that wants to implement differentiated retry logic must parse the exception message string,
which is fragile and undocumented.

Cross-reference: the same anti-pattern recurs across all layers — `StateError` (Phase 1 F-3),
`RangeError` (Phase 2 R-3), `FormatException`/`UnsupportedError` inconsistency (Phase 3 O-4).

**Draft task title:** "Introduce `OhttpClientException` typed hierarchy carrying HTTP status code and error category"

---

### Finding C-8 — Network errors (`ClientException`, TLS `HandshakeException`) propagate as untyped exceptions; no documented throw contract

| Attribute | Value |
|---|---|
| **ID** | C-8 |
| **File : lines** | `lib/src/ohttp_client.dart:81` (GET), `115` (POST), `69` (method signature) |
| **Concern category** | Error handling / API contract |
| **Severity** | HIGH |

`OhttpClient.send()` has no documented throws contract (no doc comment listing exceptions). The
method can throw:

1. `http.ClientException` — DNS failure or connection refused (wrapped by `IOClient`)
2. `_ClientSocketException` — socket-level error (private `package:http` type; is-a `ClientException` and `SocketException`)
3. `dart:io.HandshakeException` — TLS/SSL handshake failure (NOT wrapped by `package:http`; propagates as `dart:io` type)
4. `Exception` (generic) — non-200 status code (thrown by `ohttp_client.dart` itself)
5. `FormatException` — malformed KeyConfig or BHTTP response
6. `UnsupportedError` — unsupported KEM from `OhttpKeyConfig.validate()`
7. `SecretBoxAuthenticationError` — AEAD authentication failure from `ohttpDecapsulate`
8. `RangeError` — truncated BHTTP body (Phase 2 R-3, propagates through `bhttp.parseResponse`)

A wallet caller cannot write a correct catch hierarchy without importing `dart:io`, `package:http`,
and `package:cryptography` — all internal implementation details of `ohttp_dart`. The public API
surface does not own any of these exception types.

The `HandshakeException` gap is the most operationally significant: a TLS certificate error on
the gateway is indistinguishable at the catch-site from a bug in the caller's code unless the
caller explicitly imports and catches `dart:io.HandshakeException`.

**Draft task title:** "Document and own `OhttpClient.send()` exception contract; wrap foreign exception types in library-owned types"

---

### Finding C-9 — No response body/header size cap before `ohttpDecapsulate` and `bhttp.parseResponse`

| Attribute | Value |
|---|---|
| **ID** | C-9 |
| **File : lines** | `lib/src/ohttp_client.dart:129-133` (decapsulate), `136` (parseResponse) |
| **Concern category** | Parser robustness / DoS |
| **Severity** | HIGH |

`gatewayResponse.bodyBytes` (the full gateway response body, buffered by `package:http`) is passed
directly to `ohttpDecapsulate` at line 129 with no length check. The result `binaryResponse` is
then passed to `bhttp.parseResponse` at line 136 with no length check.

Two-layer amplification (cross-reference Phase 2 finding R-2):
1. `ohttp_client.dart` imposes no cap on the gateway response body before decapsulation.
2. `bhttp.parseResponse` imposes no cap on header count or total header size (`bhttp.dart:165-182`, Phase 2 R-2).

A malicious gateway can force unbounded memory allocation in the client process through a single
oversized response. The attack surface requires only a compromised or spoofed OHTTP gateway; the
client IP is already known to the gateway, so this is a targeted attack vector. The mobile
deployment context (wallet) makes memory exhaustion particularly severe.

**Draft task title:** "Add configurable `maxResponseBytes` guard in `OhttpClient.send()` before `ohttpDecapsulate`"

---

### Finding C-10 — `OhttpHeader.name` not lowercased on response side; inconsistent with `sendDirect()` response path

| Attribute | Value |
|---|---|
| **ID** | C-10 |
| **File : lines** | `lib/src/ohttp_client.dart:139` (`send` response construction), `169` (`sendDirect` response construction) |
| **Concern category** | API consistency |
| **Severity** | IMPROVEMENT |

In `send()`, response headers are built at line 139:
```dart
headers: bhttpResp.headers.map((h) => OhttpHeader(name: h.$1, value: h.$2)).toList(),
```
`h.$1` is the header name as returned by `bhttp.parseResponse` — stored as-received from the
gateway binary, with no case normalization.

In `sendDirect()`, response headers are built at line 169:
```dart
headers: streamedResponse.headers.entries.map((e) => OhttpHeader(name: e.key, value: e.value)).toList(),
```
`streamedResponse.headers` comes from `IOClient`, which internally calls `dart:io`
`HttpHeaders.forEach`. `dart:io` normalizes header names to lowercase, so `sendDirect()` response
headers are lowercase while `send()` response headers are not.

RFC 7230 §3.2 specifies that HTTP header field names are case-insensitive. The inconsistency
forces wallet callers to implement case-insensitive comparison for the `send()` path but not for
`sendDirect()`.

**Draft task title:** "Lowercase `OhttpHeader.name` at construction time in `OhttpClient.send()` response path"

---

### Finding C-11 — `SecretBoxAuthenticationError` propagates unwrapped to `OhttpClient.send()` caller (cross-layer confirmation)

| Attribute | Value |
|---|---|
| **ID** | C-11 |
| **File : lines** | `lib/src/ohttp_client.dart:129-133` (call to `ohttpDecapsulate`), `lib/src/ohttp.dart:219` (throw site) |
| **Concern category** | Error handling / API boundary |
| **Severity** | HIGH |

This finding extends Phase 3 O-9 to confirm the propagation path through the client layer. There
is no `try/catch` anywhere in `OhttpClient.send()`. `SecretBoxAuthenticationError implements
Exception` (confirmed in `cryptography-2.9.0/lib/src/cryptography/mac.dart:52`), so:

- A caller `on Exception` will catch it (masking the authentication failure as generic).
- A caller cannot distinguish "AEAD failure = possible active attacker / replay" from "format
  error = malformed gateway response" without importing `package:cryptography`.
- The existing Phase 3 TASK-O5 ("Wrap `SecretBoxAuthenticationError` in `OhttpAuthenticationException`
  in `ohttp.dart`") remains the correct remediation site. Once TASK-O5 is done, the client-level
  propagation is resolved automatically because the wrapping occurs before the error reaches the
  client layer.

**Cross-phase note:** This finding does not require a separate task from TASK-O5. It is recorded
here as confirmation that the unwrapped error reaches the wallet boundary, not just the `ohttp.dart`
layer. The existing TASK-O5 addresses the full call chain.

---

### Finding C-12 — `onLog` at line 77 logs gateway base URL at every `send()` call

| Attribute | Value |
|---|---|
| **ID** | C-12 |
| **File : lines** | `lib/src/ohttp_client.dart:77-78` |
| **Concern category** | Privacy / Observability |
| **Severity** | IMPROVEMENT |

The first `onLog` call in `send()` logs:
```
'Fetching OHTTP KeyConfig from ${gateway.gatewayBaseUrl}${gateway.configPath}...'
```
`vision.md §7` lists constraint 6: "`gatewayBaseUrl` at DEBUG or lower in production builds" must
not be logged because it "identifies the gateway identity; correlating gateway identity with timing
is a metadata leak." The current log call has no guard against production builds and no log-level
distinction. The `onLog` callback is opt-in (caller supplies it), which provides some protection,
but the URL is constructed unconditionally in the string interpolation regardless of whether
`onLog` is null (Dart evaluates the argument even though `?.call` short-circuits the actual call
for null receivers — the string is constructed regardless).

**Draft task title:** "Guard `gatewayBaseUrl` and `configPath` from `onLog` output, or add log-level parameter to `onLog` callback"

---

## Cross-Layer References

### Phase 2 BHTTP findings that compound Phase 4 findings

| Phase 2 finding | Phase 4 finding | Compounding mechanism |
|---|---|---|
| R-2 — No header-count / header-size guard in `parseResponse` | C-9 | No size cap at client layer + no cap in parser = two-layer unbounded allocation |
| R-3 — `RangeError` on truncated body in `parseResponse` | C-8 | `RangeError` propagates through `ohttp_client.dart:136` to wallet caller unchanged |
| R-5 — No body size cap in `serializeRequest` | C-9 (partial) | Large inner request body → large BHTTP blob → large HPKE ciphertext; no cap at any layer |

### Phase 3 OHTTP findings confirmed or extended at Phase 4

| Phase 3 finding | Phase 4 finding | Notes |
|---|---|---|
| O-9 — `SecretBoxAuthenticationError` propagates unwrapped | C-11 | Confirmed at wallet boundary; TASK-O5 remains the correct fix site |
| O-4 — Inconsistent exception types (`FormatException` vs `UnsupportedError`) | C-8 | Both types reachable at `OhttpClient.send()` boundary; contributes to undocumented throws contract |

---

## Draft Task Entries

All tasks target `lib/src/ohttp_client.dart` unless otherwise noted. All are independently actionable
by a single engineer.

| Task ID | Title | Severity | Primary file : lines | Concern category |
|---|---|---|---|---|
| TASK-C1 | Enforce `https`-scheme on `OhttpGatewayConfig.gatewayBaseUrl` | HIGH | `ohttp_client.dart:35-53`, `81-82`, `115-116` | Scheme enforcement |
| TASK-C2 | Replace string path concatenation with `Uri.resolve` normalization | IMPROVEMENT | `ohttp_client.dart:82`, `116`, `154` | URL construction |
| TASK-C3 | Add allow-list validation or scheme-stripping for `targetAuthority` | HIGH | `ohttp_client.dart:98-101` | Privacy / SSRF |
| TASK-C4 | Add mandatory privacy-impact warning to `sendDirect()`; add `onLog` bypass signal | HIGH | `ohttp_client.dart:148-171` | Privacy / Documentation |
| TASK-C5 | Add configurable TTL-based KeyConfig cache and timeout to KeyConfig GET | HIGH | `ohttp_client.dart:81-89` | Network reliability / KeyConfig lifecycle |
| TASK-C6 | Add configurable timeout and optional cancellation to gateway POST | HIGH | `ohttp_client.dart:115-122` | Network reliability |
| TASK-C7 | Introduce `OhttpClientException` typed hierarchy carrying HTTP status code | HIGH | `ohttp_client.dart:84-87`, `120-122` | Error handling |
| TASK-C8 | Document and own `OhttpClient.send()` exception contract; wrap foreign types | HIGH | `ohttp_client.dart:69` (method sig) | Error handling / API contract |
| TASK-C9 | Add configurable `maxResponseBytes` guard before `ohttpDecapsulate` | HIGH | `ohttp_client.dart:129-133` | Parser robustness / DoS |
| TASK-C10 | Lowercase `OhttpHeader.name` in `send()` response construction | IMPROVEMENT | `ohttp_client.dart:139` | API consistency |
| TASK-C11 | (Resolved by TASK-O5) Wrap `SecretBoxAuthenticationError` at `ohttp.dart` layer | HIGH | `ohttp.dart:219` | Error handling / API boundary |
| TASK-C12 | Guard `gatewayBaseUrl` from `onLog` output or add log-level parameter | IMPROVEMENT | `ohttp_client.dart:77-78` | Privacy / Observability |

TASK-C11 has no independent remediation work in `ohttp_client.dart` — TASK-O5 (Phase 3) already
targets the correct fix site (`ohttp.dart:219`). TASK-C11 serves as the cross-layer confirmation
record and should be closed when TASK-O5 is implemented.

TASK-C7 and TASK-C8 overlap: a typed `OhttpClientException` hierarchy (C7) is also the wrapping
vehicle for foreign exceptions (C8). These can be merged into a single Jira task.

---

## New Technical Questions

These questions surfaced during the research and were not known before reading the source.

1. **`HandshakeException` wrapping scope.** `package:http` 1.6.0 wraps `SocketException` and
   `HttpException` but NOT `dart:io.HandshakeException`. Should the remediation for C-8 wrap
   `HandshakeException` inside the new `OhttpClientException`, or should the wallet's
   `http.Client` wrapper handle it? The answer determines whether the fix belongs in
   `ohttp_client.dart` or in a custom `http.Client` subclass provided by the wallet.

2. **`effectiveDirectBaseUrl` footgun when `directBaseUrl` is null.** When `directBaseUrl` is null,
   `sendDirect()` sends to `gatewayBaseUrl` — the OHTTP relay — rather than the target server.
   Should `sendDirect()` throw `StateError` when `directBaseUrl` is null, or is sending to the
   gateway intentional in some deployment patterns? This is a design question for TASK-C4.

3. **KeyConfig cache invalidation strategy.** When the gateway rotates its key (changing `keyId`),
   a cached `OhttpKeyConfig` will produce an OHTTP request that the gateway rejects. The
   current code re-fetches on every call, which accidentally handles rotation correctly. Any
   caching solution (TASK-C5) must define a forced-refresh trigger (e.g., on 4xx gateway
   response, or on `FormatException` from decapsulation). The invalidation strategy is a
   correctness concern, not just a performance concern.

4. **`OhttpClient` lifecycle and `dispose()`.** `dispose()` calls `_httpClient.close()`. If
   `OhttpClient` is abandoned without calling `dispose()` (common in wallet lifecycle scenarios),
   the underlying `IOClient` connection pool is not closed, which prevents Dart VM process
   termination. This is a `package:http` API contract issue but may need documentation in
   `OhttpClient`'s class doc. Not classified as a finding (no wallet-correctness impact) but
   worth noting before Iteration 7 task compilation.

5. **`onLog` string interpolation is unconditional.** In Dart, `onLog?.call(expensiveString)` still
   evaluates `expensiveString` even when `onLog` is null, because Dart evaluates arguments
   before the null-check on the receiver. The logging overhead is negligible for the current
   call sites (short string interpolations), but any future log call that interpolates large
   data (e.g., response body size) should be guarded with an explicit `if (onLog != null)` to
   avoid unnecessary work. Not a finding, but an implementation note for TASK-C12.
