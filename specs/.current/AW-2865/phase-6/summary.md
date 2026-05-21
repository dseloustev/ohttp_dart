# AW-2865 Phase 6 Summary: Privacy Risk Review, Observability Gaps Audit, and Documentation Review

**Ticket:** AW-2865
**Phase:** 6 of 7
**Date completed:** 2026-05-21
**Status:** COMPLETE — QA verdict: PASS

---

## Phase Goal and Scope

Phase 6 is the final investigation-phase audit of the AW-2865 ticket. It is a read-only, static-source review — no file in `lib/`, `test/`, or `example/` was modified.

Rather than auditing a single implementation layer (as Phases 1–5 did), Phase 6 performs three cross-cutting reviews:

1. **Privacy risk review** — whether the package's current state preserves OHTTP's unlinkability guarantee when used in a non-custodial crypto wallet, and what gaps undermine it.
2. **Observability gaps audit** — cataloging the five structured gaps and six logging constraints defined in `vision.md §7`.
3. **Documentation review** — `example/ohttp_dart_example.dart` and `pubspec.yaml` for documented limitations, version pinning integrity, and absence of native/FFI dependencies.

The phase produced 13 draft task entries (P6-1 through P6-13) for Phase 7 to consolidate with findings from all prior phases.

---

## Key Findings

Severity distribution: **1 BLOCKER, 7 HIGH, 5 IMPROVEMENT**.

### BLOCKER (1)

**P6-1 — OHTTP bypass via silent `effectiveDirectBaseUrl` fallback**
File: `lib/src/ohttp_client.dart:52, 148–171`

`sendDirect()` carries a single doc comment ("Send a direct HTTP request (for comparison)."), no `@Deprecated` annotation, no `assert`, and no privacy-impact warning. The `effectiveDirectBaseUrl` getter (`directBaseUrl ?? gatewayBaseUrl`) silently resolves a null `directBaseUrl` — which is the default when no explicit direct path is configured — to `gatewayBaseUrl`. A developer who creates `OhttpGatewayConfig` with only the required parameters and then calls `sendDirect()` issues a plain HTTP request to the OHTTP relay with no OHTTP encryption and no unlinkability guarantee, and receives no runtime signal that OHTTP was bypassed. Escalated from HIGH (Phase 4 Finding C-4) to BLOCKER because Phase 6 source-confirmed that the silent fallback is the default code path.

### HIGH (7)

**P6-2 — No `https`-scheme enforcement on outer channel**
File: `lib/src/ohttp_client.dart:43–51`
The `OhttpGatewayConfig` constructor accepts any `gatewayBaseUrl` string with no scheme validation. Plain HTTP is silently accepted; the outer OHTTP ciphertext is then transmitted in cleartext, enabling timing correlation by a network observer. RFC 9458 §1 requires TLS on the outer channel. Cross-ref: Phase 4 C-1.

**P6-3 — Key material persistence in Dart heap (`hpke.dart` scope)**
File: `lib/src/hpke.dart:258–270`
`HpkeSenderContext` retains `key` (16 B AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) as `final Uint8List` fields after `seal()` returns. No explicit zeroization; Dart GC does not guarantee prompt collection. Cross-ref: Phase 1 F-2.

**P6-4 — Key material persistence in Dart heap (`ohttp.dart` scope)**
File: `lib/src/ohttp.dart:105–120, 196–207`
`OhttpEncapsulateResult.enc` (32 B ephemeral X25519 public key) and `.exportedSecret` (16 B) are not zeroed after `ohttpDecapsulate` completes. Local HKDF outputs `aeadKey` and `aeadNonce` derived inside `ohttpDecapsulate` also remain on the heap until GC. Leaking `enc` alongside captured ciphertext breaks forward secrecy. Cross-ref: Phase 3 TASK-O6.

Note: P6-3 and P6-4 are intentionally kept as two separate HIGH entries with disjoint fix scopes.

**P6-5 — No structured event on KeyConfig fetch (OG-1)**
File: `lib/src/ohttp_client.dart:81–89`
Non-200 status on the KeyConfig GET throws a generic `Exception`. A deployed wallet cannot distinguish gateway misconfiguration from a transient network failure.

**P6-6 — No structured record of gateway POST status (OG-2)**
File: `lib/src/ohttp_client.dart:115–122`
Non-200 status on the gateway POST throws a generic `Exception`. Transient 5xx and permanent 4xx are indistinguishable.

**P6-7 — AEAD authentication failure propagates without audit trail (OG-3)**
File: `lib/src/ohttp.dart:218–224`
`SecretBoxAuthenticationError` propagates unwrapped from `ohttpDecapsulate`. No audit trail for decryption failure events; the internal exception type leaks into the public API surface.

**P6-8 — No observability signal when `sendDirect()` is invoked (OG-4)**
File: `lib/src/ohttp_client.dart:148–171`
`sendDirect()` accepts no `onLog` parameter and emits no signal when the plaintext path is taken. This is the observability framing of the same code gap covered by P6-1 (privacy framing); the fix scopes differ.

### IMPROVEMENT (5)

**P6-9 — Metadata in opt-in `onLog` callback**
File: `lib/src/ohttp_client.dart:77, 113`
`onLog` messages interpolate `gatewayBaseUrl`. Acceptable in development, but violates LC-6 if wired to a production structured logger.

**P6-10 — No timing instrumentation around encapsulation (OG-5)**
File: `lib/src/ohttp.dart:131–167`
No structured timing event surrounds `ohttpEncapsulate`. Round-trip encap latency is unobservable. Severity is IMPROVEMENT per `vision.md §7`.

