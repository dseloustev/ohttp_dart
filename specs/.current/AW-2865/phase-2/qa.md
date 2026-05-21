# QA Report — AW-2865 Phase 2: BHTTP Layer Audit Against RFC 9292

**Date:** 2026-05-21
**Phase:** 2
**Scope:** `lib/src/bhttp.dart` audit against RFC 9292. Read-only — no code in `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified.
**Verdict: PASS**

---

## Phase Scope

Phase 2 targeted Layer 2 of the four-layer stack: `bhttp.dart` (191 lines). The QA plan verifies that:

1. All 8 tasks (2.1–2.8) in `tasks.md` are marked `[x]` complete.
2. Every finding in `research.md` cites a specific `bhttp.dart:NNN` line range.
3. All severity classifications are present and drawn from the `BLOCKER / HIGH / IMPROVEMENT` taxonomy.
4. Draft task entries `TASK-B1` through `TASK-B7` exist in `research.md` with RFC section and/or DoS scenario references.
5. No source file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified.
6. The `review.md` Phase 2 verdict is `APPROVED` with no Blocking issues.
7. Cross-references to Phase 1 findings F-2, F-3, F-6, and F-7 are present.

This QA report does not re-verify RFC text or re-run code. It verifies the audit artifacts against the PRD acceptance criteria, the plan's Definition of Done, and the review sign-off in `review.md`.

---

## Check 1: All 8 tasks marked `[x]` in `tasks.md`

**Criterion:** Every checklist item 2.1–2.8 in `specs/.current/AW-2865/phase-2/tasks.md` must carry `[x]`.

| Task | Description | Status in tasks.md |
|---|---|---|
| 2.1 | Read the full file (use `ast-index outline` first) | `[x]` |
| 2.2 | Verify framing indicator validation and behavior on unexpected indicator values | `[x]` |
| 2.3 | Trace `parseResponse` status code range validation (100–599) | `[x]` |
| 2.4 | Trace `parseResponse` header loop for max-header-count and total-size guards | `[x]` |
| 2.5 | Trace body sublist extraction for exception type on truncated input | `[x]` |
| 2.6 | Verify `decodeVarint` handles all four QUIC length prefixes and rejects short buffers | `[x]` |
| 2.7 | Confirm `serializeRequest` has no upper-bound check on body size | `[x]` |
| 2.8 | Record each finding as a draft task entry with file/line, RFC/DoS scenario, and severity | `[x]` |

All 8 checklist items are marked complete.

**Check 1: PASS**

---

## Check 2: Every finding cites a specific `bhttp.dart:NNN` line range

**Criterion:** Each task-finding block in `research.md` and each row in the `tasks.md` finding table cites an explicit `bhttp.dart:NNN` or `bhttp.dart:NNN–NNN` reference.

| Finding | File:Line citation (research.md) | File:Line citation (tasks.md) | Both present? |
|---|---|---|---|
| Task 2.2 — framing indicator | `bhttp.dart:140–146` | `bhttp.dart:140–146` | Yes |
| Task 2.3 — status code range | `bhttp.dart:149–163` | `bhttp.dart:148–162` (off by one, cosmetic) | Yes |
| Task 2.4 — header loop | `bhttp.dart:165–182` | `bhttp.dart:164–182` (off by one, cosmetic) | Yes |
| Task 2.5 — body sublist | `bhttp.dart:187` | `bhttp.dart:184–187` | Yes |
| Task 2.6 — `decodeVarint` | `bhttp.dart:44–67` | `bhttp.dart:44–67` | Yes |
| Task 2.7 — `serializeRequest` | `bhttp.dart:103–105` (verdict block), `bhttp.dart:74–111` (full function) | `bhttp.dart:74–111` | Yes |
| TASK-B1 | `bhttp.dart:149–163` | — (draft task table cites F2.3) | Yes |
| TASK-B2 | `bhttp.dart:165–182` | — (draft task table cites F2.4a/b/c) | Yes |
| TASK-B3 | `bhttp.dart:187` (primary), `173, 178` (secondary) | — (draft task table cites F2.5a/b) | Yes |
| TASK-B4 | `bhttp.dart:44–67` | — (draft task table cites F2.6a) | Yes |
| TASK-B5 | `bhttp.dart:74–111` | — (draft task table cites F2.7) | Yes |
| TASK-B6 | `test/bhttp_test.dart` (test file target) | — (draft task table cites R-6) | Yes |
| TASK-B7 | `bhttp.dart:57–63` (8-byte branch) | — (draft task table cites F2.6b) | Yes |

The off-by-one drift between `tasks.md` and `research.md` on tasks 2.3 and 2.4 (noted in `review.md` as cosmetic) does not fail this check — both ranges encompass the cited construct. The review already flagged this for tightening before Iteration 7 import.

**Check 2: PASS**

---

## Check 3: All severity classifications present (`BLOCKER / HIGH / IMPROVEMENT`)

**Criterion:** Every finding in `research.md` "Limitations & Risks" table (R-1..R-7) and every draft task entry (TASK-B1..TASK-B7) carries exactly one of `BLOCKER`, `HIGH`, or `IMPROVEMENT`. No "TBD" is permitted.

**Limitations & Risks table (research.md):**

| Row | Severity | Assigned |
|---|---|---|
| R-1 — status code not range-validated | HIGH | Yes |
| R-2 — header loop no size cap | HIGH | Yes |
| R-3 — body sublist raises `RangeError` not `FormatException` | HIGH | Yes |
| R-4 — `decodeVarint` no buffer-length guards | HIGH | Yes |
| R-5 — `serializeRequest` no body size upper bound | HIGH | Yes |
| R-6 — no negative tests for truncated/adversarial input | HIGH | Yes |
| R-7 — 8-byte varint untested | IMPROVEMENT | Yes |

**Draft task entries (research.md):**

| Task | Severity | Assigned |
|---|---|---|
| TASK-B1 | HIGH | Yes |
| TASK-B2 | HIGH | Yes |
| TASK-B3 | HIGH | Yes |
| TASK-B4 | HIGH | Yes |
| TASK-B5 | HIGH | Yes |
| TASK-B6 | HIGH | Yes |
| TASK-B7 | IMPROVEMENT | Yes |

Distribution: 0 BLOCKER, 6 HIGH (R-1..R-6), 1 IMPROVEMENT (R-7). All draft tasks carry one severity label. No "TBD" present in `research.md`.

Note: `tasks.md` classifies F2.3 as `IMPROVEMENT` and F2.7 as `IMPROVEMENT→HIGH for wallet`, diverging from `research.md`'s HIGH classifications for both. This severity-discipline drift was identified as an Important finding in `review.md` (Phase 2 Important #1). The downstream Jira import draws from `research.md` draft task entries (TASK-B1 = HIGH, TASK-B5 = HIGH), so the drift is a documentation consistency issue, not a PRD criteria failure. The PRD criterion "every finding has a severity" is technically satisfied per file.

**Check 3: PASS** (with recommended pre-Iteration-7 action to align `tasks.md` severities)

---

## Check 4: Draft task entries TASK-B1..TASK-B7 exist with RFC/DoS scenario references

**Criterion:** All 7 draft task entries in `research.md` "Draft Task Entries (for Iteration 7)" exist. Each must reference an RFC section or a named DoS scenario.

| Draft Task | Title | RFC / DoS Reference | Present? |
|---|---|---|---|
| TASK-B1 | Add HTTP status code range validation in `parseResponse` | RFC 9292 §4.1 | Yes |
| TASK-B2 | Add max-header-count and max-total-header-size guards in `parseResponse` | DoS: memory exhaustion from adversarial gateway with large `headersLen` | Yes |
| TASK-B3 | Wrap body sublist extraction in `parseResponse` with `FormatException` | RFC 9292 §3.4 | Yes |
| TASK-B4 | Add buffer-length guards to `decodeVarint` for 2-, 4-, and 8-byte widths | RFC 9000 §16 | Yes |
| TASK-B5 | Add request body size upper-bound guard in `serializeRequest` | DoS: oversized in-memory allocation and oversized POST to OHTTP gateway | Yes |
| TASK-B6 | Add negative tests for truncated input and adversarial inputs in `bhttp_test.dart` | Test gap (no RFC requirement citation; gap confirmed by R-3 through R-5) | Yes |
| TASK-B7 | Add 8-byte varint test and verify web/WASM integer behavior | RFC 9000 §16 | Yes |

All 7 entries exist. Every entry has a concrete RFC section reference or a named DoS scenario narrative. Each entry also carries the required "Check the upstream `ohttp_dart` GitHub repository before implementing a local patch" note per resolved-question Q5.

Note from `review.md` (Important #2): TASK-B2 currently consolidates three distinct DoS surfaces from `tasks.md` F2.4a (oversized `headersLen`), F2.4b (large header count), and F2.4c (single oversized header value). The review recommends explicitly enumerating the three sub-guards in TASK-B2's description to prevent the single-oversized-header DoS (F2.4c) from being overlooked during Iteration 7 implementation.

**Check 4: PASS** (with recommended pre-Iteration-7 action to enumerate three sub-guards in TASK-B2)

---

## Check 5: No source files modified (`lib/`, `test/`, `pubspec.yaml`, `example/`)

**Criterion:** `git diff HEAD -- lib/ test/ pubspec.yaml example/` must be empty. Only `specs/.current/AW-2865/phase-2/` artifacts and the `.active_ticket` pointer may differ from HEAD.

Evidence from `review.md` Phase 2 section (lines 270–283):

```
$ git diff HEAD -- lib/ test/ pubspec.yaml example/
(empty)

