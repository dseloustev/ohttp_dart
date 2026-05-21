# Phase 6 Research: Privacy Risk Review, Observability Gaps Audit, and Documentation Review

**Ticket:** AW-2865  
**Phase:** 6  
**Date:** 2026-05-21  
**Scope:** Cross-cutting audit — `lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, `lib/src/hpke.dart`, `example/ohttp_dart_example.dart`, `pubspec.yaml`  
**Status:** Research complete — no source files modified

---

## Phase Scope

Phase 6 is the final investigation-phase audit. It does not audit a single implementation layer; it performs three cross-cutting reviews:

1. **Privacy risk review** — whether the package's current state preserves OHTTP's unlinkability guarantee when used in a non-custodial crypto wallet, and what gaps undermine it.
2. **Observability gaps audit** — cataloging the five structured gaps and six logging constraints from `vision.md §7`.
3. **Documentation review** — `example/ohttp_dart_example.dart` and `pubspec.yaml` for documented limitations, version pinning integrity, and absence of native/FFI dependencies.

Phases 1–5 are complete; this phase cross-references their findings by ID only.

---

## 1. Resolved Questions

Per the user's answers provided before this research run:

| # | Question | Resolution |
|---|---|---|
| Q1 | `sendDirect()` escalation to BLOCKER | Severity floor is HIGH entering Phase 7. Escalate to BLOCKER only if task 6.1 confirms the silent `effectiveDirectBaseUrl` fallback at source level. Research must confirm definitively — not defer. |
| Q2 | Zeroization task merging | Two separate HIGH tasks: one for `hpke.dart` (Phase 1 F-2 scope), one for `ohttp.dart` (Phase 3 TASK-O6 scope). |
| Q3 | Example file gateway URL policy | Real but non-sensitive URL → IMPROVEMENT only. Credentials or private-only URLs → HIGH or BLOCKER. |
| Q4 | `pubspec.yaml` constraint format | Confirm actual constraint; HIGH if wildcard, IMPROVEMENT if tight caret. |
| Q5 | Encapsulation timing severity | IMPROVEMENT per `vision.md §7`. Authoritative for Phase 7 prioritization. |
| Q6 | Prior-phase cross-references | Cite by ID only; do not reproduce verbatim findings. |
| Q7 | `sendDirect()` severity | Confirmed definitively by reading source now — see P6-1 below. |
| Q8 | `pubspec.yaml` scope | Direct dependencies only (no transitive audit). |
| Q9 | Draft task entry format | Stable ID, severity, file:lines, concern category — included in §6 below. |

---

## 2. Related Modules / Services

All files audited are under `lib/src/` and `example/`. No external services are contacted during this audit; all checks are static source review.

| File | Role |
|---|---|
| `lib/src/ohttp_client.dart` | Layer 4 — high-level client; `OhttpClient`, `OhttpGatewayConfig`, `sendDirect()` |
| `lib/src/ohttp.dart` | Layer 3 — OHTTP encap/decap; `OhttpKeyConfig`, `OhttpEncapsulateResult`, `ohttpEncapsulate`, `ohttpDecapsulate` |
| `lib/src/hpke.dart` | Layer 1 — HPKE Base Mode sender; `HpkeSender`, `HpkeSenderContext` |
| `example/ohttp_dart_example.dart` | Developer-facing runnable example |
| `pubspec.yaml` | Dependency manifest |

---

## 3. Current Endpoints & Contracts

### 3.1 `OhttpGatewayConfig` — `lib/src/ohttp_client.dart:35–53`

```
gatewayBaseUrl  String    No https-scheme enforcement (Phase 4 C-1)
configPath      String    String-concatenated, not Uri-normalized
requestPath     String    String-concatenated, not Uri-normalized
targetAuthority String    Embedded verbatim into BHTTP inner request
targetScheme    String    Defaults to 'https'; not enforced on the outer channel
directBaseUrl   String?   Nullable; feeds effectiveDirectBaseUrl
```

`effectiveDirectBaseUrl` getter at line 52:
```dart
String get effectiveDirectBaseUrl => directBaseUrl ?? gatewayBaseUrl;
```

This is the silent fallback at the center of P6-1 (see §6). When `directBaseUrl` is `null` (the default), calling `sendDirect()` routes to `gatewayBaseUrl` — the OHTTP gateway endpoint — not to a separately-configured direct target. A developer who did not intend to configure a direct path gets a quietly functional non-OHTTP request to the relay.

### 3.2 `sendDirect()` — `lib/src/ohttp_client.dart:148–171`

Doc comment at line 147: `/// Send a direct HTTP request (for comparison).`

