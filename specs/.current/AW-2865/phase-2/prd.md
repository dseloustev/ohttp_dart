# AW-2865 Phase 2: BHTTP Layer Audit Against RFC 9292

Status: PRD_READY

## Context / Idea

Ticket AW-2865 is a read-only investigation of the `ohttp_dart` package to determine its production-readiness for use in a non-custodial crypto wallet. The investigation is organized into phases covering each layer of the four-layer stack.

Phase 2 targets Layer 2: `lib/src/bhttp.dart`. This file implements RFC 9292 Binary HTTP Known-Length framing. It is the only layer with no cryptographic operations — its role is to serialize inner HTTP requests before HPKE encapsulation and to parse decrypted inner HTTP responses after decapsulation. Because malformed-input handling flows directly through the OHTTP round-trip, robustness defects here have wallet-level impact.

**Inherited context (ticket-wide vision §2 severity model):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Phase dependency:** Phase 1 (HPKE layer audit) is complete. Phase 2 findings may cross-reference Phase 1 HPKE-layer results where a BHTTP defect compounds an upstream risk.

**No code is changed in this phase.** Every output is a finding mapped to a future follow-up task.

---

## Goals

1. Determine whether the framing indicator byte (`0x00` / `0x01`) is validated in `parseResponse` and establish behavior on unexpected indicator values.
2. Determine whether `parseResponse` range-validates the decoded status code against the valid HTTP range 100–599.
3. Establish whether the header loop in `parseResponse` has any max-header-count or total-size guard against memory-exhaustion from an adversarial gateway.
4. Confirm the exact exception type raised when the `body` sublist extraction encounters a truncated buffer (`RangeError` vs. `FormatException`), and assess the impact on callers.
5. Verify that `decodeVarint` correctly handles all four QUIC length-prefix widths (1/2/4/8 bytes) and rejects buffers shorter than the declared length.
6. Confirm that `serializeRequest` has no upper-bound guard on request body size before allocating the output buffer.
7. Produce a set of draft follow-up task entries for every confirmed finding, each with a file reference, line range, applicable RFC 9292 section or DoS scenario, and severity classification.

---

## User Stories

**US-1 — Security reviewer**
As a security reviewer preparing the package for wallet integration, I need a line-by-line audit of `bhttp.dart` against RFC 9292 so I can confirm the parser rejects malformed input safely and identify which gaps constitute production-blocking risks.

**US-2 — Engineering lead**
As an engineering lead, I need each BHTTP finding to be classified by severity and mapped to a specific line range and malformed-input scenario, so I can assign remediation tasks independently without further triage.

**US-3 — Wallet integrator**
As a developer integrating `ohttp_dart` into a non-custodial crypto wallet, I need to know whether a malicious or misconfigured OHTTP gateway can crash the client process or exhaust its memory through the BHTTP parsing layer, before deciding whether the package is safe to deploy.

---

## Main Scenarios

### Scenario 1 — Framing indicator validation (task 2.2)
Analyst reads `bhttp.dart` and locates the point where the framing indicator byte is read in `parseResponse`. Determines whether values other than `0x01` cause a typed error (e.g., `FormatException`) or fall through silently. Records the exact line and behavior.

### Scenario 2 — Status code range validation (task 2.3)
Analyst traces `parseResponse` from the framing indicator read through the varint decode of the status code. Checks whether any guard enforces `100 <= statusCode <= 599` after decoding. Documents the result with the line where the varint is decoded and the line (or absence) where a range check would appear.

### Scenario 3 — Header loop size guards (task 2.4)
Analyst traces the header-parsing loop in `parseResponse`. Checks for (a) a max iteration count and (b) an accumulator that tracks total bytes consumed by header names and values. Records whether either guard is absent, along with the loop boundaries and the concrete DoS scenario (unbounded memory allocation from adversarial header count or total header size).

### Scenario 4 — Body sublist truncation exception type (task 2.5)
Analyst locates the `sublist(offset, offset + contentLen)` call in `parseResponse`. Constructs a mental model of inputs where `contentLen` exceeds remaining buffer bytes. Confirms whether Dart's `Uint8List.sublist` raises `RangeError` in this case, and whether any wrapping converts it to `FormatException` before it propagates to callers. Records the exact line and the exception type that reaches `OhttpClient`.

### Scenario 5 — QUIC varint decoder correctness and truncation handling (task 2.6)
Analyst reads `decodeVarint` and traces each of the four prefix-width branches (1/2/4/8 bytes). Verifies that each branch correctly masks the length prefix bits and assembles the value. Checks whether the function guards against a buffer that has fewer bytes remaining than the width implied by the prefix bits. Records the exact line of any missing guard and the resulting behavior (likely `RangeError` on a short buffer).

### Scenario 6 — Request body size guard in `serializeRequest` (task 2.7)
Analyst reads `serializeRequest` and confirms whether any check limits the size of the body parameter before the function allocates or writes to the output buffer. Notes the line where body bytes are written and the absence of a preceding size assertion. Records the concrete risk: a caller passing a multi-megabyte body silently produces a multi-megabyte encapsulated OHTTP request with no warning.

### Scenario 7 — Draft task entry compilation (task 2.8)
For each confirmed finding from Scenarios 1–6, the analyst produces a draft task entry containing: finding description, affected file and line range, RFC 9292 section or DoS scenario, severity, and a proposed task title for Iteration 7 consolidation.

