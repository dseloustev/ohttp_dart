# AW-2865 Phase 3: OHTTP Layer Audit Against RFC 9458

Status: PRD_READY

## Context / Idea

Ticket AW-2865 is a read-only investigation of the `ohttp_dart` package to determine its production-readiness for use in a non-custodial crypto wallet. The investigation is organized into phases covering each layer of the four-layer stack.

Phase 3 targets Layer 3: `lib/src/ohttp.dart`. This file is the OHTTP encapsulation/decapsulation core. It parses the gateway's `KeyConfig`, constructs the HPKE info string, seals the pre-serialized BHTTP request with HPKE, exports a response secret, and performs response decapsulation using plain (unlabeled) HKDF. It sits between `hpke.dart` (Layer 1) and `ohttp_client.dart` (Layer 4), and every correctness or safety defect here directly affects the privacy guarantees of the OHTTP protocol.

**Inherited context (ticket-wide vision §2 severity model):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Phase dependencies:**
- Phase 1 (HPKE layer audit) is complete. Findings relevant to Phase 3: absence of zeroization for `key`, `baseNonce`, `exporterSecret` in `HpkeSenderContext`; `testKeyPair` injection hook reachable from production.
- Phase 2 (BHTTP layer audit) is complete. Findings relevant to Phase 3: `RangeError` propagation pattern from truncated BHTTP buffer — the same pattern recurs in the `OhttpKeyConfig` parser.

**No code is changed in this phase.** Every output is a finding mapped to a future follow-up task.

---

## Goals

1. Verify that `OhttpKeyConfig.parse` correctly reads the RFC 9458 §4.1 wire layout (key ID 1 B, KEM ID 2 B, public key `Npk` B, `symLen` 2 B, KDF+AEAD pairs) and confirm the exact behavior when the buffer is shorter than `symLen` implies.
2. Confirm that the parser silently drops extra KDF+AEAD cipher-suite pairs when `symLen > 4`, and establish the severity of this omission for a wallet that must enforce a specific suite.
3. Document the exception-type inconsistency: `parse()` throws `FormatException` for an unknown KEM ID, while `validate()` throws `UnsupportedError` for the same condition.
4. Verify the HPKE info string construction in `ohttpEncapsulate` against RFC 9458 §4.3 (`"message/bhttp request" || 0x00 || header`).
5. Confirm that the AAD passed to `ctx.seal` is empty (`[]`) and document this as an intentional deviation from some RFC 9458 readings — with the exact line reference and the Go interop basis that justifies it.
6. Verify that `ohttpDecapsulate` uses plain (unlabeled) `HKDF-Extract(salt=enc||response_nonce, ikm=exportedSecret)` and `HKDF-Expand` for key/nonce derivation per RFC 9458 §4.4, and confirm these are distinct code paths from the labeled `LabeledExtract`/`LabeledExpand` in `hpke.dart`.
7. Confirm that `SecretBoxAuthenticationError` (a `package:cryptography` type) propagates unwrapped to callers and establish the API-surface impact.
8. Confirm that `enc` and `exportedSecret` are not zeroized after `ohttpEncapsulate` returns, and assess this against the wallet threat model.
9. Produce a set of draft follow-up task entries for every confirmed finding, each with a file reference, line range, RFC 9458 section, and severity classification.

---

## User Stories

**US-1 — Security reviewer**
As a security reviewer preparing the package for wallet integration, I need a line-by-line audit of `ohttp.dart` against RFC 9458 so I can determine whether the OHTTP encapsulation/decapsulation layer deviates from the standard in ways that would break interoperability or compromise user privacy.

**US-2 — Engineering lead**
As an engineering lead, I need each finding to be classified by severity (BLOCKER / HIGH / IMPROVEMENT) and mapped to a specific code location and RFC section, so I can prioritize follow-up tasks and assign them to engineers without further triage.

**US-3 — Wallet integrator**
As a developer integrating `ohttp_dart` into a non-custodial crypto wallet, I need to know whether the OHTTP layer introduces risks through silent cipher-suite downgrade, unhandled decryption failures, or residual key material in memory — each of which could expose transaction data or wallet keys.