$ git status --short
 M specs/.current/.active_ticket
 M specs/.current/AW-2865/phase-2/tasks.md
?? specs/.current/AW-2865/phase-2/plan.md
?? specs/.current/AW-2865/phase-2/prd.md
?? specs/.current/AW-2865/phase-2/research.md
```

The current git status (from the conversation context) shows:

```
M specs/.current/AW-2865/tasklist.md
?? specs/.current/AW-2865/phase-2/
```

All changes are confined to `specs/.current/AW-2865/`. No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` appears in the git diff. The read-only constraint is satisfied. The `research.md` footer at line 386 independently confirms: "No files in `lib/` or `test/` were modified."

**Check 5: PASS**

---

## Check 6: Review verdict is APPROVED with no Blocking issues

**Criterion:** The Phase 2 section of `specs/.current/AW-2865/review.md` must record an APPROVED overall verdict and explicitly state "no Blocking issues."

Evidence from `review.md` Phase 2 "Overall Verdict" (lines 161–165):

> **APPROVED with notes.** Phase 2 meets all PRD success criteria. Every parsing surface (2.2–2.7) has a per-task finding grounded in a concrete `bhttp.dart` line citation, all 8 checklist items are marked `[x]` in `phase-2/tasks.md`, no source under `lib/` or `test/` was modified, and the deliverable shape mirrors Phase 1.
>
> There are **no Blocking issues.**

