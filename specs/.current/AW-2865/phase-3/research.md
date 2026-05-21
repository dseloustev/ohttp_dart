# Phase 3 Research — OHTTP Layer Audit Against RFC 9458

**Ticket:** AW-2865
**Phase:** 3
**File audited:** `lib/src/ohttp.dart` (258 lines)
**Status:** COMPLETE

---

## Phase Scope

Phase 3 is a read-only audit of `lib/src/ohttp.dart`, the OHTTP encapsulation and
decapsulation core (Layer 3 of the four-layer stack). It sits above `hpke.dart` (Layer 1,
audited in Phase 1) and `bhttp.dart` (Layer 2, audited in Phase 2), and below
`ohttp_client.dart` (Layer 4, deferred to Phase 5).

The audit covers:
- `OhttpKeyConfig.parse` wire-layout correctness against RFC 9458 §4.1
- Silent drop of extra KDF+AEAD pairs when `symLen > 4`
- `RangeError` vs `FormatException` on malformed KeyConfig input
- `FormatException` vs `UnsupportedError` inconsistency for unknown KEM
- HPKE info string construction in `ohttpEncapsulate` against RFC 9458 §4.3
- Empty AAD in request sealing (intentional deviation matching Go reference implementation)
- Plain (unlabeled) HKDF in response decapsulation against RFC 9458 §4.4
- Empty AAD in response AEAD decryption
- `SecretBoxAuthenticationError` leaking `package:cryptography` into the public API
- Absence of zeroization for `enc` and `exportedSecret` in `OhttpEncapsulateResult`
- Test coverage gaps for the ohttp layer

No code was changed. All findings are deferred to follow-up tasks.

---

## Resolved Questions

1. **Silent multi-suite drop severity** — HIGH. The parser silently ignoring extra
   KDF+AEAD pairs should be fixed before wallet use, but is not an unconditional
   deployment block.

2. **Empty-AAD deviation documentation scope** — An inline comment at the
   `ctx.seal(aad=[], ...)` call site (citing the Go reference implementation) is
   sufficient. No separate ADR is required.

3. **`SecretBoxAuthenticationError` wrapping layer** — Wrap in `ohttp.dart`, the layer
   closest to the decap source. The new exception type is defined and thrown in
   `ohttp.dart`, not in `ohttp_client.dart` or a shared file.

4. **Zeroization severity** — HIGH, unified finding with a note that AOT is the
   higher-risk deployment target. The Dart VM does not guarantee that explicit zero-writes
   prevent access via GC roots or JIT-optimized memory; best-effort zeroization (explicit
   zero-fill before nulling references) is the actionable remediation.

5. **Go reference cite** — General mention "matches Go reference implementation" is
   sufficient. No URL or commit hash required.

6. **Cross-phase references** — Reference Phase 1 and Phase 2 findings by phase number
   only. Details are in those phase documents.

---

## Related Modules / Services

- `lib/src/ohttp.dart` — subject of this audit (258 lines)
- `lib/src/hpke.dart` — Layer 1; `HpkeSender.setupBaseS`, `HpkeSender.hkdfExtract`,
  `HpkeSender.hkdfExpand`, and `HpkeSenderContext` are called directly from `ohttp.dart`
- `lib/src/ohttp_client.dart` — Layer 4; calls `ohttpEncapsulate` and `ohttpDecapsulate`;
  receives unwrapped exceptions from `ohttp.dart`
- `lib/ohttp_dart.dart` — re-exports `hpke.dart` fully including the unlabeled
  `hkdfExtract`/`hkdfExpand` static methods, making them part of the public API surface
- `test/ohttp_test.dart` — existing test file; covers happy-path parse, validate,
  `ohttpEncapsulate` structure check, and two trivial short-response rejection cases in
  `ohttpDecapsulate`; no tests for multi-suite parse, malformed-buffer `RangeError`,
  decryption failure, or round-trip decrypt correctness
- `package:cryptography` — `AesGcm.with128bits`, `SecretBox`, `SecretBoxAuthenticationError`,
  `SecretKeyData`, `Mac`, `X25519`

---

## Current Endpoints & Contracts

### `OhttpKeyConfig.parse(Uint8List data)` — `ohttp.dart:32-79`

Parses the RFC 9458 §4.1 wire layout. Returns an `OhttpKeyConfig` on success; throws on
malformed or unsupported input.

**Wire-layout reading (confirmed correct for the happy path):**

| Field | Wire bytes | Code location |
|---|---|---|
| `key_id` | byte 0 | `ohttp.dart:40` |
| `kem_id` | bytes 1–2 BE | `ohttp.dart:41` |
| `public_key` | bytes 3–34 (32 B for X25519) | `ohttp.dart:57` |
| `symLen` | bytes 35–36 BE | `ohttp.dart:60` |
| `kdf_id` | bytes 37–38 BE | `ohttp.dart:68` |
| `aead_id` | bytes 39–40 BE | `ohttp.dart:70` |

