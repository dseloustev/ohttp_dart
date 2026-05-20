# Vision: [Investigation] Create improvement tasks for ohttp_dart (AW-2865)

> Source idea: [./idea.md](./idea.md)

---

## 1. Technologies

| Concern | Tech | Status |
|---|---|---|
| HPKE implementation | `lib/src/hpke.dart` — hand-rolled DHKEM(X25519, HKDF-SHA256) + AES-128-GCM | EXISTING |
| Binary HTTP framing | `lib/src/bhttp.dart` — RFC 9292 Known-Length framing | EXISTING |
| OHTTP encapsulation | `lib/src/ohttp.dart` — RFC 9458 encap/decap | EXISTING |
| High-level HTTP client | `lib/src/ohttp_client.dart` — `OhttpClient` + `OhttpGatewayConfig` | EXISTING |
| Cryptographic primitives | `package:cryptography` (X25519, Hmac.sha256, AesGcm.with128bits) | EXISTING |
| Test framework | `package:test` | EXISTING |
| Static analysis | `dart analyze` + `analysis_options.yaml` (strict-casts, strict-raw-types) | EXISTING |

This is a read-only investigation task. No new `pubspec.yaml` entries are added. No native code. No new codegen.

---

## 2. Development principles

1. **Read and annotate only — no code changes.**
   This ticket produces a Jira task list, not a patch. Every finding must map to a future task, not to an edit of `lib/`.

2. **Ground every finding in concrete evidence.**
   Each identified risk must cite a specific file, line range, RFC section, or test gap — no vague statements like "crypto could be weak".

3. **Classify findings by production-blocking severity.**
   Each follow-up task must carry one of three labels: `BLOCKER` (must fix before wallet use), `HIGH` (should fix before wallet use), `IMPROVEMENT` (desirable but not blocking). This prevents task lists that mix critical and cosmetic items.

4. **Cover all eight investigation vectors from the idea file.**
   The output task list must address every vector listed under Technical Details (cryptographic correctness, interoperability, parser robustness, privacy risks, network reliability, KeyConfig management, test suite, documentation). No vector may be silently skipped.

5. **Keep tasks independently actionable.**
   Each follow-up Jira task must be scoped so a single engineer can pick it up without depending on another in-flight task from this list. If two findings are inseparable, merge them into one task and note why.

---

## 3. Project structure

This is a read-only investigation. Every file below is `UNCHANGED` — the ticket produces no edits to `lib/` or `test/`.

```
ohttp_dart/
├── lib/
│   ├── ohttp_dart.dart              # UNCHANGED — public re-export surface
│   └── src/
│       ├── bhttp.dart               # UNCHANGED — RFC 9292 framing; review for parser robustness
│       ├── hpke.dart                # UNCHANGED — HPKE Base Mode sender; review for RFC 9180 correctness
│       ├── ohttp.dart               # UNCHANGED — RFC 9458 encap/decap; review for AAD handling, response decap
│       └── ohttp_client.dart        # UNCHANGED — OhttpClient; review for timeout, retry, KeyConfig lifecycle
├── test/
│   ├── bhttp_test.dart              # UNCHANGED — varint round-trip tests; review for missing negative/fuzz cases
│   ├── hpke_test.dart               # UNCHANGED — RFC 9180 A.1 vector tests; review coverage gaps
│   └── ohttp_test.dart              # UNCHANGED — OHTTP encap/decap tests; review for missing failure paths
├── example/
│   └── ohttp_dart_example.dart      # UNCHANGED — usage example; review for documented limitations
├── pubspec.yaml                     # UNCHANGED — dependency manifest; review for transitive risk
└── analysis_options.yaml            # UNCHANGED — linter config
```

---

## 4. Project architecture

The package is a four-layer stack. Each layer depends only on the layer below it; there are no circular imports and no shared mutable state between layers.

