# QA Report — AW-2865 Phase 1: HPKE Layer Audit Against RFC 9180

**Date:** 2026-05-21
**Phase:** 1
**Scope:** `lib/src/hpke.dart` audit against RFC 9180. Read-only — no code in `lib/` or `test/` was modified.
**Verdict: PASS**

---

## Phase Scope

Phase 1 targeted the bottom layer of the four-layer stack (`hpke.dart`). The QA plan verifies that:

1. All PRD success criteria are met by the research deliverable.
2. Every task in `tasks.md` is complete and traceable to a concrete finding or explicit "no issue found" note.
3. The read-only constraint is proven by git state.
4. Findings are consistent with resolved questions and the vision §2 severity model.
5. The seven draft Jira tasks are independently actionable and carry the required metadata.

This QA report does not re-run code, re-read RFC text, or re-verify cryptographic algebra. It verifies the audit artifact against the PRD acceptance criteria and the review sign-off.

---

## PRD Acceptance Criteria — Verification

### AC-1: All RFC 9180 sections covered

**Criterion:** Tasks 1.2–1.6 each produce a positive ("matches RFC") or negative ("deviates: …") verdict with a cited RFC section and line reference in `hpke.dart`. No section from the labeled-KDF, KEM Encap, key-schedule, nonce-derivation, and export sub-sections is skipped.

| RFC Section | Task | Research Verdict | Status |
|---|---|---|---|
| §4 — LabeledExtract | 1.2 | "matches RFC" — suite ID split correct, label embedding correct, HMAC-SHA-256 invocation correct | PASS |
| §4 — LabeledExpand | 1.2 | "matches RFC" — 2-byte big-endian length prefix correct, concat order correct | PASS |
| §4 — hkdfExtract empty-salt handling | 1.2 | "matches RFC" — 32 zero-bytes when salt is empty, matching RFC 5869 §2.2 | PASS |
| §4 — hkdfExpand T(i) construction | 1.2 | "matches RFC" — single-byte counter, truncated to `length` bytes | PASS |
| §4.1 — KEM Encap (DHKEM X25519) | 1.3 | "matches RFC" — DH, enc, kem_context byte order, ExtractAndExpand labels all correct | PASS |
| §5.1 — Key Schedule (Base Mode) | 1.4 | "matches RFC" — mode_base=0, psk_id_hash, info_hash, secret, key, base_nonce, exp all correct | PASS |
| §5.2 — Nonce derivation and sequence counter | 1.5 | "matches RFC for single-use operating mode" — conservative overflow guard noted as F-4 | PASS |
| §5.3 — Export | 1.6 | "matches RFC" — LabeledExpand with HPKE suite ID, label "sec", exporterSecret as PRK | PASS |

All eight sub-sections are addressed. No gap. **AC-1: PASS.**

---

### AC-2: Scrutiny table fully addressed

**Criterion:** Every row in the vision §4 scrutiny table for `hpke.dart` has a corresponding finding or explicit "no issue found" note.

| Scrutiny concern | Finding | Explicit note |
|---|---|---|
| Single-suite hard-coding (DHKEM(X25519) + AES-128-GCM) | Research Limitations & Risks #1 (line 260) — design decision, not a defect; documented for wallet integration | Present |
| Sequence-number overflow not guarded with a typed exception | F-3 (IMPROVEMENT) — `StateError` used; typed exception recommended. F-4 (IMPROVEMENT) — 32-bit guard tighter than RFC. F-6 (IMPROVEMENT) — no negative test for this path. | Present |
| `testKeyPair` injection in production code path | F-1 (HIGH) — parameter present in public `setupBaseS` and threaded through `ohttpEncapsulate` | Present |

**AC-2: PASS.**

---

### AC-3: Severity assigned to every finding

**Criterion:** Every finding carries exactly one of `BLOCKER`, `HIGH`, or `IMPROVEMENT`.

