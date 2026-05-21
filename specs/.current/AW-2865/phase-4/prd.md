# AW-2865 Phase 4: OhttpClient Layer Audit — Network Reliability, KeyConfig Lifecycle, Scheme Enforcement, and sendDirect() Bypass Risks

Status: PRD_READY

## Context / Idea

Ticket AW-2865 is a read-only investigation of the `ohttp_dart` package to determine its production-readiness for use in a non-custodial crypto wallet. The investigation is organized into phases covering each layer of the four-layer stack.

Phase 4 targets Layer 4: `lib/src/ohttp_client.dart`. This file is the top of the stack and the only layer a wallet integrator directly calls. It orchestrates the complete OHTTP round trip: GET `KeyConfig` from the gateway, BHTTP-serialize the inner request, OHTTP-encapsulate, POST to the gateway, OHTTP-decapsulate, and BHTTP-parse the inner response. It also exposes `sendDirect()`, a plain-HTTP escape hatch that bypasses OHTTP entirely. Defects here are the most wallet-visible: they determine end-user reliability, privacy preservation, and error observability.

**Inherited context (ticket-wide vision §2 severity model):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Phase dependencies:**
- Phase 1 (HPKE layer audit) is complete. Relevant finding: absence of zeroization for `key`, `baseNonce`, `exporterSecret` in `HpkeSenderContext`; `testKeyPair` injection hook reachable from production.
- Phase 2 (BHTTP layer audit) is complete. Relevant finding: no size cap on headers or body in `parseResponse`; `RangeError` on truncated input instead of `FormatException`.
- Phase 3 (OHTTP layer audit) is complete. Relevant finding: `SecretBoxAuthenticationError` (a `package:cryptography` type) propagates unwrapped from `ohttpDecapsulate` to the caller; `enc` and `exportedSecret` not zeroized after `ohttpEncapsulate` returns.

**No code is changed in this phase.** Every output is a finding mapped to a future follow-up task.

---

## Goals

1. Confirm that neither the KeyConfig GET nor the gateway POST carries a timeout, retry, or cancellation hook, and record the specific `http.Client` call site lines.
2. Confirm that `KeyConfig` is fetched fresh on every `send()` invocation with no caching or TTL — establishing that each OHTTP request incurs two network round trips.
3. Confirm that `OhttpGatewayConfig.gatewayBaseUrl` has no `https`-scheme enforcement, allowing a misconfigured `http://` URL to silently transmit outer-channel traffic in plaintext.
4. Confirm that `configPath` and `requestPath` are string-concatenated rather than `Uri`-normalized, and document the potential for double-slash and path-traversal sequences.
5. Confirm that `targetAuthority` is embedded verbatim into the BHTTP request with no allow-list validation, and assess the privacy and SSRF implications for a wallet.
6. Confirm that `sendDirect()` carries no privacy-impact warning or assertion, and establish that a wallet using it as a fallback silently degrades its OHTTP privacy guarantees.
7. Confirm that all non-200 HTTP responses (both KeyConfig GET and gateway POST) throw a generic untyped `Exception` with no distinction between 4xx permanent and 5xx transient errors.
8. Confirm that the gateway response body and headers have no size cap before `bhttp.parseResponse` is called, and cross-reference the Phase 2 finding to characterize the two-layer amplification risk.
9. Confirm that `OhttpHeader.name` is not lowercased on the response side, documenting the inconsistency with the request side.
10. Produce a set of draft follow-up task entries for every confirmed finding, each with a file reference, line range, concern category, and severity classification.

---

## User Stories

**US-1 — Security reviewer**
As a security reviewer preparing the package for wallet integration, I need a line-by-line audit of `ohttp_client.dart` so I can determine whether the network client introduces reliability gaps, privacy risks, or unhandled error conditions that must be closed before the package is used to transmit wallet transactions.

**US-2 — Engineering lead**
As an engineering lead, I need each finding to be classified by severity (BLOCKER / HIGH / IMPROVEMENT) and mapped to a specific code location, so I can prioritize follow-up tasks and assign them to engineers without further triage.

