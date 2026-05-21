# Expand BHTTP test coverage (varint, truncation, framing)

**Estimate:** 1d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

`test/bhttp_test.dart` covers QUIC varint round-trips at the 1/2/4 byte boundaries and a small set of `parseResponse` happy paths, but it leaves three meaningful gaps. Each is a low-effort, high-value test that pins behaviour against malicious or malformed inputs from a gateway.

## Technical Details

**1. QUIC varint 8-byte boundary round-trip** (`test/bhttp_test.dart`, after line 68)

The existing varint round-trip group closes at line 68. RFC 9000 §16 (referenced by RFC 9292 §3.5.4) defines varint encoding boundaries at 1, 2, 4, and 8 bytes; the existing tests cover the lower boundaries but the 8-byte boundary case (values requiring the 8-byte encoding form) is not exercised. The 8-byte form is rare in BHTTP framing in practice, but content-length fields can legitimately reach into the multi-gigabyte range that requires it. A bug in the 8-byte encode/decode path would silently corrupt large response framing.

Add a test that round-trips three carefully chosen values through `encodeVarint` and `decodeVarint`:
- The largest 4-byte-encodable value.
- The smallest 8-byte-encodable value.
- The largest 8-byte-encodable value (`2^62 - 1` per RFC 9000 §16).

Assert both byte-level equality of the encoded form and exact equality after decode. Test name references RFC 9000 §16.

**2. `parseResponse` truncated body** (`test/bhttp_test.dart`, after line 185)

The existing `parseResponse` group closes around line 186. Lines 173, 178, and 187 of `lib/src/bhttp.dart` perform `data.sublist(...)` calls that throw `RangeError` on truncated input today (and `FormatException` once the parser is hardened — see the BHTTP parser-hardening task for the helper that produces the `FormatException`).

Add unit tests covering truncation at each of the documented offsets:
- Header name length.
- Header value length.
- Content length.

Assert that the exception thrown is `FormatException` with a useful message. Authored such that the tests pin the current behaviour and pass after the parser-hardening change lands (or land together with it).

**3. Unknown framing indicator** (`test/bhttp_test.dart`)

RFC 9292 §3.3 defines framing indicators `0` (Known-Length Request), `1` (Known-Length Response), `2` (Indeterminate-Length Request), and `3` (Indeterminate-Length Response). The library supports only `0` and `1`. The existing tests cover the immediate-neighbour case (`rejects non-response framing`) but do not exhaustively test all unsupported indicator values.

Add unit tests covering:
- Indicator `2` (indeterminate-length request).
- Indicator `3` (indeterminate-length response).
- Indicator `4` (out-of-range valid varint).
- A large varint value.

Assert each produces a `FormatException` with a message that includes the offending indicator. Tests reference RFC 9292 §3.3.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- Varint 8-byte boundary round-trip test added after line 68 in `test/bhttp_test.dart`; covers the three boundary values; asserts byte-level encoding equality and decoded-value equality; references RFC 9000 §16.
- Truncated-body tests added after line 185; cover header-name, header-value, and content truncation positions; assert `FormatException` with a useful message.
- Unknown-framing-indicator tests cover indicators `2`, `3`, `4`, and a large varint value; each asserts `FormatException` with a message naming the offending indicator; references RFC 9292 §3.3.

## Additional

- Tests: pure unit tests in `test/bhttp_test.dart`.
- The truncation tests rely on the BHTTP parser hardening producing `FormatException` rather than `RangeError`; if that work has not landed, the tests should be authored against the current `RangeError` and updated when the hardening lands.
