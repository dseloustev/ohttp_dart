# Eliminate `sendDirect` privacy bypass and update example

**Estimate:** 2d
**Priority:** P1: Critical
**Component:** App
**Severity:** BLOCKER

## Task Description

Currently, when `OhttpGatewayConfig` is constructed with only the required parameters (omitting the optional `directBaseUrl`), a subsequent call to `sendDirect()` issues plaintext HTTP directly to the relay host — no encryption, no unlinkability, no runtime signal that the OHTTP flow was bypassed. The silent `directBaseUrl ?? gatewayBaseUrl` fallback at `lib/src/ohttp_client.dart:52` turns a "comparison" helper into a privacy-breaking footgun.

For a non-custodial crypto wallet, this is the difference between an unlinkable RPC call and a plaintext leak of wallet activity to the relay operator. The footgun is widened by the public getter `effectiveDirectBaseUrl` on the same line, which exposes the same `null → gatewayBaseUrl` resolution as a callable, documented part of the public API. The example file `example/ohttp_dart_example.dart` does not warn about any of this — nothing in the example signals that `sendDirect()` is unsafe by default.

## Technical Details

Three changes land together in this task:

**1. `sendDirect()` no longer silently bypasses OHTTP** (`lib/src/ohttp_client.dart:147-171`)

The `sendDirect()` method currently carries a single doc line ("Send a direct HTTP request (for comparison)."), no `@Deprecated`, no `assert`, and no privacy-impact warning. The body at lines 148–171 calls `_httpClient.send()` against `effectiveDirectBaseUrl`, which resolves to `gatewayBaseUrl` when `directBaseUrl` is null.

Required:
- The method must fail fast (assertion or thrown library-owned exception) when `directBaseUrl` is null or equal to `gatewayBaseUrl` — never silently send plaintext to the relay.
- Add a `@Deprecated('...')` annotation that names the privacy impact.
- The doc comment must explicitly state that calling `sendDirect()` bypasses the OHTTP pipeline entirely — no HPKE encryption of the inner request, no relay indirection, no unlinkability — and must enumerate the RFC 9458 §1 guarantees the caller forfeits by choosing this path (confidentiality of the inner request from the relay operator, unlinkability between client identity and request content, metadata isolation from the target server's view of the client). This bullet covers the documentation of the deliberate-bypass method only; RFC 9458 compliance hardening of the actual `send()` pipeline is addressed in other tasks.

**2. Remove or privatize the `effectiveDirectBaseUrl` public getter** (`lib/src/ohttp_client.dart:52`)

The getter is currently the public surface that makes the silent fallback discoverable. Rename to `_effectiveDirectBaseUrl` (library-private) or move the resolution logic inline into the `sendDirect()` body at line 154 so the public surface never exposes the fallback by name. Update the constructor doc comment so it no longer references the fallback path as a public concept.

**3. Rework the example file** (`example/ohttp_dart_example.dart`)

- Demonstrate only `send()` in the main flow.
- If `sendDirect()` is shown at all, gate it behind a loud "do not use in production — bypasses OHTTP" comment and require an explicit non-`gatewayBaseUrl` `directBaseUrl`.
- Add doc comments explaining the meaning of every `OhttpGatewayConfig` field, especially `targetAuthority` (the privacy-sensitive backend host embedded in the inner BHTTP envelope).
- List the exceptions a caller should expect to handle.
- Reference RFC 9458 §1 for the unlinkability guarantees the example relies on.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | https://www.notion.so/adguard/OHTTP-Dart-360aa56b773080619205ff76a2a360f9 |

## Acceptance Criteria

- Calling `sendDirect()` without an explicit `directBaseUrl` distinct from `gatewayBaseUrl` fails fast (assertion or library-owned exception), never silently sending plaintext to the relay.
- The `sendDirect()` doc comment names the privacy impact (no encryption, no unlinkability) and references the RFC 9458 §1 guarantees that are bypassed.
- A `@Deprecated` annotation (or equivalent compile-time signal) is present on the method.
- `effectiveDirectBaseUrl` is no longer part of the public API of `OhttpGatewayConfig`; internal call sites continue to work.
- Doc comment on `OhttpGatewayConfig` no longer references the fallback path as a public concept.
- Example file's `main()` flow uses only `send()`; any reference to `sendDirect()` includes a prominent warning and requires an explicit `directBaseUrl`.
- Example doc comments enumerate every `OhttpGatewayConfig` field and the expected exception surface.
- Unit test exercises the silent-fallback configuration and asserts that the call is rejected.

## Additional

- Tests: unit tests for the assertion/exception path; manual verification of the example (`dart run example/ohttp_dart_example.dart` after editing the URL).
- This task is the first that should land — every other follow-up assumes the privacy footgun is closed.