| Finding | Severity | Assigned |
|---|---|---|
| F-1 `testKeyPair` injection | HIGH | Yes |
| F-2 Key-material zeroization | IMPROVEMENT | Yes |
| F-3 Untyped `StateError` on `_seq` overflow | IMPROVEMENT | Yes |
| F-4 32-bit nonce XOR cap vs RFC 96-bit maximum | IMPROVEMENT | Yes |
| F-5 Public `hkdfExtract` / `hkdfExpand` on API surface | IMPROVEMENT | Yes |
| F-6 No negative test for `_seq` overflow path | IMPROVEMENT | Yes |
| F-7 No integration test for `seal` with empty AAD | HIGH | Yes |

All 7 findings carry exactly one severity label. Distribution: 2 HIGH, 5 IMPROVEMENT, 0 BLOCKER. Matches the reported distribution in the review.

**AC-3: PASS.**

---

### AC-4: No code changes

**Criterion:** `git diff HEAD -- lib/ test/` is empty at the end of this phase.

Evidence from `review.md` (lines 95–107):

```
$ git diff HEAD -- lib/ test/ pubspec.yaml example/
(empty)

$ git status --short
 M specs/.current/.active_ticket
 M specs/.current/AW-2865/phase-1/tasks.md
?? specs/.current/AW-2865/phase-1/plan.md
?? specs/.current/AW-2865/phase-1/prd.md
?? specs/.current/AW-2865/phase-1/research.md
```

