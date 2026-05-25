# Typed `OhttpException` hierarchy and KeyConfig multi-suite negotiation

**Estimate:** 3d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

The library has no typed exception hierarchy and ignores trailing pairs in a multi-suite KeyConfig. Both defects sit on the same error/return surface and must land together so every new throw site lands on the typed hierarchy from day one:

1. **No typed exception hierarchy.** `OhttpClient.send()` at `lib/src/ohttp_client.dart:84-87, 120-122` throws bare `Exception('...')` for KeyConfig GET and gateway POST non-200 responses, embedding the status code only inside the message string. Network errors (DNS failure, connection reset) surface as untyped exceptions. AEAD authentication failures bubble up as `SecretBoxAuthenticationError` from `package:cryptography` (via `aesGcm.decrypt(...)` at `lib/src/ohttp.dart:217-225`), which makes the transitive dependency part of the public API surface. A wallet caller cannot distinguish a 5xx from a 4xx, a timeout from a parse error, or an AEAD authentication failure from a transport failure without parsing exception messages.

2. **KeyConfig multi-suite negotiation is broken.** RFC 9458 §4.1 permits a gateway to advertise multiple KDF+AEAD pairs in the symmetric algorithms section (`symLen > 4`). `OhttpKeyConfig.parse()` at `lib/src/ohttp.dart:60-78` reads `symLen`, performs a bounds check, then consumes exactly one 4-byte (`kdfId`, `aeadId`) pair at lines 67–71 and returns, silently dropping any trailing pairs. A gateway that advertises an unsupported suite first followed by the supported suite will cause the parser to lock onto the first pair and fail validation — or silently succeed on a suite the library does not implement.

3. **Companion mismatch in unsupported-suite errors.** `parse()` throws `FormatException('Unsupported KEM: ...')` at line 51, while `validate()` throws `UnsupportedError` at lines 82–85 for the same conceptual condition. Two unrelated exception hierarchies for the same error class force callers to write two `catch` blocks or fall back to broad `catch (e)`.

Doing these together avoids re-churning the exception surface: the multi-suite fix introduces new throw sites that should already use the typed hierarchy.

## Technical Details

### Part A — Typed exception hierarchy

**1. Define the typed hierarchy**

Introduce a common base class `OhttpException` (extends `Exception`) and concrete subclasses:

- `OhttpHttpException(statusCode, body?)` — non-2xx response from the relay or gateway POST. Subdivide or expose enough information that callers can branch on `statusCode >= 400 && < 500` vs `>= 500`.
- `OhttpTimeoutException` — request timeout (consumed by the timeout work that lands in L-03).
- `OhttpParseException(cause)` — malformed BHTTP/OHTTP/KeyConfig response.
- `OhttpKeyConfigException` — KeyConfig-specific parsing failures.
- `OhttpDecryptionException(cause)` — wraps the underlying `SecretBoxAuthenticationError` thrown by `aesGcm.decrypt(...)` at `lib/src/ohttp.dart:217-225`.
- `OhttpUnsupportedSuiteException` — unsupported KEM/KDF/AEAD (consumed by Part B below).

Re-export every type from `lib/ohttp_dart.dart`.

**2. Replace bare throws** (`lib/src/ohttp_client.dart:84-87, 120-122` and `lib/src/ohttp.dart:217-225`)

- Lines 84–87: `if (configResponse.statusCode != 200) { throw Exception('...'); }` → `throw OhttpHttpException(statusCode: configResponse.statusCode, ...)`.
- Lines 120–122: same pattern for the gateway POST.
- Lines 217–225 in `ohttp.dart`: catch `SecretBoxAuthenticationError` inside `ohttpDecapsulate` and re-throw `OhttpDecryptionException` wrapping the original cause without leaking the internal cause type name into the public contract.

Update doc comments on `send()`, `sendDirect()`, and `ohttpDecapsulate` to enumerate the exceptions they may throw.

### Part B — KeyConfig multi-suite parsing

**3. Rework `OhttpKeyConfig.parse` multi-pair handling** (`lib/src/ohttp.dart:60-78`)

Inspect every KDF+AEAD pair in the symmetric algorithms section. Either:

(a) Select the first pair that matches the library's single fixed supported suite (`kdfId == 0x0001`, `aeadId == 0x0001`), iterating through the section until found.