**US-3 — Wallet integrator**
As a developer integrating `ohttp_dart` into a non-custodial crypto wallet, I need to know whether the client layer exposes user IP addresses, wallet addresses, or transaction data through missing transport-layer enforcement, untyped errors that swallow security signals, or an undocumented plaintext fallback path.

**US-4 — Platform reliability engineer**
As an engineer responsible for the reliability of wallet network calls, I need to know whether `OhttpClient` provides any timeout, retry, or cancellation mechanism, or whether the wallet must implement all resilience logic itself around every `send()` call.

---

## Main Scenarios

### Scenario 1 — KeyConfig GET timeout and caching absence (tasks 4.6)
Analyst reads the `send()` method and locates the `http.Client.get()` call that fetches `KeyConfig`. Confirms there is no `.timeout(Duration(...))` chained on the returned `Future`, no `http.Client` subclass with a built-in timeout, and no stored reference to a previously fetched `OhttpKeyConfig`. Records the exact line of the `get()` call and the line where `OhttpKeyConfig.parse` is invoked on the fresh response. Notes that every `send()` invocation produces two network round trips: one for `KeyConfig` and one for the gateway POST.

### Scenario 2 — Gateway POST timeout and cancellation absence (task 4.7)
Analyst locates the `http.Client.post()` call for the gateway POST. Confirms there is no `.timeout(Duration(...))` chained on the `Future`, no cancellation token or `CancelableOperation` pattern, and no retry logic triggered by 5xx responses. Records the exact call site line.

### Scenario 3 — `https`-scheme non-enforcement in `OhttpGatewayConfig` (task 4.2)
Analyst reads `OhttpGatewayConfig` constructor or factory. Confirms that `gatewayBaseUrl` is accepted as a bare string or `Uri` with no assertion on the scheme. Traces `Uri.parse(gatewayBaseUrl)` at the call site to confirm it accepts any scheme. Documents the threat: an `http://` gateway URL silently transmits all outer-channel traffic (including the encapsulated BHTTP ciphertext) without TLS, allowing a network observer to correlate request timing and size even though the inner request remains encrypted.

### Scenario 4 — Path string concatenation vs. `Uri` normalization (task 4.3)
Analyst traces the construction of the KeyConfig URL and the gateway POST URL. Confirms both are formed by `'${config.gatewayBaseUrl}${config.configPath}'` and `'${config.gatewayBaseUrl}${config.requestPath}'` (string concatenation). Notes the double-slash risk when `gatewayBaseUrl` ends with `/` and `configPath` or `requestPath` begin with `/`. Notes that `..` path components are not normalized away, creating a path-traversal vector if either path field accepts user-supplied input. Records both lines.

### Scenario 5 — `targetAuthority` verbatim embedding (task 4.4)
Analyst locates the `bhttp.serializeRequest` call in `send()`. Confirms that `targetAuthority` from `OhttpGatewayConfig` is passed directly as the `authority` argument with no allow-list check, no URI-scheme stripping, and no validation against a set of permitted target servers. Documents the privacy implication: a misconfigured or maliciously supplied `targetAuthority` could route the inner request to an unintended server while the outer OHTTP layer provides no visibility into this misdirection.

### Scenario 6 — `sendDirect()` bypass and absence of privacy warning (task 4.5)
Analyst reads `sendDirect()`. Confirms it issues a plain HTTP request to `directBaseUrl` without any OHTTP encapsulation. Confirms there is no `assert`, no comment, no `@Deprecated`, and no `log` call warning that this method bypasses OHTTP privacy guarantees. Notes that `sendDirect()` and `send()` are methods on the same class, configured by the same `OhttpGatewayConfig` object that contains `directBaseUrl` alongside the OHTTP paths. Documents the wallet risk: a developer using `sendDirect()` as a connectivity fallback silently reveals the client IP address and full request details to the target server, defeating RFC 9458 §1 privacy goals with no runtime signal.