- No `@Deprecated` annotation.
- No `assert`.
- No privacy-impact warning.
- The comment phrase "for comparison" is the sole contextual hint.
- Internally, it calls `Uri.parse('${gateway.effectiveDirectBaseUrl}$path')` at line 154, confirming the silent fallback.

### 3.3 `send()` `onLog` callback — `lib/src/ohttp_client.dart:74`

`send()` accepts an optional `void Function(String message)? onLog` parameter. It emits the following log strings when the callback is provided:

| Line | Message template | Sensitive data? |
|---|---|---|
| 77 | `'Fetching OHTTP KeyConfig from ${gateway.gatewayBaseUrl}${gateway.configPath}...'` | `gatewayBaseUrl` — metadata leak |
| 91 | `'KeyConfig received (N bytes, keyId=K)'` | `keyId` only — low sensitivity |
| 96 | `'Serializing request to BHTTP...'` | None |
| 107 | `'Encapsulating request via OHTTP...'` | None |
| 109 | `'Request encapsulated (N bytes)'` | None |
| 113 | `'Sending to OHTTP gateway ${gateway.gatewayBaseUrl}${gateway.requestPath}...'` | `gatewayBaseUrl` — metadata leak |
| 124 | `'Gateway responded (N bytes)'` | None |
| 128 | `'Decapsulating response...'` | None |
| 142 | `'Response decapsulated: HTTP N'` | Status code only — low sensitivity |

No logging of inner request URL, method, headers, body, response body, response headers, `targetAuthority`, `enc`, `exportedSecret`, or HKDF-derived keys. The `onLog` messages at lines 77 and 113 interpolate `gatewayBaseUrl` — this is a metadata leak risk if the callback is wired to production logs (constraint 6 in §5 below).

### 3.4 `ohttpEncapsulate` — `lib/src/ohttp.dart:131–167`

No logging. Returns `OhttpEncapsulateResult` holding `encRequest`, `enc` (32 B ephemeral X25519 public key), and `exportedSecret` (16 B). Neither `enc` nor `exportedSecret` is zeroed after the caller (`OhttpClient.send()`) completes decapsulation. See Phase 3 TASK-O6 and P6-4.

### 3.5 `ohttpDecapsulate` — `lib/src/ohttp.dart:174–226`

No logging. Derives `aeadKey` (16 B) and `aeadNonce` (12 B) via plain HKDF at lines 196–207. Both are local variables; they live on the Dart heap until GC. No explicit zeroization.

### 3.6 `HpkeSenderContext` — `lib/src/hpke.dart:258–313`

Fields: `enc` (32 B), `key` (16 B AES-128), `baseNonce` (12 B), `exporterSecret` (32 B). All are `final Uint8List`. No logging. No zeroization. See Phase 1 F-2 and P6-3.

---

## 4. Patterns Used

### 4.1 Logging pattern — `onLog` callback injection

`send()` uses a caller-injected `void Function(String message)? onLog` callback rather than a hard-coded logging framework. This is the only logging mechanism in the entire `lib/` tree. Grep confirms zero instances of `print`, `debugPrint`, `dart:developer`, `log()`, or `Logger` in `lib/src/`. The `onLog` approach respects the "no hard-coded logging" constraint from `vision.md §7`, but it logs `gatewayBaseUrl` twice when the callback is provided (lines 77, 113), violating logging constraint 6.

`sendDirect()` has no `onLog` parameter at all — no log, no warning, no signal on the plaintext path. This is the observability gap for OHTTP bypass (gap 4 in §5).

### 4.2 Error pattern — untyped `Exception`

