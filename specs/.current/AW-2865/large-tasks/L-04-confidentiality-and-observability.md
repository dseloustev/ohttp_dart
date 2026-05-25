# Confidentiality hygiene & structured observability

**Estimate:** 3d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

Two workstreams about what the wallet exposes at runtime — secrets in memory and telemetry on the wire — that share no implementation surface with L-03 and can be developed in parallel.

**Secrets in memory.** A wallet running on a mobile or desktop host is a credible memory-dump target. Sensitive HPKE and OHTTP outputs currently linger in heap-allocated `Uint8List` fields long after the OHTTP request completes. The OHTTP send flow is one-shot — there is no functional reason to retain these buffers — but Dart GC does not guarantee prompt collection, so an attacker who acquires a memory snapshot of the wallet process minutes after a request can recover the AEAD key, base nonce, exporter secret, KEM ephemeral share, and response-channel root secret.

**Telemetry on the wire.** The library has no structured observability surface. There is one optional `onLog` string callback on `OhttpClient` that is used inconsistently and currently leaks sensitive path information: at `lib/src/ohttp_client.dart:77` it logs `gatewayBaseUrl + configPath`, and at line 113 it logs `gatewayBaseUrl + requestPath`. Wallet operators need actionable signals (KeyConfig fetch outcome, gateway POST status, decryption success/failure, accidental `sendDirect()` invocation), but those signals must not include cryptographic material, the inner request, the target authority, or the response body. The current ad-hoc string logging is both insufficient (no structure, no levels) and unsafe (mixes safe and unsafe fields).

Separately, response header names are not case-normalized at the OHTTP client boundary (`lib/src/ohttp_client.dart:139`): `e.key` comes directly from `streamedResponse.headers.entries`. HTTP/1.1 header names are case-insensitive (RFC 9110 §5.1), but different HTTP clients deliver them in different cases.

## Technical Details

### Part A — Zeroize sensitive key material

**1. HPKE sender context** (`lib/src/hpke.dart`, `HpkeSenderContext` fields at lines 258–263)

`HpkeSenderContext` retains `key` (16 B, AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) as `final Uint8List` fields. None are zeroized.

Provide an explicit zeroization step (method or `dispose`-style call) that overwrites each `Uint8List` field with zeros once the context is no longer needed. Callers in `ohttp.dart` invoke the step after `seal()` and `export()` complete.

**2. OHTTP encap result + intermediate HKDF outputs** (`lib/src/ohttp.dart:105-120, 192-207`)

`OhttpEncapsulateResult` (lines 105–120) stores `enc`, `encRequest`, and `exportedSecret` as `final Uint8List` fields. `exportedSecret` is the response-channel root secret; `enc` is the ephemeral KEM share. Leakage of `exportedSecret` recovers the response AEAD key and breaks confidentiality of the gateway reply (which may contain wallet balances or transaction history); leakage of `enc` alongside ciphertext breaks the response-channel unlinkability property.

Lines 192–207 are the HKDF key/nonce derivation block inside `ohttpDecapsulate` that consumes `exportedSecret`. The derived AEAD key and nonce intermediates are also sensitive.

Expose a `dispose()`-style method on `OhttpEncapsulateResult` (or reshape so the lifetime ends as soon as decap completes). After `ohttpDecapsulate` returns (success or failure), `enc` and `exportedSecret` buffers held by the call site, plus the derived AEAD key and nonce intermediates, are overwritten with zeros.

**3. Dart caveat**

AOT/JIT compilers may optimize away the write. The mitigation is best-effort and the doc comment must say so. Evaluate whether `package:cryptography` exposes a `SecretKey.destroy()`-style API and prefer it if available.

### Part B — Structured observability

**4. Typed observability interface**

Define a typed observer interface (e.g., `OhttpObserver`) with event methods such as:
- `onKeyConfigFetch({required int statusCode, required Duration elapsed})`
- `onGatewayPost({required int statusCode, required Duration elapsed})`
- `onSendDirectInvoked()` — for accidental fallback alerting (companion to the L-01 `sendDirect` privacy task).
- `onDecryptionFailure()` — emitted before/around the AEAD-failure exception path in `ohttpDecapsulate`.

Re-export from `lib/ohttp_dart.dart`.

