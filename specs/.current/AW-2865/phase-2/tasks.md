# Phase 2: Read BHTTP layer and assess parser robustness

**Goal:** Audit `lib/src/bhttp.dart` against RFC 9292 to identify length-guard gaps and malformed-input risks.

## Context

**Feature motivation (from idea):** Investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The goal is not to implement improvements but to identify risks and form subsequent engineering tasks. Every finding must map to a specific file, line range, and DoS/malformed-input scenario or RFC section.

**Severity classification (from vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Technical approach (from vision §4):** `lib/src/bhttp.dart` is Layer 2 of the four-layer stack. It implements RFC 9292 Known-Length framing — QUIC varints for lengths, `serializeRequest` and `parseResponse` as the two public entry points. It has no crypto. It is used both for serializing the inner HTTP request before encapsulation, and for parsing the decrypted response body.

**Primary investigation concerns (from vision §4 scrutiny table):**
- No length guards against oversized frames
- No negative-case tests for malformed input

**Related types/entities (from vision §5.6 — `BhttpResponse`):**

| Field | Investigation concern |
|---|---|
| `statusCode` | No range validation after varint decode (valid HTTP status is 100–599) |
| `headers` | No max header-count or total-size guard |
| `body` | `sublist(offset, offset + contentLen)` throws `RangeError` (not `FormatException`) on truncated input |

**Cross-cutting severity (from vision §5 — data model concerns):**

| Concern | Severity |
|---|---|
| No size caps on response body / headers | HIGH |
| `RangeError` instead of `FormatException` on truncated BHTTP body | HIGH |

**Data flow relevant to this phase (from vision §4 / §6.1):**
```
[inner HTTP request]
  → bhttp.serializeRequest()        # RFC 9292 Known-Length framing
  ...
  → bhttp.parseResponse(binaryResponse)
         → varint statusCode (no range check)
         → header loop: no total-size guard
         → body sublist: RangeError on truncation (not FormatException)
     [CONCERN] No max-header-count or max-body-size guard
```

**Happy path concern from vision §6.1:**
> `[CONCERN] No upper-bound on body size before serialization`

## Tasks

- [ ] 2.1 Read the full file (use `ast-index outline` first).
- [ ] 2.2 Verify that the framing indicator byte (`0x00` for request, `0x01` for response) is validated; confirm behavior on unexpected indicator values.
- [ ] 2.3 Trace `parseResponse`: check whether `statusCode` is range-validated after varint decode (valid HTTP status is 100–599).
- [ ] 2.4 Trace `parseResponse` header loop: confirm there is no max-header-count guard and no total-size cap.
- [ ] 2.5 Trace the `body` sublist extraction: confirm it raises `RangeError` (not `FormatException`) on truncated input.
- [ ] 2.6 Verify varint decode (`decodeVarint`) handles all four QUIC length prefixes (1/2/4/8 bytes) and rejects buffers shorter than the declared length.
- [ ] 2.7 Confirm `serializeRequest` has no upper-bound check on body size before writing to the buffer.
- [ ] 2.8 Record each finding as a draft task entry (file, line range, RFC section or DoS scenario, severity).

## Acceptance Criteria

**Test:** No code is changed. Verification is: each finding cites a specific line in `bhttp.dart` and a concrete malformed-input scenario (e.g., "truncated body at line 87 raises `RangeError` instead of `FormatException`").

## Dependencies

- Phase 1 complete (HPKE layer audit)

## Technical Details

**RFC 9292 Known-Length framing overview:**

A Known-Length request begins with a framing indicator byte (`0x00`), followed by:
- Request control data (method length varint + method bytes, scheme length + bytes, authority length + bytes, path length + bytes)
- Header section: zero or more `(name-length varint, name-bytes, value-length varint, value-bytes)` pairs, terminated by a `0x00` varint
- Content: content-length varint + body bytes
- Trailer section: same structure as headers, terminated by `0x00`

A Known-Length response begins with framing indicator `0x01`, followed by:
- Status code as varint
- Headers (same pair structure, terminated by `0x00`)
- Content: content-length varint + body bytes

**QUIC varint encoding (RFC 9000 §16):**
The most-significant two bits of the first byte encode the total byte length:
- `00xxxxxx` — 1 byte (max 63)
- `01xxxxxx xxxxxxxx` — 2 bytes (max 16383)
- `10xxxxxx xxxxxxxx xxxxxxxx xxxxxxxx` — 4 bytes (max 1073741823)
- `11xxxxxx ... (7 more bytes)` — 8 bytes (max 2^62 − 1)

A compliant decoder must reject a buffer whose remaining bytes are fewer than the declared length.

**Key risk scenarios to verify:**
1. Framing indicator not in `{0x00, 0x01}` — what does `parseResponse` do?
2. Status code varint decodes to 0 or 999 — is range `100–599` enforced?
3. Header count or total header bytes unbounded — memory exhaustion from adversarial gateway response.
4. `contentLen` varint larger than remaining buffer — `RangeError` vs `FormatException`.
5. `serializeRequest` with multi-megabyte body — no size guard before buffer allocation.

## Implementation Notes

This is a read-only audit iteration. Do not edit any file in `lib/` or `test/`. Record all findings as draft task entries in this file or as separate notes for use in Iteration 7 (compile into per-task Markdown files).

Phase 1 completed with findings in `phase-1/tasks.md`; reference those HPKE-layer findings when noting any cross-layer concerns (e.g., bhttp concerns that compound HPKE/OHTTP risks).
