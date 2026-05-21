# Task 19: Test OHTTP AEAD authentication failure surfaces correctly

**Severity:** HIGH
**Vector:** Test suite
**Files:** `test/ohttp_test.dart` (after line 163)

**Evidence:** Cross-ref T-O3. Line 163 closes the second `ohttpDecapsulate` test. Lines 217–225 of `lib/src/ohttp.dart` decrypt the response with `aesGcm.decrypt(...)`, which throws `SecretBoxAuthenticationError` on authentication failure (see task 07).

## Description

The test suite never exercises the path where the response ciphertext is tampered with after the gateway returns it. As a result, the behaviour of `ohttpDecapsulate` under that condition is unspecified by tests and any change in exception surface (intentional or accidental) lands without notice. A targeted test is required regardless of whether task 07 has shipped — both before and after the wrapper exception is introduced.

## Proposed change

Add a test that produces a valid encap, derives the response secret as in task 18, encrypts a response, then flips one byte in the ciphertext before calling `ohttpDecapsulate`. Assert that decap throws — initially the underlying cryptography type, and after task 07 lands, the library-owned wrapper type. The test should be authored so it pins the current behaviour and updates cleanly when task 07 changes the exception type.

## Acceptance criteria

- New test placed after line 163 in `test/ohttp_test.dart`.
- Test tampers with one byte of the response ciphertext.
- Assert that decap throws an exception (the precise type depends on task 07 status — note this in the test).
- Test name references RFC 9458 §4.4 and AEAD authentication semantics.