(b) Require an exact-single-pair config and raise `OhttpUnsupportedSuiteException` when the section length implies multiple pairs the client cannot negotiate.

Pick one and document the chosen policy in the function doc comment. Cite RFC 9458 §4.1.

Also reject `symLen` values that are not a multiple of 4 (malformed wire format) with `OhttpKeyConfigException`.

**4. Unify the unsupported-suite exception type** (`lib/src/ohttp.dart:51, 82-85`)

Throw `OhttpUnsupportedSuiteException` from both `parse()` (currently `FormatException` at line 51) and `validate()` (currently `UnsupportedError` at lines 82–85) for unsupported KEM/KDF/AEAD. The type is part of the hierarchy defined in Part A.

### Part C — Tests

**5. Exception hierarchy tests** (`test/ohttp_test.dart`, `test/ohttp_client_test.dart`)

- Mock `http.Client` to return 4xx → assert `OhttpHttpException` with `statusCode` in the 4xx range.
- Mock `http.Client` to return 5xx → assert `OhttpHttpException` with `statusCode` in the 5xx range.
- Produce a valid OHTTP encap, derive the response secret, encrypt a response, flip one byte in the ciphertext, call `ohttpDecapsulate`, assert that an `OhttpDecryptionException` is thrown (not the underlying `SecretBoxAuthenticationError`).

**6. Multi-suite tests** (`test/ohttp_test.dart`, after the existing `OhttpKeyConfig.parse` group around line 62)

Construct synthetic KeyConfig byte buffers in test fixtures (documented byte arrays so the wire format is explicit). Cover:

- Single supported pair (current happy path).
- Two pairs where the supported pair is at index 0.
- Two pairs where the supported pair is at index 1.
- Two/three pairs where no supported suite is present (expect `OhttpUnsupportedSuiteException` per the chosen policy).
- Malformed `symLen` not divisible by 4 (expect `OhttpKeyConfigException`).
- Unknown KEM via `parse()` and via `validate()` — both must throw `OhttpUnsupportedSuiteException`.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

**Hierarchy:**
- Library-owned `OhttpException` base class is exported from `lib/ohttp_dart.dart` with at least the subtypes listed in Part A.
- All `throw Exception(...)` sites in `lib/src/ohttp_client.dart` are replaced with typed subclasses carrying the relevant context (status code, cause).
- `ohttpDecapsulate` no longer surfaces `SecretBoxAuthenticationError` directly; it surfaces `OhttpDecryptionException` instead.
- Doc comments on `send()`, `sendDirect()`, and `ohttpDecapsulate` enumerate the exceptions they may throw.

**Multi-suite:**
- `OhttpKeyConfig.parse()` no longer ignores trailing bytes when `symLen > 4`; it either iterates pairs and selects a supported one, or throws `OhttpUnsupportedSuiteException` per the documented policy.
- Doc comment on `parse()` cites RFC 9458 §4.1 and documents the chosen negotiation policy.
- Malformed `symLen` (not divisible by 4) is rejected with `OhttpKeyConfigException`.
- A single library-owned exception type (`OhttpUnsupportedSuiteException`) is thrown from both `parse()` and `validate()` for unsupported KEM/KDF/AEAD.

**Tests:**
- Hierarchy: gateway 4xx → typed exception with `statusCode == 4xx`; gateway 5xx → typed exception with `statusCode == 5xx`; AEAD auth failure → `OhttpDecryptionException`.
- Multi-suite: single supported pair, supported pair at index 0, supported pair at index 1, no supported pair (typed exception), malformed `symLen` (typed exception), unsupported KEM through both `parse()` and `validate()` paths.

## Additional

- Tests: unit tests for each typed exception path and each multi-suite wire-format case enumerated above.
- Migration notes:
  - Downstream callers currently catching `Exception` from `OhttpClient.send`, `SecretBoxAuthenticationError` from decap, or `FormatException` / `UnsupportedError` from `parse`/`validate` must migrate to the new typed surface.
- This task should land second (after L-01), so subsequent tasks (L-03/L-04/L-06) can emit their new throws onto the typed surface from day one.
- Source merged tasks (in `../merged-tasks/`): M-05 (typed exception hierarchy), M-04 (KeyConfig multi-suite + unsupported-suite unification).