**US-4 — Protocol compatibility engineer**
As an engineer responsible for interoperability with third-party OHTTP gateways, I need to understand exactly where the Dart implementation deviates from RFC 9458 (specifically the empty-AAD choice and the plain-HKDF response path) so I can document these constraints and confirm compatibility with target gateway implementations before production deployment.

---

## Main Scenarios

### Scenario 1 — `OhttpKeyConfig.parse` wire-layout verification (tasks 3.2–3.3)
Analyst reads `OhttpKeyConfig.parse` and traces the byte-offset arithmetic against the RFC 9458 §4.1 wire layout table. Confirms that `key_id`, `kem_id`, `public_key`, and `sym_algorithms` are read from the correct byte offsets. Then traces the behavior when `symLen > 4`: confirms that only the first 4-byte KDF+AEAD pair is read and that subsequent pairs are not validated or surfaced. Records the exact line where the loop (or single read) terminates, along with the RFC section that defines multi-suite advertisement.

### Scenario 2 — Malformed-buffer `RangeError` confirmation (task 3.4)
Analyst constructs a mental model of a KeyConfig blob where `symLen` claims more bytes than the buffer contains. Traces the code path to the exact Dart list-access line that raises `RangeError` rather than `FormatException`. Records the throw site and notes the cross-layer impact: the same pattern was confirmed in Phase 2's BHTTP audit; the OHTTP layer has an independent instance of the same issue.

### Scenario 3 — Exception-type inconsistency documentation (task 3.5)
Analyst confirms that `parse()` throws `FormatException` when `kemId != 0x0020`, and that `validate()` throws `UnsupportedError` for the same condition. Records the exact lines of both throws and the specific inconsistency: callers who catch `FormatException` to handle "unknown KEM" will miss the `UnsupportedError` path, creating an unhandled exception risk.

### Scenario 4 — HPKE info string construction verification (task 3.6)
Analyst traces `ohttpEncapsulate` and locates the info string assembly. Confirms the construction is `"message/bhttp request" || 0x00 || header` where `header` is the 7-byte OHTTP request header (`key_id || kem_id || kdf_id || aead_id`). Cross-references with RFC 9458 §4.3. Notes the exact line and any deviations from the RFC's prescribed construction.

### Scenario 5 — Empty AAD confirmation and deviation documentation (task 3.7)
Analyst locates the `ctx.seal(aad=[], ...)` call in `ohttpEncapsulate`. Confirms the AAD is an empty list. Cross-references RFC 9458 §4.3 wording on AAD construction. Documents this as an intentional deviation (matching Go reference implementation), citing the CLAUDE.md note at `ohttp.dart:149`. The remediation is an inline comment at the `ctx.seal(aad=[], ...)` call site citing the Go reference — no separate ADR or interoperability document is required. Records the deviation as an `IMPROVEMENT` finding.

### Scenario 6 — Response decapsulation HKDF path verification (tasks 3.8–3.9)
Analyst traces `ohttpDecapsulate` through the response-nonce extraction, `HKDF-Extract(salt=enc||response_nonce, ikm=exportedSecret)`, and the two `HKDF-Expand` calls for `key` and `nonce`. Confirms these are the plain unlabeled HKDF calls (`HpkeSender.hkdfExtract`/`hkdfExpand`) and not the labeled `LabeledExtract`/`LabeledExpand` variants. Confirms the response AEAD also uses empty AAD. Cross-references RFC 9458 §4.4 for the prescribed derivation. Records verdict: compliant or deviant, with line references.

### Scenario 7 — `SecretBoxAuthenticationError` API-surface leakage confirmation (task 3.10)
Analyst confirms that AEAD decryption failure in `ohttpDecapsulate` surfaces as `SecretBoxAuthenticationError`, a type from `package:cryptography`. Notes that callers must import `package:cryptography` to catch this specific type, creating an unintended hard dependency on an internal package in the public API surface. Records the throw site. The remediation wraps the error in a library-owned exception defined in `ohttp.dart` — the layer closest to the decap source — so callers interact only with types owned by this library.

