# Task 11: Convert BHTTP `RangeError` to `FormatException` via shared bounds helper

**Severity:** HIGH
**Vector:** Parser robustness
**Files:** `lib/src/bhttp.dart:44-67, 173, 178, 187`

**Evidence:** Cross-refs F2.5a, F2.5b, F2.6a. Lines 44–67 are `decodeVarint`; `data[offset]` at line 45 and `data[offset + 1..4]` at lines 51/54/61 throw `RangeError` on truncated input. Lines 173, 178, and 187 call `data.sublist(...)` for header name, header value, and content respectively and also raise `RangeError`. `RangeError` is an `Error`, not an `Exception`. RFC 9292 §3.5.4 governs the framing.

## Description

A caller that wraps BHTTP parsing in `try { ... } on FormatException { ... }` — the natural Dart idiom for "the bytes did not match the expected format" — will not catch `RangeError` and the program will crash. Truncated or malformed input from a malicious or buggy gateway therefore turns into an unhandled error rather than a graceful parse failure. The condition is identical to a `FormatException` in intent; only the exception hierarchy is wrong.

## Proposed change

Introduce a private bounds-checking helper (e.g., `_requireBytes(data, offset, n)`) that raises a `FormatException` with a clear message when `offset + n > data.length`. Apply it before every indexed read and `sublist` call in `decodeVarint` and in the response/request parsing paths. The change is mechanical and contained; no API change is needed beyond replacing `RangeError` with `FormatException` for these specific call sites.

## Acceptance criteria

- A shared helper performs the bounds check and throws `FormatException` with a descriptive message.
- All `data[offset]`, `data[offset + i]`, and `data.sublist(...)` operations inside the parser go through the helper or have an equivalent guard.
- `try { ... } on FormatException { ... }` correctly catches every truncation scenario.
- Unit tests cover truncated input at each of the cited offsets (varint single-byte boundary, varint 2/4/8-byte boundary, header name, header value, content).
- See task 17 for the companion truncated-body test.
