# Phase 7: Compile findings into per-task Markdown files

**Goal:** Synthesize all draft task entries from iterations 1–6 into a set of independently actionable Markdown task files under `specs/.current/AW-2865/tasks/`. These files are the deliverable — no Jira tickets are created from this ticket; downstream engineers will read these `.md` files directly when picking up follow-up work.

## Context

**Feature motivation (from idea):** Investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. Within the scope of the investigation, the goal is not to implement improvements, but to identify risks and form subsequent engineering tasks. As a result, a set of Markdown task files must be prepared with a brief description, priority, and estimated urgency. Each task must be specific enough to be taken into work separately.

**Severity classification (from vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Eight investigation vectors (from idea.md) — all must appear across the task files:**
1. Cryptographic correctness (RFC 9180 / 9292 / 9458 compliance)
2. Interoperability (wire-format compatibility with real gateways)
3. Parser robustness (malformed input, DoS scenarios)
4. Privacy risks (unlinkability guarantees, key-material exposure)
5. Network reliability (timeout, retry, cancellation, error handling)
6. KeyConfig management (caching, TTL, downgrade risk)
7. Test suite (negative cases, fuzz/property tests, integration tests)
8. Documentation (threat model, limitations, observability guidance)

**Phases completed and their draft-finding sources:**
- Phase 1 (HPKE audit): findings in `phase-1/tasks.md`
- Phase 2 (BHTTP audit): draft entries F2.3–F2.7 in `phase-2/tasks.md §2.8`
- Phase 3 (OHTTP audit): draft entries TASK-O1 through TASK-O7 in `phase-3/tasks.md`
- Phase 4 (Client audit): findings C-1 through C-10 in `phase-4/tasks.md`
- Phase 5 (Test suite): draft entries T-H1..T-H3, T-B1..T-B5, T-O1..T-O6, T-X1..T-X2 in `phase-5/tasks.md`
- Phase 6 (Privacy/observability): draft entries P6-1 through P6-13 in `phase-6/tasks.md`

## Tasks

### `specs/.current/AW-2865/tasks/` (new directory)

- [ ] 7.1 Create the `tasks/` directory under `specs/.current/AW-2865/`.
- [ ] 7.2 For each independently actionable finding in the **Consolidated Findings Inventory** (Technical Details §2 below), create one file named `NN-<kebab-slug>.md` (e.g., `01-ohttp-bypass-send-direct.md`). Number sequentially; BLOCKER tasks come first, then HIGH, then IMPROVEMENT. Each file must contain at minimum:
  - `# Task NN: <Title>` heading.
  - `**Severity:** BLOCKER | HIGH | IMPROVEMENT`
  - `**Vector:** <one of the eight vectors from idea.md>`
  - `**Files:** <path[:line-range]>` — at least one file and one line range or RFC section.
  - `**Evidence:** <RFC section, line reference, or concrete malformed-input scenario>` motivating the task.
  - `## Description` — 2–5 sentences explaining the problem.
  - `## Proposed change` — what would resolve it (no code, just intent + acceptance criteria).
  - `## Acceptance criteria` — bullet list of verifiable conditions.
- [ ] 7.3 After creating all files, verify each file has all required minimum fields (heading, Severity, Vector, Files, Evidence, Description, Proposed change, Acceptance criteria).
- [ ] 7.4 Verify no two task files cover the same finding. Use the cross-reference IDs in §2 to detect duplicates (e.g., P6-2 and C-1 must become one file, not two).
- [ ] 7.5 Verify all eight investigation vectors from `idea.md` appear at least once across the task files.
- [ ] 7.6 Verify at least one task file is labelled `BLOCKER`.

### `specs/.current/AW-2865/tasks/README.md` (new file)

- [ ] 7.7 Create `README.md` with: a top-level heading, a brief executive summary (3–5 sentences) of the overall assessment, and a short note that these `.md` files are the final deliverable of AW-2865 — no Jira tickets are created automatically; engineers picking up follow-up work should reference these files directly.
- [ ] 7.8 Add a "Blockers for production use" section listing only BLOCKER-severity task files with a one-line summary and a relative link to each.
- [ ] 7.9 Add an "All follow-up tasks" table with columns: `# | Task title | Vector | Severity | File link`. The `File link` column links to the per-task `.md` file in this directory.
- [ ] 7.10 Add an "Out of scope" section repeating the non-goals from `vision.md §Out of scope`.

## Acceptance Criteria

**Test:** No code is changed. Verification is: open `tasks/README.md` and confirm:
- (a) the index table lists every file in `tasks/`;
- (b) every task file has a severity label and a file/line citation;
- (c) all eight vectors appear at least once across the task files;
- (d) at least one BLOCKER task file exists;
- (e) `ls specs/.current/AW-2865/tasks/` shows `README.md` plus one file per task with no duplicates.

## Dependencies

- Phases 1–6 complete (all audit and review iterations done).

## Technical Details

### §1 Required task file structure (template)

```markdown
# Task NN: <Title>

**Severity:** BLOCKER | HIGH | IMPROVEMENT
**Vector:** <one of the eight vectors>
**Files:** `<path:line-range>`

**Evidence:** <RFC section or concrete scenario>

## Description

2–5 sentences.

## Proposed change

Intent and acceptance criteria (no code).

## Acceptance criteria

- Bullet list of verifiable conditions.
```

### §2 Consolidated Findings Inventory

All entries below are deduplicated across Phases 1–6. Cross-reference IDs are shown for traceability. One task file per row.

---

#### BLOCKER (1 finding)

| # | Slug | Vector | Cross-refs | File(s) / Lines |
|---|------|--------|-----------|-----------------|
| 01 | `ohttp-bypass-send-direct` | Privacy risks | P6-1, C-4 | `lib/src/ohttp_client.dart:52, 148–171` |

**01 — OHTTP bypass via silent `effectiveDirectBaseUrl` fallback**
`sendDirect()` carries a single doc line ("Send a direct HTTP request (for comparison)."), no `@Deprecated`, no `assert`, and no privacy-impact warning. `effectiveDirectBaseUrl` at line 52 silently resolves a null `directBaseUrl` to `gatewayBaseUrl` (the OHTTP relay). A developer who creates `OhttpGatewayConfig` with only required parameters and then calls `sendDirect()` issues plaintext HTTP to the relay — no encryption, no unlinkability — with no runtime signal. Escalated to BLOCKER because the silent fallback default is source-confirmed.

---

#### HIGH (21 findings)

| # | Slug | Vector | Cross-refs | File(s) / Lines |
|---|------|--------|-----------|-----------------|
| 02 | `enforce-https-scheme` | Privacy risks | C-1, P6-2 | `lib/src/ohttp_client.dart:43–51, 82, 116` |
| 03 | `restrict-target-authority` | Privacy risks | C-3 | `lib/src/ohttp_client.dart:97–104` |
| 04 | `zeroize-hpke-key-material` | Privacy risks | Phase 1 F-2, P6-3 | `lib/src/hpke.dart` (HpkeSenderContext fields) |
| 05 | `zeroize-ohttp-enc-exported-secret` | Privacy risks | TASK-O6, P6-4 | `lib/src/ohttp.dart:105–120, 196–207` |
| 06 | `fix-silent-drop-extra-kdf-aead-pairs` | Cryptographic correctness | TASK-O1, T-O1 | `lib/src/ohttp.dart:60–78` |
| 07 | `wrap-aead-auth-error` | Interoperability | TASK-O5, P6-7 | `lib/src/ohttp.dart:217–225` |
| 08 | `add-http-timeouts` | Network reliability | C-6 | `lib/src/ohttp_client.dart:81–83, 115–119` |
| 09 | `implement-keyconfig-caching` | KeyConfig management | C-5 | `lib/src/ohttp_client.dart:69–89` |
| 10 | `add-typed-error-hierarchy` | Network reliability | C-7, C-8 | `lib/src/ohttp_client.dart:84–87, 120–122` |
| 11 | `bhttp-bounds-check-rangeerror-to-formatexception` | Parser robustness | F2.5a, F2.5b, F2.6a | `lib/src/bhttp.dart:44–67, 173, 178, 187` |
| 12 | `bhttp-size-cap-headers-and-body` | Parser robustness | F2.4a, F2.4b, F2.4c, C-9 | `lib/src/bhttp.dart:164–187; lib/src/ohttp_client.dart:129–133` |
| 13 | `add-structured-observability-hooks` | Documentation | P6-5, P6-6, P6-8 | `lib/src/ohttp_client.dart:81–89, 115–122, 148–171` |
| 14 | `test-hpke-sequence-number-overflow` | Test suite | T-H1 | `test/hpke_test.dart` (after line 200) |
| 15 | `test-setupbases-invalid-public-key` | Test suite | T-H3 | `test/hpke_test.dart` |
| 16 | `test-varint-8byte-boundary` | Test suite | T-B1 | `test/bhttp_test.dart` (after line 68) |
| 17 | `test-bhttp-truncated-body` | Test suite | T-B2 | `test/bhttp_test.dart` (after line 185) |
| 18 | `test-ohttp-encap-decap-round-trip` | Test suite | T-O4 | `test/ohttp_test.dart` |
| 19 | `test-ohttp-aead-auth-failure` | Test suite | T-O3 | `test/ohttp_test.dart` (after line 163) |
| 20 | `test-ohttp-keyconfig-multi-suite` | Test suite | T-O1, T-O2 | `test/ohttp_test.dart` (after line 62) |
| 21 | `test-integration-gateway-stub` | Test suite | T-X1 | new `test/integration/` directory |
| 22 | `add-structured-observability-hooks` | Interoperability | T-X1 note | — |

> **Note:** Row 22 is a placeholder; T-X1 and finding 13 both motivate structured observability — merge into one file if the scope overlaps. The engineer creating task files should review and deduplicate #13 and #21–#22 if needed.

**Selected HIGH finding details:**

**02 — Enforce HTTPS scheme on `gatewayBaseUrl`**
RFC 9458 §1 requires TLS on the outer channel. `OhttpGatewayConfig` constructor accepts any URL scheme string with no validation. `http://` is silently accepted; the outer OHTTP ciphertext is transmitted in cleartext, enabling timing correlation by a network observer. Evidence: `lib/src/ohttp_client.dart:43–51`; RFC 9458 §1.

**04 — Zeroize HPKE ephemeral key material in `HpkeSenderContext`**
`key` (16 B AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) are retained as `final Uint8List` fields after `seal()` returns. Dart GC does not guarantee prompt collection. Evidence: `lib/src/hpke.dart` (HpkeSenderContext field declarations); wallet memory-dump attack surface.

**06 — Fix silent drop of extra KDF+AEAD pairs in `OhttpKeyConfig.parse`**
RFC 9458 §4.1 allows multiple KDF+AEAD pairs (`symLen > 4`). The parser at `ohttp.dart:67–71` reads exactly the first 4-byte pair and returns without inspecting remaining pairs or raising an error. A gateway that advertises the supported suite second would cause the client to select an unsupported suite without detection. Evidence: `ohttp.dart:60–78`; RFC 9458 §4.1.

**07 — Wrap `SecretBoxAuthenticationError` in a library-owned exception**
On AEAD auth failure, `aesGcm.decrypt()` throws `SecretBoxAuthenticationError` from `package:cryptography`, which is not re-exported by `lib/ohttp_dart.dart`. Callers must add a transitive `package:cryptography` import to catch it specifically, making `package:cryptography` an unintended part of the public API surface. Evidence: `ohttp.dart:217–225`; package:cryptography internal type.

**08 — Add timeouts to KeyConfig GET and gateway POST**
Both `http.Client.get()` at `ohttp_client.dart:81–83` and `http.Client.post()` at `ohttp_client.dart:115–119` have no `.timeout(Duration(...))` chained and no retry logic. A wallet can hang indefinitely on a non-responsive gateway. Evidence: `ohttp_client.dart:81–83, 115–119`.

**09 — Implement KeyConfig caching with TTL**
`OhttpClient` has no cache field (only `_httpClient` and `gateway`). Every `send()` invocation performs a fresh GET, creating two round trips per inner request. Evidence: `ohttp_client.dart:69–89`; no `OhttpKeyConfig` cache field.

**11 — Add BHTTP bounds-checking helper (convert `RangeError` to `FormatException`)**
All varint accesses at `bhttp.dart:51, 54, 61` and `sublist` calls at `bhttp.dart:173, 178, 187` throw `RangeError` (an `Error`, not an `Exception`) on truncated input. Callers using `try { } on FormatException` will miss truncation silently. A shared `_requireBytes(data, offset, n)` helper is the appropriate mitigation. Evidence: RFC 9292 §3.5.4; `bhttp.dart:44–67, 173, 178, 187`.

**12 — Add size caps on BHTTP header section and response body**
`parseResponse` at `bhttp.dart:164–182` accepts `headersLen` without an upper bound — an adversarial gateway declaring `headersLen = 2^30` forces a ~1 GiB allocation. Response body is similarly uncapped. The gateway response body at `ohttp_client.dart:129–133` is passed to `ohttpDecapsulate` without a size check. Evidence: RFC 9292 §3.5.3; `bhttp.dart:164–187`; `ohttp_client.dart:129–133`.

---

#### IMPROVEMENT (13 findings)

| # | Slug | Vector | Cross-refs | File(s) / Lines |
|---|------|--------|-----------|-----------------|
| 23 | `unify-unsupported-kem-exception` | Cryptographic correctness | TASK-O2, T-O5 | `lib/src/ohttp.dart:51, 82–85` |
| 24 | `add-serialize-request-body-size-cap` | Parser robustness | F2.7 | `lib/src/bhttp.dart:74–111` |
| 25 | `normalize-gateway-url-paths` | Network reliability | C-2 | `lib/src/ohttp_client.dart:82, 116, 154` |
| 26 | `lowercase-response-header-names` | Interoperability | C-10 | `lib/src/ohttp_client.dart:139` |
| 27 | `validate-bhttp-status-code-range` | Parser robustness | F2.3 | `lib/src/bhttp.dart:148–162` |
| 28 | `expand-empty-aad-comment` | Documentation | TASK-O3 | `lib/src/ohttp.dart:148–150` |
| 29 | `add-plain-hkdf-maintenance-comments` | Documentation | TASK-O4 | `lib/src/ohttp.dart:192–207` |
| 30 | `add-fuzz-property-based-tests` | Test suite | T-X2 | all test files |
| 31 | `improve-example-file-documentation` | Documentation | P6-11 | `example/ohttp_dart_example.dart` |
| 32 | `make-effective-direct-base-url-private` | Privacy risks | P6-13 | `lib/src/ohttp_client.dart:52` |
| 33 | `test-export-invalid-length` | Test suite | T-H2 | `test/hpke_test.dart` (after line 241) |
| 34 | `test-bhttp-unknown-framing-indicator` | Test suite | T-B3, T-B4 | `test/bhttp_test.dart` |
| 35 | `test-ohttp-edge-cases` | Test suite | T-O5, T-O6 | `test/ohttp_test.dart` |

**Selected IMPROVEMENT details:**

**23 — Unify exception types for unsupported KEM in `parse()` and `validate()`**
`parse()` at `:51` throws `FormatException`; `validate()` at `:82–85` throws `UnsupportedError` for the same "unsupported KEM" condition. Callers must catch two unrelated exception hierarchies. Consolidate to a single library-owned `OhttpUnsupportedSuiteException`. Evidence: `ohttp.dart:51, 82–85`.

**28 — Expand empty-AAD comment at `ctx.seal` call site**
The existing comment at `ohttp.dart:148` mentions the Go reference implementation but does not cite RFC 9458 §4.3 or explain why the deviation is intentional and safe. Evidence: `ohttp.dart:148–150`; RFC 9458 §4.3.

**29 — Add plain-HKDF maintenance-hazard comments in `ohttpDecapsulate`**
`HpkeSender.hkdfExtract` and `hkdfExpand` at `ohttp.dart:193, 196, 203` are plain RFC 5869 HKDF calls, not `LabeledExtract`/`LabeledExpand`. Replacing them with labeled variants would silently derive different key material. No warning comment is present. Evidence: RFC 9458 §4.4; `ohttp.dart:192–207`.

**32 — Make `effectiveDirectBaseUrl` private or remove from public API**
`effectiveDirectBaseUrl` is a public getter that silently resolves `null → gatewayBaseUrl`. Making it private reduces the blast radius of the P6-1 footgun (companion, not duplicate — P6-1 adds a runtime warning; this changes API surface). Evidence: `ohttp_client.dart:52`.

---

### §3 Eight-vector coverage map

| Vector | Task file #s |
|--------|-------------|
| Cryptographic correctness | 04, 05, 06, 23 |
| Interoperability | 07, 26 |
| Parser robustness | 11, 12, 24, 27 |
| Privacy risks | 01, 02, 03, 04, 05, 32 |
| Network reliability | 08, 09, 10, 25 |
| KeyConfig management | 09 |
| Test suite | 14–21, 30, 33, 34, 35 |
| Documentation | 13, 28, 29, 31 |

All eight vectors are represented. At least one BLOCKER (task 01) exists.

### §4 Logging constraints (must appear in task 13 and README context)

Any future logging added to `lib/` **must never** log:
1. Key material of any kind (`HpkeSenderContext.key`, `exporterSecret`, any HKDF output).
2. The ephemeral `enc` value — leakage alongside ciphertext breaks forward secrecy.
3. The inner request URL, path, method, headers, or body.
4. `targetAuthority` from `OhttpGatewayConfig`.
5. Response body or response headers (may contain wallet balances or PII).
6. `gatewayBaseUrl` at DEBUG or lower in production builds.

Safe to log: KeyConfig fetch success/failure (status code only), gateway POST status code (not body), `sendDirect()` invocation signal (no body, no path).

### §5 Out of scope (for README.md §Out of scope)

Verbatim from `vision.md §Out of scope`:
- Implementing any of the identified improvements (this is an investigation ticket only).
- Adding new dependencies to `pubspec.yaml`.
- Changing any source file under `lib/` or `test/`.
- Designing new APIs, data models, or architecture layers.
- Writing fuzz tests, integration tests, or end-to-end tests (those are follow-up tasks).
- Evaluating alternative cryptographic libraries or cipher suites.

## Implementation Notes

This is the deliverable phase. Create every task file and the README before marking any individual task `[x]`. Work sequentially: directory → BLOCKER files → HIGH files → IMPROVEMENT files → README.

Cross-reference the tasklist's **Test** block before marking 7.3–7.6 as done:
- `ls specs/.current/AW-2865/tasks/` must show `README.md` plus exactly one `.md` file per row in §2.
- The README index table must list every file with no gaps.

Do not create or edit any file in `lib/`, `test/`, or `example/`.
