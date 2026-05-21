# Task 13: Add structured observability hooks (with strict logging constraints)

**Severity:** HIGH
**Vector:** Documentation
**Files:** `lib/src/ohttp_client.dart:81-89, 115-122, 148-171`

**Evidence:** Cross-refs P6-5, P6-6, P6-8, T-X1 (row 22 note). This file is the merged target of rows 13 AND 22 of the Phase 7 Consolidated Findings Inventory — the gap at file number 22 in this directory is intentional. The existing `onLog` callback at `ohttp_client.dart:74` is a single `void Function(String)?` parameter that currently passes formatted strings containing `gatewayBaseUrl + configPath` (line 77) and `gatewayBaseUrl + requestPath` (line 113), which violates the §4 logging constraints below.

## Description

The library has no structured observability surface: there is one optional `onLog` string callback, used inconsistently, that already leaks sensitive path information. Wallet operators need actionable signals (KeyConfig fetch outcome, gateway POST status, decryption success/failure) to diagnose production issues, but those signals must not include any of the materials enumerated in the logging constraints below. The current ad-hoc string logging is both insufficient (no structure, no levels) and unsafe (mixes safe and unsafe fields into a single string). This task is the merged scope of inventory rows 13 (documentation gap) and 22 (interoperability/T-X1 observability note).

## Proposed change

Define a structured event interface (e.g., `OhttpObserver` with typed event methods such as `onKeyConfigFetch(statusCode)`, `onGatewayPost(statusCode)`, `onSendDirectInvoked()`, `onDecryptionFailure()`). Replace the existing `onLog` callback (a breaking change) or deprecate it alongside the new typed surface; document the migration path. Every emitter must obey the constraints below.

**§4 Logging constraints — verbatim:**

Never log: (1) key material, (2) enc value, (3) inner request URL/path/method/headers/body, (4) targetAuthority, (5) response body/headers, (6) gatewayBaseUrl at DEBUG or lower in production.

Safe to log: KeyConfig fetch success/failure (status code only), gateway POST status code (not body), sendDirect() invocation signal.

These constraints must be reproduced verbatim in the public doc comment of the observability interface so they cannot be lost in a refactor.

## Acceptance criteria

- A typed observability interface is defined and re-exported from `lib/ohttp_dart.dart`.
- All emission sites in `OhttpClient` produce structured events that obey the §4 logging constraints — no key material, no `enc`, no inner request details, no `targetAuthority`, no response body/headers, no `gatewayBaseUrl` at DEBUG or lower in production builds.
- The existing string `onLog` callback is either removed (breaking change, documented) or deprecated with a migration note.
- A `sendDirect()` invocation produces a distinct, documented event so wallet operators can alert on accidental fallback use (companion to task 01).
- Doc comment on the observability interface reproduces the §4 constraints verbatim.
- Unit test verifies no emitted event contains any restricted field for a successful and a failing flow.
