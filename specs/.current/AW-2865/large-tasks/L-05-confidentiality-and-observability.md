# Confidentiality hygiene & structured observability

**Estimate:** 3d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

Two workstreams about what the wallet exposes at runtime — secrets in memory and telemetry on the wire — that share no implementation surface with L-04 and can be developed in parallel.

**Secrets in memory.** A wallet running on a mobile or desktop host is a credible memory-dump target. Sensitive HPKE and OHTTP outputs currently linger in heap-allocated `Uint8List` fields long after the OHTTP request completes. The OHTTP send flow is one-shot — there is no functional reason to retain these buffers — but Dart GC does not guarantee prompt collection, so an attacker who acquires a memory snapshot of the wallet process minutes after a request can recover the AEAD key, base nonce, exporter secret, KEM ephemeral share, and response-channel root secret.

**Telemetry on the wire.** After the L-01 restructure, the package has no observability surface at all — the old string `onLog` callback on `OhttpClient` is gone with the rest of `OhttpClient`. The new orchestrator `OhttpSession.send` is the right place to attach a typed observer surface. Wallet operators need actionable signals (KeyConfig fetch outcome, gateway POST status, decryption success/failure, cache hit/miss) without including cryptographic material, the inner request, the target authority, or the response body.

Separately, response header names are not case-normalized when `OhttpSession` materializes `OhttpResponseData` from `bhttp.parseResponse`'s `List<(String, String)>`. HTTP/1.1 header names are case-insensitive (RFC 9110 §5.1), but different gateways may emit different cases and downstream consumers should be able to rely on a normalized form.

## Technical Details

### Part A — Zeroize sensitive key material

**1. HPKE sender context** (`lib/src/hpke.dart`, `HpkeSenderContext` fields)

`HpkeSenderContext` retains `key` (16 B, AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) as `final Uint8List` fields. None are zeroized.

Provide an explicit zeroization step (method or `dispose`-style call) that overwrites each `Uint8List` field with zeros once the context is no longer needed. Callers in `ohttp.dart` invoke the step after `seal()` and `export()` complete.

**2. OHTTP encap result + intermediate HKDF outputs** (`lib/src/ohttp.dart`)

`OhttpEncapsulateResult` stores `enc`, `encRequest`, and `exportedSecret` as `final Uint8List` fields. `exportedSecret` is the response-channel root secret; `enc` is the ephemeral KEM share. Leakage of `exportedSecret` recovers the response AEAD key and breaks confidentiality of the gateway reply (which may contain wallet balances or transaction history); leakage of `enc` alongside ciphertext breaks the response-channel unlinkability property.

The HKDF key/nonce derivation block inside `ohttpDecapsulate` (consuming `exportedSecret`) produces derived AEAD key and nonce intermediates that are also sensitive.

Expose a `dispose()`-style method on `OhttpEncapsulateResult` (or reshape so the lifetime ends as soon as decap completes). After `ohttpDecapsulate` returns (success or failure), `enc` and `exportedSecret` buffers held by the call site, plus the derived AEAD key and nonce intermediates, are overwritten with zeros.

`OhttpSession.send` is the call site that holds the `OhttpEncapsulateResult` between encap and decap — it must invoke the zeroization step after decap completes (in `try`/`finally`).

**3. Dart caveat**

AOT/JIT compilers may optimize away the write. The mitigation is best-effort and the doc comment must say so. Evaluate whether `package:cryptography` exposes a `SecretKey.destroy()`-style API and prefer it if available.

### Part B — Structured observability

**4. Typed observer interface**

Define `lib/src/observer.dart` with `abstract interface class OhttpObserver` carrying event methods:

- `void onKeyConfigFetch({required int statusCode, required Duration elapsed})` — fired after the transport's `fetchKeyConfig` returns (success or failure). On `OhttpHttpException`, `statusCode` is the gateway status; on timeout, callers may emit a sentinel (e.g., -1) or a separate `onKeyConfigTimeout` event — pick one and document it.
- `void onKeyConfigCacheHit()` — fired when `KeyConfigCache.get()` returns a cached value (no fetch needed). Lets wallet operators see cache effectiveness without inferring it.
- `void onGatewayPost({required int statusCode, required Duration elapsed})` — fired after the transport's `postToGateway` returns (success or failure).
- `void onDecryptionFailure()` — fired inside `OhttpSession.send` before the `OhttpDecryptionException` propagates.

Re-export from `lib/ohttp_dart.dart`.

Add an optional `OhttpObserver? observer` parameter to `OhttpSession` (and `OhttpSession.withTransport`). When null, no events are emitted. The session is the **only** emission site — `HttpClientTransport` does not call the observer directly (it has no reference to it). Instead, `OhttpSession.send` wraps `transport.fetchKeyConfig` / `postToGateway` calls in `Stopwatch`-based timing and translates outcomes to observer events. This keeps the transport interface narrow.

