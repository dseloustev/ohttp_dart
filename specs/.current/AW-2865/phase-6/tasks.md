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
- [ ] 6.1 Confirm that `sendDirect()` is not documented with a privacy-impact warning anywhere in the source.
- [ ] 6.2 Confirm no logging of inner request URL/method/headers/body or response body/headers anywhere in `lib/`.

### `lib/src/ohttp.dart`
- [ ] 6.3 Confirm no logging of `enc`, `exportedSecret`, or HKDF-derived key material.

### `lib/src/hpke.dart`
- [ ] 6.4 Confirm no logging of ephemeral key material.

### `example/ohttp_dart_example.dart`
- [ ] 6.5 Read the file.
- [ ] 6.6 Note whether the example documents: (a) that a real gateway URL is required, (b) that `sendDirect()` bypasses OHTTP, (c) any known limitations or prerequisites.

### `pubspec.yaml`
- [ ] 6.7 Read the file.
- [ ] 6.8 Note `package:cryptography` version pinning — confirm no wildcard `any` constraint that could silently pull in a breaking update.
- [ ] 6.9 Note absence of any dependency that introduces native code or FFI.

### Observability gaps (cross-cutting)
- [ ] 6.10 Catalog all five observability gaps from `vision.md §7` (KeyConfig fetch, gateway POST status, AEAD failure, `sendDirect()` invocation, encapsulation timing) and assign severity per vision.md.
- [ ] 6.11 Note the six logging constraints from `vision.md §7` that any future logging must respect.

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