### Scenario 7 — Generic `Exception` on non-200 responses (task 4.8)
Analyst locates all `throw Exception(...)` or equivalent statements that are triggered by non-200 status codes in both the KeyConfig GET path and the gateway POST path. Confirms there is no typed exception hierarchy distinguishing permanent errors (4xx) from transient errors (5xx), and no structured error object carrying the status code or response body. Documents that callers cannot implement differentiated retry logic without inspecting the exception message string, which is fragile.

### Scenario 8 — Response body and header size caps (task 4.10)
Analyst confirms that `gatewayResponse.bodyBytes` is passed directly to `ohttpDecapsulate` with no length check. Confirms that the resulting decrypted bytes are passed to `bhttp.parseResponse` with no size limit. Cross-references the Phase 2 finding (no max-header-count or max-body-size guard in `parseResponse`) to establish the two-layer amplification: a malicious gateway can force unbounded memory allocation in the client process through a single oversized response. Records as a compounded `HIGH` finding.

### Scenario 9 — `OhttpHeader.name` case inconsistency (task 4.11)
Analyst locates the response header parsing code in `ohttp_client.dart`. Confirms that `OhttpHeader.name` is stored as-received from the gateway (not lowercased). Confirms that on the request side headers are treated in a way that differs from the response side. Records the line where response headers are constructed and notes that HTTP/1.1 header field names are case-insensitive (RFC 7230 §3.2); inconsistent casing on the response side forces callers to perform case-insensitive comparison manually.

### Scenario 10 — Draft task entry compilation (task 4.12)
For each confirmed finding from Scenarios 1–9, the analyst produces a draft task entry containing: finding description, affected file and line range, concern category (network reliability / KeyConfig lifecycle / scheme enforcement / privacy / error handling / observability), severity classification, and a proposed task title for Iteration 7 consolidation.

---

## Success / Metrics

| Criterion | Definition of done |
|---|---|
| All ten investigation goals covered | Tasks 4.2–4.11 each produce a positive ("confirmed as described") or negative ("not present — no issue") verdict with a specific line reference in `ohttp_client.dart`. No goal from the vision §4 scrutiny table for `ohttp_client.dart` is skipped. |
| Timeout and caching findings cite call-site lines | The "no timeout" finding and the "no KeyConfig caching" finding each cite the specific `http.Client` call site line number in `ohttp_client.dart`. |
| `sendDirect()` finding documents privacy impact | The `sendDirect()` finding explicitly states that OHTTP privacy is bypassed, references RFC 9458 §1, and proposes a concrete remediation target (documentation or assertion). |
| Two-layer response-size finding cross-references Phase 2 | The size-cap finding explicitly references the Phase 2 BHTTP finding on `parseResponse` to characterize the amplification. |
| Severity assigned to every finding | Every finding carries exactly one of `BLOCKER`, `HIGH`, or `IMPROVEMENT`. |
| No code changes | `git diff HEAD -- lib/ test/` is empty at the end of this phase. |
| Draft tasks independently actionable | Each draft task can be assigned to a single engineer without requiring another in-flight task from this list to be completed first. |

---

## Constraints and Assumptions