Replace the existing string `onLog` callback in `OhttpClient`. Either remove it (breaking change, documented in migration note) or deprecate it alongside the new typed surface and document the migration path. The existing emission sites at lines 77 and 113 — which log full URLs containing both base and path — must be replaced by typed events that carry only the status code.

**5. Strict logging constraints**

Emission sites must obey these constraints (reproduced verbatim in the public doc comment of the observer interface so they cannot be lost in a refactor):

> Never log: (1) key material, (2) enc value, (3) inner request URL/path/method/headers/body, (4) targetAuthority, (5) response body/headers, (6) gatewayBaseUrl at DEBUG or lower in production.
>
> Safe to log: KeyConfig fetch success/failure (status code only), gateway POST status code (not body), sendDirect() invocation signal.

**6. Lowercase response header names** (`lib/src/ohttp_client.dart:139`)

At line 139 (`.map((e) => OhttpHeader(name: e.key, value: e.value))`), lowercase the header name before constructing `OhttpHeader`. Document the normalization in the `OhttpHeader` doc comment so downstream consumers know they can rely on it. Cite RFC 9110 §5.1.

### Part C — Tests

**7. Zeroization tests**
- Unit test verifies that after the zeroization call on `HpkeSenderContext`, each field's bytes are all zero (or that the field reference is replaced with a zeroed buffer), on both happy-path and decryption-failure paths.
- Unit test verifies the same for `OhttpEncapsulateResult` (`enc`, `exportedSecret`) and the derived AEAD key/nonce intermediates after `ohttpDecapsulate` completes (success and failure).

**8. Observability tests**
- Drive a successful `OhttpClient.send` flow against a mock client and capture every emitted event. Assert that no event payload contains: gateway path, target authority, response body, key material strings, `enc` bytes.
- Drive a failing flow (4xx, 5xx, AEAD auth failure) and assert the same. Assert that the failure-specific events fire with the correct status code.
- Drive `sendDirect()` and assert that `onSendDirectInvoked()` fires.
- Feed mixed-case response header names through `OhttpClient.send` and assert the resulting `OhttpHeader.name` values are lowercase.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

**Zeroization:**
- `HpkeSenderContext` exposes an explicit zeroization step (method or `dispose`-style call) that overwrites `key`, `baseNonce`, and `exporterSecret` with zeros.
- Callers in `ohttp.dart` invoke the zeroization step after the seal/export operations complete.
- `OhttpEncapsulateResult` exposes a zeroization API that wipes `enc` and `exportedSecret`.
- After `ohttpDecapsulate` completes (success or failure), `enc`, `exportedSecret`, and the derived AEAD key/nonce intermediates are overwritten with zeros.
- Doc comments on each zeroization API document the Dart best-effort limitation.

**Observability:**
- A typed observability interface (e.g., `OhttpObserver`) is defined and re-exported from `lib/ohttp_dart.dart`.
- All emission sites in `OhttpClient` produce structured events that obey the logging constraints above — no key material, no `enc`, no inner request details, no `targetAuthority`, no response body/headers, no full `gatewayBaseUrl + path` strings.
- The existing string `onLog` callback is either removed (breaking change, documented) or deprecated with a migration note.
- A `sendDirect()` invocation produces a distinct, documented event so wallet operators can alert on accidental fallback use.
- The observer interface's public doc comment reproduces the logging constraints verbatim.
- Response header names are lowercased at `lib/src/ohttp_client.dart:139` before constructing `OhttpHeader`.
- `OhttpHeader` doc comment documents that `name` is lowercase and cites RFC 9110 §5.1.

**Tests:**
- Zeroization unit tests cover each layer on both happy-path and failure paths.
- Observer-event capture tests verify no emitted event contains any restricted field for a successful and a failing flow.
- Mixed-case response header names are normalized to lowercase.

## Additional

- This task can run in parallel with L-03 (touches `hpke.dart`, `ohttp.dart` decap-side fields, and the observability surface on `ohttp_client.dart` — no overlap with L-03's `OhttpClient.send` orchestration changes).
- The observer's failure-classification events should align with the typed exception hierarchy from L-02 (e.g., `onDecryptionFailure()` fires before the `OhttpDecryptionException` is thrown).
- Migration note: removal/deprecation of `onLog` is breaking; document the new surface clearly.
- Source merged tasks (in `../merged-tasks/`): M-03 (zeroize key material), M-08 (observability hooks + header name normalization).