---

## Success / Metrics

| Criterion | Definition of done |
|---|---|
| All RFC 9292 parsing surfaces covered | Tasks 2.2–2.7 each produce a positive ("guard present and correct") or negative ("guard absent: …") verdict with a cited line reference in `bhttp.dart`. No parsing surface from the scrutiny table in `vision.md §4` is skipped. |
| Framing indicator behavior documented | A clear statement of what `parseResponse` does on an unexpected indicator byte (throws typed error, throws untyped error, or silently proceeds) with the exact line cited. |
| Every finding has a severity | Each finding carries exactly one of `BLOCKER`, `HIGH`, or `IMPROVEMENT`. |
| No code changes | `git diff HEAD -- lib/ test/` is empty at the end of this phase. |
| Draft tasks independently actionable | Each draft task can be assigned to a single engineer without requiring another in-flight task from this list. |
| Exception-type mismatch confirmed or denied | A definitive verdict on whether `RangeError` or `FormatException` propagates to callers on truncated body input, with the specific call-stack path identified. |

---

## Constraints and Assumptions

1. **Read-only.** No file in `lib/` or `test/` is modified during this phase.
2. **RFC 9292 Known-Length framing only.** The Indeterminate-Length framing variant defined in RFC 9292 is not implemented in `bhttp.dart`; evaluating it is out of scope.
3. **No cryptographic operations.** `bhttp.dart` has no crypto; HPKE/AEAD correctness is out of scope for this phase and was addressed in Phase 1.
4. **`package:cryptography` not in scope.** The audit covers only `bhttp.dart` itself; transitive dependency risks are reviewed in Phase 6 (privacy and documentation).
5. **Dart runtime exception behavior assumed standard.** `Uint8List.sublist` is assumed to throw `RangeError` on out-of-range indices per the Dart core library specification; this is a verification target, not a given.
6. **Test gaps are noted but not fixed.** Identified missing negative-case tests are recorded as draft task entries for Iteration 7; no new test code is written in this phase.
7. **QUIC varint encoding follows RFC 9000 §16.** The varint format is the two-bit-prefix scheme from RFC 9000; this is the normative reference for `decodeVarint` correctness.

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Absent header-size guard enables memory-exhaustion DoS from an adversarial or compromised OHTTP gateway; relevant for a wallet deployed on mobile where memory is constrained | HIGH | Scenario 3 directly targets this; any confirmed absence is recorded as a HIGH task entry |
| `RangeError` (instead of `FormatException`) on truncated body propagates unwrapped through `OhttpClient`; callers cannot distinguish a parsing failure from a logic error, making error handling unreliable | HIGH | Scenario 4 captures the exact throw site; remediation task would wrap the sublist call |
| No status-code range validation means a gateway can return a varint value of `0`, `1000`, or `2^62-1` without the client detecting an invalid response | HIGH | Scenario 2 captures this; remediation is a one-line guard |
| Missing framing indicator validation may allow a request-framed blob to be silently parsed as a response (or vice versa), producing corrupt `OhttpResponse` fields with no error signal | HIGH | Scenario 1 captures this; severity may be BLOCKER if cross-framing is exploitable |
| `serializeRequest` with no body-size upper bound could silently produce a multi-megabyte OHTTP request, exhausting memory or causing oversized POST to the gateway | HIGH | Scenario 6 captures this; remediation task adds a configurable size guard |
| `decodeVarint` may accept a four-byte or eight-byte prefix while the remaining buffer has fewer bytes than the prefix implies, causing a `RangeError` instead of a clean parse error | HIGH | Scenario 5 captures each prefix-width branch; any missing guard is a HIGH finding |
| Audit findings may be incomplete if `bhttp.dart` has grown or diverged since the vision was written | LOW | Task 2.1 (full outline read before targeted slices) serves as a completeness gate |

---

## Resolved Open Questions

All questions below were resolved by accepting analyst defaults.

1. **Framing indicator severity (HIGH vs. BLOCKER).** Resolution: keep as originally drafted — severity is `HIGH`, with a note in the draft task that the security reviewer may escalate to `BLOCKER` during Iteration 7 consolidation if cross-framing exploit potential is confirmed. No change to the risks table classification at this stage.

2. **Body size guard ownership (BHTTP layer vs. `OhttpClient` layer).** Resolution: scope decision left as originally noted in the draft — both layers are candidates; the follow-up task will identify the appropriate owner. The draft task entry records the finding without pre-assigning a layer.

3. **Maximum expected BHTTP response body size for the wallet use case.** Resolution: left open as a practical DoS calibration question. The draft task will document the absence of a cap as a `HIGH` risk regardless of the specific size threshold; the wallet team will supply the threshold during remediation.

4. **Exception-type mismatch consolidation vs. per-site tasks.** Resolution: keep as originally structured — findings are recorded as per-site task entries to preserve independent assignability, consistent with the success criterion that each task can be assigned to a single engineer without dependencies.

5. **Upstream tracking of BHTTP robustness gaps.** Resolution: upstream status is unknown; to be verified separately. Each draft task entry will include a note to check the upstream `ohttp_dart` GitHub repository before implementing a local patch.
