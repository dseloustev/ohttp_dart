# Task 20: Test `OhttpKeyConfig.parse` with multiple KDF+AEAD pairs

**Severity:** HIGH
**Vector:** Test suite
**Files:** `test/ohttp_test.dart` (after line 62)

**Evidence:** Cross-refs T-O1, T-O2. Line 62 is blank; line 63 closes the `OhttpKeyConfig.parse` group. Lines 60–78 of `lib/src/ohttp.dart` read only the first KDF+AEAD pair (see task 06). No existing test feeds a KeyConfig with `symLen > 4`.

## Description

Without a multi-pair test, the silent-drop bug described in task 06 is invisible to the test suite. A regression that "fixes" the parser to iterate pairs in the wrong order (or that re-introduces the drop) would land undetected. The test must cover the three meaningful inputs: supported suite first, supported suite second, no supported suite present.

## Proposed change

Add unit tests that construct synthetic KeyConfig byte buffers containing two and three KDF+AEAD pairs in the symmetric algorithms section. Cover: supported suite at index 0 (current happy path); supported suite at index 1; no supported suite (expect typed exception per task 06 policy). Test inputs should be documented byte arrays so the wire format is explicit in the test.

## Acceptance criteria

- New tests placed after line 62 in `test/ohttp_test.dart`.
- Cover at least the three cases enumerated above.
- Assert the correct selected suite (or the correct typed exception) per task 06's resolved policy.
- Test name references RFC 9458 §4.1.
