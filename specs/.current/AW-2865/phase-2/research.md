# Phase 2 Research: BHTTP Layer Audit Against RFC 9292
# AW-2865 — ohttp_dart Investigation

**Phase scope:** `lib/src/bhttp.dart` only — RFC 9292 Known-Length framing.
No cryptographic operations are in scope. No file in `lib/` or `test/` was modified during this research.

---

## Phase Scope

This phase audits Layer 2 of the four-layer stack: `lib/src/bhttp.dart` (191 lines).
The file implements RFC 9292 Known-Length Binary HTTP framing with two public entry points:
`serializeRequest` (request → binary blob) and `parseResponse` (binary blob → `BhttpResponse`).
It also provides `encodeVarint` and `decodeVarint` (QUIC RFC 9000 §16 variable-length integers).

The audit covers:
- Framing indicator validation in `parseResponse` (task 2.2)
- Status code range validation in `parseResponse` (task 2.3)
- Header loop size guards in `parseResponse` (task 2.4)
- Body sublist truncation exception type (task 2.5)
- QUIC varint decoder correctness and truncation handling (task 2.6)
- Request body size upper bound in `serializeRequest` (task 2.7)

Out of scope for this phase: `hpke.dart` (Phase 1), `ohttp.dart` (Phase 3), `ohttp_client.dart` (Phase 5),
and the indeterminate-length framing variant of RFC 9292 (not implemented).

---

## Resolved Questions

| # | Question | Resolution |
|---|---|---|
| 1 | Framing indicator severity (HIGH vs. BLOCKER) | **HIGH**, with a reviewer note. Framing indicator IS validated (finding F-B1 is positive: guard present and correct). Severity classification for the absence of a guard in the hypothetical case is moot; the guard exists. |
| 2 | Body size guard ownership (BHTTP layer vs. `OhttpClient` layer) | Scope decision left open in each draft task entry. Both layers are candidates; the follow-up task identifies the appropriate layer. |
| 3 | Maximum expected BHTTP response body size for the wallet use case | Left open — the wallet team will supply the threshold during remediation. The draft task records the absence of a cap as HIGH regardless of the specific threshold. |
| 4 | Exception-type mismatch: consolidate vs. per-site tasks | Per-site task entries; each independently assignable. |
| 5 | Phase 1 cross-references | Phase 1 finding IDs cited from `specs/.current/AW-2865/phase-1/research.md` as `F-1` through `F-7`. |
| 6 | Varint truncation reference | Confirmed via direct code inspection of `bhttp.dart` lines 44–67 only. No external SDK reference used. |

---

## Related Modules / Services

| File | Role | Relationship to bhttp.dart |
|---|---|---|
| `lib/src/bhttp.dart` | RFC 9292 Known-Length framing | **Subject of this audit** |
| `lib/src/ohttp_client.dart` | High-level OHTTP client | Calls `bhttp.serializeRequest` (line 97) and `bhttp.parseResponse` (line 136) directly. Errors from `parseResponse` propagate unwrapped to `OhttpClient.send()` callers. |
| `lib/src/ohttp.dart` | RFC 9458 encap/decap | Does not call `bhttp.dart` directly. Consumes BHTTP output via `ohttp_client.dart`. |
| `lib/src/hpke.dart` | HPKE Base Mode Sender | No dependency on `bhttp.dart`. |
| `test/bhttp_test.dart` | BHTTP unit tests | Covers happy-path round-trips and one negative-case framing test. No truncation, overflow, or adversarial-input tests. |

---

## Current Endpoints & Contracts

`bhttp.dart` exposes four public symbols:

**`encodeVarint(int value) → Uint8List`** (lines 15–40)
Encodes a non-negative integer as a 1/2/4/8-byte QUIC varint.
No upper-bound guard on `value` before branching; the 8-byte branch (lines 29–39) will produce a result for any Dart `int`.

**`decodeVarint(Uint8List data, int offset) → (int, int)`** (lines 44–67)
Returns `(decodedValue, bytesConsumed)`. Does not validate that `data` is long enough to satisfy the width implied by the two-bit prefix of `data[offset]`. All four branches access `data` at indices `offset+1` through `offset+7` without length guards.