The review further confirms under "Findings (priority taxonomy)" → "Blocking": "None."

The three Important findings in the review (severity drift, TASK-B2 consolidation, TASK-B6 severity rationale) are documentation consistency and pre-Iteration-7 polish items. None block Phase 2 closure.

**Check 6: PASS**

---

## Check 7: Cross-references to Phase 1 findings F-2, F-3, F-6, F-7

**Criterion:** `research.md` must cite Phase 1 finding IDs F-2, F-3, F-6, and F-7 in its "Cross-References to Phase 1 Findings" section.

Evidence from `research.md` "Cross-References to Phase 1 Findings" table (lines 302–308):

| Phase 2 Finding | Phase 1 ID cited | Location in research.md |
|---|---|---|
| R-3 (RangeError on truncated body) | Phase 1 F-3 (`StateError` on HPKE overflow — same untyped-error anti-pattern) | Line 305 |
| R-5 (no body size cap in `serializeRequest`) | Phase 1 F-2 (no size cap on `encRequest` in `ohttp.dart`) | Line 306 |
| R-6 (no negative tests in `bhttp_test.dart`) | Phase 1 F-6 (no `_seq` overflow test) and Phase 1 F-7 (no empty-AAD integration test) | Line 307 |

All four required Phase 1 IDs are cited: F-2 (R-5), F-3 (R-3), F-6 (R-6), F-7 (R-6). The cross-references were independently verified in `review.md` Phase 2 "Cross-phase reference verification" (lines 254–264): "All three required cross-references are present. The Phase 1 IDs (F-2, F-3, F-6, F-7) match the actual finding IDs in `phase-1/research.md`."