```
┌─────────────────────────────────────────────────────────┐
│  Caller  (wallet app / example / test)                  │
└────────────────────────┬────────────────────────────────┘
                         │ OhttpClient.send()
┌────────────────────────▼────────────────────────────────┐
│  Layer 4 — ohttp_client.dart                            │
│  OhttpClient  OhttpGatewayConfig  OhttpResponse         │
│  Orchestrates: GET KeyConfig → BHTTP-serialize →        │
│  OHTTP-encap → POST gateway → decap → BHTTP-parse       │
└──────┬───────────────────────────┬───────────────────────┘
       │ ohttpEncapsulate /        │ bhttp.serializeRequest /
       │ ohttpDecapsulate          │ bhttp.parseResponse
┌──────▼──────────────┐  ┌────────▼───────────────────────┐
│  Layer 3 — ohttp.dart│  │  Layer 2 — bhttp.dart          │
│  OhttpKeyConfig      │  │  serializeRequest              │
│  OhttpEncapsulateResult  │  parseResponse                │
│  ohttpEncapsulate    │  │  encodeVarint / decodeVarint   │
│  ohttpDecapsulate    │  └────────────────────────────────┘
└──────┬───────────────┘
       │ HpkeSender.setupBaseS / seal / export
       │ HpkeSender.hkdfExtract / hkdfExpand
┌──────▼──────────────────────────────────────────────────┐
│  Layer 1 — hpke.dart                                    │
│  HpkeSender  HpkeSenderContext                          │
│  setupBaseS  LabeledExtract  LabeledExpand  KEM Encap   │
│  Wraps: package:cryptography (X25519, Hmac, AesGcm)     │
└─────────────────────────────────────────────────────────┘
```

**Request data flow (happy path):**

```
[inner HTTP request]
  → bhttp.serializeRequest()        # RFC 9292 Known-Length framing
  → ohttpEncapsulate()
      → OhttpKeyConfig.parse/validate
      → HpkeSender.setupBaseS()     # DHKEM(X25519) Encap + HPKE key schedule
      → ctx.seal(aad=[], plaintext) # AES-128-GCM, seq-nonce XOR
      → ctx.export("message/bhttp response", 16)
      → assemble: header(7) || enc(32) || ciphertext
  → HTTP POST gateway (Content-Type: message/ohttp-req)
  → ohttpDecapsulate()
      → split response_nonce(16) || ciphertext
      → plain HKDF-Extract(salt=enc||response_nonce, ikm=exportedSecret)
      → plain HKDF-Expand(..., "key", 16) + HKDF-Expand(..., "nonce", 12)
      → AES-128-GCM decrypt(aad=[])
  → bhttp.parseResponse()
[OhttpResponse]
```

**Investigation scrutiny points per layer:**

| Layer | File | Primary investigation concern |
|---|---|---|
| 4 — Client | `lib/src/ohttp_client.dart` | No timeout/retry/cancellation; KeyConfig fetched on every call (no caching); no downgrade detection |
| 3 — OHTTP | `lib/src/ohttp.dart` | Empty AAD deviation from RFC 9458; response decap uses plain HKDF (not labeled); KeyConfig parser reads only the first KDF+AEAD pair silently |
| 2 — BHTTP | `lib/src/bhttp.dart` | No length guards against oversized frames; no negative-case tests for malformed input |
| 1 — HPKE | `lib/src/hpke.dart` | Single-suite hard-coding; sequence-number counter not guarded against overflow; `testKeyPair` injection leaves test seam in production code path |

### Invariants

1. Dependency graph is strictly acyclic: `bhttp` ← `hpke` ← `ohttp` ← `ohttp_client`.
2. All crypto is delegated to `package:cryptography`; `hpke.dart` composes primitives but does not implement them.
3. `HpkeSenderContext` is single-use in the OHTTP flow.
4. Response decapsulation uses plain (unlabeled) HKDF intentionally — must not be unified with `LabeledExtract`/`LabeledExpand` without RFC re-validation.
5. The only supported cipher suite is DHKEM(X25519, HKDF-SHA256) + AES-128-GCM; `OhttpKeyConfig.validate()` enforces this at runtime.

---

## 5. Data model

This package has no persistent storage and no file I/O. All types are in-memory Dart objects.

### 5.1 `OhttpKeyConfig` — `lib/src/ohttp.dart`

| Field | Type | Wire source | Investigation concern |
|---|---|---|---|
| `keyId` | `int` (uint8) | byte 0 | No validation against a pinned key-ID set; gateway can rotate freely |
| `kemId` | `int` (uint16 BE) | bytes 1–2 | Only `0x0020` accepted; `parse()` throws `FormatException`, `validate()` throws `UnsupportedError` — inconsistent |
| `publicKey` | `Uint8List` (32 B) | bytes 3–34 | No pinning; point-on-curve check delegated to `package:cryptography` |
| `kdfId` / `aeadId` | `int` (uint16 BE) | bytes 37–40 | Parser reads only the **first** KDF+AEAD pair when `symLen > 4`; extra pairs are silently ignored |

Parser gap: malformed blob with large `symLen` and short buffer throws `RangeError`, not `FormatException`.

### 5.2 `OhttpEncapsulateResult` — `lib/src/ohttp.dart`