**`serializeRequest({method, scheme, authority, path, headers, body}) → Uint8List`** (lines 74–111)
Serializes a Known-Length request. Framing indicator `0x00` is written at line 85. No upper-bound check on `body.length` before writing to the `BytesBuilder`.

**`parseResponse(Uint8List data) → BhttpResponse`** (lines 136–190)
Parses a Known-Length response. Framing indicator check at lines 142–146. Status code loop at lines 150–163. Header loop at lines 165–182. Body extraction at line 187.

---

## Patterns Used

| Pattern | Location | Notes |
|---|---|---|
| `BytesBuilder` accumulation | `serializeRequest` lines 82–110 | Standard Dart pattern; grows without bound |
| QUIC two-bit prefix dispatch (`switch (prefix)`) | `decodeVarint` lines 47–66 | Correct prefix masking in each branch; no bounds-checking on subsequent index reads |
| `Uint8List.sublist` for field extraction | `parseResponse` lines 173, 178, 187 | Throws `RangeError` (not `FormatException`) on out-of-range indices |
| `while (offset < headersEnd)` loop | `parseResponse` lines 170–182 | Boundary from a single `headersLen` varint; no secondary cap |
| `while (true)` + break for 1xx skip | `parseResponse` lines 151–163 | Status code used as loop-exit condition; accepted range is only 100–199 for continue |
| `utf8.decode` for name/value | `parseResponse` lines 173–179 | Throws `FormatException` on invalid UTF-8; this is the only `FormatException` path in the header loop |
| `_writeField` helper | `serializeRequest` lines 113–116 | Writes length-prefixed field with `encodeVarint` + raw bytes |

---

## Per-Task Findings (2.2–2.7)

### Task 2.2 — Framing Indicator Validation

**File:** `lib/src/bhttp.dart` lines 140–146

```dart
final (framing, framingLen) = decodeVarint(data, offset);
offset += framingLen;
if (framing != 1) {
  throw FormatException(
    'Expected known-length response (framing=1), got $framing',
  );
}
```

**Verdict: guard present and correct.** Any byte that does not decode to the value `1` causes `parseResponse` to throw a typed `FormatException` with a descriptive message. This includes `0` (request frame), `2`, and any multi-byte varint. The exception is not caught inside `bhttp.dart` and propagates directly to `OhttpClient.send()` (line 136 of `ohttp_client.dart`), where it also propagates unwrapped to the caller of `send()`.

**Note for security reviewer:** The validation is correct. The remaining question (out of scope for code audit) is whether a `FormatException` from a higher network layer can be confused with this one at the `OhttpClient` API boundary. This is a caller-error-handling concern, not a BHTTP-layer defect.

**RFC 9292 reference:** §3.2, §3.4 (Known-Length Request/Response framing indicator values).

---

### Task 2.3 — Status Code Range Validation

**File:** `lib/src/bhttp.dart` lines 149–163

```dart
int statusCode;
int statusLen;
while (true) {
  (statusCode, statusLen) = decodeVarint(data, offset);
  offset += statusLen;

  if (statusCode >= 100 && statusCode < 200) {
    // Skip informational response header section
    final (hLen, hLenLen) = decodeVarint(data, offset);
    offset += hLenLen + hLen;
  } else {
    break;
  }
}
```

**Verdict: guard absent for the final accepted status code.** The loop correctly skips 1xx informational responses. However, after `break`, the value of `statusCode` is only guaranteed to be outside `[100, 200)`. It is not checked against the valid HTTP range `[100, 599]`. A gateway can return a varint encoding `0`, `600`, `999`, `1000`, or any value up to `2^62 - 1`; `parseResponse` accepts all of these and returns a `BhttpResponse` with that `statusCode` value. No exception is raised.

The value propagates verbatim to `OhttpClient.send()` → `OhttpResponse.statusCode`. A caller checking `response.statusCode == 200` would still work correctly; a caller iterating `statusCode` by range class (2xx, 4xx, etc.) would receive unexpected values silently.