RFC 9458 §4.1 prescribes exactly this layout. The parser correctly reads the 7-byte
minimum guard (`data.length < 7`), the public-key length check (`data.length < offset +
pkLen + 2`), and the symmetric-algorithms section check (`symLen < 4 || data.length <
offset + symLen`). The happy-path layout is **compliant with RFC 9458 §4.1**.

### `OhttpKeyConfig.validate()` — `ohttp.dart:81-97`

Post-parse semantic validation. Throws `UnsupportedError` for any non-X25519 KEM, non-
HKDF-SHA256 KDF, or non-AES-128-GCM AEAD. Called unconditionally at the start of
`ohttpEncapsulate`.

### `ohttpEncapsulate(OhttpKeyConfig, Uint8List, {SimpleKeyPairData?})` — `ohttp.dart:131-167`

Async. Calls `config.validate()`, builds the HPKE info string, calls
`HpkeSender.setupBaseS`, seals the plaintext with empty AAD, exports a 16-byte response
secret, and assembles `header(7) || enc(32) || ciphertext`. Returns
`OhttpEncapsulateResult`.

### `ohttpDecapsulate(Uint8List enc, Uint8List exportedSecret, Uint8List encResponse)` — `ohttp.dart:174-226`

Async. Validates response length, splits `response_nonce` (16 B) and ciphertext, derives
AEAD key/nonce via plain HKDF, decrypts with `AesGcm.with128bits()` using empty AAD,
returns the decrypted plaintext as `Uint8List`.

### `_buildHpkeInfo(OhttpKeyConfig)` — `ohttp.dart:232-245`

Private helper. Builds the HPKE info string: `utf8("message/bhttp request") || 0x00 ||
key_id(1) || kem_id(2 BE) || kdf_id(2 BE) || aead_id(2 BE)`.

### `_buildRequestHeader(OhttpKeyConfig)` — `ohttp.dart:249-257`

Private helper. Builds the 7-byte OHTTP request header: `key_id(1) || kem_id(2 BE) ||
kdf_id(2 BE) || aead_id(2 BE)`.

---

## Patterns Used

1. **Sequential byte-offset parsing with manual length guards** — same pattern as
   `bhttp.dart`. Length guards are present for the minimum-length, public-key section, and
   symmetric-algorithms section, but a gap exists for the first KDF+AEAD read when
   `symLen == 4` exactly (see Finding O-4 below — the existing guard `data.length < offset +
   symLen` covers this case, so no gap for the first pair; the gap is silently ignoring
   subsequent pairs when `symLen > 4`).

2. **`FormatException` for structural parse failures** — used at `ohttp.dart:34`,
   `ohttp.dart:51`, `ohttp.dart:55`, `ohttp.dart:64`, `ohttp.dart:180-182`,
   `ohttp.dart:212-213`. Consistent within the parser itself, but inconsistent with
   `validate()` which uses `UnsupportedError`.

3. **`UnsupportedError` for semantic rejection in `validate()`** — `ohttp.dart:82-96`. A
   different exception hierarchy than the `FormatException` used in `parse()` for the same
   "unknown KEM" condition.

4. **`BytesBuilder` for buffer assembly** — used in `_buildHpkeInfo`; avoids spread-list
   allocations for the info construction.

5. **Spread-list concatenation** — used in `ohttpEncapsulate` at `ohttp.dart:160`
   (`[...header, ...ctx.enc, ...ct]`) and in `ohttpDecapsulate` at `ohttp.dart:190`
   (`[...enc, ...responseNonce]`). These create intermediate `List<int>` objects followed
   by `Uint8List.fromList`; not a correctness issue but contributes to allocation pressure
   on large requests.

6. **Plain (unlabeled) HKDF in response decap** — `HpkeSender.hkdfExtract` and
   `HpkeSender.hkdfExpand` called directly at `ohttp.dart:193`, `ohttp.dart:196-200`,
   `ohttp.dart:203-207`. These are the RFC 5869 primitives, distinct from the labeled
   `LabeledExtract`/`LabeledExpand` called internally during `HpkeSender.setupBaseS`.

7. **`testKeyPair` injection parameter on `ohttpEncapsulate`** — `ohttp.dart:134`. Passed
   through to `HpkeSender.setupBaseS`. Carries forward the Phase 1 F-1 finding into Layer 3.

---

## Findings

### O-1 — Wire layout of `OhttpKeyConfig.parse` complies with RFC 9458 §4.1 (POSITIVE)

**File:** `ohttp.dart:32-79`
**RFC section:** RFC 9458 §4.1

The parser reads `key_id` (1 B), `kem_id` (2 B BE), `public_key` (32 B for X25519),
`symLen` (2 B BE), and the first `kdf_id`/`aead_id` pair (4 B) from the correct byte
offsets. All four length guards are present and ordered correctly. The happy-path wire
layout matches RFC 9458 §4.1 exactly.