**Check 7: PASS**

---

## PRD Acceptance Criteria — Full Verification

The following table maps the PRD success criteria (from `phase-2/prd.md` "Success / Metrics" table) to evidence in the research deliverable.

| Criterion | Definition of done | Status | Evidence |
|---|---|---|---|
| All RFC 9292 parsing surfaces covered | Tasks 2.2–2.7 each produce a verdict with a cited `bhttp.dart` line | PASS | `research.md` lines 87–285: six per-task verdict blocks, each with bolded **Verdict** and `bhttp.dart:NNN` reference |
| Framing indicator behavior documented | Clear statement of what `parseResponse` does on unexpected indicator byte | PASS | Task 2.2 verdict: "guard present and correct — any byte that does not decode to the value `1` causes `parseResponse` to throw a typed `FormatException`" |
| Every finding has a severity | Each finding carries exactly one of `BLOCKER`, `HIGH`, or `IMPROVEMENT` | PASS | R-1..R-6 = HIGH; R-7 = IMPROVEMENT; TASK-B1..B6 = HIGH; TASK-B7 = IMPROVEMENT |
| No code changes | `git diff HEAD -- lib/ test/` is empty | PASS | Confirmed by `review.md` and git status |
| Draft tasks independently actionable | Each task assignable without depending on another in-flight task | PASS | TASK-B1..TASK-B7 each have single-file targets; TASK-B6 dependency on TASK-B3/B4 for post-fix assertions is called out inline |
| Exception-type mismatch confirmed or denied | Definitive verdict on `RangeError` vs. `FormatException` with call-stack path | PASS | Task 2.5 verdict traces `bhttp.dart:187` → `ohttp_client.dart:136` → caller; confirms `RangeError` propagates unwrapped |

All six PRD success criteria are satisfied.

---

## Positive Scenarios

### PS-1: Framing indicator validation — guard present and correct

`parseResponse` reads the framing indicator at `bhttp.dart:140–146` via `decodeVarint` and explicitly rejects any value other than `1` with `FormatException('Expected known-length response (framing=1), got $framing')`. This covers `0` (request frame), `2`, `3`, and any multi-byte varint value. The exception propagates to `OhttpClient.send()` without being caught. RFC 9292 §3.2/§3.4 compliance is confirmed.

### PS-2: QUIC varint decoder correctness — all four prefix widths correct

`decodeVarint` at `bhttp.dart:44–67` correctly implements all four RFC 9000 §16 prefix-width branches:
- 1-byte: `first & 0x3F`, no additional reads — correct.
- 2-byte: `(first & 0x3F) << 8 | data[offset+1]` — correct big-endian assembly.
- 4-byte: unrolled big-endian with correct `0x3F` mask on first byte — correct.
- 8-byte: iterative `(value << 8) | data[offset+i]` for i=1..7 — correct mask, correct assembly for native VM.

The `switch` on `first >> 6` covers all four values of a 2-bit prefix. Non-canonical encodings (e.g., value `5` encoded as a 2-byte varint) are legal per RFC 9000 and are decoded correctly.

