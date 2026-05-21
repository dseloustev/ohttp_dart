# Task 06: Fix silent drop of extra KDF+AEAD pairs in `OhttpKeyConfig.parse`

**Severity:** HIGH
**Vector:** Cryptographic correctness
**Files:** `lib/src/ohttp.dart:60-78`

**Evidence:** Cross-refs TASK-O1, T-O1. RFC 9458 §4.1 permits a gateway to advertise multiple KDF+AEAD pairs in the symmetric algorithms section (`symLen > 4`). At lines 60–78, `parse()` reads `symLen`, performs a bounds check, then consumes exactly one 4-byte (`kdfId`, `aeadId`) pair starting at lines 67–71 and returns; lines 72–78 build the `OhttpKeyConfig` without inspecting the remainder.

## Description

A real-world gateway that advertises two suites — for example, an unsupported suite first followed by the supported suite — will cause the parser to lock onto the first pair and either fail validation (if the first suite is unsupported) or, worse, succeed in selecting a suite that the library does not actually implement. Either outcome is wrong: the parser is selecting blindly instead of negotiating. For a wallet, the failure mode silently degrades interoperability with any gateway operator who lists multiple suites.

## Proposed change

Rework `OhttpKeyConfig.parse` so it inspects every KDF+AEAD pair in the symmetric algorithms section. Either (a) select the first pair that matches the library's single fixed supported suite, or (b) require an exact-single-pair config and raise a clearly typed library exception when the section length implies multiple pairs that the client cannot negotiate. Document the chosen policy in the function doc comment. Provide a typed exception so callers can distinguish "gateway advertises no supported suite" from other parse failures.

## Acceptance criteria

- `parse()` no longer ignores trailing bytes when `symLen > 4`; it either iterates pairs or throws a typed exception.
- Doc comment cites RFC 9458 §4.1 and documents the chosen negotiation policy.
- Unit test covers: single supported pair (current behaviour), two pairs where the supported pair is second, two pairs where neither is supported (typed exception), malformed `symLen` not divisible by 4 (typed exception).