| Field | Type | Size | Investigation concern |
|---|---|---|---|
| `encRequest` | `Uint8List` | 7 + 32 + plaintext + 16 | No upper-bound guard on payload size |
| `enc` | `Uint8List` | 32 B | Ephemeral X25519 public key; no zeroization after use |
| `exportedSecret` | `Uint8List` | 16 B | Sensitive; no zeroization after use |

Lifetime must be a single `send()` invocation. Not persisted or cached.

### 5.3 `HpkeSenderContext` — `lib/src/hpke.dart`

| Field | Type | Investigation concern |
|---|---|---|
| `key` | `Uint8List` (16 B) | AES-128 key; no zeroization after `seal()` |
| `baseNonce` | `Uint8List` (12 B) | Copy made per `_computeNonce()` call — correct |
| `exporterSecret` | `Uint8List` (32 B) | Retained for `export()`; no zeroization |
| `_seq` | `int` | Overflow guard throws `StateError`, not a typed catchable exception |

### 5.4 `OhttpGatewayConfig` — `lib/src/ohttp_client.dart`

| Field | Investigation concern |
|---|---|
| `gatewayBaseUrl` | No `https`-scheme enforcement; plain HTTP leaks requests |
| `configPath` / `requestPath` | String concatenation, not `Uri` normalization |
| `targetAuthority` | Embedded verbatim into BHTTP request; no allow-list |
| `directBaseUrl` | Non-OHTTP escape path in the same config object |

### 5.5 `OhttpResponse` / `OhttpHeader` — `lib/src/ohttp_client.dart`

| Field | Investigation concern |
|---|---|
| `headers` | No size cap; malicious gateway can produce arbitrarily large lists |
| `body` | No size cap; buffered in memory |
| `OhttpHeader.name` | Not lowercased on response side (inconsistent with request side) |

### 5.6 `BhttpResponse` — `lib/src/bhttp.dart`

| Field | Investigation concern |
|---|---|
| `statusCode` | No range validation after varint decode |
| `headers` | No max header-count or total-size guard |
| `body` | `sublist(offset, offset + contentLen)` throws `RangeError` (not `FormatException`) on truncated input |

### Cross-cutting data model concerns

| Concern | Severity |
|---|---|
| No zeroization of key material after use | HIGH |
| No size caps on response body / headers | HIGH |
| Silent ignore of extra KDF+AEAD pairs in KeyConfig | HIGH |
| No `https`-scheme enforcement in gateway config | HIGH |
| `RangeError` instead of `FormatException` on truncated BHTTP body | HIGH |
| Inconsistent exception types in `OhttpKeyConfig` | IMPROVEMENT |
| `directBaseUrl` enables accidental non-OHTTP path | IMPROVEMENT |

---

## 6. Workflows

These workflows describe the **existing runtime behaviour** as read from the source. Investigation concerns are noted inline.

### 6.1 Happy path — `OhttpClient.send()` (`lib/src/ohttp_client.dart`)

```
caller.send(method, path, headers, body)
  │
  ├─ 1. GET ${gatewayBaseUrl}${configPath}
  │       → http.Client.get() — no timeout, no retry
  │       → on statusCode != 200: throw Exception (generic, not typed)
  │       → OhttpKeyConfig.parse(bodyBytes)
  │           → extra KDF+AEAD pairs: silently ignored
  │       → config.validate()
  │  [CONCERN] KeyConfig fetched fresh on every call — no caching, no TTL
  │  [CONCERN] No scheme check; plain-HTTP gateway URL silently accepted
  │
  ├─ 2. bhttp.serializeRequest(method, scheme, authority, path, headers, body)
  │       → RFC 9292 Known-Length framing
  │  [CONCERN] No upper-bound on body size before serialization
  │
  ├─ 3. ohttpEncapsulate(config, binaryRequest)
  │       → DHKEM(X25519) Encap + HPKE key schedule
  │       → ctx.seal(aad=[], binaryRequest)   — AES-128-GCM
  │       → ctx.export("message/bhttp response", 16)
  │       → assemble: header(7) || enc(32) || ciphertext
  │  [CONCERN] Empty AAD deviates from RFC 9458 §4.6.1 (header-as-AAD wording)
  │  [CONCERN] enc and exportedSecret never zeroed after this call returns
  │
  ├─ 4. POST ${gatewayBaseUrl}${requestPath}
  │       Content-Type: message/ohttp-req
  │       → http.Client.post() — no timeout, no retry, no cancellation
  │       → on statusCode != 200: throw Exception (generic)
  │  [CONCERN] Network errors surface as untyped exceptions
  │
  ├─ 5. ohttpDecapsulate(enc, exportedSecret, gatewayResponse.bodyBytes)
  │       → responseNonce = bodyBytes[0..15]
  │       → plain HKDF-Extract(salt=enc||responseNonce, ikm=exportedSecret)
  │       → plain HKDF-Expand → key(16 B) + nonce(12 B)
  │       → AES-128-GCM decrypt(aad=[])
  │           → auth failure → SecretBoxAuthenticationError (package:cryptography type)
  │  [CONCERN] Plain HKDF labels differ from labeled variants — any unification is a protocol break
  │  [CONCERN] No size cap on gatewayResponse.bodyBytes before decapsulation
  │  [CONCERN] AEAD failure type leaks package:cryptography into public API surface
  │
  └─ 6. bhttp.parseResponse(binaryResponse)
         → varint statusCode (no range check)
         → header loop: no total-size guard
         → body sublist: RangeError on truncation (not FormatException)
     [CONCERN] No max-header-count or max-body-size guard
```