Both the KeyConfig GET (line 85) and gateway POST (line 121) throw `Exception('...')` on non-200 status. This matches the pattern confirmed in Phase 4 (C-2/C-3 scope). Observability gaps 1 and 2 in §5 correspond to these two throw sites.

### 4.3 No native code pattern

`pubspec.yaml` direct dependencies:
- `cryptography: ^2.9.0` — tight caret constraint; major-version breaking changes blocked.
- `http: ^1.6.0` — tight caret constraint.

Dev dependencies: `lints: ^3.0.0`, `test: ^1.25.6`. No `dart:ffi`, `package:ffi`, platform-specific packages, or native assets present. `publish_to: none` — not published to pub.dev.

### 4.4 Example file pattern

`example/ohttp_dart_example.dart` (39 lines):
- Line 1: `// ignore_for_file: avoid_print` — suppresses lint for `print` calls. This is the only `print` in the repo visible from outside `lib/`.
- Lines 9–14: `OhttpGatewayConfig` with placeholder-style URLs: `'https://your-gateway.example.com'`, `'/ohttp/config'`, `'/ohttp/gateway'`, `'your-gateway.example.com'`. These are obviously-placeholder strings — not real credentials, not real private URLs.
- Lines 18–24: Calls `client.send()` with `onLog: print`. The `onLog` callback wires the gateway metadata (lines 77, 113 of `ohttp_client.dart`) to `print` — in an example context this is acceptable; in production code wired to a structured logger it would violate constraint 6.
- Lines 27–37: Second `send()` call (POST) with `onLog: print`. Passes `body: utf8.encode('{"hello": "ohttp"}')` — not sensitive in an example.
- No call to `sendDirect()` anywhere in the example.
- No comment explaining that `sendDirect()` bypasses OHTTP.
- No comment explaining that a real OHTTP relay URL is required (the placeholder strings are self-descriptive but not explicitly documented).
- No prerequisites block (Dart SDK version, relay setup, etc.).
- `targetAuthority` is set to `'your-gateway.example.com'` — same as `gatewayBaseUrl` host, which is technically incorrect (should be the target server, not the relay) but is a documentation gap, not a security finding.

---

## 5. Limitations & Risks

### 5.1 Confirmed privacy risks from source review

**Risk 1 — `sendDirect()` no privacy warning (compound of Phase 4 C-4 + new P6-1 finding)**

Source confirmed: `sendDirect()` at `lib/src/ohttp_client.dart:148–171` has one doc comment ("Send a direct HTTP request (for comparison)."), no `@Deprecated`, no `assert`, and no privacy-impact warning. The `effectiveDirectBaseUrl` getter at line 52 silently falls back to `gatewayBaseUrl` when `directBaseUrl` is `null`. This means a developer who creates `OhttpGatewayConfig` with only the required parameters (leaving `directBaseUrl` unset) and then calls `sendDirect()` produces a plain HTTP request to the OHTTP relay endpoint — no OHTTP encryption, no unlinkability. The call succeeds (or fails with a relay-specific 4xx) without any runtime warning that OHTTP was bypassed. Severity: **BLOCKER** — the silent fallback is now source-confirmed, satisfying the escalation condition stated in the PRD (Scenario A and Constraint 10). The fallback is not merely a configuration possibility; it is the default code path.

**Risk 2 — Plain HTTP outer channel (Phase 4 C-1 cross-reference)**

`OhttpGatewayConfig` constructor at `lib/src/ohttp_client.dart:43–51` performs no scheme validation. Plain HTTP `gatewayBaseUrl` is silently accepted. Severity: HIGH (confirmed prior finding; no new source evidence added in Phase 6).

**Risk 3 — Key material in heap — `hpke.dart` scope (Phase 1 F-2 cross-reference)**

`HpkeSenderContext` at `lib/src/hpke.dart:258–270` retains `key`, `baseNonce`, and `exporterSecret` as `final Uint8List` fields. No zeroization after `seal()` returns. Severity: HIGH.

**Risk 4 — Key material in heap — `ohttp.dart` scope (Phase 3 TASK-O6 cross-reference)**

