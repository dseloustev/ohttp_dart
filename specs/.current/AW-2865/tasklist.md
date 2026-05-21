# Development Tasklist: [Investigation] Create improvement tasks for ohttp_dart (AW-2865)

Based on [vision.md](./vision.md).

---

## Progress Report

| # | Iteration | Status | Notes |
|---|-----------|--------|-------|
| 1 | Read HPKE layer and compare with RFC 9180 | ✅ Done | 10/10 |
| 2 | Read BHTTP layer and assess parser robustness | ✅ Done | 8/8 |
| 3 | Read OHTTP layer and assess encap/decap correctness | ✅ Done | 11/11 |
| 4 | Read client layer and assess network reliability and KeyConfig lifecycle | ✅ Done | 12/12 |
| 5 | Review test suite coverage gaps | ✅ Done | 11/11 |
| 6 | Review privacy risks, observability gaps, and documentation | ⬜ Pending |  |
| 7 | Compile findings into per-task Markdown files | ⬜ Pending |  |

**Legend:** ⬜ Pending | 🔄 In Progress | ✅ Done | ❌ Blocked

**Current Phase:** 6

---

## Iteration 1: Read HPKE layer and compare with RFC 9180

**Goal:** Audit `lib/src/hpke.dart` against RFC 9180 to identify deviations, missing guards, and test-seam risks.

### `lib/src/hpke.dart`
- [x] Read the full file (use `ast-index outline lib/src/hpke.dart` first, then read targeted slices).
- [x] Verify that `LabeledExtract` and `LabeledExpand` match the labeled-KDF construction in RFC 9180 §4 (suite ID, label strings, lengths).
- [x] Verify that `KEM Encap` matches RFC 9180 §4.1 (DH step, `ExtractAndExpand`, `shared_secret` derivation).
- [x] Verify that `setupBaseS` key-schedule matches RFC 9180 §5.1 (`key_schedule_s`, `mode_base = 0`).
- [x] Verify nonce derivation in `HpkeSenderContext.seal` matches RFC 9180 §5.2 (XOR of `baseNonce` with counter as big-endian `Nn`-byte integer).
- [x] Verify `HpkeSenderContext.export` matches RFC 9180 §5.3 (`LabeledExpand(exporterSecret, "sec", context, L)`).
- [x] Note whether `_seq` overflow guard throws a typed exception that callers can catch cleanly.
- [x] Note the `testKeyPair` injection point: confirm it is reachable from production call sites (`setupBaseS` optional parameter).
- [x] Note absence of zeroization for `key`, `baseNonce`, `exporterSecret` after use.
- [x] Record each finding as a draft task entry (file, line range, RFC section, severity: BLOCKER / HIGH / IMPROVEMENT).

**Test:** No code is changed. Verification is: all findings are supported by a specific line reference in `hpke.dart` and an RFC section number. Review your notes against the scrutiny table in `vision.md §4` before proceeding.

---

## Iteration 2: Read BHTTP layer and assess parser robustness

**Goal:** Audit `lib/src/bhttp.dart` against RFC 9292 to identify length-guard gaps and malformed-input risks.

### `lib/src/bhttp.dart`
- [x] Read the full file (use `ast-index outline` first).
- [x] Verify that the framing indicator byte (`0x00` for request, `0x01` for response) is validated; confirm behavior on unexpected indicator values.
- [x] Trace `parseResponse`: check whether `statusCode` is range-validated after varint decode (valid HTTP status is 100–599).
- [x] Trace `parseResponse` header loop: confirm there is no max-header-count guard and no total-size cap.
- [x] Trace the `body` sublist extraction: confirm it raises `RangeError` (not `FormatException`) on truncated input.
- [x] Verify varint decode (`decodeVarint`) handles all four QUIC length prefixes (1/2/4/8 bytes) and rejects buffers shorter than the declared length.
- [x] Confirm `serializeRequest` has no upper-bound check on body size before writing to the buffer.
- [x] Record each finding as a draft task entry (file, line range, RFC section or DoS scenario, severity).

**Test:** No code is changed. Verification is: each finding cites a specific line in `bhttp.dart` and a concrete malformed-input scenario (e.g., "truncated body at line 87 raises `RangeError` instead of `FormatException`").

---

## Iteration 3: Read OHTTP layer and assess encap/decap correctness

**Goal:** Audit `lib/src/ohttp.dart` against RFC 9458 to identify AAD deviations, response-decap correctness, and KeyConfig parser gaps.