### 6.2 KeyConfig fetch error path

All error types collapse to `Exception` or propagate raw — no retry logic, no typed error hierarchy.
`RangeError` from malformed KeyConfig body propagates unwrapped; caller cannot distinguish it from a logic bug.

### 6.3 Gateway POST error path

AEAD auth failure (`SecretBoxAuthenticationError`) propagates unwrapped — `package:cryptography` type leaks into the public API surface.
No distinction between transient (5xx) and permanent (4xx) gateway errors; no retry or back-off.

### 6.4 `sendDirect()` bypass path (`lib/src/ohttp_client.dart`)

Same `OhttpGatewayConfig` object supports both OHTTP and non-OHTTP paths; a misconfiguration silently sends plaintext to the target server.
No assertion or warning that `sendDirect()` bypasses OHTTP privacy guarantees.

---

## 7. Logging approach

The `ohttp_dart` package currently contains **no logging whatsoever** — no `dart:developer` calls, no `print` statements, no structured event emission. All observable state is returned via return values or thrown exceptions.

### Observability gaps identified during investigation

| Missing event | File | Gap severity |
|---|---|---|
| KeyConfig fetch succeeded / failed | `lib/src/ohttp_client.dart` | HIGH — no way to distinguish gateway misconfiguration from network failure in a deployed wallet |
| Gateway POST status code | `lib/src/ohttp_client.dart` | HIGH — non-200 responses throw generic `Exception`; no structured record |
| AEAD authentication failure on decap | `lib/src/ohttp.dart` | HIGH — `SecretBoxAuthenticationError` propagates silently; no audit trail |
| `sendDirect()` invoked (OHTTP bypass) | `lib/src/ohttp_client.dart` | HIGH — plaintext path has no warning signal |
| OHTTP encapsulation completed | `lib/src/ohttp.dart` | IMPROVEMENT — request round-trip timing is unobservable |

### Constraints for future logging

Any future logging implementation **must never** log:

1. Key material of any kind (`HpkeSenderContext.key`, `exporterSecret`, any HKDF output).
2. The ephemeral `enc` value — leaking it alongside the ciphertext breaks forward secrecy.
3. The inner request URL, path, method, headers, or body — defeats OHTTP's purpose.
4. `targetAuthority` from `OhttpGatewayConfig` — identifies the privacy-preserving target server; defeats unlinkability.
5. Response body or response headers — may contain wallet balances or PII.
6. `gatewayBaseUrl` at DEBUG or lower in production builds — correlating gateway identity with timing is a metadata leak.

---

## Out of scope

- Implementing any of the identified improvements (this is an investigation ticket only).
- Adding new dependencies to `pubspec.yaml`.
- Changing any source file under `lib/` or `test/`.
- Designing new APIs, data models, or architecture layers.
- Writing fuzz tests, integration tests, or end-to-end tests (those are follow-up tasks).
- Evaluating alternative cryptographic libraries or cipher suites.

---

## References

- [Jira: AW-2865](https://adguard.atlassian.net/browse/AW-2865) — source ticket
- [Blocks: AW-2857](https://adguard.atlassian.net/browse/AW-2857)
- [GitHub: ohttp_dart](https://github.com/AdguardTeam/ohttp_dart)
- [Notion: OHTTP Dart](https://www.notion.so/adguard/OHTTP-Dart-360aa56b773080619205ff76a2a360f9)
- RFC 9458 — Oblivious HTTP Application Intermediaries
- RFC 9180 — Hybrid Public Key Encryption
- RFC 9292 — Binary Representation of HTTP Messages