`OhttpEncapsulateResult` at `lib/src/ohttp.dart:105–120` retains `enc` and `exportedSecret`. No zeroization after `ohttpDecapsulate` completes. Local HKDF outputs `aeadKey` (line 196) and `aeadNonce` (line 203) are also not explicitly zeroed. Severity: HIGH.

**Risk 5 — Inner request content in logs — confirmed absent**

Grep across `lib/src/` for `print`, `debugPrint`, `dart:developer`, `log(`, `Logger` returned zero matches. The `onLog` callback does not log inner request URL, method, headers, body, `targetAuthority`, `enc`, or key material. Risk of inner content in logs: **NOT PRESENT** in current source. This finding closes tasks 6.2, 6.3, 6.4 as confirmed-absent.

**Risk 6 — `gatewayBaseUrl` logged via `onLog` callback**

Lines 77 and 113 of `ohttp_client.dart` interpolate `gatewayBaseUrl` into `onLog` messages. The callback is opt-in, but if wired to a production structured logger it violates logging constraint 6. Severity: IMPROVEMENT (callback is not invoked without explicit caller opt-in; no hard-coded logger).

### 5.2 `pubspec.yaml` assessment

`cryptography: ^2.9.0` — tight caret constraint. Breaking changes above `2.x.y` are blocked. No wildcard constraint found. No `dart:ffi`, `package:ffi`, or native-asset dependency in either `dependencies` or `dev_dependencies`. Severity of this finding: **IMPROVEMENT** (tight caret is correct practice; no HIGH finding warranted).

### 5.3 Example file assessment

- No real credentials or private URLs. Placeholder strings are clearly marked. Severity: IMPROVEMENT (not HIGH or BLOCKER).
- No comment documenting that a real OHTTP relay URL is required. Severity: IMPROVEMENT.
- No comment or any mention of `sendDirect()` in the example, and no warning that such a call bypasses OHTTP. Severity: IMPROVEMENT (the example does not call `sendDirect()`, so no immediate bypass risk; but the omission leaves a documentation gap).
- `targetAuthority` is set to the same host as `gatewayBaseUrl` — this is an example correctness gap (target and relay should differ). Severity: IMPROVEMENT.
- `onLog: print` is passed in both `send()` calls, which causes `gatewayBaseUrl` to be printed to stdout. Acceptable in an example file. Severity: IMPROVEMENT.

---

## 6. Observability Gaps Catalog (Task 6.10) and Logging Constraints (Task 6.11)

### 6.1 Five observability gaps (vision.md §7)

| Gap ID | Missing event | File | Lines | Gap severity | Notes |
|---|---|---|---|---|---|
| OG-1 | KeyConfig fetch succeeded / failed | `lib/src/ohttp_client.dart` | 81–89 | HIGH | Non-200 throws generic `Exception`; no structured record of fetch outcome |
| OG-2 | Gateway POST status code | `lib/src/ohttp_client.dart` | 115–122 | HIGH | Non-200 throws generic `Exception`; no structured record of gateway response code |
| OG-3 | AEAD authentication failure on decap | `lib/src/ohttp.dart` | 218–224 | HIGH | `SecretBoxAuthenticationError` propagates unwrapped; no audit trail for decryption failure |
| OG-4 | `sendDirect()` invoked (OHTTP bypass) | `lib/src/ohttp_client.dart` | 148–171 | HIGH | Plaintext path has no log, no warning signal; `sendDirect()` has no `onLog` parameter |
| OG-5 | OHTTP encapsulation completed (timing) | `lib/src/ohttp.dart` | 131–167 | IMPROVEMENT | Round-trip encap timing is unobservable; no structured timing event |

### 6.2 Six logging constraints (vision.md §7)

Any future logging added to `lib/` **must never** log:

| # | Constraint | Identifiers affected |
|---|---|---|
| LC-1 | No key material of any kind | `HpkeSenderContext.key`, `exporterSecret`, any HKDF output (`prk`, `aeadKey`, `aeadNonce`) |
| LC-2 | No ephemeral `enc` value | `ctx.enc`, `OhttpEncapsulateResult.enc` — leaking alongside ciphertext breaks forward secrecy |
| LC-3 | No inner request URL, path, method, headers, or body | Any field passed to `serializeRequest()` or `send()` |
| LC-4 | No `targetAuthority` from `OhttpGatewayConfig` | Identifies privacy-preserving target server; defeats unlinkability |
| LC-5 | No response body or response headers | May contain wallet balances or PII |
| LC-6 | No `gatewayBaseUrl` at DEBUG or lower in production builds | Correlating gateway identity with timing is a metadata leak |

---

## 7. Draft Task Entries

The following entries are ready for Phase 7 compilation into per-task Markdown files.

| Task ID | Severity | File | Lines | Concern category | Description |
|---|---|---|---|---|---|
| P6-1 | BLOCKER | `lib/src/ohttp_client.dart` | 52, 148–171 | Privacy — OHTTP bypass | `sendDirect()` has no privacy-impact warning, no `@Deprecated`, no `assert`. `effectiveDirectBaseUrl` silently falls back to `gatewayBaseUrl` when `directBaseUrl` is null (default). A call to `sendDirect()` without explicit `directBaseUrl` configuration routes plaintext to the OHTTP relay, bypassing all unlinkability guarantees without any runtime signal. Escalated from HIGH to BLOCKER: silent fallback confirmed at source. Cross-refs: Phase 4 C-4. |
| P6-2 | HIGH | `lib/src/ohttp_client.dart` | 43–51 | Privacy — transport security | No `https`-scheme enforcement in `OhttpGatewayConfig` constructor. Plain HTTP `gatewayBaseUrl` accepted silently. Outer OHTTP ciphertext transmitted in cleartext on a plain-HTTP channel. RFC 9458 §1 requires TLS on outer channel. Cross-refs: Phase 4 C-1. |
| P6-3 | HIGH | `lib/src/hpke.dart` | 258–270 | Privacy — key material persistence | `HpkeSenderContext` fields `key` (16 B AES-128), `baseNonce` (12 B), `exporterSecret` (32 B) are retained as `final Uint8List` after `seal()` returns. No explicit zeroization. Dart GC does not guarantee prompt collection. Memory-dump or heap-analysis attack could expose these values. Cross-refs: Phase 1 F-2. |
| P6-4 | HIGH | `lib/src/ohttp.dart` | 105–120, 196–207 | Privacy — key material persistence | `OhttpEncapsulateResult.enc` (32 B) and `.exportedSecret` (16 B) not zeroed after `ohttpDecapsulate` completes. Local HKDF outputs `aeadKey` and `aeadNonce` at lines 196–207 also not explicitly zeroed. Cross-refs: Phase 3 TASK-O6. |
| P6-5 | HIGH | `lib/src/ohttp_client.dart` | 81–89 | Observability — KeyConfig fetch | No structured event on KeyConfig fetch success or failure. Non-200 throws generic `Exception`. No way to distinguish gateway misconfiguration from network failure in a deployed wallet (OG-1). |
| P6-6 | HIGH | `lib/src/ohttp_client.dart` | 115–122 | Observability — gateway POST | No structured record of gateway POST status code. Non-200 throws generic `Exception`. Cannot distinguish transient 5xx from permanent 4xx without structured event (OG-2). |
| P6-7 | HIGH | `lib/src/ohttp.dart` | 218–224 | Observability — AEAD auth failure | `SecretBoxAuthenticationError` propagates unwrapped from `ohttpDecapsulate`. No audit trail for decryption failure event. `package:cryptography` internal type leaks into public API surface (OG-3). |
| P6-8 | HIGH | `lib/src/ohttp_client.dart` | 148–171 | Observability — OHTTP bypass signal | `sendDirect()` has no `onLog` parameter and emits no signal when the plaintext path is taken. No way to detect OHTTP bypass in deployed observability infrastructure (OG-4). Combines with P6-1 for privacy concern; listed separately for observability framing. |
| P6-9 | IMPROVEMENT | `lib/src/ohttp_client.dart` | 77, 113 | Privacy — metadata in opt-in logs | `onLog` messages at lines 77 and 113 interpolate `gatewayBaseUrl`. Compliant with LC-6 only if the caller does not wire `onLog` to a production structured logger. No hard-coded logger; caller opt-in only. Requires documentation guidance that `onLog` must not be wired to persistent production logs. |
| P6-10 | IMPROVEMENT | `lib/src/ohttp.dart` | 131–167 | Observability — encap timing | No structured timing event around `ohttpEncapsulate`. Round-trip encap timing unobservable (OG-5). IMPROVEMENT per `vision.md §7`. |
| P6-11 | IMPROVEMENT | `example/ohttp_dart_example.dart` | 1–39 | Documentation — example limitations | Example lacks: (a) comment noting real OHTTP relay URL required, (b) any mention of `sendDirect()` privacy bypass, (c) prerequisites block (Dart SDK, relay setup). Placeholder URLs present (`'https://your-gateway.example.com'`). `targetAuthority` set to same host as relay — example correctness gap. |
| P6-12 | IMPROVEMENT | `pubspec.yaml` | 13 | Dependency — version pinning | `cryptography: ^2.9.0` uses tight caret constraint. No wildcard constraint. No FFI/native deps. Finding is positive (correct practice). No engineering task needed; document as confirmed-clean for Phase 7 triage record. |
| P6-13 | IMPROVEMENT | `lib/src/ohttp_client.dart` | 52 | Privacy — API footgun | `effectiveDirectBaseUrl` is a public getter that silently resolves `null` directBaseUrl to `gatewayBaseUrl`. Making this getter private or removing it from the public API surface would reduce the blast radius of the P6-1 footgun. Separate from P6-1 (fix scopes differ: P6-1 = add warning/assert; P6-13 = API visibility change). |

