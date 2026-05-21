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

- [x] 2.1 Read the full file (use `ast-index outline` first).
- [x] 2.2 Verify that the framing indicator byte (`0x00` for request, `0x01` for response) is validated; confirm behavior on unexpected indicator values.
  - **Finding (file/line):** `lib/src/bhttp.dart:140-146` — `parseResponse` decodes the framing indicator via `decodeVarint` and explicitly rejects values other than `1` with `FormatException('Expected known-length response (framing=1), got $framing')`.
  - **Behavior on unexpected values:** any varint-decodable first byte that is not exactly `1` (e.g., `0x00` request indicator, `0x02` known-length unused, `0x03` indeterminate-length response) raises `FormatException` — the intended error type. Indeterminate-length response (`0x03`) is *not* supported.
  - **Asymmetry vs. serializer:** `serializeRequest` (line 85) writes the indicator as `encodeVarint(0)` unconditionally; there is no reciprocal `parseRequest` entry point, so request-side framing validation is N/A for this library (sender-only).
  - **Edge case — empty buffer:** `decodeVarint(data, 0)` on an empty `Uint8List` throws `RangeError` (not `FormatException`) before the framing check at line 142 can run. This is a separate concern tracked under task 2.5 (truncated-input error type).
  - **RFC mapping:** RFC 9292 §3.2 defines framing indicators `0` (known-length request), `1` (known-length response), `2` (indeterminate-length request), `3` (indeterminate-length response). Only `1` is accepted on the parse path; this matches the library's documented Known-Length-only scope (`bhttp.dart:4-8` doc comment).
  - **Severity:** `IMPROVEMENT` — the framing check itself is correct; the only gap is that an empty/truncated input surfaces as `RangeError` rather than `FormatException`, which is covered by the broader truncation concern (HIGH) in task 2.5.
- [x] 2.3 Trace `parseResponse`: check whether `statusCode` is range-validated after varint decode (valid HTTP status is 100–599).
  - **Finding (file/line):** `lib/src/bhttp.dart:148-162` — `statusCode` is decoded as a QUIC varint and the only check applied is `statusCode >= 100 && statusCode < 200` to detect informational (1xx) responses for skip-loop purposes. No upper bound (`< 600`) and no lower bound (`>= 100`) is asserted on the final non-1xx value before it is stored on `BhttpResponse`.
  - **Concrete malformed-input scenarios accepted silently:**
    - `statusCode == 0` — passes the 1xx-skip branch (0 is not in `[100, 200)`), falls through to `break`, and is returned as-is.
    - `statusCode == 999`, `statusCode == 2^62 - 1` — same path; any non-1xx varint value is returned without validation, including values that exceed the HTTP status range `100–599` (RFC 9110 §15) and values that exceed Dart's safe-int representable range on JS-compiled targets.
    - Negative values are not representable (varints are unsigned), so the only direction of bogus input is "too large" or "below 100".
  - **Downstream impact:** `OhttpClient` and `parseResponse` callers receive a `BhttpResponse.statusCode` they cannot trust as a valid HTTP status without re-validating. Wallet code branching on `statusCode == 200` is safe, but any range-based logic (`statusCode >= 500`, `statusCode ~/ 100`) is not.
  - **RFC mapping:** RFC 9292 §3.5.2 states the status code is a varint but does not itself constrain the range; RFC 9110 §15 defines valid HTTP status codes as 100–599. The parser conforms to RFC 9292 but propagates out-of-range values without flagging them.
  - **Severity:** `IMPROVEMENT` — does not enable DoS or memory exhaustion; consumer-side concern. Worth a one-line guard `if (statusCode < 100 || statusCode > 599) throw FormatException(...)` after the 1xx skip-loop exits.
