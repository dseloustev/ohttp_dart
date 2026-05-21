# Task 14: Test HPKE sequence-number overflow in `HpkeSenderContext.seal`

**Severity:** HIGH
**Vector:** Test suite
**Files:** `test/hpke_test.dart` (after line 200)

**Evidence:** Cross-ref T-H1. `HpkeSenderContext.seal` computes the per-call nonce by XOR-ing `_seq` into `baseNonce`. RFC 9180 §5.2 caps the sequence number at `2^(8*Nn) - 1`; on overflow, the implementation must refuse to seal. The existing `seal seq=0` and `seal seq=1` tests end around line 200; no overflow test exists.

## Description

OHTTP itself only seals once per context, so overflow is not reached during normal flow. But the counter machinery exists in `HpkeSenderContext` and could be reused in future paths. A missing overflow test means a regression that allows nonce reuse after overflow could land silently — and AES-GCM nonce reuse is catastrophic. A targeted unit test pins the current behaviour and protects future maintainers.

## Proposed change

Add a unit test that forces `_seq` to the maximum allowed value, calls `seal` once successfully, then calls `seal` again and asserts that the call fails with a clearly-typed exception (per RFC 9180 §5.2). If the current implementation does not enforce the cap, the task expands to include the implementation fix.

## Acceptance criteria

- New test placed after line 200 in `test/hpke_test.dart`.
- Test exercises the boundary at `_seq == 2^(8*Nn) - 1` and the overflow case at `_seq == 2^(8*Nn)`.
- If the current code lacks the cap, the test fails until the implementation is updated; the implementation fix is in scope of this task.
- Test name references RFC 9180 §5.2.