Verdict: **COMPLIANT**

---

### O-2 — Silent drop of extra KDF+AEAD pairs when `symLen > 4` (HIGH)

**File:** `ohttp.dart:60-78`
**RFC section:** RFC 9458 §4.1

RFC 9458 §4.1 allows a gateway to advertise multiple KDF+AEAD pairs (`sym_algorithms`
field is a variable-length list). `symLen > 4` means more than one pair is present. The
parser at `ohttp.dart:67-71` reads exactly the first 4-byte pair and returns without
checking or storing remaining pairs. There is no loop, no validation that subsequent pairs
are rejected, and no error thrown.

Consequences for a wallet:
- If a gateway advertises suites in order of preference with the preferred suite second
  (e.g., a future cipher-suite migration), the client silently uses the first advertised
  suite regardless of gateway preference — implicit cipher-suite downgrade with no
  diagnostic.
- The guard at `ohttp.dart:63` (`data.length < offset + symLen`) confirms that the extra
  bytes are present in the buffer, yet they are never examined.
- `validate()` at `ohttp.dart:81-97` then enforces that the selected (first) suite is
  `0x0020 / 0x0001 / 0x0001`. If the first pair is the supported suite, `validate()`
  passes silently even when additional unsupported pairs were advertised.

No `RangeError` risk from this specific path: the guard already confirms sufficient buffer
length for all `symLen` bytes before reading the first pair.

**Cross-phase reference:** This is an independent instance of the silent-ignore pattern
documented at the BHTTP layer in Phase 2. No direct Phase 2 finding covers this; it is
a new gap at Layer 3.

**Test gap:** `ohttp_test.dart` has no test with `symLen > 4` (multi-suite KeyConfig).

---

### O-3 — `RangeError` on malformed KeyConfig with `symLen < 4` or inconsistently short buffer (HIGH)

**File:** `ohttp.dart:63-65`, `ohttp.dart:68-71`
**RFC section:** RFC 9458 §4.1

The guard at `ohttp.dart:63`:
```
if (symLen < 4 || data.length < offset + symLen) {
  throw const FormatException('Invalid symmetric algorithms section');
}
```

This guard is correct for the two conditions it tests. However, there is a latent gap:
the buffer length guard at `ohttp.dart:54-56` checks `data.length < offset + pkLen + 2`,
ensuring there are enough bytes for the public key plus the 2-byte `symLen` field itself,
but the reads of `kdf_id` and `aead_id` at lines 68 and 70 occur after `offset +=
2` at line 61 increments past `symLen`. If a caller constructs a buffer that passes the
guard at line 63 (because `data.length >= offset + symLen`) but `symLen` is exactly 2
(which fails `symLen < 4` — so this is caught), the `FormatException` is thrown.

The more relevant scenario: a buffer where `symLen >= 4` but the buffer is exactly
`offset + symLen - 1` bytes long. The guard at line 63 catches this (`data.length <
offset + symLen`), throwing `FormatException`. This is correct.

However, the guard at line 54 only checks for `pkLen + 2` bytes beyond `offset`, not
for `pkLen + 2 + symLen` bytes. This means the `symLen` field itself is safely readable,
but if an adversary crafts a buffer where `data.length == offset + pkLen + 2` (exactly the
minimum for the symmetric-algorithms length field) and `symLen` is large, the guard at
line 63 correctly catches it. No `RangeError` escapes through this specific path.

**Actual `RangeError` exposure:** If a buffer is 6 bytes (passes the `< 7` guard at line
33 fails — wait, `data.length < 7` throws `FormatException` at line 34). Any buffer of 7+
bytes passes line 33. A buffer of exactly 7 bytes: `data[0]` (keyId), `data[1-2]` (kemId),
then the `switch` at line 46 executes `pkLen = 32`. The guard at line 54 checks
`data.length < offset + 32 + 2 = 3 + 32 + 2 = 37`. With 7 bytes, `7 < 37` is true —
`FormatException` thrown. So all truncation cases before the `symLen` read are caught.

After `symLen` is read (line 60), the guard at line 63 checks `data.length < offset +
symLen`. If this passes, the reads at lines 68 and 70 access `data[offset]`,
`data[offset+1]`, `data[offset+2]`, `data[offset+3]`. The guard at line 63 guarantees
`data.length >= offset + symLen` and `symLen >= 4`, so `data[offset+3]` is safe. **No
`RangeError` escapes through the parser in the current code when all guards are examined.**