Alternative if Stopwatch wrapping turns out to be awkward: thread the observer through `OhttpSession` and `KeyConfigCache`, with the cache emitting `onKeyConfigCacheHit` (it's the only component that knows about hits). `KeyConfigCache` already wraps the underlying `transport.fetchKeyConfig` call; it can time the cold path and emit `onKeyConfigFetch` there.

**5. Strict logging constraints**

Emission sites must obey these constraints (reproduced verbatim in the public doc comment of the observer interface so they cannot be lost in a refactor):

> Never log: (1) key material, (2) `enc` value, (3) inner request URL / path / method / headers / body, (4) `authority` (target host) from `OhttpRequestData`, (5) response body / headers, (6) gateway URLs at DEBUG or lower in production.
>
> Safe to log: KeyConfig fetch success/failure (status code only), gateway POST status code (not body), cache hit/miss, decryption-failure signal.

**6. Lowercase response header names** (`lib/src/ohttp_session.dart`)

When `OhttpSession.send` constructs `OhttpResponseData` from `bhttp.parseResponse(...).headers` (a `List<(String, String)>`), lowercase each header name before placing it into `OhttpResponseData.headers`. Order and duplicates are preserved. Document the normalization in the `OhttpResponseData.headers` doc comment with a reference to RFC 9110 §5.1.

The adapter `OhttpHttpClient.send` collapses `List<(String, String)>` into a `Map<String, String>` to satisfy `http.StreamedResponse.headers`; the lowercase guarantee from `OhttpSession` carries through this collapse.

### Part C — Tests

**7. Zeroization tests**
- Unit test verifies that after the zeroization call on `HpkeSenderContext`, each field's bytes are all zero (or that the field reference is replaced with a zeroed buffer), on both happy-path and decryption-failure paths.
- Unit test verifies the same for `OhttpEncapsulateResult` (`enc`, `exportedSecret`) after `ohttpDecapsulate` completes (success and failure).
- Unit test verifies that `OhttpSession.send` invokes the zeroization step on the success and failure paths (use a test double / spy on `OhttpEncapsulateResult.dispose`).

**8. Observability tests** (`test/ohttp_session_test.dart`, `test/key_config_cache_test.dart`)
- Drive a successful `OhttpSession.send` flow against a fake `OhttpTransport` and capture every emitted event. Assert that no event payload contains: inner authority, response body, key material strings, `enc` bytes, gateway URLs.
- Drive a failing flow: `OhttpHttpException` → `onGatewayPost` with the failure status; AEAD auth failure → `onDecryptionFailure` fires before the exception propagates.
- Drive a cache-hit flow: first `send()` populates the cache, second `send()` fires `onKeyConfigCacheHit` but **not** `onKeyConfigFetch`.
- Feed mixed-case response header names through `OhttpSession.send` and assert the resulting `OhttpResponseData.headers` tuples have lowercase names.

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
- `OhttpSession.send` invokes the zeroization step in a `try`/`finally` around the decap call so it runs on success and failure paths.
- After `ohttpDecapsulate` completes (success or failure), `enc`, `exportedSecret`, and the derived AEAD key/nonce intermediates are overwritten with zeros.
- Doc comments on each zeroization API document the Dart best-effort limitation.

**Observability:**
- `OhttpObserver` interface defined in `lib/src/observer.dart` and re-exported from `lib/ohttp_dart.dart`. Methods: `onKeyConfigFetch`, `onKeyConfigCacheHit`, `onGatewayPost`, `onDecryptionFailure`.
- `OhttpSession` (and `OhttpSession.withTransport`) accept an optional `OhttpObserver` parameter; when null, no events are emitted.
- All emission sites obey the logging constraints above — no key material, no `enc`, no inner request details, no `authority`, no response body/headers, no gateway URLs in event payloads.
- Observer interface's public doc comment reproduces the logging constraints verbatim.
- Response header names are lowercased in `OhttpSession.send` when materializing `OhttpResponseData` from BHTTP-parsed headers.
- `OhttpResponseData.headers` doc comment documents the lowercase guarantee and cites RFC 9110 §5.1.

**Tests:**
- Zeroization unit tests cover each layer on both happy-path and failure paths, including the `OhttpSession.send` invocation.
- Observer-event capture tests verify no emitted event contains any restricted field for a successful and a failing flow.
- Cache-hit flow fires `onKeyConfigCacheHit` and skips `onKeyConfigFetch`.
- Mixed-case response header names are normalized to lowercase.

## Additional

- This task can run in parallel with L-04 (touches `hpke.dart`, `ohttp.dart` decap-side fields, a new `observer.dart`, and the response-materialization step in `ohttp_session.dart` — no overlap with L-04's `HttpClientTransport` timeout work, `OhttpSession.send` cap check, or BHTTP parser changes). Modest coordination is needed if both tasks edit `ohttp_session.dart` in the same week (this task adds the observer wiring and zeroization-around-decap; L-04 adds the response cap check).
- The observer's `onDecryptionFailure` event should fire before the `OhttpDecryptionException` from L-03 propagates.
- The `onSendDirectInvoked` event from the original L-04 is **dropped** — `sendDirect()` no longer exists in the new architecture (removed by L-01). Wallet operators no longer need an alerting hook for accidental fallback because there is no fallback API.
- The `onLog` migration note from the original L-04 is **dropped** — the old `onLog` string callback was deleted with `OhttpClient` by L-01; this task introduces the new observer surface from scratch, no migration path needed.
- Depends on L-01 (the `OhttpSession`, `KeyConfigCache`, `OhttpResponseData` types do not exist before the restructure lands).
- Depends on L-03 (the `OhttpDecryptionException` and `OhttpHttpException` types are the failure conditions the observer hooks fire around).
- Source merged tasks (in `../merged-tasks/`): M-03 (zeroize key material) — preserved as-is; M-08 (observability hooks + header name normalization) — observer surface retargeted from `OhttpClient` to `OhttpSession`; header lowercasing retargeted from `ohttp_client.dart:139` to `OhttpSession.send`.
