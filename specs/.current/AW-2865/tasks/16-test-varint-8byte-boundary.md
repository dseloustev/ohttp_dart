# Task 16: Test QUIC varint encode/decode at the 8-byte boundary

**Severity:** HIGH
**Vector:** Test suite
**Files:** `test/bhttp_test.dart` (after line 68)

**Evidence:** Cross-ref T-B1. The existing varint round-trip group closes at line 68 of `test/bhttp_test.dart`. RFC 9000 §16 (referenced by RFC 9292 §3.5.4) defines varint encoding boundaries at 1, 2, 4, and 8 bytes; the existing tests cover the lower boundaries but the 8-byte boundary case (values requiring the 8-byte encoding form) is not exercised.

## Description

The 8-byte varint form is rare in BHTTP request/response framing in practice, but content-length fields can legitimately reach into the multi-gigabyte range that requires the 8-byte form. A bug in the 8-byte encode/decode path would silently corrupt large response framing. A boundary round-trip test pins this behaviour cheaply.

## Proposed change

Add a unit test that round-trips three carefully chosen values through `encodeVarint` and `decodeVarint`: the largest 4-byte-encodable value, the smallest 8-byte-encodable value, and the largest 8-byte-encodable value (`2^62 - 1` per RFC 9000). Assert exact byte-level equality of the encoded form and exact equality after decode.

## Acceptance criteria

- New test placed after line 68 in `test/bhttp_test.dart`.
- Covers the three boundary values above.
- Asserts both byte-level encoding and decoded-value equality.
- Test name references RFC 9000 §16.