### PS-3: Request serialization — framing indicator written correctly

`serializeRequest` writes framing indicator `0x00` at `bhttp.dart:85` via `encodeVarint(0)`, conforming to RFC 9292 §3.2 (Known-Length Request framing indicator). The function correctly sequences control data, header section, and content.

### PS-4: RFC 9292 §4.1 Known-Length response fields — partially conformant

`parseResponse` correctly handles the 1xx informational-response skip loop (status code decoded as varint, loop while in `[100, 200)`). The header section and body content are parsed in the RFC 9292 §3.4 structure order. The framing indicator guard correctly gates all subsequent parsing.

---

## Negative Scenarios and Edge Cases

### NS-1: Status code not range-validated after 1xx skip — R-1, HIGH

After the `while (true)` loop exits at `bhttp.dart:148–162`, the accepted `statusCode` value is not checked against `[100, 599]`. Values of `0`, `600`, `999`, and up to `2^62 - 1` are silently returned in `BhttpResponse.statusCode`. A wallet checking `response.statusCode == 200` works correctly, but any range-based branching (`statusCode >= 500`) is unreliable. RFC 9292 §4.1 implicitly requires valid HTTP status codes; RFC 9110 §15 defines the range as 100–599.

### NS-2: Header loop — no max-header-count or max-total-size guard — R-2, HIGH

`parseResponse` at `bhttp.dart:165–182` accepts `headersLen` as a 62-bit varint with no secondary bound check. Three distinct DoS surfaces exist:
1. `headersLen` larger than remaining buffer — `RangeError` (not `FormatException`) when `sublist` calls run out of data.
2. Large header count — unbounded `headers.add(...)` iterations exhaust heap on mobile.
3. Single oversized header value — `nameLen`/`valueLen` up to `2^30 - 1` causes a single contiguous heap allocation of declared size.

A compromised or malicious OHTTP gateway is the threat actor; wallet code cannot verify gateway integrity at the BHTTP layer.

### NS-3: Body sublist raises `RangeError` not `FormatException` on truncated input — R-3, HIGH

`data.sublist(offset, offset + contentLen)` at `bhttp.dart:187` throws `RangeError` (a Dart `Error`, not `Exception`) when `contentLen` exceeds remaining bytes. The same applies to header name/value `sublist` calls at lines 173 and 178. The `RangeError` propagates unwrapped through `ohttp_client.dart:136` to `OhttpClient.send()` callers. Callers that catch `FormatException` do not catch this error; callers that catch `Exception` also do not catch it (`RangeError extends Error`). A generic `catch (e)` is required, conflating BHTTP truncation with any other Dart runtime error.

### NS-4: `decodeVarint` — no buffer-length guards for 2-, 4-, 8-byte widths — R-4, HIGH

Cases 1, 2, and 3 of `decodeVarint` (`bhttp.dart:44–67`) access `data[offset + 1]` through `data[offset + 7]` without checking `data.length >= offset + width`. A short buffer (e.g., `0x80` for a declared 4-byte varint but only 1 byte available) throws `RangeError` rather than a catchable `FormatException`. This affects every length-prefix decode in `parseResponse` — framing, status, headers, and body.

### NS-5: `serializeRequest` — no body size or header size upper bound — R-5, HIGH

`serializeRequest` at `bhttp.dart:74–111` accepts any `Uint8List body` without a size constraint. A multi-megabyte body causes `BytesBuilder.toBytes()` to attempt a single contiguous allocation of `|body| + |headers| + overhead` bytes, which is then fed to HPKE AES-128-GCM encryption (doubling memory pressure) and posted to the gateway. No constant, assertion, or configurable limit is present.

### NS-6: No negative tests for truncated or adversarial input — R-6, HIGH