### `lib/src/ohttp.dart`
- [x] Read the full file (use `ast-index outline` first).
- [x] Trace `OhttpKeyConfig.parse`: confirm the wire layout matches RFC 9458 §4.1 (key ID 1 B, KEM ID 2 B, public key `Npk` B, `symLen` 2 B, KDF+AEAD pairs).
- [x] Confirm that when `symLen > 4` the parser reads only the first KDF+AEAD pair and silently ignores the rest — note this as a finding.
- [x] Confirm that a short buffer with large `symLen` raises `RangeError`, not `FormatException` — note the exact throw site.
- [x] Note the inconsistency: `parse()` throws `FormatException` for unknown KEM, but `validate()` throws `UnsupportedError` for same condition.
- [x] Trace `ohttpEncapsulate`: verify HPKE info string construction (`"message/bhttp request" || 0x00 || header`) against RFC 9458 §4.3.
- [x] Confirm that AAD passed to `ctx.seal` is empty (`[]`) and cross-reference with RFC 9458 §4.3 wording; note the intentional deviation (matches Go reference implementation).
- [x] Trace `ohttpDecapsulate`: verify response HKDF usage — confirm it is plain (unlabeled) `HKDF-Extract(salt=enc||response_nonce, ikm=exportedSecret)` and `HKDF-Expand` for key/nonce derivation per RFC 9458 §4.4.
- [x] Confirm response AEAD also uses empty AAD.
- [x] Note that `SecretBoxAuthenticationError` (a `package:cryptography` type) propagates unwrapped to callers.
- [x] Record each finding as a draft task entry (file, line range, RFC section, severity).

**Test:** No code is changed. Verification is: the empty-AAD finding and the plain-HKDF-vs-labeled-HKDF finding each cite the exact line in `ohttp.dart` plus the RFC 9458 section that motivates the concern.

---

## Iteration 4: Read client layer and assess network reliability and KeyConfig lifecycle

**Goal:** Audit `lib/src/ohttp_client.dart` for timeout/retry/cancellation gaps, KeyConfig caching policy, scheme enforcement, and `sendDirect()` bypass risks.

### `lib/src/ohttp_client.dart`
- [x] Read the full file (use `ast-index outline` first).
- [x] Confirm that `OhttpGatewayConfig.gatewayBaseUrl` has no `https`-scheme enforcement — note that plain HTTP silently accepted.
- [x] Confirm that `configPath` and `requestPath` are concatenated as strings (not `Uri`-normalized) — note potential double-slash or path-traversal risk.
- [x] Confirm that `targetAuthority` is embedded verbatim into the BHTTP request with no allow-list check.
- [x] Confirm that `directBaseUrl` exists in the same config object as the OHTTP paths and that `sendDirect()` has no warning about bypassing OHTTP.
- [x] Trace the KeyConfig GET call: confirm `http.Client.get()` has no timeout and no retry, and that a fresh fetch occurs on every `send()` invocation (no caching).
- [x] Trace the gateway POST call: confirm `http.Client.post()` has no timeout, no retry, and no cancellation hook.
- [x] Confirm that all non-200 HTTP responses throw generic `Exception` (no typed error hierarchy, no distinction between 4xx and 5xx).
- [x] Confirm that network errors (DNS failure, connection reset) surface as untyped exceptions.
- [x] Confirm that response body and headers have no size cap before `bhttp.parseResponse`.
- [x] Note that `OhttpHeader.name` is not lowercased on the response side (inconsistent with request side).
- [x] Record each finding as a draft task entry (file, line range, concern category, severity).

**Test:** No code is changed. Verification is: the "no timeout" finding and the "no KeyConfig caching" finding each cite the specific `http.Client` call site line in `ohttp_client.dart`.

---

## Iteration 5: Review test suite coverage gaps

**Goal:** Read all three test files and identify negative-case, fuzz/property, and end-to-end test gaps relative to the implementation surface.

### `test/hpke_test.dart`
- [x] Read the file.
- [x] Confirm it uses `testKeyPair` injection to reproduce RFC 9180 Appendix A.1 vectors — note which specific vectors are covered.
- [x] Identify missing cases: sequence-number overflow, `export()` with wrong length, invalid public key input.

### `test/bhttp_test.dart`
- [x] Read the file.
- [x] Confirm varint round-trip tests cover 1/2/4/8-byte boundaries.
- [x] Identify missing cases: truncated body, oversized `symLen`, unknown framing indicator, malformed header (no colon separator).

### `test/ohttp_test.dart`
- [x] Read the file.
- [x] Identify missing cases: malformed KeyConfig (short buffer, extra KDF+AEAD pairs), AEAD auth failure (tampered ciphertext), empty response body, KeyConfig with unknown KEM/KDF/AEAD IDs.
- [x] Note that there are no integration tests against a live gateway.
- [x] Note absence of fuzz / property-based tests across all three files.
- [x] Record each gap as a draft task entry with the specific test file, missing scenario, and severity.

**Test:** No code is changed. Verification is: the gap list is grounded in specific test-file line counts and identified scenario names — no vague "needs more tests" entries.

---

## Iteration 6: Review privacy risks, observability gaps, and documentation

**Goal:** Assess the package against privacy requirements for a non-custodial crypto wallet, review the example file for documented limitations, and catalog observability gaps.