**P6-11 — Example file lacks documentation of key limitations**
File: `example/ohttp_dart_example.dart:1–39`
The example omits: (a) a comment noting a real relay URL is required, (b) any mention of `sendDirect()` bypass implications, (c) a prerequisites block. Also, `targetAuthority` is set to the same host as `gatewayBaseUrl` — an example-correctness gap.

**P6-12 — Dependency version pinning (confirmed clean)**
File: `pubspec.yaml:12–18`
`cryptography: ^2.9.0` and `http: ^1.6.0` use tight caret constraints; no FFI/native deps. Recorded to preserve audit trail; no engineering task required.

**P6-13 — `effectiveDirectBaseUrl` exposed as a public getter**
File: `lib/src/ohttp_client.dart:52`
Making this getter private would reduce the blast radius of the P6-1 footgun. Separate fix scope from P6-1 (P6-1 adds a runtime warning; P6-13 changes API visibility).

---

## Observability Gaps Catalog (OG-1 .. OG-5)

| Gap ID | Missing event | File | Lines | Severity | Draft task |
|---|---|---|---|---|---|
| OG-1 | KeyConfig fetch succeeded / failed | `lib/src/ohttp_client.dart` | 81–89 | HIGH | P6-5 |
| OG-2 | Gateway POST status code | `lib/src/ohttp_client.dart` | 115–122 | HIGH | P6-6 |
| OG-3 | AEAD authentication failure on decap | `lib/src/ohttp.dart` | 218–224 | HIGH | P6-7 |
| OG-4 | `sendDirect()` invoked (OHTTP bypass) | `lib/src/ohttp_client.dart` | 148–171 | HIGH | P6-8 |
| OG-5 | OHTTP encapsulation completed (timing) | `lib/src/ohttp.dart` | 131–167 | IMPROVEMENT | P6-10 |

---

## Logging Constraints (LC-1 .. LC-6)

Any future logging added to `lib/` must never log:

| # | Constraint | Identifiers affected |
|---|---|---|
| LC-1 | No key material of any kind | `HpkeSenderContext.key`, `exporterSecret`, any HKDF output (`prk`, `aeadKey`, `aeadNonce`) |
| LC-2 | No ephemeral `enc` value | `ctx.enc`, `OhttpEncapsulateResult.enc` — leaking alongside ciphertext breaks forward secrecy |
| LC-3 | No inner request URL, path, method, headers, or body | Any field passed to `serializeRequest()` or `OhttpClient.send()` |
| LC-4 | No `targetAuthority` from `OhttpGatewayConfig` | Identifies the privacy-preserving target server |
| LC-5 | No response body or response headers | May contain wallet balances or PII |
| LC-6 | No `gatewayBaseUrl` at DEBUG or lower in production builds | Correlating gateway identity with timing is a metadata leak |

**Confirmed logging absence:** A grep sweep of `lib/src/` for `print`, `debugPrint`, `dart:developer`, `log(`, and `Logger` returned zero matches. The only logging mechanism is the opt-in `onLog` callback, which logs operation phases and `gatewayBaseUrl` (subject to LC-6 in production).

---

## Acceptance Criteria Verification

| Criterion | Status |
|---|---|
| All 11 tasks (6.1–6.11) checked off with concrete findings | PASS |
| At least one BLOCKER or HIGH finding tied to `sendDirect()` bypass | PASS — P6-1 (BLOCKER), P6-8 (HIGH) |
| At least one HIGH finding tied to key-material zeroization in `hpke.dart` | PASS — P6-3 (HIGH) |
| At least one HIGH finding tied to key-material zeroization in `ohttp.dart` | PASS — P6-4 (HIGH) |
| Five observability gaps (OG-1..OG-5) cataloged | PASS |
| Six logging constraints (LC-1..LC-6) documented | PASS |
| `pubspec.yaml` dependency review complete | PASS — P6-12 (confirmed clean) |
| All 13 draft entries conform to `plan.md §3` schema | PASS |
| No file in `lib/`, `test/`, or `example/` was modified | PASS |

**QA verdict: PASS** (see `specs/.current/AW-2865/phase-6/qa.md`).
**Review verdict: APPROVED** (see `specs/.current/AW-2865/review.md`).

---

## What Feeds Into Phase 7

Phase 7 performs a cross-phase severity triage across all phases (1–6), reconciles duplicates, confirms severity classifications, and produces a consolidated prioritized task list under `specs/.current/AW-2865/tasks/`.

**Phase 6 contributions:**
- 13 draft task entries (P6-1 through P6-13).
- Observability gaps catalog (OG-1..OG-5) and logging constraints matrix (LC-1..LC-6).
- Four cross-references to prior findings that Phase 7 must handle (P6-1↔C-4, P6-2↔C-1, P6-3↔F-2, P6-4↔TASK-O6).

### Four Phase 7 Carry-Forward Questions

1. **`onLog` production guidance.** Does P6-9 need a dedicated API-documentation task, or can it be absorbed into P6-11?
2. **`sendDirect()` retention rationale.** Testing utility or production fallback? Determines whether the P6-1 fix is "add warning" vs. "deprecate/remove."
3. **`OhttpEncapsulateResult` ownership for P6-4 fix.** New `dispose()` method vs. explicit zeroing in `OhttpClient.send()`?
4. **AEAD failure audit trail for P6-7.** Rate-limited security alert vs. structured log event only?