`bhttp_test.dart` covers only happy-path round-trips and one framing-indicator rejection. There are no tests for: truncated body (`contentLen > remaining bytes`), truncated varint (2/4/8-byte with insufficient buffer), out-of-range status codes (`0`, `600`, `65535`), large header count, or `FormatException` vs. `RangeError` behavior on malformed input. Future refactoring of these code paths has no regression backstop.

### NS-7: 8-byte varint — untested and JS-integer-overflow risk — R-7, IMPROVEMENT

Case 3 of `decodeVarint` (`bhttp.dart:57–63`) is functionally correct on the Dart VM (63-bit signed `int`), but has no test vector. Values above `2^53` would silently truncate on dart2js / dart2wasm targets. If `ohttp_dart` is deployed in a web/WASM context, the 8-byte varint case could produce incorrect values for adversarial inputs with high bits set.

### NS-8: Empty buffer — `decodeVarint` raises `RangeError` before framing check

An empty `Uint8List` passed to `parseResponse` causes `decodeVarint(data, 0)` at line 140 to access `data[0]` on an empty buffer, throwing `RangeError` before the framing-indicator guard at lines 142–146 can execute. This is a subset of R-3/R-4's broader truncation gap; callers receive an untyped `RangeError` instead of `FormatException`.

---

## Automated Test Coverage Assessment

| Area | Existing coverage | Gap identified |
|---|---|---|
| Happy-path `serializeRequest` + `parseResponse` round-trip | `bhttp_test.dart` — covers one content-type header, basic body | No gap for correctness path |
| Framing-indicator rejection (non-`0x01` framing) | `bhttp_test.dart` — one test present | Covered |
| QUIC varint round-trips (1/2/4-byte, offset decode) | `bhttp_test.dart` lines 33–57 | 8-byte varint not tested (R-7 / TASK-B7) |
| Truncated body (`contentLen` > remaining bytes) | None | TASK-B3 + TASK-B6 |
| Truncated varint (2/4/8-byte short buffer) | None | TASK-B4 + TASK-B6 |
| Out-of-range status codes (`0`, `600`, `2^62-1`) | None | TASK-B1 + TASK-B6 |
| Large header count / oversized `headersLen` | None | TASK-B2 + TASK-B6 |
| `serializeRequest` with oversized body | None | TASK-B5 + TASK-B6 |
| `FormatException` propagation post-fix (relies on TASK-B3/B4) | None | TASK-B6 (post-fix assertions) |

---

## Manual Checks Needed

1. **Severity reconciliation before Iteration 7.** The `tasks.md` F2.3 classification (`IMPROVEMENT`) and F2.7 classification (`IMPROVEMENT→HIGH for wallet`) conflict with `research.md`'s `HIGH` for both. The Jira import must draw from one canonical source — `research.md` HIGH is the more defensible position given the wallet deployment context. Confirm and update `tasks.md` before Iteration 7 compilation.

2. **TASK-B2 sub-guard enumeration.** TASK-B2 consolidates three distinct DoS surfaces (F2.4a oversized `headersLen`, F2.4b large header count, F2.4c single oversized header value). The current TASK-B2 description mentions guards (1) and (2) but only obliquely covers (3). The single-oversized-header DoS (F2.4c, `nameLen = 1`, `valueLen = 2^30 - 1`) is the most novel. Explicitly enumerate all three sub-guards in TASK-B2 before Iteration 7 compilation.