1. **Read-only.** No file in `lib/` or `test/` is modified during this phase.
2. **`package:http` primitives assumed correct.** The audit does not evaluate the `package:http` client implementation. It verifies only how `ohttp_client.dart` uses the provided `http.Client` interface.
3. **Empty AAD is intentional (inherited from Phase 3).** The empty-AAD choice in `ohttpEncapsulate` is already documented in Phase 3. This phase does not re-examine it.
4. **`sendDirect()` scope.** The audit documents the absence of a privacy warning. It does not propose removing `sendDirect()` — that is a design decision for follow-up tasks. The finding records severity and proposes remediation options (assertion, `@Deprecated`, or inline documentation).
5. **Scheme enforcement scope.** The audit documents that no `https` check exists. Whether the check belongs in `OhttpGatewayConfig` constructor, in `send()`, or in a validation method is a remediation design question left to the follow-up task.
6. **No live gateway.** This phase contains no integration testing against a real OHTTP gateway. Findings are derived from static code reading.
7. **Cross-layer concerns.** Findings that compound Phase 2 or Phase 3 risks (response size caps, `SecretBoxAuthenticationError` propagation) explicitly reference the earlier phase findings rather than re-deriving them.
8. **`OhttpHeader.name` inconsistency is not a protocol defect.** The response header case inconsistency is classified as `IMPROVEMENT`. It does not affect correctness of the OHTTP protocol implementation.

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| No timeout on `http.Client.get()` or `http.Client.post()` means a stalled gateway can hang the wallet indefinitely, blocking all OHTTP-protected operations with no recovery path | HIGH | Scenario 1 and Scenario 2 confirm the absence; remediation task adds a configurable `Duration` timeout to both calls |
| No KeyConfig caching doubles the network cost of every OHTTP request; under poor network conditions, the KeyConfig GET may fail intermittently, causing `send()` to throw even when the gateway POST would have succeeded | HIGH | Scenario 1 confirms the fresh-fetch behavior; remediation task adds a configurable TTL cache with forced refresh on 4xx |
| No `https`-scheme enforcement means a misconfigured wallet silently transmits all OHTTP requests over plaintext HTTP, exposing the outer channel to passive observation even though the inner BHTTP payload is encrypted | HIGH | Scenario 3 documents the gap; remediation adds a `Uri.parse(gatewayBaseUrl).scheme == 'https'` assertion or constructor validation |
| String path concatenation (`gatewayBaseUrl + configPath`) produces double-slash URLs for common trailing-slash / leading-slash configurations and does not normalize `..` path traversal | IMPROVEMENT | Scenario 4 documents the two call sites; remediation replaces concatenation with `Uri.parse(base).resolve(path)` |
| `targetAuthority` embedded verbatim with no allow-list exposes wallets to misconfigured or adversarially supplied target servers; inner requests may reach unintended hosts with no indication in the outer OHTTP response | HIGH | Scenario 5 documents the embedding; remediation task adds an allow-list check or at minimum a URI-scheme validation before `serializeRequest` |
| `sendDirect()` has no privacy warning; a developer who uses it as a fallback silently routes wallet traffic to the target server without OHTTP, revealing the client IP address to the target server in violation of the wallet's privacy model | HIGH | Scenario 6 documents the absence; remediation adds a mandatory inline doc comment, `assert`, or `@Deprecated` annotation |
| Generic `Exception` on non-200 responses prevents callers from distinguishing permanent gateway errors (4xx) from transient ones (5xx), making reliable retry logic impossible without fragile string parsing | HIGH | Scenario 7 documents all throw sites; remediation introduces a typed `OhttpClientException` hierarchy carrying the status code |
| No response body size cap before `ohttpDecapsulate` and `bhttp.parseResponse` means a malicious gateway can force unbounded memory allocation; combined with the Phase 2 finding (no header cap in `parseResponse`), this is a two-layer amplification risk | HIGH | Scenario 8 documents the two-layer risk; remediation adds a configurable `maxResponseBytes` guard before decapsulation |
| `OhttpHeader.name` stored as-received on the response side forces callers to implement case-insensitive header lookup manually, contrary to RFC 7230 §3.2 | IMPROVEMENT | Scenario 9 documents the inconsistency; remediation lowercases `name` at parse time on the response side |
| Audit findings may be incomplete if `ohttp_client.dart` was modified after the vision was written | LOW | Task 4.1 (full outline read via `ast-index outline`) serves as a completeness gate |

---

## Open Questions

All questions have been resolved prior to finalizing this PRD. No open questions remain. See context notes below for standing decisions.

- **`sendDirect()` severity** — Classified as `HIGH` (not `BLOCKER`), because it requires a deliberate misconfiguration to activate; it is not triggered by the default `send()` path. The finding is still treated as "should fix before wallet use".
- **Timeout default value** — The specific default `Duration` is left to the follow-up task author; the PRD requires only that a configurable timeout be present, not a specific value.
- **`https`-scheme enforcement placement** — Whether validation belongs in the constructor, a `validate()` method, or the `send()` call is a remediation design question. The PRD records the absence; the follow-up task resolves placement.