- [x] 2.4 Trace `parseResponse` header loop: confirm there is no max-header-count guard and no total-size cap.
  - **Finding (file/line):** `lib/src/bhttp.dart:164-182` — the header section is bounded only by the `headersLen` varint that prefixes it (`headersEnd = offset + headersLen`, line 167). The loop `while (offset < headersEnd)` will iterate until `headersEnd` is reached, with no cap on the number of `(name, value)` pairs appended to the `headers` list and no cap on `headersLen` itself.
  - **Concrete malformed-input scenarios:**
    - **Adversarial gateway sends `headersLen = 2^30 − 1` (max 4-byte varint, ~1 GiB)** — line 165-167 accepts the declared length without bound-checking against `data.length - offset`. The loop then runs against `data.sublist(...)` calls (lines 173, 178) which will eventually throw `RangeError` once the underlying buffer is exhausted, but only after attempting to allocate `Uint8List` views for each header name/value. On an 8-byte varint (`headersLen` up to `2^62 − 1`), the integer arithmetic `offset + headersLen` can overflow on JS-compiled targets.
    - **Many tiny headers** — `headersLen = 100_000_000`, each header `("a", "")` (~3 bytes overhead per pair) yields ~33M `(String, String)` records in the returned list. The `List<(String, String)>` grows unboundedly with no cap on `headers.length` and no early-abort heuristic.
    - **Single oversized header value** — `nameLen = 1`, `valueLen = 2^30 − 1`. `utf8.decode(data.sublist(offset, offset + valueLen))` (line 178) allocates a contiguous `Uint8List` of declared size and a `String` of equal character count; both succeed if the gateway actually supplies that many bytes, exhausting heap.
    - **`headersLen` exceeds remaining buffer** — `headersEnd = offset + headersLen` can point past `data.length`; the loop will run until a `sublist` call at line 173 or 178 throws `RangeError` (not `FormatException`) — same error-type gap as task 2.5.
  - **DoS vector:** an OHTTP relay-to-gateway path is end-to-end encrypted, so this only fires against a *compromised or malicious* gateway. However, since the wallet trusts the gateway only for transport (not for content), a malicious gateway sending an oversized header section can OOM the wallet process before the inner request response is parsed.
  - **RFC mapping:** RFC 9292 §3.5.3 defines the header section as a length-prefixed sequence of field-line pairs and explicitly does not impose limits; RFC 9113 §10.5.1 (HTTP/2) and common HTTP/1.1 implementations cap total header size at 8–64 KiB. `parseResponse` imposes no equivalent cap.
  - **Severity:** `HIGH` — matches the vision §5 cross-cutting severity ("No size caps on response body / headers"). Mitigation: introduce a constant (e.g., `_kMaxHeaderSectionBytes = 64 * 1024`) and reject `headersLen` exceeding it before allocating; additionally cap `headers.length` (e.g., 256 entries).
- [x] 2.5 Trace the `body` sublist extraction: confirm it raises `RangeError` (not `FormatException`) on truncated input.
  - **Finding (file/line):** `lib/src/bhttp.dart:184-187` — `contentLen` is decoded at line 185, then `data.sublist(offset, offset + contentLen)` is invoked at line 187 with no prior check that `offset + contentLen <= data.length`. `Uint8List.sublist` (via `ListBase.sublist`) throws `RangeError` when `end > length`, not `FormatException`.
  - **Concrete malformed-input scenario:** gateway returns a truncated BHTTP response where the body content is cut off after `contentLen` is declared. Example: header declares `contentLen = 1024`, but only 512 bytes of body follow. `data.sublist(offset, offset + 1024)` throws `RangeError (end): Invalid value: Not in inclusive range 0..N`.
  - **Same gap applies to:**
    - `utf8.decode(data.sublist(offset, offset + nameLen))` (line 173) and `(offset, offset + valueLen)` (line 178) in the header loop.
    - `decodeVarint` itself (lines 45, 51, 54, 61) when `offset + N > data.length` — also raises `RangeError`, not `FormatException`.
    - The framing-indicator varint at line 140 on an empty buffer (noted under task 2.2).
  - **API-contract impact:** callers that wrap `parseResponse` in `try { ... } on FormatException catch (...)` will *not* catch truncation; the `RangeError` propagates as an `Error` (not an `Exception`), which conventionally indicates a programmer bug, not a recoverable malformed-input condition. In wallet code this can crash an isolate or be misclassified as an internal error rather than a network/peer fault.
  - **RFC mapping:** RFC 9292 §3.5.4 defines content as `Content Length (varint) || Content (bytes)`; the parser must be robust to receiving fewer bytes than declared (malicious or truncated transport). The current code conflates "transport truncation" with "internal programmer error".
  - **Severity:** `HIGH` — matches the vision §5 cross-cutting severity ("`RangeError` instead of `FormatException` on truncated BHTTP body"). Mitigation: introduce a helper `_safeSublist(data, start, end)` that throws `FormatException('Truncated BHTTP response: declared $end bytes, have ${data.length}')` and route all `sublist` calls and varint decodes through bound checks.