**RFC 9292 reference:** §4.1 (Known-Length Response control data — status code as an integer in the HTTP status code range).

---

### Task 2.4 — Header Loop Size Guards

**File:** `lib/src/bhttp.dart` lines 165–182

```dart
final (headersLen, headersLenLen) = decodeVarint(data, offset);
offset += headersLenLen;
final headersEnd = offset + headersLen;

final headers = <(String, String)>[];
while (offset < headersEnd) {
  ...
  headers.add((name, value));
}
```

**Verdict: no max-header-count guard and no total-size cap.** The loop terminates when `offset` reaches `headersEnd = offset_before + headersLen`. The only bound is the varint value `headersLen`, which can be up to `2^62 - 1` (approximately 4.6 × 10^18 bytes). There is no:

1. Maximum iteration count (max header count).
2. Maximum `headersLen` value check before computing `headersEnd`.
3. Accumulated size limit independent of `headersLen`.

An adversarial gateway that sends a well-formed BHTTP blob with 10,000 headers (each small) would cause 10,000 `headers.add(...)` calls. A gateway that crafts a large `headersLen` varint pointing past the end of the actual buffer would cause `data[offset]` accesses beyond `data.length`, resulting in `RangeError` — not a clean `FormatException`.

There is no test exercising the header loop with an adversarial input. The `bhttp_test.dart` suite tests exactly one header (`'content-type': 'application/json'`).

**DoS scenario:** Adversarial or compromised OHTTP gateway returns a response with `headersLen` large enough to trigger repeated heap allocations via `headers.add(...)`. On mobile, this can exhaust available memory. The risk is bounded in practice by the network response size limit of the underlying `http.Client`, but `bhttp.dart` imposes no independent limit.

**RFC 9292 reference:** §3.2 (Known-Length header section structure); RFC does not define a required max-header-count or max-header-size; these are implementation-defined robustness decisions.

**Cross-reference Phase 1:** This finding combines with Phase 1 F-7 (no integration test for the full OHTTP round-trip). The absence of an adversarial-input test means this path has never been exercised end-to-end.

---

### Task 2.5 — Body Sublist Truncation Exception Type

**File:** `lib/src/bhttp.dart` line 187

```dart
final body = Uint8List.fromList(data.sublist(offset, offset + contentLen));
```

**Verdict: `RangeError` propagates unwrapped.** If `contentLen > data.length - offset` (i.e., the declared body length exceeds remaining bytes), `Uint8List.sublist` throws `RangeError` per the Dart core library specification for typed data lists.

There is no `try/catch` block around this call, inside either `parseResponse` or in the `OhttpClient.send()` call site (line 136 of `ohttp_client.dart`). The `RangeError` propagates unwrapped to the caller of `send()`.

This applies equally to the `sublist` calls within the header loop (lines 173, 178): if a header name or value length varint exceeds remaining bytes, those also throw unwrapped `RangeError`.

**Call-stack path for truncated body:**
```
adversarial gateway response
  → ohttpDecapsulate() returns binaryResponse
  → bhttp.parseResponse(binaryResponse)          // ohttp_client.dart:136
      → data.sublist(offset, offset + contentLen) // bhttp.dart:187
          → RangeError (thrown by Dart runtime)
  [propagates unwrapped to OhttpClient.send() caller]
```

**Impact:** Callers of `OhttpClient.send()` that catch `FormatException` to handle BHTTP parsing errors will not catch this `RangeError`. Callers that only catch `Exception` (the base class) will catch `RangeError` since `RangeError extends Error`, not `Exception` — meaning a generic `catch (e)` is required, which conflates BHTTP truncation with any other runtime error.

**RFC 9292 reference:** §3.2, §3.4 (content length field handling). The RFC requires compliant implementations to reject malformed input; the specific error type is implementation-defined.

---

### Task 2.6 — QUIC Varint Decoder Correctness and Truncation Handling

**File:** `lib/src/bhttp.dart` lines 44–67

Full decoder:

```dart
(int, int) decodeVarint(Uint8List data, int offset) {
  final first = data[offset];
  final prefix = first >> 6;
  switch (prefix) {
    case 0:
      return (first & 0x3F, 1);
    case 1:
      return (((first & 0x3F) << 8) | data[offset + 1], 2);
    case 2:
      return (
        ((first & 0x3F) << 24) | (data[offset + 1] << 16) | (data[offset + 2] << 8) | data[offset + 3],
        4,
      );
    case 3:
      var value = (first & 0x3F);
      for (var i = 1; i <= 7; i++) {
        value = (value << 8) | data[offset + i];
      }
      return (value, 8);
    default:
      throw StateError('unreachable');
  }
}
```

**Correctness verdict (per RFC 9000 §16):**

| Prefix bits | Width | Mask | Assembly | Verdict |
|---|---|---|---|---|
| `00` (case 0) | 1 byte | `first & 0x3F` | single byte, no additional reads | Correct |
| `01` (case 1) | 2 bytes | `(first & 0x3F) << 8 \| data[offset+1]` | big-endian, correct mask | Correct |
| `10` (case 2) | 4 bytes | `(first & 0x3F) << 24 \| ...` | big-endian, correct mask | Correct |
| `11` (case 3) | 8 bytes | iterative `(value << 8) \| data[offset+i]` for `i=1..7` | big-endian, correct mask | Correct |

The `switch` covers all four values of a 2-bit prefix (0–3). The `default: throw StateError('unreachable')` is dead code given a 2-bit prefix but is harmless.

**Truncation-guard verdict: no guard in any branch.** Every branch that reads beyond `data[offset]` does so without checking `data.length >= offset + width`:

- Case 1: reads `data[offset + 1]` — no check that `offset + 1 < data.length`.
- Case 2: reads `data[offset + 1..3]` — no check that `offset + 3 < data.length`.
- Case 3: reads `data[offset + 1..7]` — no check that `offset + 7 < data.length`.

All three truncation scenarios throw `RangeError` (Dart typed-data out-of-bounds). The error propagates unwrapped through all callers: the `parseResponse` field-length decodes, header name/value reads, status code read, and framing indicator read all funnel through `decodeVarint`.

Additionally, `decodeVarint` does not verify that the bytes remaining after the varint are sufficient to hold the declared field length. That responsibility is left to the caller; neither `parseResponse` nor `serializeRequest` performs this validation.

**Test coverage gap:** `bhttp_test.dart` tests the 1-byte, 2-byte, and 4-byte round-trips and an offset decode (lines 33–57 of `bhttp_test.dart`). There is no test for the 8-byte varint decode, and no negative test for a truncated buffer of any width.

**RFC 9000 §16 reference:** A compliant decoder must be able to indicate an error when the buffer is too short to read the declared varint width; RFC 9000 does not prescribe the error type, but the error must be distinguishable from a valid decode.

---

### Task 2.7 — Request Body Size Guard in `serializeRequest`

**File:** `lib/src/bhttp.dart` lines 103–105

```dart
// Content (known-length)
buf.add(encodeVarint(body.length));
if (body.isNotEmpty) buf.add(body);
```

**Verdict: no upper-bound guard present.** Before this point in `serializeRequest`, `body` is accepted as a `required Uint8List body` parameter with no constraint. `BytesBuilder.add` appends without limit. A caller supplying a 50 MB `body` Uint8List would produce a 50 MB + overhead BHTTP blob, which is then passed to `ohttpEncapsulate` → AES-128-GCM encryption → HTTP POST to the gateway — all in-memory, without any warning.

The absence of a guard is confirmed by reading the full `serializeRequest` function (lines 74–111): there is no assertion, length check, or early-return before `buf.add(body)`.

The same absence applies to header content: the `headers.entries` loop (lines 95–101) and the individual field writes in `_writeField` (lines 113–116) also impose no limit on combined header bytes.

**Concrete risk:** In a non-custodial crypto wallet, the inner HTTP request body is typically small (a JSON-encoded transaction payload, a few kilobytes at most). However, if a caller mistakenly passes a large blob — or if `targetAuthority` is misconfigured to point to a non-OHTTP-aware server that returns large responses — the BHTTP layer has no backstop. The limit must be enforced by the caller or by `OhttpClient` before calling `serializeRequest`.