### `lib/src/ohttp_client.dart`
- [ ] Confirm that `sendDirect()` is not documented with a privacy-impact warning anywhere in the source.
- [ ] Confirm no logging of inner request URL/method/headers/body or response body/headers anywhere in `lib/`.

### `lib/src/ohttp.dart`
- [ ] Confirm no logging of `enc`, `exportedSecret`, or HKDF-derived key material.

### `lib/src/hpke.dart`
- [ ] Confirm no logging of ephemeral key material.

### `example/ohttp_dart_example.dart`
- [ ] Read the file.
- [ ] Note whether the example documents: (a) that a real gateway URL is required, (b) that `sendDirect()` bypasses OHTTP, (c) any known limitations or prerequisites.

### `pubspec.yaml`
- [ ] Read the file.
- [ ] Note `package:cryptography` version pinning — confirm no wildcard `any` constraint that could silently pull in a breaking update.
- [ ] Note absence of any dependency that introduces native code or FFI.

### Observability gaps (cross-cutting)
- [ ] Catalog all five observability gaps from `vision.md §7` (KeyConfig fetch, gateway POST status, AEAD failure, `sendDirect()` invocation, encapsulation timing) and assign severity per vision.md.
- [ ] Note the six logging constraints from `vision.md §7` that any future logging must respect.

**Test:** No code is changed. Verification is: the privacy findings include at least one BLOCKER or HIGH item tied to the `sendDirect()` bypass and at least one tied to absence of key-material zeroization.

---

## Iteration 7: Compile findings into per-task Markdown files

**Goal:** Synthesize all draft task entries from iterations 1–6 into a set of independently actionable Markdown task files under `specs/.current/AW-2865/tasks/`. These files are the deliverable — no Jira tickets are created from this ticket; downstream engineers will read these `.md` files directly when picking up the follow-up work.

### `specs/.current/AW-2865/tasks/` (new directory)
- [ ] Create the `tasks/` directory under `specs/.current/AW-2865/`.
- [ ] For each independently actionable finding from iterations 1–6, create one file named `NN-<kebab-slug>.md` (e.g., `01-enforce-https-scheme.md`). Number sequentially; group by severity so BLOCKER tasks come first, then HIGH, then IMPROVEMENT.
- [ ] Every task file MUST contain, at minimum:
  - `# Task NN: <Title>` heading.
  - `**Severity:** BLOCKER | HIGH | IMPROVEMENT`
  - `**Vector:** <one of the eight vectors from idea.md>`
  - `**Files:** <path[:line-range]>` — at least one file and one line range or RFC section.
  - `**Evidence:** <RFC section, line reference, or concrete malformed-input scenario>` motivating the task.
  - `## Description` — 2–5 sentences explaining the problem.
  - `## Proposed change` — what would resolve it (no code, just intent + acceptance criteria).
  - `## Acceptance criteria` — bullet list of verifiable conditions.
- [ ] Verify no two task files cover the same finding (deduplicate iterations 1–6 draft notes).
- [ ] Verify all eight investigation vectors from `idea.md` are represented across the task files: cryptographic correctness, interoperability, parser robustness, privacy risks, network reliability, KeyConfig management, test suite, documentation.
- [ ] Verify at least one task file is labelled `BLOCKER`.

### `specs/.current/AW-2865/tasks/README.md` (new file)
- [ ] Create the file with a top-level heading and a brief executive summary (3–5 sentences) covering the overall assessment of `ohttp_dart`.
- [ ] Add a short note at the top clarifying that these `.md` files are the final deliverable of AW-2865 — no Jira tickets are created automatically; engineers picking up follow-up work should reference these files.
- [ ] Add a "Blockers for production use" section listing only BLOCKER-severity task files with a one-line summary and a relative link to each.
- [ ] Add an "All follow-up tasks" table with columns: `# | Task title | Vector | Severity | File link`. The `File link` column links to the per-task `.md` file in this directory.
- [ ] Add an "Out of scope" section repeating the non-goals from `vision.md §Out of scope`.

**Test:** No code is changed. Verification is: open `tasks/README.md` and confirm (a) the index table lists every file in `tasks/`, (b) every task file has a severity label and a file/line citation, (c) all eight vectors appear at least once across the task files, (d) at least one BLOCKER task file exists, and (e) `ls specs/.current/AW-2865/tasks/` shows `README.md` plus one file per task with no duplicates.

---

## Final Verification

Run after **all iterations above are complete and checked off**. This is the end-of-feature gate — heavier than the per-iteration `mcp__dart__analyze_files` step, so do not run it after every edit.

- [ ] Run `make verify` — must pass with zero warnings, zero failing tests, zero DCM violations (runs `make analyze` + `make test-unit` + `make dcm-analyze`)
- [ ] Run `ast-index update` to reindex the codebase

**Gate:** Do not mark the ticket as done until `make verify` exits clean.