All changes are confined to `specs/.current/AW-2865/phase-1/` (spec artifacts) and the `.active_ticket` pointer. No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` is modified. The code review agent independently verified this.

**AC-4: PASS.**

---

### AC-5: Draft tasks independently actionable

**Criterion:** Each draft task can be assigned to a single engineer without requiring another in-flight task from this list to be completed first.

| Task | Single-file target | Independent? |
|---|---|---|
| TASK-1: Remove `testKeyPair` from production signature | `hpke.dart:47–48`, `ohttp.dart:134–146` | Yes |
| TASK-2: Add key-material zeroization | `hpke.dart:259–262`, `ohttp.dart:113–114` | Yes |
| TASK-3: Typed exception for `_seq` overflow | `hpke.dart:275–277` | Yes |
| TASK-4: Extend nonce XOR to full 12 bytes | `hpke.dart:275–285` | Yes |
| TASK-5: Restrict `hkdfExtract`/`hkdfExpand` from public API | `hpke.dart:225–254`, `lib/ohttp_dart.dart` | Yes |
| TASK-6: Add negative test for `_seq` overflow | `test/hpke_test.dart`, `hpke.dart:274–277` | Yes |
| TASK-7: Integration test for `seal` with empty AAD | `test/hpke_test.dart`, `ohttp.dart:149` | Yes — blocked only on gateway availability, not on any TASK-1..6 |

Note: TASK-3 and TASK-4 address overlapping code in `_computeNonce`. If assigned concurrently, a minor merge conflict is expected (both modify lines 275–285). They can be ordered sequentially without semantic dependency, or merged into a single task. This is a scheduling note, not a criteria failure.

**AC-5: PASS.**

---

## Task Completeness Verification

All 10 tasks in `tasks.md` are marked `[x]` complete:

| Task | Description | Finding produced |
|---|---|---|
| 1.1 | Full file outline and targeted slice reads | Baseline for all subsequent tasks |
| 1.2 | LabeledExtract / LabeledExpand vs §4 | §4 verdict (matches RFC), hkdfExtract/hkdfExpand verdicts (matches RFC) |
| 1.3 | KEM Encap vs §4.1 | §4.1 verdict (matches RFC) |
| 1.4 | Key schedule vs §5.1 | §5.1 verdict (matches RFC) |
| 1.5 | Nonce derivation vs §5.2 | §5.2 verdict (matches RFC, conservative guard noted) |
| 1.6 | Export vs §5.3 | §5.3 verdict (matches RFC) |
| 1.7 | `_seq` overflow guard typed exception check | F-3 (IMPROVEMENT) |
| 1.8 | `testKeyPair` injection reachability | F-1 (HIGH) |
| 1.9 | Zeroization absence | F-2 (IMPROVEMENT) |
| 1.10 | Draft task entry compilation | TASK-1 through TASK-7 |

**All 10 tasks: complete and traceable.**

---

## Positive Scenarios

### PS-1: RFC 9180 §4 labeled KDF correctness

The `LabeledExtract` and `LabeledExpand` functions in `hpke.dart` correctly:
- Prefix IKM with `"HPKE-v1"` || suiteId || label as specified by RFC 9180 §4.
- Use the KEM suite ID (`"KEM" || 0x00 0x20`) for KEM-scoped calls and the HPKE suite ID (`"HPKE" || 0x00 0x20 || 0x00 0x01 || 0x00 0x01`) for key-schedule calls.
- Prefix info with `I2OSP(L, 2)` in `LabeledExpand`, using correct big-endian encoding.
- Apply 32 zero-byte fallback salt for empty salt inputs (RFC 5869 §2.2 compliance).
- Correctly compute HKDF-Expand with single-byte counter and truncated output.

These are substantiated by RFC 9180 Appendix A.1 vector tests passing with `testKeyPair` injection (confirmed by existing `test/hpke_test.dart` test suite).

### PS-2: RFC 9180 §4.1 KEM Encap correctness

The `_kemEncap` function correctly follows the DHKEM(X25519) Encap procedure: generates or accepts an ephemeral key pair, serializes `enc` as the 32-byte X25519 public key, computes the DH shared value, assembles `kem_context = enc || recipientPkBytes` in the correct byte order, and calls `ExtractAndExpand` with `"eae_prk"` and `"shared_secret"` labels.

### PS-3: RFC 9180 §5.1 key schedule correctness

`setupBaseS` correctly derives `psk_id_hash`, `info_hash`, `ks_context`, `secret`, `key` (Nk=16), `base_nonce` (Nn=12), and `exporter_secret` (Nh=32) using `mode_base = 0` and empty PSK/PSK-ID for Base Mode. All label strings and input ordering match RFC §5.1 pseudo-code.

### PS-4: RFC 9180 §5.2 nonce derivation correctness

`HpkeSenderContext.seal` correctly computes `nonce = baseNonce XOR I2OSP(_seq, 12)` via 4 big-endian byte-level XOR operations into the last 4 bytes of a copy of `baseNonce`. Post-increment sequencing is correct (counter read before XOR, incremented after). For all `_seq` values in `[0, 2^32 - 1]` — the only values reachable in the current implementation — the result is bitwise equivalent to a full 12-byte `I2OSP(_seq, 12)` XOR, because the high 8 bytes of `I2OSP(_seq, 12)` are always zero in this range.

### PS-5: RFC 9180 §5.3 export correctness

`HpkeSenderContext.export` correctly calls `LabeledExpand(exporterSecret, "sec", exporterContext, length)` with the HPKE suite ID.

### PS-6: RFC 9180 A.1 vector test pass (existing)

The existing `test/hpke_test.dart` suite exercises `SetupBaseS`, `seal` (seq=0 and seq=1), and `export` using RFC 9180 Appendix A.1 test vectors via the `testKeyPair` injection hook. All pass. This is an independent correctness signal corroborating the line-by-line audit verdicts.

---

## Negative Scenarios and Edge Cases

### NS-1: `testKeyPair` injection in production builds (F-1, HIGH)

Any caller of `HpkeSender.setupBaseS` or `ohttpEncapsulate` can supply a fixed, known, or weak ephemeral key by passing `testKeyPair`. The Dart compiler and runtime provide no barrier. In a wallet context, a compromised caller or a supply-chain attack that replaces a dependency could silently downgrade X25519 to a deterministic ephemeral key, breaking forward secrecy. No test currently exercises this negative path to assert that production builds reject `testKeyPair != null`.

**Gap:** No runtime guard, no `@visibleForTesting` annotation, no assert. The attack surface extends to `ohttp.dart:134–146` because `ohttpEncapsulate` also threads the parameter.

### NS-2: `_seq` overflow — untyped exception and truncated nonce space (F-3, F-4, IMPROVEMENT)

The overflow guard at `hpke.dart:275` throws `StateError`, which collides with many unrelated Dart APIs. A caller relying on typed catch semantics cannot distinguish this from, for example, a `StateError` thrown by a stream already listened to. Additionally, the nonce XOR covers only 4 of 12 bytes, capping the usable sequence space at `2^32 - 1` rather than the RFC maximum of `2^96 - 1`. For the confirmed single-use context this is safe, but the restriction is undocumented in the public API.

**Gap:** No typed exception class; no test for the overflow boundary; no API documentation stating the single-use constraint.

### NS-3: Key-material persistence in heap (F-2, IMPROVEMENT)

`HpkeSenderContext.key`, `baseNonce`, and `exporterSecret` — as well as `OhttpEncapsulateResult.exportedSecret` in `ohttp.dart` — are `final Uint8List` fields with no `dispose()` method. After `seal()` and `export()` complete, these arrays remain allocated until GC collects the context object. In a wallet process under memory-dump attack, the per-request OHTTP session key (16 B AES-128) and exporter secret remain recoverable from heap, enabling decryption of captured inner HTTP content (traffic deanonymization).

**Gap:** No `dispose()` method, no documentation of key residency, no test asserting post-use state.

### NS-4: Unlabeled HKDF on public API surface (F-5, IMPROVEMENT)

`HpkeSender.hkdfExtract` and `hkdfExpand` are plain (unlabeled) HKDF variants exposed as public static methods and re-exported from `lib/ohttp_dart.dart`. A future maintainer or consumer could invoke them in place of `LabeledExtract`/`LabeledExpand`, silently removing the domain-separation prefix without any compiler error.

**Gap:** No `@internal` annotation, no package-private boundary, no documentation distinguishing them from the labeled variants.

### NS-5: Empty-AAD integration gap (F-7, HIGH)

The RFC 9180 A.1 vector tests confirm HPKE correctness using non-empty AAD values (`"Count-0"`, `"Count-1"`). The actual OHTTP usage at `ohttp.dart:149` calls `ctx.seal(Uint8List(0), binaryRequest)` — empty AAD. No test exercises this combination end-to-end or confirms interoperability with a real OHTTP gateway decapsulator.

**Gap:** The happy-path HPKE tests and the empty-AAD OHTTP path are never tested together. An interoperability regression (e.g., a gateway expecting non-empty AAD) would not be caught by the current test suite.

### NS-6: No test for `_seq` overflow boundary (F-6, IMPROVEMENT)

The `hpke_test.dart` suite tests `seal` at seq=0 and seq=1. There is no test that pre-sets `_seq` to `(1 << 32) - 1`, calls `seal()` once (the last valid call), then calls `seal()` again and asserts the expected exception type is thrown. A refactoring of `_computeNonce` could silently break the overflow guard.

**Gap:** No boundary test, no regression coverage.

### NS-7: `export()` length bound absent (New Technical Question #4)

RFC 5869 §2.3 constrains `L <= 255 * HashLen = 8160` for SHA-256. `HpkeSenderContext.export` accepts an arbitrary positive `length` with no upper-bound check. Passing `length > 8160` would require more than 255 HKDF-Expand rounds, which is undefined behavior per RFC 5869. In practice, the OHTTP call uses `length = 16`, so this is not triggered, but the absence of a guard means misuse goes undetected.

**Status:** Left as New Technical Question #4 in `research.md`. Not promoted to a numbered finding in Phase 1. Recommended for explicit deferral to Phase 3 or promotion to F-8 to avoid loss during Jira import.

---

## Automated Test Coverage Assessment

| Area | Existing coverage | Gap identified |
|---|---|---|
| RFC 9180 A.1 vectors (`SetupBaseS`, `seal` seq=0/1, `export` × 3 contexts) | `hpke_test.dart` — all pass | No gap for correctness path |
| `_seq` overflow boundary | None | F-6: add boundary test |
| `seal` with empty AAD + gateway decapsulation | None | F-7: add integration / round-trip test |
| `testKeyPair` production guard | None | Part of TASK-1 remediation |
| `export()` length bound guard | None | Not yet a numbered finding; defer to Phase 3 or add guard with test |
| `hkdfExtract` / `hkdfExpand` API boundary misuse | None | Part of TASK-5 remediation |

---

## Manual Checks Needed

The following items cannot be verified by automated tests and require human or tooling-assisted review:

1. **Jira import readiness.** TASK-1 through TASK-7 must be reviewed by an engineering lead to confirm priority assignments (HIGH vs IMPROVEMENT) align with the wallet threat model before import. In particular, whether F-7 (empty-AAD integration gap, HIGH) is filed as a Phase 1 task or deferred to Phase 3 as a Phase 3 item.

2. **Resolution of New Technical Question #4.** The `export()` length guard question must be explicitly resolved — either promote to F-8 / TASK-8 or record a Phase 3 deferral note in `research.md`. Leaving it in "New Technical Questions" risks loss during Jira triage.

3. **F-4 finding row clarification (nice-to-have).** The review recommends adding a one-line clarifier in F-4's findings-table row stating that for `_seq < 2^32` the truncated XOR is bitwise equivalent to the full 12-byte XOR. This prevents a reader who skips the §5.2 verdict from misreading F-4 as a correctness defect.

4. **F-7 scope placement (nice-to-have).** The review recommends either moving F-7 to a "Cross-Phase References" sub-table or downgrading its Phase 1 severity to IMPROVEMENT with a pointer to Phase 3. This is a presentation choice with no effect on correctness.

5. **`package:cryptography` version pinning.** New Technical Question #2 flags the dependency version range for CVE and breaking-change review. This is out of Phase 1 scope but must be captured in the backlog before wallet integration.

6. **Web/WASM integer semantics.** New Technical Question #3 flags `dart2js` 53-bit integer behavior for `_seq` arithmetic. If the package targets Flutter Web or WASM, manual verification of `(1 << 32)` evaluation and `_seq` arithmetic on those runtimes is required.

---

## Phase-Specific Risk Zone

| Risk | Likelihood | Impact | Phase 1 status |
|---|---|---|---|
| `testKeyPair` injection abused in production (F-1, HIGH) | Low (requires deliberate caller action) | High (forward secrecy broken for affected requests) | Captured as TASK-1; must close before wallet use |
| Empty-AAD integration gap causes gateway interop failure (F-7, HIGH) | Medium (untested path) | High (OHTTP request fails at gateway) | Captured as TASK-7; scope placement to be resolved per manual check #1 |
| `_seq` overflow silently swallowed by generic `StateError` catch (F-3, IMPROVEMENT) | Very low (single-use context; overflow unreachable) | Medium (silent HPKE nonce reuse if multi-use introduced) | Captured as TASK-3; low priority given confirmed single-use context |
| Key-material heap residency exploited via memory dump (F-2, IMPROVEMENT) | Low (requires process compromise) | Medium (traffic deanonymization for one request) | Captured as TASK-2; threat model does not elevate to HIGH |
| `hkdfExtract`/`hkdfExpand` misuse by future maintainer (F-5, IMPROVEMENT) | Low (no external consumers today) | High (domain-separation broken silently) | Captured as TASK-5 |
| `export()` length bound violation (uncaptured) | Very low (OHTTP uses length=16) | Low (undefined HKDF behavior, not a security break at current call sites) | Unresolved in Phase 1; defer to Phase 3 |

---

## Final Verdict

**PASS**

All five PRD acceptance criteria are met:

- **AC-1 PASS** — All RFC 9180 sections (§4, §4.1, §5.1, §5.2, §5.3) are covered with per-section verdicts and `hpke.dart` line references.
- **AC-2 PASS** — All three vision §4 scrutiny-table items (single-suite hard-coding, `_seq` overflow, `testKeyPair` injection) are addressed with findings or explicit notes.
- **AC-3 PASS** — All 7 findings carry exactly one severity label; distribution (2 HIGH, 5 IMPROVEMENT, 0 BLOCKER) is consistent with resolved questions and the review sign-off.
- **AC-4 PASS** — `git diff HEAD -- lib/ test/` is empty; read-only constraint is satisfied, verified independently by the code review agent.
- **AC-5 PASS** — All 7 draft Jira tasks are independently actionable with single-file targets and concrete remediation directions.

The HPKE layer implementation matches RFC 9180 for all audited sections within the single-use Base Mode Sender constraint. No BLOCKER was found. Two HIGH-severity items (F-1 `testKeyPair` injection, F-7 empty-AAD integration gap) require remediation before production wallet use. Five IMPROVEMENT items are recommended for the follow-up backlog.

Three optional polish items from the code review (F-4 clarifier, F-7 scope placement, New Technical Question #4 resolution) are recorded above and should be addressed before Phase 1 artifacts are closed and archived.

Phase 1 is ready to close.
