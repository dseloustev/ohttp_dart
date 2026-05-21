# Task 26: Lowercase response header names

**Severity:** IMPROVEMENT
**Vector:** Interoperability
**Files:** `lib/src/ohttp_client.dart:139`

**Evidence:** Cross-ref C-10. Line 139: `.map((e) => OhttpHeader(name: e.key, value: e.value))`. The `e.key` comes directly from `streamedResponse.headers.entries` with no case normalization.

## Description

HTTP/1.1 header names are case-insensitive (RFC 9110 §5.1), but different HTTP clients deliver them in different cases. Downstream consumers that compare header names with `==` or look up by exact string see inconsistent behaviour depending on which version of the gateway, the relay, or the transport stack is in play. Lowercasing at the boundary normalizes the contract.

## Proposed change

Lowercase the header name when constructing `OhttpHeader` at line 139. Document the normalization in the `OhttpHeader` doc comment so downstream consumers know they can rely on it.

## Acceptance criteria

- Line 139 lowercases the header name before constructing `OhttpHeader`.
- `OhttpHeader` doc comment documents that the `name` field is lowercase.
- Unit test feeds mixed-case header names through `OhttpClient.send` and asserts the resulting `OhttpHeader.name` is lowercase.
