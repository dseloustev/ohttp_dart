# Validate `gatewayBaseUrl` scheme and `targetAuthority` on `OhttpGatewayConfig`

**Estimate:** 1d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

`OhttpGatewayConfig` accepts both `gatewayBaseUrl` (the outer relay/gateway URL) and `targetAuthority` (the privacy-sensitive backend host embedded inside the encrypted BHTTP envelope) as raw `String` parameters with no validation. The library silently accepts `http://` URLs — RFC 9458 §1 requires TLS on the outer channel — and the outer OHTTP ciphertext is then transmitted in cleartext, giving a passive on-path observer the ability to perform timing correlation against the gateway and undermining the unlinkability property that OHTTP exists to provide. Similarly, `targetAuthority` accepts any string, so a malformed value (`"https://host"`, leading/trailing whitespace, embedded path) silently produces a malformed inner BHTTP request.

A wallet caller using a typo'd configuration or a misconfigured staging URL has no signal that the deployment is broken until a network observer is already capturing traffic, or until the gateway returns a parse error on the inner request.

## Technical Details

Two related validations land at construction time on `OhttpGatewayConfig` (`lib/src/ohttp_client.dart:43-51`):

**1. HTTPS scheme enforcement**

The constructor (lines 43–50) and the call sites that build URIs from these fields (line 82 for KeyConfig GET, line 116 for gateway POST) currently accept any scheme. Reject any base URL whose scheme is not `https://`, for both `gatewayBaseUrl` and `directBaseUrl` (when supplied). Throw a library-owned exception at construction time. Document the rationale (RFC 9458 §1) in the constructor doc comment.

If an escape hatch for tests is needed, gate it behind an explicit named flag (e.g., `allowInsecureScheme: false` default) so production code paths cannot accidentally pass through it.

**2. `targetAuthority` shape validation** (`lib/src/ohttp_client.dart:97-104`)

`targetAuthority` is embedded verbatim into the inner BHTTP request at `bhttp.serializeRequest(...)` (lines 97–104). The accepted shape is the RFC 3986 `authority` production: `host` or `host:port`. Reject:
- empty values,
- values containing a scheme prefix (`http://`, `https://`, etc.),
- values containing whitespace,
- values containing path or query characters (`/`, `?`, `#`),
- values containing fragment characters.

Throw a library-owned exception at construction time. Document the expected shape in the constructor doc comment with reference to RFC 3986 §3.2.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- Constructing `OhttpGatewayConfig` with `http://` (or any non-`https://`) `gatewayBaseUrl` throws a library-owned exception at construction time.
- The same scheme validation rejects `http://` `directBaseUrl` when supplied.
- Constructing `OhttpGatewayConfig` with an invalid `targetAuthority` (empty, scheme-prefixed, whitespace-containing, path-containing) throws a library-owned exception at construction time.
- Constructor doc comment cites RFC 9458 §1 for the scheme requirement and RFC 3986 §3.2 for the authority shape.
- Unit tests cover, for the scheme: valid `https://`, rejected `http://`, rejected `ftp://`, rejected empty string.
- Unit tests cover, for the authority: valid `host`, valid `host:port`, rejected empty, rejected scheme-prefixed, rejected whitespace-containing, rejected path-containing values.

## Additional

- Tests: unit tests on `OhttpGatewayConfig` construction covering each accepted/rejected input class.
