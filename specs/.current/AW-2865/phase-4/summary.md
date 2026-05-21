# Phase 4 Summary — OhttpClient Layer Audit

**Ticket:** AW-2865
**Phase:** 4 of 7
**Status:** CLOSED
**Date completed:** 2026-05-21
**Verdict:** PASS (QA) / APPROVED (review)

---

## What was audited and why

Phase 4 is a read-only, line-by-line audit of `lib/src/ohttp_client.dart` (177 lines) — Layer 4 of the four-layer stack and the only layer a wallet integrator directly calls. It orchestrates the complete OHTTP round trip: GET `KeyConfig` from the gateway, BHTTP-serialize the inner request, OHTTP-encapsulate, POST to the gateway, OHTTP-decapsulate, and BHTTP-parse the inner response. It also exposes `sendDirect()`, a plain-HTTP escape hatch that bypasses OHTTP entirely.

Defects at this layer are the most wallet-visible: they directly determine end-user reliability, privacy preservation, and error observability. The driver is AW-2865: assessing `ohttp_dart` for production use in a non-custodial crypto wallet.

No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified. All writes were confined to `specs/.current/AW-2865/phase-4/`.

---

## File Audited

| File | Lines | Role |
|---|---|---|
| `lib/src/ohttp_client.dart` | 177 | `OhttpClient`, `OhttpGatewayConfig`, `OhttpResponse`, `OhttpHeader` |

---

## Findings Summary

Twelve findings were produced. None are BLOCKER. Nine are HIGH (should fix before wallet use). Three are IMPROVEMENT (recommended backlog items).

| ID | Title | Severity | File : Line Range | Concern Category |
|---|---|---|---|---|
| C-1 | `OhttpGatewayConfig.gatewayBaseUrl` has no `https`-scheme enforcement | HIGH | `ohttp_client.dart:35-53, 82, 116` | Scheme enforcement |
| C-2 | `configPath`/`requestPath` string-concatenated, not `Uri`-normalized | IMPROVEMENT | `ohttp_client.dart:82, 116, 154` | URL construction |
| C-3 | `targetAuthority` embedded verbatim with no allow-list | HIGH | `ohttp_client.dart:98-101` | Privacy / SSRF |
| C-4 | `sendDirect()` has no privacy-impact warning; co-located with OHTTP paths | HIGH | `ohttp_client.dart:41, 52, 148-171` | Privacy / Documentation |
| C-5 | KeyConfig GET: no timeout, no retry, no caching | HIGH | `ohttp_client.dart:81-89` | Network reliability / KeyConfig lifecycle |
| C-6 | Gateway POST: no timeout, no retry, no cancellation hook | HIGH | `ohttp_client.dart:115-122` | Network reliability |
| C-7 | Non-200 responses throw generic `Exception`; no typed hierarchy | HIGH | `ohttp_client.dart:84-87, 120-122` | Error handling |
| C-8 | Network errors propagate as untyped exceptions; no documented throws contract | HIGH | `ohttp_client.dart:69` | Error handling / API contract |
| C-9 | No response body/header size cap before `ohttpDecapsulate` and `bhttp.parseResponse` | HIGH | `ohttp_client.dart:129-133, 136` | Parser robustness / DoS |
| C-10 | `OhttpHeader.name` not lowercased in `send()` response path | IMPROVEMENT | `ohttp_client.dart:139, 168` | API consistency |
| C-11 | `SecretBoxAuthenticationError` propagates unwrapped to `OhttpClient.send()` caller | HIGH | `ohttp_client.dart:129-133`, `ohttp.dart:219` | Error handling / API boundary |
| C-12 | `onLog` unconditionally interpolates `gatewayBaseUrl` at every `send()` call | IMPROVEMENT | `ohttp_client.dart:77-78` | Privacy / Observability |

Severity assignments were resolved before research began and are recorded in `research.md` "Resolved Questions". Key resolutions: C-1/C-3/C-4/C-5/C-6 are HIGH (not BLOCKER) because each requires a deliberate misconfiguration or specific network condition to manifest, rather than triggering on the default happy-path; C-4 (`sendDirect()`) is HIGH because activation requires a deliberate developer choice; C-11 is HIGH via cross-layer confirmation (carries no independent remediation work); C-2/C-10/C-12 are IMPROVEMENT because they do not affect correctness of the OHTTP protocol or the wallet privacy model under normal operation.

---

## Draft Jira Tasks Produced (TASK-C1 through TASK-C12)

