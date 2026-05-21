# Phase 2 Summary: BHTTP Layer Audit Against RFC 9292
# AW-2865 — ohttp_dart Investigation

**Date completed:** 2026-05-21
**Phase:** 2 of 7 (Iteration 2)
**Status:** APPROVED — ready to close

---

## Scope and Goal

Phase 2 was a read-only audit of `lib/src/bhttp.dart` (191 lines) against RFC 9292
Binary HTTP Known-Length framing and RFC 9000 section 16 (QUIC variable-length integers).
No file in `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified.

The goal was to determine whether the two public entry points -- `serializeRequest`
(request to binary blob) and `parseResponse` (binary blob to `BhttpResponse`) -- safely
handle malformed or adversarial inputs from a compromised OHTTP gateway, and to produce a
set of independently actionable draft task entries for Iteration 7.

The `bhttp.dart` layer is the only layer in the four-layer stack with no cryptographic
operations. Its robustness matters because malformed-input errors propagate directly through
`ohttp_client.dart` to the wallet caller without being wrapped.

---

## Key Findings (R-1 through R-7)

Seven findings were recorded. One positive verdict (framing indicator guard is correct)
and six gaps, all in the parser.

### R-1 -- Status code not range-validated (HIGH)

`parseResponse` (`bhttp.dart:149-163`) correctly skips 1xx informational responses but
does not check the accepted `statusCode` against the valid HTTP range [100, 599] after
the loop breaks. Values of 0, 600, 999, or 2^62 minus 1 pass through verbatim to
`OhttpResponse.statusCode`.

RFC 9292 section 4.1 / RFC 9110 section 15.

### R-2 -- Header loop has no max-count or max-size guard (HIGH)

The header-parsing loop in `parseResponse` (`bhttp.dart:165-182`) accepts `headersLen`
as a 62-bit varint with no secondary limit. Three distinct DoS surfaces exist:
(1) `headersLen` larger than the remaining buffer causes an unwrapped `RangeError`;
(2) a large header count causes unbounded `headers.add()` heap growth on mobile;
(3) a single header value with valueLen up to 2^30 minus 1 causes a single large
contiguous allocation. The network layer's response size limit is the only backstop.

DoS: memory exhaustion from adversarial or compromised OHTTP gateway.

### R-3 -- Body sublist throws RangeError not FormatException on truncated input (HIGH)

`data.sublist(offset, offset + contentLen)` at `bhttp.dart:187` throws `RangeError`
(a Dart Error, not Exception) when `contentLen` exceeds remaining bytes. The same
applies to header name/value sublist calls at lines 173 and 178. The `RangeError`
propagates unwrapped through `ohttp_client.dart:136` to `OhttpClient.send()` callers.
Callers that catch `FormatException` or `Exception` do not intercept this error.

Cross-reference Phase 1 F-3 (StateError on HPKE overflow -- same untyped-error
anti-pattern recurring across layers).

### R-4 -- decodeVarint has no buffer-length guards for 2-, 4-, and 8-byte widths (HIGH)

`decodeVarint` (`bhttp.dart:44-67`) implements all four RFC 9000 section 16 prefix
widths correctly for bit masking and value assembly. However, cases 1, 2, and 3 access
`data[offset + 1]` through `data[offset + 7]` without checking
`data.length >= offset + width`. A short buffer throws unwrapped `RangeError`. This
affects every length-prefix decode in `parseResponse`.

RFC 9000 section 16.

### R-5 -- serializeRequest has no body size or header size upper bound (HIGH)

`serializeRequest` (`bhttp.dart:74-111`) accepts any `Uint8List body` without a size
constraint. `BytesBuilder.toBytes()` attempts a single contiguous allocation of
`|body| + |headers| + overhead` bytes, which then feeds HPKE AES-128-GCM encryption
(doubling memory pressure) and a network POST to the gateway.

Cross-reference Phase 1 F-2 (no size cap on `OhttpEncapsulateResult.encRequest` -- a
large BHTTP blob becomes a large HPKE ciphertext).

### R-6 -- No negative tests for truncated or adversarial input (HIGH)

`bhttp_test.dart` covers only happy-path round-trips and one framing-indicator rejection
test. There are no tests for: truncated body, truncated varint (2/4/8-byte with
insufficient buffer), out-of-range status codes, large header count, or
FormatException vs. RangeError behavior on malformed input.

Cross-reference Phase 1 F-6 and F-7 (test-gap pattern recurring across layers).

### R-7 -- 8-byte varint untested; JS-integer-overflow risk on web target (IMPROVEMENT)

Case 3 of `decodeVarint` (`bhttp.dart:57-63`) is correct for the Dart VM (63-bit signed
int) and has no test vector. On dart2js / dart2wasm targets, values above 2^53 would
silently truncate during the shift accumulation loop.

RFC 9000 section 16.

---

## Positive Verdict

Framing indicator validation -- guard present and correct. `parseResponse` at
`bhttp.dart:140-146` rejects any indicator value other than 1 with
`FormatException('Expected known-length response (framing=1), got $framing')`. This
covers request-frame bytes (0x00), unused indicator values (2, 3), and any multi-byte
varint value.

---

## Draft Task Entries for Iteration 7 (TASK-B1 through TASK-B7)

Full descriptions are in `specs/.current/AW-2865/phase-2/research.md`
"Draft Task Entries (for Iteration 7)".

| ID | Title | Severity | Primary file and lines | RFC / DoS reference |
|---|---|---|---|---|
| TASK-B1 | Add HTTP status code range validation in `parseResponse` | HIGH | `bhttp.dart:149-163` | RFC 9292 section 4.1 |
| TASK-B2 | Add max-header-count and max-total-header-size guards in `parseResponse` | HIGH | `bhttp.dart:165-182` | DoS: memory exhaustion (three sub-guards: max headersLen, max header count, max single-field length) |
| TASK-B3 | Wrap body sublist extraction in `parseResponse` with FormatException | HIGH | `bhttp.dart:187`, `173`, `178` | RFC 9292 section 3.4 |
| TASK-B4 | Add buffer-length guards to `decodeVarint` for 2-, 4-, and 8-byte widths | HIGH | `bhttp.dart:44-67` | RFC 9000 section 16 |
| TASK-B5 | Add request body size upper-bound guard in `serializeRequest` | HIGH | `bhttp.dart:74-111` | DoS: oversized in-memory allocation and oversized POST to gateway |
| TASK-B6 | Add negative tests for truncated input and adversarial inputs in `bhttp_test.dart` | HIGH | `test/bhttp_test.dart` | Test gap (five missing scenario groups) |
| TASK-B7 | Add 8-byte varint test and verify web/WASM integer behavior | IMPROVEMENT | `test/bhttp_test.dart`, `bhttp.dart:57-63` | RFC 9000 section 16 |

All seven entries carry the upstream-check note: verify the ohttp_dart GitHub repository
for prior fixes before implementing a local patch.

---

## Cross-Phase References to Phase 1

| Phase 2 finding | Phase 1 finding | Pattern |
|---|---|---|
| R-3 (RangeError on truncated body) | F-3 (StateError on HPKE overflow) | Domain errors thrown as untyped Dart errors rather than typed exceptions -- catch-site handling is unreliable across the whole stack |
| R-5 (no body size cap in serializeRequest) | F-2 (no size cap on encRequest) | A large BHTTP blob becomes a large HPKE plaintext; no cap at either layer |
| R-6 (no negative tests in bhttp_test.dart) | F-6, F-7 (test gaps in HPKE layer) | Test-gap pattern repeats at every layer; Iteration 7 should address all layers in a coordinated effort |

---

## Pre-Iteration-7 Action Items (from review.md)

The Phase 2 review is APPROVED with no Blocking issues. Three Important items must be
resolved before Iteration 7 compiles draft entries into Jira tickets:

1. Severity reconciliation. `tasks.md` classifies F2.3 (status code) as IMPROVEMENT and
   F2.7 (body cap) as IMPROVEMENT->HIGH for wallet, while `research.md` classifies both
   as HIGH. The Jira import must draw from one canonical source. The `research.md` HIGH
   classification is the more defensible position for the wallet deployment context.

2. TASK-B2 sub-guard enumeration. TASK-B2 consolidates three distinct DoS surfaces
   (oversized headersLen, large header count, single oversized header value). The
   single-oversized-header DoS (F2.4c) is the most novel and is currently underspecified.
   All three sub-guards must be explicitly enumerated before Iteration 7 compilation.

3. Trailer section decision (New Technical Question #4). RFC 9292 section 3.4 defines a
   trailer section after the content field; `parseResponse` silently discards any bytes
   after the body. This must be promoted to R-8 / TASK-B8 (IMPROVEMENT) or explicitly
   deferred to Phase 5 before Iteration 7 to prevent loss.

---

## QA Verdict

PASS. All 7 QA checks in `specs/.current/AW-2865/phase-2/qa.md` pass:

- All 8 tasks (2.1-2.8) in `tasks.md` are marked [x] complete.
- Every finding cites a specific `bhttp.dart:NNN` line range.
- All severity classifications are present: 0 BLOCKER, 6 HIGH (R-1..R-6), 1 IMPROVEMENT (R-7). No TBD.
- TASK-B1 through TASK-B7 exist with RFC section and/or DoS scenario references.
- `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty -- no source file was modified.
- `review.md` Phase 2 verdict is APPROVED with no Blocking issues.
- Cross-references to Phase 1 findings F-2, F-3, F-6, and F-7 are present in `research.md`.

No BLOCKER was identified at the BHTTP layer. Six HIGH-severity items require remediation
before production wallet use. One IMPROVEMENT item is recommended for the follow-up backlog.

---

## Downstream Handoff

- Phase 3 (ohttp.dart audit): cite TASK-B3 / TASK-B4 when documenting the RangeError
  anti-pattern in `OhttpKeyConfig.parse` -- the same truncated-buffer problem recurs
  there per `vision.md` section 5.1.
- Phase 5 (ohttp_client.dart audit): canonical decision point for TASK-B5 body-size-guard
  layer ownership (BHTTP layer vs. OhttpClient layer).
- Phase 6 (privacy and documentation): address New Technical Questions #1 (dart2js 8-byte
  varint semantics) and #2 (1xx flood loop bound) as platform-support documentation items.
- Iteration 7 (task compilation): draws directly from `research.md` Draft Task Entries
  (TASK-B1..TASK-B7). Pre-Iteration-7 actions above should be completed first.
