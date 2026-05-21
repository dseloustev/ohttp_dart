# Task 31: Improve example file documentation

**Severity:** IMPROVEMENT
**Vector:** Documentation
**Files:** `example/ohttp_dart_example.dart`

**Evidence:** Cross-ref P6-11. The example file exists but does not document the privacy-impact distinction between `send()` and `sendDirect()`, does not warn about the `effectiveDirectBaseUrl` fallback (task 01 BLOCKER), and does not enumerate the expected exception surface.

## Description

The example is the first file a wallet developer reads when integrating the library. Its responsibility is therefore disproportionately large: a missing comment about `sendDirect()` directly contributes to the BLOCKER 01 footgun, because the developer will assume the example is showing safe usage. Improving the example does not change behaviour but materially reduces misuse.

## Proposed change

Rework the example to:
1. Demonstrate only `send()` in the main flow.
2. If `sendDirect()` is shown, mark it loudly as "do not use in production — bypasses OHTTP — see task 01" and require an explicit non-`gatewayBaseUrl` `directBaseUrl`.
3. List the exceptions a caller should expect to handle (per task 10's typed hierarchy once landed).
4. Add doc comments explaining the meaning of every `OhttpGatewayConfig` field, especially `targetAuthority`.
5. Reference RFC 9458 §1 for the unlinkability guarantees the example relies on.

## Acceptance criteria

- Example file's `main()` flow uses only `send()`.
- Any reference to `sendDirect()` includes a prominent warning and is gated behind an explicit comment.
- Doc comments enumerate fields and expected exceptions.
- RFC reference is present.
