# Task 24: Add size cap on `serializeRequest` inputs

**Severity:** IMPROVEMENT
**Vector:** Parser robustness
**Files:** `lib/src/bhttp.dart:74-111`

**Evidence:** Cross-ref F2.7. Lines 74–111 of `lib/src/bhttp.dart` are `serializeRequest`. No size cap is applied to `method`, `scheme`, `authority`, `path`, `headers`, or `body` before they are varint-encoded into the output buffer.

## Description

An accidental wallet bug that passes a multi-megabyte body or pathological header set to `serializeRequest` would produce a giant BHTTP request, allocate the corresponding ciphertext, and POST it to the gateway. The result is wasted network traffic and a likely gateway rejection, but the error surfaces only after the round trip. A client-side cap fails fast with a clear message.

## Proposed change

Apply configurable upper bounds on the request inputs before serialization begins. Reject oversized inputs with a typed library exception (coordinate with task 10). Defaults should be conservative and documented; wallet RPC payloads rarely exceed a few hundred kilobytes.

## Acceptance criteria

- `serializeRequest` rejects inputs that exceed configured upper bounds for method/scheme/authority/path/headers/body length.
- Defaults are documented and conservatively low.
- A typed library exception is thrown on overflow.
- Unit tests cover each over-cap input class.
