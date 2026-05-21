# Phase 6: Review privacy risks, observability gaps, and documentation

**Goal:** Assess the package against privacy requirements for a non-custodial crypto wallet, review the example file for documented limitations, and catalog observability gaps.

## Context

**Feature motivation (from idea):** Investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The goal is not to implement improvements but to identify risks and form subsequent engineering tasks. Every finding must map to a specific file, line range, and privacy/observability concern.

**Severity classification (from vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Technical approach (from vision §4):** This is a cross-cutting audit covering all four layers plus the example file and `pubspec.yaml`. Privacy risk assessment is especially important for a non-custodial crypto wallet: OHTTP's purpose is unlinkability, but implementation gaps (undocumented bypass paths, absent key-material zeroization, no transport-layer enforcement) can silently undermine the privacy guarantee.

**Primary investigation concerns (from vision §7 — observability gaps):**

| Missing event | File | Gap severity |
|---|---|---|
| KeyConfig fetch succeeded / failed | `lib/src/ohttp_client.dart` | HIGH — no way to distinguish gateway misconfiguration from network failure in a deployed wallet |
| Gateway POST status code | `lib/src/ohttp_client.dart` | HIGH — non-200 responses throw generic `Exception`; no structured record |
| AEAD authentication failure on decap | `lib/src/ohttp.dart` | HIGH — `SecretBoxAuthenticationError` propagates silently; no audit trail |
| `sendDirect()` invoked (OHTTP bypass) | `lib/src/ohttp_client.dart` | HIGH — plaintext path has no warning signal |
| OHTTP encapsulation completed | `lib/src/ohttp.dart` | IMPROVEMENT — request round-trip timing is unobservable |

**Logging constraints (from vision §7):** Any future logging **must never** log:
1. Key material of any kind (`HpkeSenderContext.key`, `exporterSecret`, any HKDF output).
2. The ephemeral `enc` value — leaking it alongside the ciphertext breaks forward secrecy.
3. The inner request URL, path, method, headers, or body — defeats OHTTP's purpose.
4. `targetAuthority` from `OhttpGatewayConfig` — identifies the privacy-preserving target server.
5. Response body or response headers — may contain wallet balances or PII.
6. `gatewayBaseUrl` at DEBUG or lower in production builds — metadata leak.

**`sendDirect()` bypass context (from vision §6.4):**
```
Same OhttpGatewayConfig object → sendDirect() → plaintext HTTP to directBaseUrl
  [CONCERN] No assertion or warning that sendDirect() bypasses OHTTP privacy
  [CONCERN] Misconfiguration silently routes wallet data in cleartext
```

**Cross-phase context:**
- Phase 1 (HPKE) found no zeroization of `key`, `baseNonce`, `exporterSecret` after use (F-2).
- Phase 3 (OHTTP) found no zeroization of `enc` and `exportedSecret` in `OhttpEncapsulateResult` (TASK-O6).
- Phase 4 (Client) confirmed `sendDirect()` has no doc comment, no assert, and no privacy warning (Finding C-4).
- Phase 4 confirmed no `https`-scheme enforcement on `gatewayBaseUrl` (Finding C-1).

**Related data model concerns (from vision §5):**

| Concern | Severity |
|---|---|
| No zeroization of key material after use | HIGH |
| `directBaseUrl` enables accidental non-OHTTP path | IMPROVEMENT |
| No `https`-scheme enforcement in gateway config | HIGH |

## Tasks

### `lib/src/ohttp_client.dart`
- [x] 6.1 Confirm that `sendDirect()` is not documented with a privacy-impact warning anywhere in the source.
- [x] 6.2 Confirm no logging of inner request URL/method/headers/body or response body/headers anywhere in `lib/`.

### `lib/src/ohttp.dart`
- [x] 6.3 Confirm no logging of `enc`, `exportedSecret`, or HKDF-derived key material.

### `lib/src/hpke.dart`
- [x] 6.4 Confirm no logging of ephemeral key material.

### `example/ohttp_dart_example.dart`
- [x] 6.5 Read the file.
- [x] 6.6 Note whether the example documents: (a) that a real gateway URL is required, (b) that `sendDirect()` bypasses OHTTP, (c) any known limitations or prerequisites.

### `pubspec.yaml`
- [x] 6.7 Read the file.
- [x] 6.8 Note `package:cryptography` version pinning — confirm no wildcard `any` constraint that could silently pull in a breaking update.
- [x] 6.9 Note absence of any dependency that introduces native code or FFI.

### Observability gaps (cross-cutting)
- [x] 6.10 Catalog all five observability gaps from `vision.md §7` (KeyConfig fetch, gateway POST status, AEAD failure, `sendDirect()` invocation, encapsulation timing) and assign severity per vision.md.
- [x] 6.11 Note the six logging constraints from `vision.md §7` that any future logging must respect.

## Acceptance Criteria

**Test:** No code is changed. Verification is: the privacy findings include at least one BLOCKER or HIGH item tied to the `sendDirect()` bypass and at least one tied to absence of key-material zeroization.

## Dependencies

- Phase 5 complete (test suite coverage gaps review)

## Technical Details

**Privacy threat model for a non-custodial crypto wallet:**

OHTTP's core guarantee is unlinkability: the relay sees the client's IP but not the inner request content or target; the target server sees the request content but not the client's IP. This guarantee is undermined by:
1. **`sendDirect()` without a warning** — any misconfiguration that routes traffic through the non-OHTTP path exposes the wallet's IP directly to the target server.
2. **Plain HTTP on the outer channel** — a relay using `http://` instead of `https://` exposes the outer OHTTP ciphertext to a network observer, who can correlate timing even though content is encrypted.
3. **Key material in memory longer than necessary** — wallet processes are high-value targets for memory-dump attacks (cold-boot, device compromise, JVM/Dart VM heap dump); zeroizing ephemeral key material after use reduces exposure.
4. **Inner request metadata in logs** — even a single `print(method + ' ' + path)` in `lib/` would defeat OHTTP's inner-request privacy for anyone with log access.

**Scope of the logging audit:**

The package currently has **no logging** (confirmed in vision §7). This phase confirms that no logging was silently added and checks that the absence is consistent across all four layers and the example. Checking for: `dart:developer` calls, `print()`, `debugPrint()`, `log()`, `Logger` instances, or any string interpolation that could leak sensitive data into a structured logging framework.

**`pubspec.yaml` version pinning concern:**

`package:cryptography` is the sole cryptographic primitive source. A wildcard version constraint (`any` or `^x.y.z` with a major bump) could silently pull in a version with changed API or different cipher behavior. Confirm the constraint is a tight caret (`^x.y.z`) or exact pin, and that no transitive dependency introduces `dart:ffi`, `package:ffi`, or platform-specific code.

**Audit checklist for `example/ohttp_dart_example.dart`:**

The example is a developer-facing document as much as runnable code. Check for:
- A comment near `gatewayBaseUrl` noting that a real OHTTP relay URL is required.
- Any reference to `sendDirect()` — if called, does a comment explain the privacy implications?
- A note on prerequisites (Dart SDK, gateway setup, `pubspec.yaml` entries).
- Any hardcoded URLs, tokens, or private data that should not be in source control.

**Five observability gaps to catalog (from vision §7 — required for task 6.10):**

1. KeyConfig fetch succeeded / failed — file: `ohttp_client.dart`, severity: HIGH.
2. Gateway POST status code — file: `ohttp_client.dart`, severity: HIGH.
3. AEAD authentication failure on decap — file: `ohttp.dart`, severity: HIGH.
4. `sendDirect()` invoked (OHTTP bypass) — file: `ohttp_client.dart`, severity: HIGH.
5. OHTTP encapsulation completed (timing) — file: `ohttp.dart`, severity: IMPROVEMENT.

**Six logging constraints to document (from vision §7 — required for task 6.11):**

1. No key material of any kind.
2. No ephemeral `enc` value.
3. No inner request URL/path/method/headers/body.
4. No `targetAuthority`.
5. No response body or response headers.
6. No `gatewayBaseUrl` at DEBUG or lower in production builds.

## Implementation Notes

This is a read-only audit iteration. Do not edit any file in `lib/`, `test/`, or `example/`. Record all findings as draft task entries in this file for use in Iteration 7 (compile into per-task Markdown files under `specs/.current/AW-2865/tasks/`).

Phases 1–5 are complete with findings in their respective `tasks.md` files. Cross-reference those findings when noting privacy or observability concerns that compound lower-layer risks:
- Phase 1 F-2 (no HPKE key zeroization) + Phase 3 TASK-O6 (no OHTTP key zeroization) → combined BLOCKER or HIGH privacy task.
- Phase 4 Finding C-4 (`sendDirect()` no warning) → HIGH privacy task.
- Phase 4 Finding C-1 (no `https` enforcement) → HIGH privacy/transport task.

---

## Phase 6 Audit Findings (Draft Entries for Phase 7)

**Status:** Audit complete on 2026-05-21. No file in `lib/`, `test/`, or `example/` was modified. All entries below follow the schema defined in `phase-6/plan.md §3`.

### Confirmation summary per task

| Task | Result | Evidence |
|---|---|---|
| 6.1 | Confirmed — no privacy warning on `sendDirect()` | `lib/src/ohttp_client.dart:147` (single doc line "Send a direct HTTP request (for comparison)."); no `@Deprecated`, no `assert`, no warning. `effectiveDirectBaseUrl` at line 52 silently falls back to `gatewayBaseUrl`. |
| 6.2 | Confirmed — no inner request/response logging in `lib/` | Grep over `lib/` for `print\(|debugPrint|dart:developer|Logger|log\(` returned zero matches. The `onLog` callback in `ohttp_client.dart` logs only operation phases and `gatewayBaseUrl`. |
| 6.3 | Confirmed — no logging of `enc`, `exportedSecret`, or HKDF outputs in `ohttp.dart` | Same grep sweep; manual outline of `lib/src/ohttp.dart` confirms zero logging calls. |
| 6.4 | Confirmed — no logging of ephemeral key material in `hpke.dart` | Same grep sweep; manual outline of `lib/src/hpke.dart` confirms zero logging calls. |
| 6.5 | Read `example/ohttp_dart_example.dart` (39 lines). | — |
| 6.6 | (a) No comment documenting that a real OHTTP relay URL is required — placeholder strings are self-descriptive only. (b) No mention of `sendDirect()` and no bypass warning. (c) No prerequisites block. `targetAuthority` equals `gatewayBaseUrl` host (example-correctness gap). | `example/ohttp_dart_example.dart:9–14, 18–37` |
| 6.7 | Read `pubspec.yaml` (18 lines). | — |
| 6.8 | Confirmed tight caret pinning: `cryptography: ^2.9.0`. No wildcard `any` constraint. | `pubspec.yaml:13` |
| 6.9 | Confirmed absence of FFI/native deps. Direct deps: `cryptography ^2.9.0`, `http ^1.6.0`. Dev deps: `lints ^3.0.0`, `test ^1.25.6`. No `dart:ffi`, `package:ffi`, or platform-specific packages. | `pubspec.yaml:12–18` |
| 6.10 | Five observability gaps cataloged below as OG-1 through OG-5. | See §"Observability Gaps Catalog" |
| 6.11 | Six logging constraints recorded below as LC-1 through LC-6. | See §"Logging Constraints Matrix" |

### Draft Task Entries (P6-1 ... P6-13)

#### P6-1 — BLOCKER — Privacy — OHTTP bypass via silent `effectiveDirectBaseUrl` fallback
- **File:** `lib/src/ohttp_client.dart`
- **Lines:** 52, 148–171
- **Description:** `sendDirect()` carries one doc comment ("Send a direct HTTP request (for comparison)."), no `@Deprecated`, no `assert`, and no privacy-impact warning. The `effectiveDirectBaseUrl` getter silently resolves a null `directBaseUrl` to `gatewayBaseUrl` (the OHTTP relay endpoint). A developer who creates `OhttpGatewayConfig` with only required parameters and then calls `sendDirect()` issues plaintext HTTP to the OHTTP relay — no OHTTP encryption, no unlinkability — with no runtime signal. Escalated from HIGH (Phase 4 C-4) to BLOCKER because Phase 6 source-confirmed the silent fallback default.
- **Cross-refs:** Phase 4 C-4.

#### P6-2 — HIGH — Privacy — No `https`-scheme enforcement on outer channel
- **File:** `lib/src/ohttp_client.dart`
- **Lines:** 43–51
- **Description:** `OhttpGatewayConfig` constructor accepts any `gatewayBaseUrl` string with no scheme validation. Plain HTTP is silently accepted; the outer OHTTP ciphertext is then transmitted in cleartext, enabling timing correlation by a network observer even though the inner content is HPKE-encrypted. RFC 9458 §1 requires TLS on the outer channel.
- **Cross-refs:** Phase 4 C-1.

#### P6-3 — HIGH — Privacy — Key material persistence (`hpke.dart` scope)
- **File:** `lib/src/hpke.dart`
- **Lines:** 258–270
- **Description:** `HpkeSenderContext` retains `key` (16 B AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) as `final Uint8List` fields after `seal()` returns. No explicit zeroization; Dart GC does not guarantee prompt collection. A memory-dump or heap-analysis attack on a wallet process could expose these values long after the OHTTP exchange.
- **Cross-refs:** Phase 1 F-2.

#### P6-4 — HIGH — Privacy — Key material persistence (`ohttp.dart` scope)
- **File:** `lib/src/ohttp.dart`
- **Lines:** 105–120, 196–207
- **Description:** `OhttpEncapsulateResult.enc` (32 B ephemeral X25519 public key) and `.exportedSecret` (16 B) are not zeroed after `ohttpDecapsulate` completes. Local HKDF outputs `aeadKey` and `aeadNonce` derived inside `ohttpDecapsulate` are also retained on the heap until GC. Leak of `enc` alongside captured ciphertext breaks forward secrecy.
- **Cross-refs:** Phase 3 TASK-O6.

#### P6-5 — HIGH — Observability — KeyConfig fetch (OG-1)
- **File:** `lib/src/ohttp_client.dart`
- **Lines:** 81–89
- **Description:** No structured event on KeyConfig fetch success or failure. Non-200 throws generic `Exception('...')`. A deployed wallet cannot distinguish gateway misconfiguration from a transient network failure without code changes.

#### P6-6 — HIGH — Observability — Gateway POST status (OG-2)
- **File:** `lib/src/ohttp_client.dart`
- **Lines:** 115–122
- **Description:** No structured record of gateway POST response status. Non-200 throws generic `Exception('...')`. Transient 5xx and permanent 4xx are indistinguishable in observability output.

#### P6-7 — HIGH — Observability — AEAD authentication failure (OG-3)
- **File:** `lib/src/ohttp.dart`
- **Lines:** 218–224
- **Description:** `SecretBoxAuthenticationError` (a `package:cryptography` internal type) propagates unwrapped from `ohttpDecapsulate`. No audit trail for decryption failure; the internal exception type leaks into the public API surface.

#### P6-8 — HIGH — Observability — OHTTP bypass signal (OG-4)
- **File:** `lib/src/ohttp_client.dart`
- **Lines:** 148–171
- **Description:** `sendDirect()` accepts no `onLog` parameter and emits no signal when the plaintext path is taken. No way to detect OHTTP bypass via observability infrastructure. Companion to P6-1 (privacy framing) — separate task for the observability fix scope.

#### P6-9 — IMPROVEMENT — Privacy — Metadata in opt-in `onLog`
- **File:** `lib/src/ohttp_client.dart`
- **Lines:** 77, 113
- **Description:** `onLog` messages interpolate `gatewayBaseUrl` at lines 77 and 113. Compliant with LC-6 only if the caller does not wire `onLog` to a production structured logger. Requires documentation guidance on the API surface that `onLog` is for development/diagnostics only and must not be persisted in production logs.

#### P6-10 — IMPROVEMENT — Observability — Encapsulation timing (OG-5)
- **File:** `lib/src/ohttp.dart`
- **Lines:** 131–167
- **Description:** No structured timing event around `ohttpEncapsulate`. Round-trip encap latency is unobservable. Per `vision.md §7`, severity is IMPROVEMENT.

#### P6-11 — IMPROVEMENT — Documentation — Example file limitations
- **File:** `example/ohttp_dart_example.dart`
- **Lines:** 1–39
- **Description:** Example lacks: (a) a comment noting that a real OHTTP relay URL is required, (b) any mention of `sendDirect()` and its privacy bypass implications, (c) a prerequisites block (Dart SDK version, relay setup). Placeholder URLs `'https://your-gateway.example.com'` are obviously-placeholder but undocumented. `targetAuthority` is set to the same host as `gatewayBaseUrl`, which is technically incorrect (target server should differ from relay) — an example-correctness gap, not a security finding.

#### P6-12 — IMPROVEMENT — Dependency — Version pinning (confirmed-clean)
- **File:** `pubspec.yaml`
- **Lines:** 12–18
- **Description:** `cryptography: ^2.9.0` and `http: ^1.6.0` use tight caret constraints. No wildcard `any` constraint anywhere in the manifest. No `dart:ffi`, `package:ffi`, or platform-specific packages in direct dependencies. Recorded as IMPROVEMENT only to preserve the audit trail; no engineering task required.

#### P6-13 — IMPROVEMENT — Privacy — `effectiveDirectBaseUrl` API visibility
- **File:** `lib/src/ohttp_client.dart`
- **Lines:** 52
- **Description:** `effectiveDirectBaseUrl` is a public getter on `OhttpGatewayConfig` that silently resolves a null `directBaseUrl` to `gatewayBaseUrl`. Making this getter private (or removing it from the public surface) would reduce the blast radius of the P6-1 footgun. Separate fix scope from P6-1 (which adds runtime warnings); P6-13 changes API visibility.

### Observability Gaps Catalog (OG-1 .. OG-5)

| Gap ID | Missing event | File | Lines | Severity |
|---|---|---|---|---|
| OG-1 | KeyConfig fetch succeeded / failed | `lib/src/ohttp_client.dart` | 81–89 | HIGH |
| OG-2 | Gateway POST status code | `lib/src/ohttp_client.dart` | 115–122 | HIGH |
| OG-3 | AEAD authentication failure on decap | `lib/src/ohttp.dart` | 218–224 | HIGH |
| OG-4 | `sendDirect()` invoked (OHTTP bypass) | `lib/src/ohttp_client.dart` | 148–171 | HIGH |
| OG-5 | OHTTP encapsulation completed (timing) | `lib/src/ohttp.dart` | 131–167 | IMPROVEMENT |

### Logging Constraints Matrix (LC-1 .. LC-6)

Any future logging added to `lib/` **must never** log:

| # | Constraint | Identifiers affected |
|---|---|---|
| LC-1 | No key material of any kind | `HpkeSenderContext.key`, `exporterSecret`, any HKDF output (`prk`, `aeadKey`, `aeadNonce`) |
| LC-2 | No ephemeral `enc` value | `ctx.enc`, `OhttpEncapsulateResult.enc` — leakage alongside ciphertext breaks forward secrecy |
| LC-3 | No inner request URL, path, method, headers, or body | Any field passed to `serializeRequest()` or `OhttpClient.send()` |
| LC-4 | No `targetAuthority` from `OhttpGatewayConfig` | Identifies privacy-preserving target server |
| LC-5 | No response body or response headers | May contain wallet balances or PII |
| LC-6 | No `gatewayBaseUrl` at DEBUG or lower in production builds | Correlating gateway identity with timing is a metadata leak |

### Phase 7 Carry-Forward Questions

Surfaced during Phase 6 review; not gating items for Phase 6 closure.

1. **`onLog` production guidance.** Does P6-9 need a dedicated API-documentation task in Phase 7, or can it be absorbed into P6-11 (example documentation)?
2. **`sendDirect()` retention rationale.** Is `sendDirect()` a testing utility (per "for comparison" doc) or a production fallback? Affects whether the Phase 7 fix for P6-1 is "add warning" vs. "deprecate/remove."
3. **`OhttpEncapsulateResult` ownership for P6-4 fix.** Should zeroization live in a new `dispose()` on `OhttpEncapsulateResult`, or should `OhttpClient.send()` explicitly zero fields before returning?
4. **AEAD failure audit trail (P6-7).** Should AEAD failures trigger a rate-limited security alert, or a structured log event only?

### Acceptance Criteria — Verification

Per `phase-6/tasks.md` Acceptance Criteria block:
- BLOCKER finding tied to `sendDirect()` bypass — **satisfied** by P6-1.
- HIGH finding tied to absence of key-material zeroization — **satisfied** by P6-3 and P6-4.
- No code changed in `lib/`, `test/`, or `example/` — **satisfied**.
