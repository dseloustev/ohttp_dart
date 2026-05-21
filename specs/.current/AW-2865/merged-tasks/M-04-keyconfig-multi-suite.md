# Fix KeyConfig multi-suite negotiation and unify unsupported-suite exception

**Estimate:** 1d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

RFC 9458 §4.1 permits a gateway to advertise multiple KDF+AEAD pairs in the symmetric algorithms section of a KeyConfig (`symLen > 4`). The current `OhttpKeyConfig.parse()` at `lib/src/ohttp.dart:60-78` reads `symLen`, performs a bounds check, then consumes exactly one 4-byte (`kdfId`, `aeadId`) pair at lines 67–71 and returns, silently dropping any trailing pairs. A real-world gateway that advertises two suites — for example, an unsupported suite first followed by the supported suite — will cause the parser to lock onto the first pair and either fail validation or succeed in selecting a suite that the library does not actually implement.

The companion mismatch: `parse()` throws `FormatException('Unsupported KEM: ...')` at line 51, while `validate()` throws `UnsupportedError` at lines 82–85 for the same conceptual condition. Two unrelated exception hierarchies for the same error class force callers to write two `catch` blocks or fall back to broad `catch (e)`.

For a wallet, both bugs silently degrade interoperability with any gateway operator who lists multiple suites or uses an unsupported KEM, and prevent meaningful error handling.

## Technical Details

**1. Rework `OhttpKeyConfig.parse` multi-pair handling** (`lib/src/ohttp.dart:60-78`)

Inspect every KDF+AEAD pair in the symmetric algorithms section. Either:

(a) Select the first pair that matches the library's single fixed supported suite (`kdfId == 0x0001`, `aeadId == 0x0001`), iterating through the section until found.

(b) Require an exact-single-pair config and raise a library-owned typed exception when the section length implies multiple pairs that the client cannot negotiate.

Pick one and document the chosen policy in the function doc comment. Cite RFC 9458 §4.1. Provide a typed exception so callers can distinguish "gateway advertises no supported suite" from other parse failures.

Also reject `symLen` values that are not a multiple of 4 (malformed wire format) with a clearly typed exception.

**2. Unify the unsupported-suite exception type** (`lib/src/ohttp.dart:51, 82-85`)

Introduce a single library-owned exception, e.g., `OhttpUnsupportedSuiteException`, and throw it from both `parse()` (currently `FormatException` at line 51) and `validate()` (currently `UnsupportedError` at lines 82–85) for unsupported KEM/KDF/AEAD. Re-export from `lib/ohttp_dart.dart`. If a broader exception hierarchy is being built in parallel, this exception slots into that hierarchy.

**3. Tests** (`test/ohttp_test.dart`, after the existing `OhttpKeyConfig.parse` group around line 62)

Construct synthetic KeyConfig byte buffers in test fixtures (documented byte arrays so the wire format is explicit in the test). Cover:

- Single supported pair (current happy path).
- Two pairs where the supported pair is at index 0.
- Two pairs where the supported pair is at index 1.
- Two/three pairs where no supported suite is present (expect typed exception per the chosen policy).
- Malformed `symLen` not divisible by 4 (typed exception).
- Unknown KEM via `parse()` and via `validate()` — both must throw the unified exception type.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- `OhttpKeyConfig.parse()` no longer ignores trailing bytes when `symLen > 4`; it either iterates pairs and selects a supported one, or throws a typed library exception per the documented policy.
- Doc comment on `parse()` cites RFC 9458 §4.1 and documents the chosen negotiation policy.
- Malformed `symLen` (not divisible by 4) is rejected with a clearly typed exception.
- A single library-owned exception type (e.g., `OhttpUnsupportedSuiteException`) is thrown from both `parse()` and `validate()` for unsupported KEM/KDF/AEAD; the type is re-exported from `lib/ohttp_dart.dart`.
- Unit tests cover: single supported pair, supported pair at index 0, supported pair at index 1, no supported pair (typed exception), malformed `symLen` (typed exception), unsupported KEM through both `parse()` and `validate()` paths.

## Additional

- Tests: unit tests covering all wire-format cases enumerated above.
- Migration note in the change description: callers previously catching `FormatException` or `UnsupportedError` from these paths must be updated.
