# Task 17: Test BHTTP `parseResponse` with truncated body

**Severity:** HIGH
**Vector:** Test suite
**Files:** `test/bhttp_test.dart` (after line 185)

**Evidence:** Cross-ref T-B2. The existing `parseResponse` group closes around line 186. Lines 173, 178, and 187 of `lib/src/bhttp.dart` perform `data.sublist(...)` calls that throw `RangeError` on truncated input (see task 11). No test currently exercises truncation.

## Description

A test that drives truncated input through `parseResponse` is the natural companion to task 11 (converting `RangeError` to `FormatException`). It must be added so that the cited truncation surfaces correctly. Without the test, a regression that re-introduces unguarded `sublist` calls — or that fails to wrap a future code path with the bounds helper — would land silently.

## Proposed change

Add unit tests covering truncation at each of the documented offsets: header name length, header value length, and content length. Assert that the exception thrown is `FormatException` once task 11 lands (the test should be authored such that it fails on the current `RangeError` and passes after the fix, or the two tasks must be applied together).

## Acceptance criteria

- New tests placed after line 185 in `test/bhttp_test.dart`.
- Cover at least three truncation positions: header name, header value, content.
- Assert exception is `FormatException` with a useful message.
- Cross-reference task 11 in the test file's comments.