**Cross-reference Phase 1:** Phase 1 F-2 (no size cap on `OhttpEncapsulateResult.encRequest`) is the direct downstream consequence of this gap: if `serializeRequest` produces a large blob, `ohttpEncapsulate` allocates `7 + 32 + plaintext_len + 16` bytes for `encRequest` with no cap.

**RFC 9292 reference:** RFC 9292 does not define a maximum message size. The constraint is implementation-defined and use-case-dependent.

---

## Limitations & Risks

| # | Description | Severity | RFC / DoS Scenario |
|---|---|---|---|
| R-1 | **Status code range not validated.** `parseResponse` accepts any varint value outside `[100, 200)` as a final status code. Values of `0`, `600`, `999`, or `2^62 - 1` are passed verbatim to `OhttpResponse.statusCode`. Callers that range-check the status code silently receive unexpected values. | HIGH | RFC 9292 §4.1 |
| R-2 | **Header loop has no max-count or max-size cap.** `headersLen` is a 62-bit varint with no secondary limit. An adversarial gateway that sends a response with a very large header section causes unbounded heap allocation on the client. On mobile, this can exhaust available memory before the buffer is exhausted. | HIGH | DoS: memory exhaustion from adversarial gateway |
| R-3 | **Body sublist throws `RangeError` on truncated input, not `FormatException`.** A `contentLen` varint that exceeds remaining bytes causes an unwrapped `RangeError` to propagate through `OhttpClient.send()`. Callers cannot distinguish BHTTP truncation from a logic error without catching `Error` (not `Exception`). | HIGH | RFC 9292 §3.4; call stack: `bhttp.dart:187` → `ohttp_client.dart:136` → caller |
| R-4 | **`decodeVarint` does not guard against short buffers.** Cases 1, 2, and 3 access `data[offset + 1]` through `data[offset + 7]` without verifying `data.length >= offset + width`. Short buffers throw unwrapped `RangeError`. This applies to every length-prefix decode in `parseResponse` — framing, status, headers, and body. | HIGH | RFC 9000 §16; same call-stack path as R-3 |
| R-5 | **`serializeRequest` has no upper-bound on body size.** Any `body` Uint8List is accepted and serialized. The resulting BHTTP blob feeds directly into HPKE encryption and network POST with no size limit. Multi-megabyte requests are produced silently. | HIGH | DoS: oversized in-memory allocation + oversized POST to gateway |
| R-6 | **No negative-case tests for truncated input or adversarial header counts.** `bhttp_test.dart` covers only the happy path (valid frames) and one framing-indicator rejection. No test exercises a truncated body, a truncated varint, an out-of-range status code, or a large header count. Any future refactoring of these code paths has no regression backstop. | HIGH | Test gap: silent regression risk |
| R-7 | **8-byte varint decode is untested.** Case 3 of `decodeVarint` handles values ≥ `0x40000000` (≥ 1 073 741 824). The test suite has no test for 8-byte varints. The implementation appears correct by inspection, but is unverified by any test vector. | IMPROVEMENT | RFC 9000 §16 |

---

## Cross-References to Phase 1 Findings

| Phase 2 Finding | Phase 1 Cross-Reference | Description |
|---|---|---|
| R-3 (RangeError on truncated body) | Phase 1 F-3 (`StateError` on HPKE overflow) | Both findings represent the same pattern: domain errors thrown as untyped Dart errors (`RangeError`, `StateError`) rather than typed exceptions, making catch-site handling unreliable across the stack. |
| R-5 (no body size cap in `serializeRequest`) | Phase 1 F-2 (no size cap on `OhttpEncapsulateResult.encRequest`) | A large body from `serializeRequest` becomes a large `binaryRequest` fed into `ohttpEncapsulate`, which allocates `7 + 32 + len + 16` bytes for `encRequest` (see `ohttp.dart` line roughly 150–160). No cap at either layer. |
| R-6 (no negative tests in `bhttp_test.dart`) | Phase 1 F-6 (no `_seq` overflow test), Phase 1 F-7 (no integration test for empty-AAD seal) | The test-gap pattern recurs across layers. Every layer has untested error paths. Iteration 7 should address all layers together in a single test-hardening task or a suite of layer-specific tasks. |