**Revised assessment:** The parser's guards are sufficient for the current read pattern.
The risk documented in `vision.md §5.1` ("malformed blob with large `symLen` and short
buffer throws `RangeError`, not `FormatException`") is **not present in the current code**
because the guard at line 63 catches both `symLen < 4` and `data.length < offset + symLen`
before any list access. This is a **positive finding** — the parser handles short-buffer
conditions with `FormatException`, not `RangeError`.

**However**, the *error message* at line 64 is `'Invalid symmetric algorithms section'`
for both `symLen < 4` and short-buffer conditions. These are structurally different errors
(semantically invalid vs. truncated) combined into one message. An IMPROVEMENT would
separate them.

**Test gap:** `ohttp_test.dart` has a test for `symLen == 2` at line 50-62 (throws
`FormatException` — confirmed correct). No test for a buffer where `symLen` is valid but
the buffer is truncated mid-symmetric-section.

---

### O-4 — Exception-type inconsistency: `parse()` throws `FormatException`, `validate()` throws `UnsupportedError` for the same unknown-KEM condition (IMPROVEMENT)

**File:** `ohttp.dart:51` (`parse`) and `ohttp.dart:82-85` (`validate`)
**RFC section:** RFC 9458 §4.1

`parse()` at line 51:
```dart
throw FormatException('Unsupported KEM: 0x${kemId.toRadixString(16)}');
```

`validate()` at line 82-85:
```dart
if (kemId != 0x0020) {
  throw UnsupportedError(
    'Unsupported KEM: 0x${kemId.toRadixString(16)} (expected X25519 0x0020)',
  );
}
```

Both conditions express "this KEM is not supported." A caller who catches `FormatException`
to handle unsupported-KEM from `parse()` will not catch the `UnsupportedError` path from
`validate()`. Conversely, a caller who catches `UnsupportedError` for `validate()` will
not catch the `FormatException` from `parse()`.

In the normal call sequence (`parse()` then `validate()`), the `parse()` throw at line 51
is reached first (since `parse()` rejects unknown KEMs via its `switch`). The `validate()`
`UnsupportedError` for KEM is reachable only when an `OhttpKeyConfig` is constructed
directly (bypassing `parse()`) with an unsupported `kemId`, or when the `OhttpKeyConfig`
constructor is called from test code with a crafted value (as in `ohttp_test.dart:79-85`).
The test at line 85 confirms `validate()` throws `UnsupportedError`.

**Practical impact:** The inconsistency creates an unreliable catch surface.
`ohttp_client.dart` calls both `parse()` and `validate()` indirectly via
`ohttpEncapsulate`. Any catch clause in the wallet that handles "unsupported cipher suite"
must catch both `FormatException` and `UnsupportedError` to be complete.

**Cross-phase reference:** This is the same inconsistent exception-type anti-pattern
identified at the BHTTP and HPKE layers in Phase 2 (R-3) and Phase 1 (F-3).

---

### O-5 — HPKE info string construction matches RFC 9458 §4.3 (POSITIVE)

**File:** `ohttp.dart:232-245` (`_buildHpkeInfo`)
**RFC section:** RFC 9458 §4.3

RFC 9458 §4.3 prescribes:
```
info = "message/bhttp request" || 0x00 || hdr
```
where `hdr` is the 7-byte OHTTP request header: `key_id(1) || kem_id(2 BE) || kdf_id(2 BE)
|| aead_id(2 BE)`.

`_buildHpkeInfo` at `ohttp.dart:232-245` constructs:
1. `utf8.encode('message/bhttp request')` — 22 bytes
2. `0x00` (null terminator)
3. `config.keyId` (1 B)
4. `(config.kemId >> 8) & 0xFF`, `config.kemId & 0xFF` (2 B BE)
5. `(config.kdfId >> 8) & 0xFF`, `config.kdfId & 0xFF` (2 B BE)
6. `(config.aeadId >> 8) & 0xFF`, `config.aeadId & 0xFF` (2 B BE)

Total info length: 22 + 1 + 7 = 30 bytes. This exactly matches RFC 9458 §4.3.

The `_buildRequestHeader` helper at `ohttp.dart:249-257` produces the same 7-byte header
for the encapsulated-request prefix. Both helpers encode `kem_id`, `kdf_id`, and `aead_id`
as big-endian uint16, consistent with the RFC.

Verdict: **COMPLIANT**

---

### O-6 — Empty AAD in request sealing: intentional deviation matching Go reference implementation (IMPROVEMENT)

**File:** `ohttp.dart:149`
**RFC section:** RFC 9458 §4.3 / §4.6.1

The seal call at `ohttp.dart:149`:
```dart
final ct = await ctx.seal(Uint8List(0), binaryRequest);
```

RFC 9458 §4.3 wording (§4.6.1 in some numbering) suggests using the OHTTP request header
as the AAD for the HPKE `Seal` call. Some readings of the RFC would expect:
`HPKE.Seal(aad=header, plaintext=binaryRequest)`.

The implementation passes an empty `Uint8List(0)` as AAD. This is an intentional
deviation that matches the Go reference implementation. The `CLAUDE.md` note at line 149
documents this explicitly: `"Seal with empty AAD (per reference Go implementation, not RFC
header)"`. The existing test suite (`ohttp_test.dart`) and the bundled encap/decap tests
both pass with empty AAD.

**Impact assessment:** This deviation does not affect confidentiality or integrity of the
plaintext — AES-128-GCM with empty AAD still provides authenticated encryption of the
BHTTP request body. The difference from the RFC's header-as-AAD is that the header bytes
are not authenticated as associated data. In practice, the header is included verbatim in
the `encRequest` output at `ohttp.dart:160`, so the gateway can observe it anyway.

**Remediation:** An inline comment at the `ctx.seal(...)` call site citing the Go
reference implementation is sufficient. No separate ADR is required.

**Test gap:** There is no integration test confirming that a reference Go gateway accepts
requests sealed with empty AAD. This gap was also identified in Phase 1 (F-7).

---

### O-7 — Response decapsulation uses plain (unlabeled) HKDF, correctly distinct from labeled variants (POSITIVE with MAINTENANCE HAZARD)

**File:** `ohttp.dart:192-207`
**RFC section:** RFC 9458 §4.4

RFC 9458 §4.4 prescribes plain (unlabeled) HKDF for response key derivation:
```
salt = enc || response_nonce
prk = HKDF-Extract(salt, exportedSecret)
key = HKDF-Expand(prk, "key", Nk)
nonce = HKDF-Expand(prk, "nonce", Nn)
```

`ohttpDecapsulate` at `ohttp.dart:192-207` implements exactly this:
- `ohttp.dart:193`: `HpkeSender.hkdfExtract(salt, exportedSecret)` — RFC 5869 plain HKDF-Extract
- `ohttp.dart:196-200`: `HpkeSender.hkdfExpand(prk, utf8.encode('key'), _nk)` — plain HKDF-Expand with label `"key"`, length 16
- `ohttp.dart:203-207`: `HpkeSender.hkdfExpand(prk, utf8.encode('nonce'), _nn)` — plain HKDF-Expand with label `"nonce"`, length 12

These are `static` methods on `HpkeSender` that implement RFC 5869 HMAC-SHA256 primitives
without the RFC 9180 suite ID prefix. They are structurally distinct from
`LabeledExtract`/`LabeledExpand` which prepend `"HPKE-v1"` and the suite ID. They cannot
be substituted for each other without producing different key material.

**Salt construction:** `ohttp.dart:190` builds `salt = [...enc, ...responseNonce]`. For
AES-128-GCM, `enc` is 32 B and `responseNonce` is 16 B, producing a 48-byte salt. This
matches RFC 9458 §4.4.

**Response nonce extraction:** `ohttp.dart:186-187` splits the first 16 bytes as
`responseNonce` (= `max(Nn, Nk) = max(12, 16) = 16` for AES-128-GCM). This matches RFC
9458 §4.4.

Verdict: **COMPLIANT**

**Maintenance hazard:** `HpkeSender.hkdfExtract` and `HpkeSender.hkdfExpand` are public
static methods re-exported through `lib/ohttp_dart.dart` (confirmed at
`lib/ohttp_dart.dart:10`). Any future refactoring that replaces these calls with
`LabeledExtract`/`LabeledExpand` would silently derive different key material and break
decryption with no compile-time signal. A code comment at the call sites explicitly
warning against this substitution is the recommended remediation.

---

### O-8 — Response AEAD uses empty AAD: compliant with RFC 9458 §4.4 (POSITIVE)

**File:** `ohttp.dart:219-223`
**RFC section:** RFC 9458 §4.4

The response AES-GCM decrypt at `ohttp.dart:219-223`:
```dart
final plaintext = await aesGcm.decrypt(
  secretBox,
  secretKey: SecretKeyData(aeadKey),
  aad: [],
);
```

RFC 9458 §4.4 specifies that response decryption uses empty AAD (`aad = ""`). The
implementation uses `aad: []` (empty list, equivalent to empty byte string). This is
**compliant** with RFC 9458 §4.4.

Verdict: **COMPLIANT**

---

### O-9 — `SecretBoxAuthenticationError` propagates unwrapped to callers: `package:cryptography` type leaks into public API (HIGH)

**File:** `ohttp.dart:219-223`
**RFC section:** RFC 9458 §4.4

When AEAD authentication fails during `ohttpDecapsulate`, `aesGcm.decrypt()` at
`ohttp.dart:219` throws `SecretBoxAuthenticationError`, which is defined in
`package:cryptography/cryptography.dart`. This exception propagates unwrapped through
`ohttpDecapsulate` → `ohttp_client.dart` → the wallet caller.

**API surface impact:**
- `SecretBoxAuthenticationError` is not re-exported from `lib/ohttp_dart.dart`. A caller
  who wants to catch it specifically must add `import 'package:cryptography/cryptography.dart'`
  directly.
- This creates an unintended transitive dependency: the public API surface of `ohttp_dart`
  effectively requires callers to depend on `package:cryptography` to write complete error
  handling, even though `package:cryptography` is an internal implementation detail.
- If `package:cryptography` is replaced or its exception hierarchy changes, all callers
  must update their catch clauses.
- A wallet-layer developer who does not add the `package:cryptography` import will receive
  `Object` at the catch site, making authentication failure indistinguishable from other
  runtime exceptions.

**Confirmed by `ohttp_test.dart`:** The test at `ohttp_test.dart:149-163` tests short-
response rejection (`FormatException`) but has no test for a response with a valid length
but an incorrect authentication tag. The `SecretBoxAuthenticationError` path is untested.

**Remediation direction:** Define a library-owned exception type (e.g.,
`OhttpAuthenticationException`) in `ohttp.dart` and wrap the `aesGcm.decrypt()` call in a
`try/catch (SecretBoxAuthenticationError)` block that rethrows as the library-owned type.

---

### O-10 — `enc` and `exportedSecret` lack zeroization after `ohttpEncapsulate` returns (HIGH)

**File:** `ohttp.dart:104-120` (`OhttpEncapsulateResult`), `ohttp.dart:162-166`
**RFC section:** RFC 9458 §4.3 (response-secret export)

`OhttpEncapsulateResult` at `ohttp.dart:104-120` is a plain data class with three
`final Uint8List` fields:
- `encRequest` — the complete POST payload (not sensitive after transmission)
- `enc` — the 32-byte ephemeral X25519 public key (semi-sensitive; combining with the
  private key's output would partially recover DH material)
- `exportedSecret` — the 16-byte HPKE export used for response decryption (directly
  sensitive: this is the key material that authenticates and decrypts the gateway response)

Neither `enc` nor `exportedSecret` is zeroed (overwritten with zeros) before the object
becomes eligible for garbage collection. The `Uint8List` backing arrays remain in heap
memory until the GC decides to reclaim them.

**Wallet threat model relevance:**
- `exportedSecret` is directly equivalent to a symmetric key protecting the OHTTP response.
  If an attacker can read heap memory (via a memory dump, cold-boot attack, or a
  vulnerability in a co-located native library), they could recover `exportedSecret` and
  decrypt any intercepted gateway response.
- `enc` enables partial reconstruction of the HPKE session when combined with the
  gateway's ephemeral private key (which the client does not hold). Lower risk than
  `exportedSecret` but still sensitive per the forward-secrecy model.
- AOT-compiled deployments (the primary wallet target) have deterministic memory layout
  without JIT reordering; explicit zero-writes in AOT are more reliably observed than
  under the JIT compiler's optimization passes. AOT is therefore the higher-risk deployment
  target for zeroization absence.

**Dart VM limitation note:** The Dart VM does not guarantee that explicit zero-writes
prevent access via GC roots or JIT-optimized memory. Explicit zero-fill before nulling
references is best-effort, not a cryptographic guarantee. It is still the recommended
actionable remediation.

**Cross-phase reference:** This finding extends the Phase 1 zeroization finding (F-2) to
the OHTTP layer. Phase 1 F-2 documented absence of zeroization in `HpkeSenderContext`
fields (`key`, `baseNonce`, `exporterSecret`). The same absence recurs here in
`OhttpEncapsulateResult`, with `exportedSecret` being the most sensitive field at this layer.

---

### O-11 — Test suite covers only happy-path and minimal negative cases; no round-trip decrypt test, no multi-suite test, no authentication-failure test (HIGH)

**File:** `test/ohttp_test.dart`
**RFC section:** RFC 9458 §4.3, §4.4

`ohttp_test.dart` current coverage:
- `OhttpKeyConfig.parse`: happy path, too-short data, unsupported KEM, short symmetric
  section — four cases, all for the parser guard paths. No test for `symLen > 4`
  (multi-suite drop).
- `OhttpKeyConfig.validate`: supported suite, unsupported KEM/KDF/AEAD — four cases.
- `ohttpEncapsulate`: one structural test that checks output byte counts. No test
  against RFC 9458 test vectors (none exist in a standard appendix, but Go reference
  vectors could be used).
- `ohttpDecapsulate`: two short-response rejection tests. No round-trip test (encapsulate
  then decapsulate), no test for authentication failure (wrong tag), no test for truncated
  ciphertext (after the nonce).

Missing test scenarios:
1. Round-trip: `ohttpEncapsulate` → `ohttpDecapsulate` with matching keys, confirm
   plaintext recovered.
2. Authentication failure: `ohttpDecapsulate` with a valid-length response but a
   bit-flipped ciphertext — confirm `SecretBoxAuthenticationError` propagation path and
   the future `OhttpAuthenticationException` wrapper.
3. Multi-suite KeyConfig: `symLen = 8` (two pairs), first pair is supported — confirm
   parse succeeds and extra pair is silently dropped (documents current behavior as a known
   gap, not a regression guard).
4. Response too short after nonce: 17 bytes (1 nonce byte short of ciphertext minimum) —
   confirm `FormatException` from `ohttp.dart:212-213`.

**Cross-phase reference:** The test-gap pattern recurs at every layer. Phase 1 documented
it as F-6 and F-7; Phase 2 documented it as R-6. Phase 3 is an independent instance.

---

## Limitations & Risks

| ID | Finding | Severity | File : Lines | RFC Section |
|---|---|---|---|---|
| O-1 | Wire layout complies with RFC 9458 §4.1 (positive) | — | `ohttp.dart:32-79` | RFC 9458 §4.1 |
| O-2 | Silent drop of extra KDF+AEAD pairs when `symLen > 4` | HIGH | `ohttp.dart:60-71` | RFC 9458 §4.1 |
| O-3 | `RangeError` risk on short buffer: NOT present (positive); parser guards are complete | — | `ohttp.dart:33-71` | RFC 9458 §4.1 |
| O-4 | `FormatException` (parse) vs `UnsupportedError` (validate) for same unknown-KEM condition | IMPROVEMENT | `ohttp.dart:51`, `ohttp.dart:82-85` | RFC 9458 §4.1 |
| O-5 | HPKE info string construction matches RFC 9458 §4.3 (positive) | — | `ohttp.dart:232-245` | RFC 9458 §4.3 |
| O-6 | Empty AAD in request sealing: intentional deviation matching Go reference implementation | IMPROVEMENT | `ohttp.dart:149` | RFC 9458 §4.3 |
| O-7 | Plain HKDF in response decap is compliant; maintenance hazard if unified with labeled variants | MAINTENANCE | `ohttp.dart:192-207` | RFC 9458 §4.4 |
| O-8 | Response AEAD uses empty AAD: compliant with RFC 9458 §4.4 (positive) | — | `ohttp.dart:219-223` | RFC 9458 §4.4 |
| O-9 | `SecretBoxAuthenticationError` propagates unwrapped; `package:cryptography` leaks into public API | HIGH | `ohttp.dart:219` | RFC 9458 §4.4 |
| O-10 | `enc` and `exportedSecret` not zeroized in `OhttpEncapsulateResult`; AOT is higher-risk target | HIGH | `ohttp.dart:104-120`, `162-166` | RFC 9458 §4.3 |
| O-11 | Test suite missing round-trip, auth-failure, multi-suite, and truncated-ciphertext tests | HIGH | `test/ohttp_test.dart` | RFC 9458 §4.3, §4.4 |

**Risk summary:**
- No BLOCKER findings were identified at the OHTTP layer.
- Four HIGH findings (O-2, O-9, O-10, O-11) should be remediated before wallet use.
- One IMPROVEMENT finding (O-4) and one MAINTENANCE finding (O-7) are recommended
  backlog items.
- Four findings are positive verdicts (O-1, O-3, O-5, O-8).

---

## Draft Task Entries

### TASK-O1 — Fix silent drop of extra KDF+AEAD pairs in `OhttpKeyConfig.parse`

- File: `ohttp.dart:60-78`
- Line range: 60–78
- RFC section: RFC 9458 §4.1
- Severity: HIGH
- Description: When `symLen > 4`, the parser reads only the first KDF+AEAD pair and
  silently returns. Add a loop or explicit rejection: either iterate all pairs and confirm
  only the supported suite is present, or throw `FormatException` when `symLen > 4` to
  reject multi-suite advertisements until multi-suite negotiation is implemented.

### TASK-O2 — Unify exception types for unsupported KEM in `parse()` and `validate()`

- File: `ohttp.dart:51`, `ohttp.dart:82-85`
- Line range: 51, 82–85
- RFC section: RFC 9458 §4.1
- Severity: IMPROVEMENT
- Description: `parse()` throws `FormatException` and `validate()` throws `UnsupportedError`
  for the same "unsupported KEM" condition. Consolidate to a single exception type (e.g.,
  `UnsupportedError` throughout, or a library-owned `OhttpUnsupportedSuiteException`) so
  callers can write a single catch clause. Cross-phase reference: same inconsistent
  exception-type anti-pattern as Phase 1 F-3 and Phase 2 R-3.

### TASK-O3 — Add inline comment at empty-AAD `ctx.seal` call site

- File: `ohttp.dart:149`
- Line range: 148–150
- RFC section: RFC 9458 §4.3
- Severity: IMPROVEMENT
- Description: The comment at line 148 mentions "per reference Go implementation, not RFC
  header" but does not fully explain the intentional deviation. Expand the inline comment
  to: cite that RFC 9458 §4.3 wording suggests header-as-AAD; explain that the
  implementation deliberately uses empty AAD matching the Go reference implementation for
  interoperability; note that this is confirmed by the bundled test suite.

### TASK-O4 — Add maintenance-hazard comment at plain-HKDF call sites in `ohttpDecapsulate`

- File: `ohttp.dart:192-207`
- Line range: 192–207
- RFC section: RFC 9458 §4.4
- Severity: MAINTENANCE (IMPROVEMENT)
- Description: Add a code comment at lines 193, 196, and 203 warning that these are plain
  (unlabeled) RFC 5869 HKDF calls per RFC 9458 §4.4 and must NOT be replaced with
  `LabeledExtract`/`LabeledExpand` from `hpke.dart`. Substitution would silently derive
  different key material and break response decryption.

### TASK-O5 — Wrap `SecretBoxAuthenticationError` in a library-owned exception

- File: `ohttp.dart:217-225`
- Line range: 217–225
- RFC section: RFC 9458 §4.4
- Severity: HIGH
- Description: Wrap the `aesGcm.decrypt()` call in a try/catch for
  `SecretBoxAuthenticationError` and rethrow as a new library-owned exception (e.g.,
  `OhttpAuthenticationException`) defined in `ohttp.dart`. This prevents
  `package:cryptography` from leaking into the public API surface. Callers can then catch
  `OhttpAuthenticationException` without importing `package:cryptography`.

### TASK-O6 — Add best-effort zeroization for `OhttpEncapsulateResult.enc` and `exportedSecret`

- File: `ohttp.dart:104-120`, `ohttp.dart:162-166`
- Line range: 104–120, 162–166
- RFC section: RFC 9458 §4.3
- Severity: HIGH
- Description: After `ohttpDecapsulate` has consumed `enc` and `exportedSecret`, add an
  explicit zero-fill of both `Uint8List` backing buffers before the reference is released.
  This is best-effort (Dart VM does not guarantee GC-root clearing) but reduces exposure
  in AOT-compiled deployments, which are the primary wallet target and the higher-risk
  scenario for memory-dump attacks. Cross-phase reference: extends Phase 1 F-2
  (zeroization absence in `HpkeSenderContext`) to the OHTTP layer.

### TASK-O7 — Add missing test cases for `ohttp_test.dart`

- File: `test/ohttp_test.dart`
- Line range: all
- RFC section: RFC 9458 §4.3, §4.4
- Severity: HIGH
- Description: Add four test groups:
  1. Round-trip test: `ohttpEncapsulate` followed by a gateway-simulated
     `ohttpDecapsulate` using the same exported secret — confirm plaintext recovery.
  2. Authentication failure test: valid-length encResponse with a bit-flipped byte in
     the ciphertext — confirm the future `OhttpAuthenticationException` (from TASK-O5) is
     thrown.
  3. Multi-suite KeyConfig test: `symLen = 8` with supported first pair — confirm parse
     succeeds and documents that the second pair is silently dropped (current behavior).
  4. Truncated ciphertext test: `encResponse` length between `_responseNonceLen + 1` and
     `_responseNonceLen + tagLen - 1` — confirm `FormatException` from `ohttp.dart:212-213`.
  Cross-phase reference: same test-gap pattern as Phase 1 F-6/F-7 and Phase 2 R-6.

---

## New Technical Questions

1. **Is there a companion `symLen == 0` test?** The guard at `ohttp.dart:63` rejects
   `symLen < 4`, which covers `symLen == 0`, 1, 2, and 3. A test with `symLen == 0` would
   confirm the exact error message. Currently only `symLen == 2` is tested. Low priority
   but worth adding to TASK-O7.

2. **Does `ohttp_client.dart` provide any caller-visible wrapping of `FormatException`
   or `UnsupportedError` from `ohttpEncapsulate`?** If not, the inconsistent exception
   types (O-4) propagate all the way to the wallet with no normalization. This should be
   confirmed in the Phase 5 (`ohttp_client.dart`) audit.

3. **`OhttpEncapsulateResult` lifetime contract.** The PRD and vision describe the lifetime
   as "a single `send()` invocation." Is this contract enforced anywhere in the code, or
   is it only a documentation statement? If the result object is accidentally retained
   (e.g., stored in a cache or a BLoC state), `exportedSecret` persists indefinitely.
   TASK-O6 covers the zeroization remediation, but an assertion or a `dispose()`-style
   API would provide an additional guard.

4. **`testKeyPair` parameter on `ohttpEncapsulate`.** The Phase 1 audit identified the
   `testKeyPair` injection hook in `HpkeSender.setupBaseS` as F-1 (HIGH). The same
   parameter is exposed on the `ohttpEncapsulate` public API at `ohttp.dart:134`. Since
   `ohttpEncapsulate` is re-exported through `lib/ohttp_dart.dart`, any caller can pass
   a fixed ephemeral key in production, silently breaking forward secrecy. This was not
   independently classified as a Phase 3 finding because it is a consequence of Phase 1
   F-1 (TASK-1). Phase 5 should confirm whether `ohttp_client.dart` ever passes a
   non-null `testKeyPair` in its production path.
