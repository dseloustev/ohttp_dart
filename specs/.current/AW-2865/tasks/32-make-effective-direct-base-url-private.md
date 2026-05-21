# Task 32: Make `effectiveDirectBaseUrl` private or remove from public API

**Severity:** HIGH
**Vector:** Privacy risks
**Files:** `lib/src/ohttp_client.dart:52`

**Evidence:** Cross-ref P6-13.

- **Original classification:** IMPROVEMENT.
- **New classification:** HIGH.
- **Escalation rationale:** `effectiveDirectBaseUrl` is a **public getter** on `OhttpGatewayConfig` (line 52 of `lib/src/ohttp_client.dart`). Because it is public, any caller — including test harnesses, logging utilities, and debugging code in downstream wallet apps — can read it and observe that the value is `gatewayBaseUrl` when `directBaseUrl` is null. This widens the blast radius of the BLOCKER task 01 footgun: a developer who calls the getter to "check what URL will be used" receives a silently incorrect answer (sees `gatewayBaseUrl`, may conclude `sendDirect()` targets the gateway/relay). The getter is the public surface that makes the silent fallback discoverable and the footgun reachable via IDE autocomplete. Task 01 (BLOCKER) addresses the unsafe action (`sendDirect()`); this task is its **companion** — it removes the discoverable API surface entirely rather than just warning about it after the fact. For a wallet context where unlinkability is a first-class security property, the escalation from IMPROVEMENT to HIGH reflects the importance of shrinking the public API footprint that exposes this trap.

## Description

The getter silently resolves `null → gatewayBaseUrl` in a context where conflating the two is the BLOCKER described in task 01. The same `directBaseUrl ?? gatewayBaseUrl` pattern that drives the BLOCKER is exposed by name on the public surface. Hiding it (making it library-private or removing it) does not by itself prevent the BLOCKER — task 01 handles the runtime path — but it eliminates the discoverable, named, plausible-looking accessor that makes the unsafe behaviour easy to find and easy to misuse.

## Proposed change

Make `effectiveDirectBaseUrl` library-private (rename to `_effectiveDirectBaseUrl`, or move the resolution logic into the `sendDirect()` body so the public surface never exposes the fallback). The internal call site at line 154 of `ohttp_client.dart` must continue to work; everything else is hidden. Coordinate with task 01 so the deprecation/assertion of `sendDirect()` and the API-surface change land together (or in adjacent commits).

## Acceptance criteria

- `effectiveDirectBaseUrl` is no longer part of the public API of `OhttpGatewayConfig`.
- The internal `sendDirect()` flow continues to work without exposing the fallback by name.
- Doc comment on `OhttpGatewayConfig` no longer references the fallback path as a public concept.
- Any test that previously read `effectiveDirectBaseUrl` for assertion purposes is rewritten to test behaviour, not the resolved string.
- Migration note is added (the package is unpublished — `publish_to: none` — so breaking-change scope is internal consumers only).
- Cross-reference task 01 in the change description.
