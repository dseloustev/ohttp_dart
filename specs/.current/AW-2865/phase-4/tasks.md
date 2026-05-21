# Phase 4: Read client layer and assess network reliability and KeyConfig lifecycle

**Goal:** Audit `lib/src/ohttp_client.dart` for timeout/retry/cancellation gaps, KeyConfig caching policy, scheme enforcement, and `sendDirect()` bypass risks.

## Context

**Feature motivation (from idea):** Investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The goal is not to implement improvements but to identify risks and form subsequent engineering tasks. Every finding must map to a specific file, line range, and concern category.

**Severity classification (from vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Technical approach (from vision §4):** `lib/src/ohttp_client.dart` is Layer 4 — the top of the four-layer stack. It orchestrates the full OHTTP round trip: GET KeyConfig → BHTTP-serialize → OHTTP-encap → POST gateway → decap → BHTTP-parse. It wraps `package:http` for both the KeyConfig fetch and the gateway POST. Findings here are the most wallet-visible — they determine reliability, privacy, and error observability for the end user.

**Primary investigation concerns (from vision §4 scrutiny table):**
- No timeout/retry/cancellation on `http.Client.get()` or `http.Client.post()`
- KeyConfig fetched fresh on every `send()` invocation (no caching, no TTL)
- No downgrade detection when gateway rotates key or cipher suite
- `sendDirect()` escape path co-located in the same config object as OHTTP paths

**Related classes/entities (from vision §5.4 — `OhttpGatewayConfig`):**

| Field | Investigation concern |
|---|---|
| `gatewayBaseUrl` | No `https`-scheme enforcement; plain HTTP leaks inner requests to a network observer |
| `configPath` / `requestPath` | String concatenation, not `Uri` normalization — double-slash or path-traversal possible |
| `targetAuthority` | Embedded verbatim into the BHTTP request with no allow-list check |
| `directBaseUrl` | Non-OHTTP escape path co-located in the same config object; `sendDirect()` has no privacy-impact warning |

**Related classes/entities (from vision §5.5 — `OhttpResponse` / `OhttpHeader`):**

| Field | Investigation concern |
|---|---|
| `headers` | No size cap; malicious gateway can produce arbitrarily large header lists |
| `body` | No size cap; entire gateway response body buffered in memory before decap |
| `OhttpHeader.name` | Not lowercased on the response side (inconsistent with request side) |

**Data flow relevant to this phase (from vision §6.1 — happy path with concerns):**
```
caller.send(method, path, headers, body)
  │
  ├─ 1. GET ${gatewayBaseUrl}${configPath}
  │       → http.Client.get() — no timeout, no retry
  │       → on statusCode != 200: throw Exception (generic, not typed)
  │       → OhttpKeyConfig.parse(bodyBytes)
  │  [CONCERN] KeyConfig fetched fresh on every call — no caching, no TTL
  │  [CONCERN] No scheme check; plain-HTTP gateway URL silently accepted
  │
  ├─ 2. bhttp.serializeRequest(...)
  │
  ├─ 3. ohttpEncapsulate(config, binaryRequest)
  │
  ├─ 4. POST ${gatewayBaseUrl}${requestPath}
  │       Content-Type: message/ohttp-req
  │       → http.Client.post() — no timeout, no retry, no cancellation
  │       → on statusCode != 200: throw Exception (generic)
  │  [CONCERN] Network errors surface as untyped exceptions
  │
  ├─ 5. ohttpDecapsulate(enc, exportedSecret, gatewayResponse.bodyBytes)
  │  [CONCERN] No size cap on gatewayResponse.bodyBytes before decapsulation
  │
  └─ 6. bhttp.parseResponse(binaryResponse)
```

**`sendDirect()` bypass path (from vision §6.4):**
```
Same OhttpGatewayConfig object → sendDirect() → plaintext HTTP to directBaseUrl
  [CONCERN] No assertion or warning that sendDirect() bypasses OHTTP privacy
  [CONCERN] Misconfiguration silently routes wallet data in cleartext
```

**Error paths (from vision §6.2 / §6.3):**
- All error types collapse to `Exception` or propagate raw — no retry logic, no typed error hierarchy.
- `RangeError` from malformed KeyConfig body propagates unwrapped; caller cannot distinguish it from a logic bug.
- No distinction between transient (5xx) and permanent (4xx) gateway errors; no retry or back-off.

**Observability gaps in this layer (from vision §7):**

| Missing event | Gap severity |
|---|---|
| KeyConfig fetch succeeded / failed | HIGH — no way to distinguish gateway misconfiguration from network failure in a deployed wallet |
| Gateway POST status code | HIGH — non-200 responses throw generic `Exception`; no structured record |
| `sendDirect()` invoked (OHTTP bypass) | HIGH — plaintext path has no warning signal |

**Cross-layer context:**
- Phase 2 (BHTTP) found no size cap on `parseResponse` headers/body — the response size concern extends through this layer because `OhttpClient` buffers the full gateway response body before calling `parseResponse`.
- Phase 3 (OHTTP) found that `SecretBoxAuthenticationError` propagates unwrapped from `ohttpDecapsulate`; in the client layer it surfaces to the wallet caller with no wrapping.

## Tasks

- [ ] 4.1 Read the full file (use `ast-index outline` first).
- [ ] 4.2 Confirm that `OhttpGatewayConfig.gatewayBaseUrl` has no `https`-scheme enforcement — note that plain HTTP silently accepted.
- [ ] 4.3 Confirm that `configPath` and `requestPath` are concatenated as strings (not `Uri`-normalized) — note potential double-slash or path-traversal risk.
- [ ] 4.4 Confirm that `targetAuthority` is embedded verbatim into the BHTTP request with no allow-list check.
- [ ] 4.5 Confirm that `directBaseUrl` exists in the same config object as the OHTTP paths and that `sendDirect()` has no warning about bypassing OHTTP.
- [ ] 4.6 Trace the KeyConfig GET call: confirm `http.Client.get()` has no timeout and no retry, and that a fresh fetch occurs on every `send()` invocation (no caching).
- [ ] 4.7 Trace the gateway POST call: confirm `http.Client.post()` has no timeout, no retry, and no cancellation hook.
- [ ] 4.8 Confirm that all non-200 HTTP responses throw generic `Exception` (no typed error hierarchy, no distinction between 4xx and 5xx).
- [ ] 4.9 Confirm that network errors (DNS failure, connection reset) surface as untyped exceptions.
- [ ] 4.10 Confirm that response body and headers have no size cap before `bhttp.parseResponse`.
- [ ] 4.11 Note that `OhttpHeader.name` is not lowercased on the response side (inconsistent with request side).
- [ ] 4.12 Record each finding as a draft task entry (file, line range, concern category, severity).

## Acceptance Criteria

**Test:** No code is changed. Verification is: the "no timeout" finding and the "no KeyConfig caching" finding each cite the specific `http.Client` call site line in `ohttp_client.dart`.

## Dependencies

- Phase 3 complete (OHTTP layer audit)

## Technical Details

**`OhttpClient` entry points to audit:**

The client exposes two public methods:
1. `send(method, path, headers, body)` — full OHTTP round trip (KeyConfig GET + encap POST + decap).
2. `sendDirect(method, path, headers, body)` — plain HTTP to `directBaseUrl`, bypassing OHTTP entirely.

Both methods are on the same class; `OhttpGatewayConfig` configures both paths in one object.

**Expected `http.Client` call sites (to verify in audit):**

```dart
// KeyConfig fetch — look for:
final configResponse = await _httpClient.get(Uri.parse('${config.gatewayBaseUrl}${config.configPath}'));

// Gateway POST — look for:
final gatewayResponse = await _httpClient.post(
  Uri.parse('${config.gatewayBaseUrl}${config.requestPath}'),
  headers: {'Content-Type': 'message/ohttp-req'},
  body: encResult.encRequest,
);
```

Neither call should have a `timeout` attached — confirm by checking for `.timeout(Duration(...))` chained on the future or a `http.Client` subclass with a built-in timeout.

**KeyConfig caching — expected absence:**

In a correctly cached implementation, `OhttpKeyConfig` would be stored after the first successful GET and reused until a configurable TTL expires. The audit should confirm that `send()` always re-fetches, creating two round trips per inner request (one for config, one for the OHTTP POST).

**Scheme enforcement — expected gap:**

`Uri.parse(gatewayBaseUrl)` accepts any scheme. A wallet deploying `ohttp_client` with an `http://` gateway URL would silently transmit plaintext OHTTP requests to the relay, stripping all transport-layer protection while the inner BHTTP request remains encrypted. The OHTTP threat model requires TLS on the outer channel; without it, the relay can observe request metadata.

**`sendDirect()` privacy implications:**

RFC 9458 §1 defines Oblivious HTTP specifically to prevent the target server from learning the client's IP address. `sendDirect()` short-circuits this by sending directly to `directBaseUrl`. A wallet using `sendDirect()` as a fallback (e.g., when the OHTTP gateway is unreachable) would silently degrade its own privacy model without any indication to the wallet user.

**String path concatenation vs. `Uri` normalization:**

`'${gatewayBaseUrl}${configPath}'` produces `"https://gw.example.com//config"` if `configPath` starts with `/` (common convention) and `gatewayBaseUrl` ends with `/`. `Uri.parse` tolerates double-slashes, so the request reaches the gateway — but some gateway implementations are sensitive to path normalization. More critically, if `configPath` is user-supplied and contains `..` segments, string concatenation does not normalize them away.

**Fixed cipher suite IDs (cross-layer reference):**
- KEM ID: `0x0020` (DHKEM(X25519, HKDF-SHA256)), `Npk = 32`
- KDF ID: `0x0001` (HKDF-SHA256)
- AEAD ID: `0x0001` (AES-128-GCM), `Nk = 16`, `Nn = 12`

**Logging constraints (from vision §7):**

Any future logging in this layer MUST NOT log:
1. Key material of any kind.
2. The inner request URL, path, method, headers, or body.
3. `targetAuthority` from `OhttpGatewayConfig`.
4. Response body or response headers.
5. `gatewayBaseUrl` at DEBUG or lower in production builds.

Safe to log: KeyConfig fetch success/failure (status code only), gateway POST status code (not body), `sendDirect()` invocation (method + path stripped, no body).

## Implementation Notes

This is a read-only audit iteration. Do not edit any file in `lib/` or `test/`. Record all findings as draft task entries in this file for use in Iteration 7 (compile into per-task Markdown files).

Phases 1 (HPKE), 2 (BHTTP), and 3 (OHTTP) are complete with findings in their respective `tasks.md` files. Cross-reference those findings when noting client-layer concerns that compound lower-layer risks (e.g., no response size cap here + no header cap in bhttp.dart = two-layer amplification).