---

## New Technical Questions

These questions surfaced during the audit and are not resolved by the PRD or user responses above.

1. **`decodeVarint` integer overflow on 8-byte varints.** Case 3 (lines 57–63) accumulates the 62-bit value into a Dart `int` using `(value << 8) | data[offset + i]`. On the Dart native VM, `int` is 63-bit signed, so values up to `2^62 - 1` fit without overflow. On the Dart web/JS target, `int` is a 53-bit double, which would silently truncate values above `2^53`. Is `ohttp_dart` ever used in a web/WASM context? If yes, the 8-byte varint case could produce incorrect values silently. (Related to Phase 1 New Technical Question #3 about web/WASM target.)

2. **`parseResponse` informational-response loop termination.** The `while (true)` loop (lines 151–163) reads status codes until it finds one outside `[100, 200)`. There is no guard that limits the number of informational responses consumed. An adversarial gateway could send a stream of 1xx responses (each with an empty header section) before the final status, causing `decodeVarint` to be called many times. Is there an expected maximum number of 1xx responses to accept?

3. **`utf8.decode` exception type in header loop.** Lines 173 and 178 call `utf8.decode(data.sublist(...))`. Invalid UTF-8 in a header name or value throws `FormatException` (from `dart:convert`). This `FormatException` is distinct from the framing-indicator `FormatException` at line 143–145. At the `OhttpClient.send()` level, these are indistinguishable. Does the wallet integrator care about distinguishing UTF-8 decode errors from structural framing errors? If so, the error message is the only differentiator, and error message parsing is fragile.

4. **Trailer section in `parseResponse`.** RFC 9292 §3.4 defines a Known-Length response as: framing indicator + status code + headers + content + trailers. `parseResponse` reads framing, status, headers, and body — but does not consume or validate the trailer section. After line 187, any remaining bytes in `data` are silently ignored. Is this intentional? If a gateway sends a non-empty trailer section, it is silently discarded. The offset is not checked against `data.length` after the body sublist.

---

## Draft Task Entries (for Iteration 7)

Each entry follows the format established in Phase 1 research.

---

**TASK-B1: Add HTTP status code range validation in `parseResponse`**
- Severity: HIGH
- File: `bhttp.dart:149–163`
- RFC section: RFC 9292 §4.1
- Description: After the 1xx-skip loop exits, `statusCode` is not validated against the HTTP range `[100, 599]`. Add a range check immediately after the `break` line (after line 163). Throw `FormatException('Invalid HTTP status code: $statusCode')` for values outside the valid range. Note: RFC 9292 §4.1 defines the status field as an integer in the HTTP status code range; accepting `0` or `2^62-1` is a protocol violation. Check the upstream `ohttp_dart` GitHub repository before implementing a local patch.

---

**TASK-B2: Add max-header-count and max-total-header-size guards in `parseResponse`**
- Severity: HIGH
- File: `bhttp.dart:165–182`
- DoS scenario: Memory exhaustion from adversarial gateway with large `headersLen` varint or high header count
- Description: The header parsing loop has no limit on the number of header fields or the total bytes consumed by headers. Add two guards before the loop begins: (1) a configurable or hard-coded maximum `headersLen` value (e.g., 64 KiB or 256 KiB for wallet use), throwing `FormatException` if exceeded; (2) optionally, a maximum header-count check inside the loop (`if (headers.length > maxHeaderCount) throw FormatException(...)`). The threshold values should be documented and the wallet team should supply the appropriate limits. Both layers (`bhttp.dart` and `OhttpClient`) are candidate enforcement points — the follow-up task should decide the canonical location. Check upstream for prior art before patching locally.

---

**TASK-B3: Wrap body sublist extraction in `parseResponse` with `FormatException`**
- Severity: HIGH
- File: `bhttp.dart:187`
- RFC section: RFC 9292 §3.4
- Description: `data.sublist(offset, offset + contentLen)` throws `RangeError` (a Dart `Error`, not `Exception`) when `contentLen` exceeds remaining bytes. Callers that catch `Exception` miss this error; callers that catch `FormatException` also miss it. Wrap the call in a try/catch that converts `RangeError` to `FormatException('BHTTP body truncated: declared $contentLen bytes but only ${data.length - offset} remain')`. Apply the same wrapping to the header name and value sublist calls at lines 173 and 178, and to any other `sublist` call in `parseResponse`. Check upstream for prior art before patching locally.

---

**TASK-B4: Add buffer-length guards to `decodeVarint` for 2-, 4-, and 8-byte widths**
- Severity: HIGH
- File: `bhttp.dart:44–67`
- RFC section: RFC 9000 §16
- Description: `decodeVarint` cases 1, 2, and 3 access `data[offset + 1]` through `data[offset + 7]` without checking `data.length >= offset + width`. A short buffer throws unwrapped `RangeError`. Add length assertions or throw `FormatException` at the start of each multi-byte case: `if (offset + 2 > data.length) throw FormatException('varint truncated: need 2 bytes at offset $offset, have ${data.length - offset}')` (and similarly for 4 and 8 bytes). This converts the raw `RangeError` into a typed, catchable `FormatException` consistent with the framing-indicator guard. Check upstream for prior art before patching locally.

---

**TASK-B5: Add request body size upper-bound guard in `serializeRequest`**
- Severity: HIGH
- File: `bhttp.dart:74–111`
- DoS scenario: Oversized in-memory allocation and oversized POST to OHTTP gateway
- Description: `serializeRequest` accepts a `body` Uint8List of any size and serializes it without an upper-bound check. A multi-megabyte body silently produces a multi-megabyte BHTTP blob, a multi-megabyte HPKE ciphertext, and a multi-megabyte POST to the gateway. Add a configurable size assertion at the entry of `serializeRequest` (or at the `OhttpClient.send()` call site before the `serializeRequest` call at `ohttp_client.dart:97`). The wallet team should specify the threshold. The draft task should choose one canonical enforcement location (either BHTTP layer with a parameter, or `OhttpClient` layer with a hard-coded or config-driven limit) and document the decision. Cross-reference Phase 1 F-2 (no size cap on `encRequest`). Check upstream for prior art before patching locally.

---

**TASK-B6: Add negative tests for truncated input and adversarial inputs in `bhttp_test.dart`**
- Severity: HIGH
- File: `test/bhttp_test.dart`
- Description: The current test suite has no tests for: (a) truncated body (contentLen > remaining bytes), (b) truncated varint (2/4/8-byte varint with insufficient remaining bytes), (c) out-of-range status codes (0, 600, 65535), (d) large header count (e.g., 1000 headers), (e) `FormatException` vs. `RangeError` behavior on malformed input. Add one test group for each of these five scenarios. For (a)–(d), the primary assertion is the exception type that reaches the test boundary. For (e), assert that after TASK-B3 and TASK-B4 are implemented, only `FormatException` propagates (not raw `RangeError`). This task depends on TASK-B3 and TASK-B4 for the exception-type assertions; tests for the current behavior (pre-fix) can be added independently. Cross-reference Phase 1 TASK-6 (overflow test for `_seq`).

---

**TASK-B7: Add 8-byte varint test and verify web/WASM integer behavior**
- Severity: IMPROVEMENT
- File: `test/bhttp_test.dart`, `bhttp.dart:57–63`
- RFC section: RFC 9000 §16
- Description: The 8-byte varint case in `decodeVarint` has no test. Add at least one round-trip test for an 8-byte varint value (e.g., `0x40000000` = 1,073,741,824 and a value near `2^62 - 1`). Additionally, if `ohttp_dart` is ever deployed in a web/WASM context (see Phase 1 New Technical Question #3), verify that the 8-byte accumulation `(value << 8) | data[offset + i]` produces correct results under dart2js 53-bit integer semantics. Document the platform support scope.

---

*Research completed: 2026-05-21. Phase 2 only. No files in `lib/` or `test/` were modified.*
