# Task 03: Restrict and validate `targetAuthority`

**Severity:** HIGH
**Vector:** Privacy risks
**Files:** `lib/src/ohttp_client.dart:97-104`

**Evidence:** Cross-ref C-3. At lines 97–104, `bhttp.serializeRequest(...)` embeds `gateway.targetAuthority` verbatim into the inner BHTTP request without validation. `targetAuthority` is the privacy-sensitive endpoint that the gateway will forward the inner request to.

## Description

`targetAuthority` is the host name (and optional port) embedded inside the encrypted BHTTP envelope; it identifies the wallet's true backend to the gateway. The constructor accepts any string with no shape check — a malformed value (`"https://host"`, leading/trailing whitespace, embedded path) silently produces a malformed inner request, and a wildcard or empty value can be used unintentionally. Tight validation reduces the chance that a misconfigured wallet sends ill-formed requests through the relay or accidentally targets a different backend.

## Proposed change

Validate `targetAuthority` at `OhttpGatewayConfig` construction time. Reject empty values, values containing a scheme prefix, values containing whitespace, and values containing path or query characters. Document the expected shape (`host` or `host:port`) in the constructor doc comment with reference to the RFC 3986 `authority` production.

## Acceptance criteria

- Constructing `OhttpGatewayConfig` with an invalid `targetAuthority` throws a library-owned exception at construction time.
- The constructor doc comment documents the accepted authority syntax.
- Unit tests cover: valid `host`, valid `host:port`, rejected empty, rejected scheme-prefixed, rejected whitespace-containing, rejected path-containing values.