### Scenario 8 — Zeroization absence confirmation (cross-layer, tasks 3.6–3.7 follow-on)
Analyst confirms that `OhttpEncapsulateResult.enc` (the ephemeral X25519 public key) and `OhttpEncapsulateResult.exportedSecret` are `Uint8List` fields with no explicit overwrite (zero-fill) before the object is garbage-collected. Cross-references the Phase 1 finding on `HpkeSenderContext` field zeroization to note that the same pattern recurs at the OHTTP layer. Records as `HIGH` with a note that Dart VM does not guarantee zero-writes prevent access via GC roots or JIT-optimized memory — the finding acknowledges this platform limitation and recommends best-effort zeroization (explicit zero-fill before nulling references) as the actionable remediation, especially for AOT-compiled deployments.

### Scenario 9 — Draft task entry compilation (task 3.11)
For each confirmed finding from Scenarios 1–8, the analyst produces a draft task entry containing: finding description, affected file and line range, RFC 9458 section (or Go interop reference), severity classification, and a proposed task title for Iteration 7 consolidation.

---

## Success / Metrics

| Criterion | Definition of done |
|---|---|
| All RFC 9458 §4 subsections covered | Tasks 3.2–3.9 each produce a positive ("matches RFC") or negative ("deviates: …") verdict for every inspection target. No subsection from the scrutiny table in `vision.md §4` for `ohttp.dart` is skipped. |
| Empty-AAD and plain-HKDF findings fully cited | Both the empty-AAD finding and the plain-HKDF-vs-labeled-HKDF finding cite the exact line in `ohttp.dart` plus the RFC 9458 section that motivates the concern. |
| `OhttpKeyConfig` parser gaps documented | The silent-extra-pairs behavior and the `RangeError`-on-short-buffer behavior each have a finding with an exact line reference. |
| Severity assigned to every finding | Every finding carries exactly one of `BLOCKER`, `HIGH`, or `IMPROVEMENT`. |
| No code changes | `git diff HEAD -- lib/ test/` is empty at the end of this phase. |
| Draft tasks independently actionable | Each draft task can be assigned to a single engineer without requiring another in-flight task from this list to be completed first. |
| Cross-layer references included | Findings that compound Phase 1 or Phase 2 risks (zeroization, `RangeError` propagation) explicitly reference the earlier phase findings. |

---

## Constraints and Assumptions