All tasks are independently actionable. Full descriptions including remediation directions are in `research.md` "Draft Task Entries".

| Task | Title | Severity | Primary file target |
|---|---|---|---|
| TASK-C1 | Enforce `https`-scheme on `OhttpGatewayConfig.gatewayBaseUrl` | HIGH | `ohttp_client.dart:35-53, 82, 116` |
| TASK-C2 | Replace string path concatenation with `Uri.resolve` normalization | IMPROVEMENT | `ohttp_client.dart:82, 116, 154` |
| TASK-C3 | Add allow-list validation or scheme-stripping for `targetAuthority` | HIGH | `ohttp_client.dart:98-101` |
| TASK-C4 | Add mandatory privacy-impact doc comment (or `@Deprecated`) to `sendDirect()`; add `onLog` bypass signal | HIGH | `ohttp_client.dart:148-171` |
| TASK-C5 | Add configurable TTL-based KeyConfig cache and timeout to KeyConfig GET | HIGH | `ohttp_client.dart:81-89` |
| TASK-C6 | Add configurable timeout and optional cancellation to gateway POST | HIGH | `ohttp_client.dart:115-122` |
| TASK-C7 | Introduce `OhttpClientException` typed hierarchy carrying HTTP status code | HIGH | `ohttp_client.dart:84-87, 120-122` |
| TASK-C8 | Document and own `OhttpClient.send()` exception contract; wrap foreign exception types | HIGH | `ohttp_client.dart:69` |
| TASK-C9 | Add configurable `maxResponseBytes` guard in `OhttpClient.send()` before `ohttpDecapsulate` | HIGH | `ohttp_client.dart:129-133` |
| TASK-C10 | Lowercase `OhttpHeader.name` at construction time in `send()` response path | IMPROVEMENT | `ohttp_client.dart:139` |
| TASK-C11 | (Resolved by TASK-O5) Wrap `SecretBoxAuthenticationError` at `ohttp.dart` layer | HIGH | `ohttp.dart:219` |
| TASK-C12 | Guard `gatewayBaseUrl` from `onLog` output or add log-level parameter | IMPROVEMENT | `ohttp_client.dart:77-78` |

Note: TASK-C11 has no independent remediation work in `ohttp_client.dart`. Closing TASK-O5 (Phase 3) at `ohttp.dart:219` closes TASK-C11 automatically. TASK-C7 and TASK-C8 overlap — a typed `OhttpClientException` hierarchy (C-7) is also the wrapping vehicle for foreign exceptions (C-8); these can be merged into a single Jira task at Iteration 7.

---

## Cross-Layer References

Two compounding patterns were identified and recorded.

### Phase 2 BHTTP findings that compound Phase 4 findings

| Phase 2 finding | Phase 4 finding | Compounding mechanism |
|---|---|---|
| R-2 — No header-count / header-size guard in `parseResponse` | C-9 | No size cap at client layer + no cap in parser = two-layer unbounded allocation |
| R-3 — `RangeError` on truncated body in `parseResponse` | C-8 | `RangeError` propagates through `ohttp_client.dart:136` to wallet caller unchanged |
| R-5 — No body size cap in `serializeRequest` | C-9 (partial) | Large inner request body produces a large HPKE ciphertext; no cap at any layer |

### Phase 3 OHTTP findings confirmed or extended at Phase 4

| Phase 3 finding | Phase 4 finding | Notes |
|---|---|---|
| O-9 — `SecretBoxAuthenticationError` propagates unwrapped | C-11 | Confirmed at wallet boundary; TASK-O5 remains the correct fix site |
| O-4 — Inconsistent exception types (`FormatException` vs `UnsupportedError`) | C-8 | Both types reachable at `OhttpClient.send()` boundary; contributes to undocumented throws contract |

---

## Key Wallet-Impact Conclusions

**Reliability.** `OhttpClient.send()` issues two unguarded network calls per invocation (KeyConfig GET at line 81 + gateway POST at line 115) with no timeout on either. A stalled gateway hangs the wallet indefinitely with no recovery path. Every invocation re-fetches the `KeyConfig` with no caching, doubling the network cost and creating a second independent failure point under poor connectivity.

**Privacy.** Three independent mechanisms can silently degrade or eliminate the RFC 9458 §1 privacy guarantee: (a) an `http://` gateway URL transmits the outer channel in plaintext with no validation at any call site; (b) `targetAuthority` has no allow-list, so a misconfigured authority routes inner requests to an unintended server; (c) `sendDirect()` fully bypasses OHTTP with no runtime warning, exposing the wallet IP address and full request details to the target server.

