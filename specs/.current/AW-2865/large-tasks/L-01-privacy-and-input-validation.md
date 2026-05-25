# Privacy & input-validation foundation for `OhttpGatewayConfig` / `sendDirect`

**Estimate:** 3d
**Priority:** P1: Critical
**Component:** App
**Severity:** BLOCKER

## Task Description

`OhttpGatewayConfig` and the companion `sendDirect()` helper sit on the trust boundary between the wallet and the relay/gateway. Two compounding defects open privacy footguns that must be closed together before any other follow-up lands:

1. **Silent `sendDirect` privacy bypass.** When `OhttpGatewayConfig` is constructed without the optional `directBaseUrl`, a subsequent call to `sendDirect()` issues plaintext HTTP directly to the relay host — no encryption, no unlinkability, no runtime signal that the OHTTP flow was bypassed. The silent `directBaseUrl ?? gatewayBaseUrl` fallback at `lib/src/ohttp_client.dart:52` turns a "comparison" helper into a privacy-breaking footgun, widened further by the public getter `effectiveDirectBaseUrl` on the same line. For a non-custodial crypto wallet, this is the difference between an unlinkable RPC call and a plaintext leak of wallet activity to the relay operator. The example file `example/ohttp_dart_example.dart` does not warn about any of this.

2. **No validation of `gatewayBaseUrl` scheme or `targetAuthority` shape.** Both fields are accepted as raw `String` with no validation. The library silently accepts `http://` URLs — RFC 9458 §1 requires TLS on the outer channel — and the outer OHTTP ciphertext is then transmitted in cleartext, giving a passive on-path observer the ability to perform timing correlation against the gateway and undermining the unlinkability property that OHTTP exists to provide. `targetAuthority` accepts any string, so a malformed value (`"https://host"`, leading/trailing whitespace, embedded path) silently produces a malformed inner BHTTP request.

A wallet caller using a typo'd configuration or a misconfigured staging URL has no signal that the deployment is broken until a network observer is already capturing traffic, or until the gateway returns a parse error on the inner request.

## Technical Details

All changes land together on `OhttpGatewayConfig` / `OhttpClient.sendDirect()` (`lib/src/ohttp_client.dart`) plus the example file.

### Part A — Close the `sendDirect` privacy bypass

**1. `sendDirect()` no longer silently bypasses OHTTP** (`lib/src/ohttp_client.dart:147-171`)

The `sendDirect()` method currently carries a single doc line ("Send a direct HTTP request (for comparison)."), no `@Deprecated`, no `assert`, and no privacy-impact warning. The body at lines 148–171 calls `_httpClient.send()` against `effectiveDirectBaseUrl`, which resolves to `gatewayBaseUrl` when `directBaseUrl` is null.

