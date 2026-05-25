# Input validation for `HttpClientTransport` and `OhttpRequestData`

**Estimate:** 1.5d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

After the L-01 restructure, two construction surfaces live on the trust boundary between the wallet (or any consumer) and the relay/gateway: `HttpClientTransport` (which holds the gateway URLs) and `OhttpRequestData` (which carries the inner BHTTP target authority). Both currently accept raw strings/URIs without validation:

1. **No scheme validation on `keysUrl` / `gatewayUrl`** (`HttpClientTransport` constructor). RFC 9458 §1 requires TLS on the outer channel between client and relay. A wallet using a typo'd configuration or a misconfigured staging URL could silently fall back to `http://`, transmitting the outer OHTTP ciphertext in cleartext — giving a passive on-path observer the ability to perform timing correlation against the gateway and undermining the unlinkability property that OHTTP exists to provide.

2. **No shape validation on `authority`** (`OhttpRequestData` constructor; also extracted from `request.url` inside `OhttpHttpClient.send`). The field is embedded verbatim into the inner BHTTP request via `bhttp.serializeRequest`. A malformed value (`"https://host"`, leading/trailing whitespace, embedded path) silently produces a malformed inner BHTTP request, and the consumer has no signal until the gateway returns a parse error.

A non-custodial crypto wallet hitting either of these failure modes has no runtime warning that the deployment is broken until traffic is already being captured or until the gateway returns a parse error on the inner request.

Note: the privacy-bypass concern raised by the original L-01 (silent `sendDirect()` fallback to `gatewayBaseUrl`) is **resolved by the L-01 restructure** — `sendDirect()` and `OhttpGatewayConfig` no longer exist. A consumer wanting plaintext HTTP in the new architecture just uses a raw `http.Client` and skips constructing `OhttpHttpClient`. There is no longer a single API surface whose name suggests "OHTTP" but silently sends plaintext. The remaining work is purely input validation on the two new construction surfaces.

## Technical Details

### Part A — HTTPS scheme enforcement (`HttpClientTransport`)

**1. Reject non-HTTPS schemes at construction time** (`lib/src/adapters/http/http_transport.dart`)

`HttpClientTransport({required http.Client client, required Uri keysUrl, required Uri gatewayUrl})` is the single place gateway URLs enter the package. Validate both URIs at construction time:

- Reject any `keysUrl.scheme != 'https'`.
- Reject any `gatewayUrl.scheme != 'https'`.
- Throw a library-owned exception (`OhttpConfigException`, or — if L-03 has already landed its hierarchy — an appropriate subclass of `OhttpException`; coordinate naming with L-03).

Document the rationale in the constructor doc comment with a reference to RFC 9458 §1.

**2. Optional test escape hatch**

If a tests-only escape hatch is needed (e.g., `MockClient` test fixtures using `http://localhost`), gate it behind an explicit named flag (e.g., `allowInsecureScheme: false` default). Production code paths must not be able to accidentally pass through it. Document the flag's intended scope (testing only) in the parameter doc comment.

### Part B — Authority shape validation (`OhttpRequestData`)

**3. Validate `authority` at construction time** (`lib/src/ohttp_session.dart`)

`OhttpRequestData.authority` is the RFC 3986 `authority` production: `host` or `host:port`. Validate in the constructor:

- Reject empty values.
- Reject values containing a scheme prefix (`http://`, `https://`, anything matching `^[a-zA-Z][a-zA-Z0-9+.-]*://`).
- Reject values containing whitespace.
- Reject values containing path / query / fragment characters (`/`, `?`, `#`).
- Throw a library-owned exception (`OhttpConfigException` or equivalent — same hierarchy as Part A).

Document the expected shape in the constructor doc comment with a reference to RFC 3986 §3.2.

**4. Extraction-side validation in `OhttpHttpClient`** (`lib/src/adapters/http/ohttp_http_client.dart`)

`OhttpHttpClient.send` already extracts `authority` from `request.url.host` (plus port logic). Since the extraction goes through `OhttpRequestData(...)`, Part B's constructor validation covers this path automatically — no separate validation step in the adapter. Verify by inspection that the adapter does not bypass the constructor (i.e., it does not assign to a private field directly).

### Part C — Tests

**5. Scheme validation tests** (`test/adapters/http_adapter_test.dart`)

Add cases:
- Valid `https://gateway.example.com/keys` and `https://gateway.example.com/gw` — succeeds.
- `http://gateway.example.com/keys` for `keysUrl` — rejected with the typed exception.
- `http://gateway.example.com/gw` for `gatewayUrl` — rejected.
- `ftp://gateway.example.com/keys` — rejected.
- A `Uri` with empty scheme — rejected.
- If the `allowInsecureScheme` escape hatch is added, one positive test with `allowInsecureScheme: true` and `http://` — succeeds; document this is a test-only flag.

**6. Authority validation tests** (`test/ohttp_session_test.dart`)

Add cases for `OhttpRequestData` construction:
- Valid `host.example.com` — succeeds.
- Valid `host.example.com:8443` — succeeds.
- Empty string — rejected.
- `https://host.example.com` (scheme-prefixed) — rejected.
- `host.example.com /path` (whitespace) — rejected.
- `host.example.com/path` — rejected.
- `host.example.com?q=1` — rejected.
- `host.example.com#frag` — rejected.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | https://www.notion.so/adguard/OHTTP-Dart-360aa56b773080619205ff76a2a360f9 |

## Acceptance Criteria

**Scheme validation:**
- Constructing `HttpClientTransport` with a non-`https://` `keysUrl` or `gatewayUrl` throws a library-owned exception at construction time.
- Constructor doc comment cites RFC 9458 §1 for the scheme requirement.
- Any escape-hatch flag (if added) defaults to off and is documented as test-only.

**Authority validation:**
- Constructing `OhttpRequestData` with empty / scheme-prefixed / whitespace-containing / path-containing / query-containing / fragment-containing `authority` throws a library-owned exception at construction time.
- Constructor doc comment cites RFC 3986 §3.2 for the authority shape.
- `OhttpHttpClient.send` goes through `OhttpRequestData(...)` (does not bypass the constructor) so the same validation applies to authorities extracted from `request.url`.

**Tests:**
- Scheme tests cover: valid `https://`, rejected `http://`, rejected `ftp://`, rejected empty-scheme URI.
- Authority tests cover: valid `host`, valid `host:port`, rejected empty, rejected scheme-prefixed, rejected whitespace, rejected path, rejected query, rejected fragment.

## Additional

- Tests: unit tests on `HttpClientTransport` and `OhttpRequestData` constructors covering each accepted/rejected input class. No example or README changes — the L-01 restructure already rewrote the example to demonstrate only the new (no-bypass) integration paths.
- Depends on L-01 (the `HttpClientTransport` and `OhttpRequestData` types do not exist before the restructure lands).
- Coordinate exception naming with L-03: if L-03 lands first, use its `OhttpException` hierarchy directly. If this task lands first, introduce a placeholder library-owned exception that L-03 then folds into the hierarchy.
- Source: the privacy-bypass half of the original L-01 (M-01 in `../merged-tasks/`) is **dropped** — the restructure removes `sendDirect()` and `OhttpGatewayConfig.directBaseUrl`. The input-validation half (M-02) survives, retargeted as described above.