**Error surface.** `OhttpClient.send()` has no `try/catch` block anywhere in its 77-line body. It exposes a heterogeneous mix of exception types to callers — generic `Exception`, `FormatException`, `UnsupportedError`, `SecretBoxAuthenticationError`, `RangeError`, `http.ClientException`, and `dart:io.HandshakeException` — none owned by the `ohttp_dart` library. A wallet caller cannot write a complete and correct catch hierarchy without importing three external libraries. There is no typed distinction between permanent (4xx) and transient (5xx) gateway errors, making reliable retry logic impossible without fragile string parsing.

**DoS surface.** The combination of no response body size cap at the client layer (C-9, `ohttp_client.dart:129-133`) and no header-count or body cap inside `bhttp.parseResponse` (Phase 2 R-2, `bhttp.dart:165-182`) creates a two-layer unbounded memory allocation path. A single oversized response from a malicious or compromised OHTTP gateway can exhaust wallet process memory.

No BLOCKER was identified at the client layer. All eight HIGH findings are targeted, independently remediable gaps that do not require deep architectural changes.

---

## PRD Acceptance Criteria — Outcome

| Criterion | Result |
|---|---|
| All ten investigation goals covered (tasks 4.2–4.11 each produce a verdict with a line ref) | PASS — all 10 tasks have inline verdicts with at least one `ohttp_client.dart:NNN` citation |
| Timeout and caching findings cite call-site lines | PASS — C-5 cites line 81 (`_httpClient.get`); C-6 cites line 115 (`_httpClient.post`) |
| `sendDirect()` finding documents privacy impact | PASS — C-4 references RFC 9458 §1 and proposes concrete remediation options |
| Two-layer response-size finding cross-references Phase 2 | PASS — C-9 explicitly references Phase 2 finding R-2 |
| Severity assigned to every finding | PASS — C-1..C-12 each carry exactly one of HIGH or IMPROVEMENT; 0 BLOCKER |
| No code changes | PASS — `git diff HEAD -- lib/ test/` is empty |
| Draft tasks independently actionable | PASS — TASK-C1..TASK-C12 each carry a stable ID, single-line title, severity, primary file:line, and concern category |

---

## Open Items Deferred to Later Phases

1. **`HandshakeException` wrapping scope** — Does TASK-C8 wrap `dart:io.HandshakeException` inside the new `OhttpClientException`, or does the wallet's `http.Client` wrapper handle it? Deferred to TASK-C8 design.
2. **`effectiveDirectBaseUrl` null footgun** — When `directBaseUrl` is null, `sendDirect()` sends to `gatewayBaseUrl` (the OHTTP relay), not the intended target server. Should `sendDirect()` throw `StateError` on null? Deferred to TASK-C4 design.
3. **KeyConfig cache invalidation strategy** — Any TTL cache (TASK-C5) must define a forced-refresh trigger on key rotation (4xx response, `FormatException` from decapsulation). Correctness concern, not just performance. Deferred to TASK-C5 design.
4. **`OhttpClient` lifecycle and `dispose()`** — Abandoning `OhttpClient` without `dispose()` leaks the `IOClient` connection pool. Candidate addendum to TASK-C12 or a new IMPROVEMENT task.
5. **`onLog` string interpolation is unconditional** — Dart evaluates the argument of `?.call(string)` even when the receiver is null. Negligible at current call sites; guard needed for future additions. Implementation note for TASK-C12.

---

## Next Steps

Iteration 7 (task compilation) consumes the `research.md` "Draft Task Entries" table (TASK-C1..TASK-C12) directly alongside the Phase 1–3 outputs to produce per-task Markdown files under `specs/.current/AW-2865/tasks/`. The Iteration 7 step is the only step that produces the final per-task files for engineering pickup.

Before Iteration 7 begins, three housekeeping items from the review should be resolved: (I-1) reconcile PRD §Risks coverage for C-11 and C-12; (I-2) fix research.md C-10 line citation drift (169 → 168); (I-3) soften research.md §"Patterns Used" item 3 wording to align with C-12. None affect the correctness of any finding or draft task.

---

## Artifacts

- `specs/.current/AW-2865/phase-4/prd.md`
- `specs/.current/AW-2865/phase-4/plan.md`
- `specs/.current/AW-2865/phase-4/tasks.md`
- `specs/.current/AW-2865/phase-4/research.md`
- `specs/.current/AW-2865/phase-4/qa.md`
- `specs/.current/AW-2865/review.md` (Phase 4 section)
