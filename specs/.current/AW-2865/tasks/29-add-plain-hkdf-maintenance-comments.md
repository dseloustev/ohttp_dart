# Task 29: Add plain-HKDF maintenance-hazard comments in `ohttpDecapsulate`

**Severity:** IMPROVEMENT
**Vector:** Documentation
**Files:** `lib/src/ohttp.dart:192-207`

**Evidence:** Cross-ref TASK-O4. Lines 192–207 are the plain-HKDF block in `ohttpDecapsulate`. Line 192: `// prk = HKDF-Extract(salt, secret) — plain HKDF, not labeled`. Line 193: `final prk = await HpkeSender.hkdfExtract(...)`. Lines 195–200: `aeadKey` derivation. Lines 202–207: `aeadNonce` derivation. The HPKE key schedule elsewhere uses `_labeledExtract` / `_labeledExpand`.

## Description

The response decapsulation path intentionally uses plain (unlabeled) RFC 5869 HKDF — distinct from the labeled variants used by the HPKE key schedule. A future maintainer who notices the `HpkeSender.hkdfExtract` call and "harmonizes" it to the labeled form would silently derive different key material and break the response-channel decryption with no test failure (the labeled variant produces a valid result; it is just the wrong one). RFC 9458 §4.4 specifies plain HKDF here. The existing comment on line 192 hints at this but does not state the consequences.

## Proposed change

Expand the comment block to explicitly:
1. Cite RFC 9458 §4.4.
2. State the trap: replacing `HpkeSender.hkdfExtract`/`hkdfExpand` with the labeled variants will produce different key material and break decryption against any real gateway.
3. State that this is intentional and required by the RFC.
4. Cross-reference task 28 (empty-AAD comment) as the companion maintenance-hazard note.

## Acceptance criteria

- Comment block at lines 192, 195, 202 (or a single block at the top of the function) is expanded to at least the four points above.
- A maintainer skimming the function cannot accidentally "harmonize" to labeled HKDF without first dismissing the warning.