- [x] 2.6 Verify varint decode (`decodeVarint`) handles all four QUIC length prefixes (1/2/4/8 bytes) and rejects buffers shorter than the declared length.
  - **Finding (file/line):** `lib/src/bhttp.dart:44-67` — all four QUIC varint prefixes are recognized via the `first >> 6` switch:
    - prefix `0` → 1-byte varint, value range `[0, 63]` (line 48-49)
    - prefix `1` → 2-byte varint, value range `[0, 16383]` (line 50-51)
    - prefix `2` → 4-byte varint, value range `[0, 2^30 − 1]` (line 52-56)
    - prefix `3` → 8-byte varint, value range `[0, 2^62 − 1]` (line 57-63), implemented as an unrolled 7-iteration shift loop.
  - **Length-handling correctness:** the prefix-to-length mapping matches RFC 9000 §16 exactly. The encoder (`encodeVarint`, lines 15-40) and decoder are symmetric.
  - **Buffer-underrun handling:** `data[offset + N]` accesses at lines 51, 54, 61 throw `RangeError` (not `FormatException`) when the buffer is shorter than the declared varint length. There is **no explicit length check** before indexing. Example: a 1-byte buffer `0x80` (declares a 4-byte varint) raises `RangeError` on `data[offset + 1]`.
  - **Integer overflow on 8-byte varint:** on the Dart VM (64-bit ints), values up to `2^62 − 1` are representable. On dart2js / dart2wasm (53-bit safe integer range), an 8-byte varint with the high bits set silently overflows during the shift loop at line 61. The library is documented as "pure-Dart, no Flutter" but `CLAUDE.md` does not restrict the runtime to the VM; a JS deployment would be subtly broken for adversarial varints.
  - **Non-canonical encoding:** RFC 9000 §16 does not require canonical (minimum-length) encoding, so a value of `5` encoded as `0x4005` (2-byte form) is technically legal and is decoded correctly. No issue here, but worth noting that the encoder always produces the minimum-length form (lines 16-39 select strictly by value range).
  - **RFC mapping:** RFC 9000 §16 defines the four prefix lengths and their value ranges; the decoder conforms to the format but not to the recommended defensive practice of explicit bound checks before indexed access (RFC 8949 §3 patterns for length-prefixed decoders).
  - **Severity:** `HIGH` — feeds directly into the task 2.5 finding. Mitigation: add a one-line precondition at the start of each `case`, e.g., `if (offset + 4 > data.length) throw FormatException('Truncated varint')`, or extract a single `_requireBytes(data, offset, n)` helper.
- [x] 2.7 Confirm `serializeRequest` has no upper-bound check on body size before writing to the buffer.
  - **Finding (file/line):** `lib/src/bhttp.dart:74-111` — `serializeRequest` accepts a `Uint8List body` and a `Map<String, String> headers` without any size validation. The body is written verbatim at lines 104-105 (`buf.add(encodeVarint(body.length)); if (body.isNotEmpty) buf.add(body);`). Headers are accumulated into `headerBuf` at lines 94-99 with no per-header or aggregate length cap.
  - **Concrete scenarios:**
    - Wallet code passes a multi-megabyte body (e.g., an attached document or a large JSON payload). `BytesBuilder.toBytes()` at line 110 allocates a contiguous buffer of `~|body| + |headers| + overhead` bytes; for a 100 MiB body this attempts a single 100 MiB allocation. No streaming path exists.
    - The serialized output is then passed to `HpkeSenderContext.seal` (one-shot AEAD over the entire buffer) by `ohttp.dart`, doubling memory pressure during encryption.
    - The encoded length itself is written as a QUIC varint with a max of `2^62 − 1`, so the wire format imposes no ceiling — the only ceiling is process memory.
  - **Cross-layer impact:** the gateway likewise has no negotiated body-size cap (RFC 9458 does not define one); a wallet that does not self-limit body size can be tricked by application code into constructing an unbounded request. Combined with task 2.4 (no header cap), the serializer is a memory-amplification primitive.
  - **No mitigation present:** no constants, no asserts, no `ArgumentError` for oversized inputs.
  - **RFC mapping:** RFC 9292 §3.5.4 imposes no body-size limit (the varint extends to `2^62 − 1`); RFC 9458 §3 recommends but does not require gateways to enforce limits. The library's role as a *sender* means it should self-impose a sane cap (e.g., 4 MiB default, configurable).
  - **Severity:** `IMPROVEMENT` (sender-controlled) leaning `HIGH` for wallet use — matches vision §6.1 "No upper-bound on body size before serialization". Mitigation: introduce `_kMaxRequestBodyBytes` (e.g., 4 * 1024 * 1024) and an optional override parameter; throw `ArgumentError` when exceeded.