1. **Read-only.** No file in `lib/` or `test/` is modified during this phase.
2. **Fixed cipher suite.** Only DHKEM(X25519, HKDF-SHA256) + AES-128-GCM (KEM `0x0020`, KDF `0x0001`, AEAD `0x0001`) is in scope. Evaluating alternative suite advertisement or migration is out of scope.
3. **Empty AAD is intentional.** Per CLAUDE.md, the empty AAD in `ctx.seal` is a deliberate design choice matching the Go reference implementation and passing the bundled test suite. It must not be reclassified as a defect. The finding documents it as an intentional deviation. Remediation is a single inline comment at the `ctx.seal(aad=[], ...)` call site — no ADR is required.
4. **Plain HKDF in response decap must not be unified with labeled HKDF.** RFC 9458 §4.4 prescribes plain (unlabeled) HKDF for response key derivation. Any future refactoring that substitutes `LabeledExtract`/`LabeledExpand` would constitute a protocol break. This constraint is recorded as a maintenance hazard, not a defect.
5. **`package:cryptography` primitives assumed correct.** The audit does not re-verify AES-128-GCM, HMAC-SHA-256, or X25519. It verifies only how `ohttp.dart` composes them.
6. **Sender-only flow.** RFC 9458 defines both client (encapsulate) and gateway (decapsulate) roles. Only the client role is implemented in `ohttp.dart`; the gateway role is out of scope.
7. **Receiver-side KeyConfig negotiation out of scope.** Multi-suite advertisement (when a gateway offers more than one KDF+AEAD pair) is identified as a parser gap but full multi-suite negotiation logic is not evaluated or designed in this phase.
8. **`SecretBoxAuthenticationError` wrapping target is `ohttp.dart`.** The wrapper exception is defined in `ohttp.dart` (closest to the decap source), not in `ohttp_client.dart` or a shared exceptions file. This keeps the fix minimal and co-located with the throw site.
9. **Zeroization finding acknowledges Dart VM limitations.** The `HIGH` severity for zeroization absence reflects wallet threat model relevance. The Dart VM does not guarantee that explicit zero-writes prevent access via GC roots or JIT-optimized memory; the finding records this limitation explicitly and recommends best-effort zeroization as the actionable remediation.

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Silent drop of extra KDF+AEAD pairs means the client always uses the first advertised suite regardless of gateway preference; if the gateway rotates to a preferred suite, the client silently applies downgrade negotiation — a potential cipher-suite downgrade risk | HIGH | Scenario 1 captures the exact parser behavior; should fix before wallet use but is not an unconditional deployment block; remediation task would add explicit multi-suite validation or rejection |
| `RangeError` on malformed KeyConfig propagates unwrapped; callers cannot distinguish a tampered gateway response from a logic error, creating unreliable error handling in a wallet that must distinguish security failures from transient errors | HIGH | Scenario 2 confirms the throw site; mirrors Phase 2 finding — cross-layer remediation task expected |
| `SecretBoxAuthenticationError` leaking `package:cryptography` into the public API surface means callers cannot catch authentication failures without a direct dependency on an internal package, creating a brittle integration surface that breaks if `package:cryptography` is updated or replaced | HIGH | Scenario 7 captures the exact type; remediation wraps in a library-owned exception defined in `ohttp.dart` |
| Exception-type inconsistency (`FormatException` vs. `UnsupportedError`) for the same "unknown KEM" condition means catch clauses cannot reliably handle this error, creating uncaught exception risk in wallet error-handling code | IMPROVEMENT | Scenario 3 documents both throw sites; remediation consolidates to one exception type |
| `enc` and `exportedSecret` not zeroized after `ohttpEncapsulate` returns; ephemeral key material persists in heap memory until GC, relevant to wallet threat model (memory dump, cold-boot on mobile); Dart VM does not guarantee zero-writes are effective under all GC/JIT conditions | HIGH | Scenario 8 confirms the absence; cross-references Phase 1 zeroization finding; best-effort zeroization (explicit zero-fill before nulling references) recommended, especially for AOT-compiled deployments |
| Any future refactoring that unifies response-decap HKDF with the labeled variants in `hpke.dart` would silently produce different key material and break decryption with no compile-time signal | HIGH | Scenario 6 documents the explicit distinction; recorded as a maintenance hazard task with a code-comment remediation |
| Audit findings may be incomplete if `ohttp.dart` has been modified since the vision was written | LOW | Task 3.1 (full outline read before targeted slices) serves as a completeness gate |

---

## Open Questions

All questions have been resolved. See resolved questions below.

---

## Resolved Questions

1. **Silent multi-suite drop severity** — Resolved: **HIGH**. The parser silently ignoring extra KDF+AEAD pairs should be fixed before wallet use, but is not an unconditional deployment block (not BLOCKER). The risk and remediation task are captured in Scenario 1 and the Risks table.

2. **Empty-AAD deviation documentation scope** — Resolved: an **inline comment** at the `ctx.seal(aad=[], ...)` call site (citing the Go reference) is sufficient. No separate ADR or interoperability document is required. See Constraint 3 and Scenario 5.

3. **`SecretBoxAuthenticationError` wrapping layer** — Resolved: wrap in **`ohttp.dart`**, the layer closest to the decap source. The new exception type is defined and thrown in `ohttp.dart`, not in `ohttp_client.dart` or a shared file. See Constraint 8 and Scenario 7.

4. **Zeroization severity** — Resolved: **HIGH**, with an explicit note about Dart VM zeroization limitations (no guarantee that zero-writes prevent access via GC roots or JIT-optimized memory). Best-effort zeroization (explicit zero-fill before nulling references) is the recommended remediation, particularly for AOT-compiled deployments. See Constraint 9 and Scenario 8.