Required:
- The method must fail fast (assertion or thrown library-owned exception) when `directBaseUrl` is null or equal to `gatewayBaseUrl` — never silently send plaintext to the relay.
- Add a `@Deprecated('...')` annotation that names the privacy impact.
- The doc comment must explicitly state that calling `sendDirect()` bypasses the OHTTP pipeline entirely — no HPKE encryption of the inner request, no relay indirection, no unlinkability — and must enumerate the RFC 9458 §1 guarantees the caller forfeits by choosing this path (confidentiality of the inner request from the relay operator, unlinkability between client identity and request content, metadata isolation from the target server's view of the client).

**2. Remove or privatize the `effectiveDirectBaseUrl` public getter** (`lib/src/ohttp_client.dart:52`)

The getter is currently the public surface that makes the silent fallback discoverable. Rename to `_effectiveDirectBaseUrl` (library-private) or move the resolution logic inline into the `sendDirect()` body at line 154 so the public surface never exposes the fallback by name. Update the constructor doc comment so it no longer references the fallback path as a public concept.

**3. Rework the example file** (`example/ohttp_dart_example.dart`)

- Demonstrate only `send()` in the main flow.
- If `sendDirect()` is shown at all, gate it behind a loud "do not use in production — bypasses OHTTP" comment and require an explicit non-`gatewayBaseUrl` `directBaseUrl`.
- Add doc comments explaining the meaning of every `OhttpGatewayConfig` field, especially `targetAuthority` (the privacy-sensitive backend host embedded in the inner BHTTP envelope).
- List the exceptions a caller should expect to handle.
- Reference RFC 9458 §1 for the unlinkability guarantees the example relies on.

### Part B — Validate `OhttpGatewayConfig` inputs

Two related validations land at construction time on `OhttpGatewayConfig` (`lib/src/ohttp_client.dart:43-51`):

**4. HTTPS scheme enforcement**

The constructor (lines 43–50) and the call sites that build URIs from these fields (line 82 for KeyConfig GET, line 116 for gateway POST) currently accept any scheme. Reject any base URL whose scheme is not `https://`, for both `gatewayBaseUrl` and `directBaseUrl` (when supplied). Throw a library-owned exception at construction time. Document the rationale (RFC 9458 §1) in the constructor doc comment.

If an escape hatch for tests is needed, gate it behind an explicit named flag (e.g., `allowInsecureScheme: false` default) so production code paths cannot accidentally pass through it.

**5. `targetAuthority` shape validation** (`lib/src/ohttp_client.dart:97-104`)

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
| **Notion** | https://www.notion.so/adguard/OHTTP-Dart-360aa56b773080619205ff76a2a360f9 |

## Acceptance Criteria

**sendDirect privacy bypass:**
- Calling `sendDirect()` without an explicit `directBaseUrl` distinct from `gatewayBaseUrl` fails fast (assertion or library-owned exception), never silently sending plaintext to the relay.
- The `sendDirect()` doc comment names the privacy impact (no encryption, no unlinkability) and references the RFC 9458 §1 guarantees that are bypassed.
- A `@Deprecated` annotation (or equivalent compile-time signal) is present on the method.
- `effectiveDirectBaseUrl` is no longer part of the public API of `OhttpGatewayConfig`; internal call sites continue to work.
- Doc comment on `OhttpGatewayConfig` no longer references the fallback path as a public concept.
- Example file's `main()` flow uses only `send()`; any reference to `sendDirect()` includes a prominent warning and requires an explicit `directBaseUrl`.
- Example doc comments enumerate every `OhttpGatewayConfig` field and the expected exception surface.

**Input validation:**
- Constructing `OhttpGatewayConfig` with `http://` (or any non-`https://`) `gatewayBaseUrl` throws a library-owned exception at construction time.
- The same scheme validation rejects `http://` `directBaseUrl` when supplied.
- Constructing `OhttpGatewayConfig` with an invalid `targetAuthority` (empty, scheme-prefixed, whitespace-containing, path-containing) throws a library-owned exception at construction time.
- Constructor doc comment cites RFC 9458 §1 for the scheme requirement and RFC 3986 §3.2 for the authority shape.

**Tests:**
- Unit test exercises the silent-fallback `sendDirect` configuration and asserts that the call is rejected.
- Scheme validation tests cover: valid `https://`, rejected `http://`, rejected `ftp://`, rejected empty string.
- Authority validation tests cover: valid `host`, valid `host:port`, rejected empty, rejected scheme-prefixed, rejected whitespace-containing, rejected path-containing values.

## Additional

- Tests: unit tests on `OhttpGatewayConfig` construction covering each accepted/rejected input class plus the `sendDirect` rejection path; manual verification of the example (`dart run example/ohttp_dart_example.dart` after editing the URL).
- This task is the first that should land — every other follow-up assumes the privacy footgun is closed and that the construction surface rejects malformed inputs.
- Source merged tasks (in `../merged-tasks/`): M-01 (`sendDirect` privacy bypass), M-02 (gateway config validation).