---

## 8. New Technical Questions

The following questions surfaced during this phase's source review — not resolvable by static audit alone:

1. **`onLog` production guidance:** Is there a plan to document that `onLog: print` in the example must not be adapted as `onLog: structuredLogger.debug` in production? If so, does this belong in Phase 7 as an IMPROVEMENT task for API documentation, or is it covered by the existing example-documentation task (P6-11)?

2. **`sendDirect()` retention rationale:** Was `sendDirect()` designed as a testing utility (hence "for comparison" in the doc comment), or is it intended as a production fallback path? If it is a testing utility only, the Phase 7 fix task might consider deprecation or removal rather than just adding a warning. This affects fix-scope of P6-1.

3. **`OhttpEncapsulateResult` ownership:** After `OhttpClient.send()` calls `ohttpDecapsulate`, the `OhttpEncapsulateResult` object is no longer referenced. Is the zeroization fix (P6-4) best placed in a `dispose()` method on `OhttpEncapsulateResult`, or should `OhttpClient.send()` explicitly zero the fields before returning? The answer affects whether a new `dispose()` pattern is introduced at layer 3.

4. **AEAD failure audit trail:** OG-3 (P6-7) notes that `SecretBoxAuthenticationError` propagates without audit trail. In the wallet integration context, should AEAD failures trigger a security alert (rate-limited) or just a structured log event? This affects whether the Phase 7 task is "add structured log" or "add typed exception + alert hook."

---

## Cross-Phase Reference Map

| Prior finding | Phase | Confirmed relevant in Phase 6? | Phase 6 task |
|---|---|---|---|
| F-2 (HPKE key zeroization) | Phase 1 | Yes — `HpkeSenderContext` fields at `hpke.dart:258–270` | P6-3 |
| TASK-O6 (OHTTP key zeroization) | Phase 3 | Yes — `OhttpEncapsulateResult` at `ohttp.dart:105–120`, local HKDF outputs at `ohttp.dart:196–207` | P6-4 |
| C-1 (no https enforcement) | Phase 4 | Yes — `OhttpGatewayConfig` constructor at `ohttp_client.dart:43–51` | P6-2 |
| C-4 (sendDirect no warning) | Phase 4 | Yes — confirmed and **escalated to BLOCKER** via source-confirmed silent fallback | P6-1 |