3. **Trailer section silent drop (New Technical Question #4).** RFC 9292 §3.4 defines a trailer section after the content field. `parseResponse` returns after the body sublist with no trailer validation or offset check. `review.md` (Nice-to-have #3) recommends promoting this to R-8 / TASK-B8 (IMPROVEMENT) or explicitly deferring to Phase 5. Decision must be made before Iteration 7 to prevent loss.

4. **TASK-B6 severity rationale.** `review.md` (Important #3) notes that classifying TASK-B6 (negative tests) as HIGH is inconsistent with Phase 1's treatment of analogous test gaps (F-6, F-7 were IMPROVEMENT or HIGH for different reasons). Add a rationale note to TASK-B6, or downgrade to IMPROVEMENT for cross-phase consistency.

5. **TASK-B7 split.** `review.md` (Nice-to-have #4) recommends splitting TASK-B7 into (a) add 8-byte varint round-trip test and (b) document/verify platform support for dart2js/WASM. Bundled in one task, either concern may be dropped by the implementing engineer.

6. **Jira import readiness.** TASK-B1 through TASK-B7 should be reviewed by the engineering lead before import to confirm: (a) severity assignments align with the wallet threat model, (b) the upstream `ohttp_dart` GitHub repository has been checked for prior fixes, and (c) the layer-ownership decision for body-size enforcement (BHTTP vs. `OhttpClient`) is resolved before TASK-B5 is picked up.

---

## Phase-Specific Risk Zone

| Risk | Likelihood | Impact | Phase 2 status |
|---|---|---|---|
| Header loop memory-exhaustion DoS from adversarial gateway (R-2, HIGH) | Medium (requires compromised/malicious gateway) | High (OOM on mobile wallet process) | Captured as TASK-B2; three sub-guards required; must close before wallet production use |
| Truncated-input `RangeError` crashes isolate or is misclassified as programmer error (R-3, R-4, HIGH) | Medium (any truncated/malformed gateway response) | High (unreliable error handling; `catch (FormatException)` silently misses these) | Captured as TASK-B3 and TASK-B4; single `_safeSublist`/`_requireBytes` helper pattern recommended |
| Oversized request body exhausts process memory before gateway POST (R-5, HIGH) | Low (requires caller to pass oversized body) | High (memory exhaustion + oversized network request) | Captured as TASK-B5; layer-ownership decision deferred to Phase 5 per plan.md |
| Status code out-of-range silently accepted (R-1, HIGH) | Medium (compromised gateway can send `statusCode = 0`) | Medium (incorrect client branching; not a DoS) | Captured as TASK-B1; one-line guard fix |
| No negative tests for truncated/adversarial inputs (R-6, HIGH) | N/A (test gap, not runtime risk) | High (silent regression risk on refactoring) | Captured as TASK-B6 |
| 8-byte varint truncation under dart2js (R-7, IMPROVEMENT) | Very low (VM-only deployment assumed) | Medium (silent incorrect decode for large varint values) | Captured as TASK-B7; platform scope documentation needed |

---

## Final Verdict

**PASS**

All 7 QA checks pass:

- **Check 1 PASS** — All 8 tasks (2.1–2.8) in `tasks.md` are marked `[x]`.
- **Check 2 PASS** — Every finding cites a specific `bhttp.dart:NNN` line range; minor off-by-one drift between `tasks.md` and `research.md` on tasks 2.3 and 2.4 is cosmetic and does not affect traceability.
- **Check 3 PASS** — All severity classifications are present (`BLOCKER / HIGH / IMPROVEMENT`); 0 BLOCKER, 6 HIGH, 1 IMPROVEMENT in `research.md`. Cross-file severity drift (F2.3 and F2.7) flagged for pre-Iteration-7 reconciliation.
- **Check 4 PASS** — TASK-B1 through TASK-B7 exist with RFC section and/or DoS scenario references. TASK-B2 sub-guard enumeration flagged for pre-Iteration-7 expansion.
- **Check 5 PASS** — `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty. No source file was modified.
- **Check 6 PASS** — `review.md` Phase 2 verdict is APPROVED with no Blocking issues.
- **Check 7 PASS** — Cross-references to Phase 1 findings F-2, F-3, F-6, and F-7 are present in `research.md` lines 305–307.

No BLOCKER was identified at the BHTTP layer. Six HIGH-severity items (R-1 through R-6) require remediation before production wallet use. One IMPROVEMENT item (R-7) is recommended for the follow-up backlog. The four manual-check action items above should be resolved before Iteration 7 compiles `research.md` draft tasks into per-task Jira entries.

Phase 2 is ready to close.