- [x] 2.8 Record each finding as a draft task entry (file, line range, RFC section or DoS scenario, severity).
  - **Draft task entries (compile into Iteration 7 per-task Markdown):**

    | # | File / Lines | Concern | RFC Section / Scenario | Severity |
    |---|---|---|---|---|
    | F2.3 | `lib/src/bhttp.dart:148-162` | `statusCode` not range-checked (100–599); accepts 0, 999, varint up to 2^62−1 | RFC 9110 §15 (status code range); RFC 9292 §3.5.2 silent on range | IMPROVEMENT |
    | F2.4a | `lib/src/bhttp.dart:164-182` | `headersLen` accepted without bound check vs. remaining buffer; no max-section-byte cap | RFC 9292 §3.5.3; DoS via 1 GiB declared header section | HIGH |
    | F2.4b | `lib/src/bhttp.dart:169-182` | No max-header-count cap; `headers` list grows unboundedly | RFC 9113 §10.5.1 analogue; DoS via 33M tiny header pairs | HIGH |
    | F2.4c | `lib/src/bhttp.dart:173,178` | Single header name/value `utf8.decode(sublist(...))` allocates declared size verbatim | RFC 9292 §3.5.3; DoS via single 1 GiB header value | HIGH |
    | F2.5a | `lib/src/bhttp.dart:187` | `data.sublist(offset, offset + contentLen)` raises `RangeError` (not `FormatException`) on truncated body | RFC 9292 §3.5.4; truncation scenario | HIGH |
    | F2.5b | `lib/src/bhttp.dart:173,178` | Header-loop `sublist` calls raise `RangeError` on truncation (same gap as body) | RFC 9292 §3.5.3 | HIGH |
    | F2.5c | `lib/src/bhttp.dart:140` | Empty input → `decodeVarint` on empty buffer raises `RangeError` before framing check | RFC 9292 §3.2 | IMPROVEMENT |
    | F2.6a | `lib/src/bhttp.dart:51,54,61` | Varint decode lacks pre-index bound checks; `RangeError` instead of `FormatException` on short buffer | RFC 9000 §16 | HIGH |
    | F2.6b | `lib/src/bhttp.dart:57-63` | 8-byte varint silently overflows on dart2js / dart2wasm (53-bit safe int) | RFC 9000 §16; JS-target runtime | IMPROVEMENT |
    | F2.7 | `lib/src/bhttp.dart:74-111` | `serializeRequest` has no body-size or header-size guard; unbounded `BytesBuilder.toBytes()` allocation | RFC 9292 §3.5.4 (no wire cap); RFC 9458 §3 (no negotiated cap); memory-amplification | IMPROVEMENT→HIGH for wallet |
  - **Cross-layer notes:**
    - F2.4* and F2.7 compound the Phase 1 HPKE concern that `HpkeSenderContext.seal` operates on the entire serialized buffer in one shot — oversized BHTTP inputs directly inflate AEAD plaintext size with no streaming option.
    - F2.5* and F2.6a together motivate a single shared helper (e.g., `_safeSublist` / `_requireBytes`) rather than per-call-site guards; this should be one engineering task in Iteration 7, not seven.
    - F2.6b is the only finding that depends on deployment target (JS); the wallet's Dart-VM-only deployment may reclassify it as N/A.
  - **Output for Iteration 7:** the table above maps 1:1 to per-task Markdown files in the `tasks/` directory; severities feed the BLOCKER/HIGH/IMPROVEMENT grouping in the final report (`vision §2`).

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
