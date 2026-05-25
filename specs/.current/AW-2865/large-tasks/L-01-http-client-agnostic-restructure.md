# HTTP-client-agnostic package restructure (core + opt-in `http` adapter)

**Estimate:** 5d
**Priority:** P1: Critical
**Component:** App
**Severity:** BLOCKER

## Task Description

`ohttp_dart` today ships a single high-level `OhttpClient` that owns its own `package:http` plumbing and returns a custom `OhttpResponse`/`OhttpHeader` shape. The wallet's `ohttp_integration` branch needed neither of those: it bypassed `OhttpClient` entirely and re-orchestrated the primitives (`serializeRequest` → `ohttpEncapsulate` → POST → `ohttpDecapsulate` → `parseResponse`) inside a hand-rolled `http.BaseClient` subclass, plus a hand-rolled KeyConfig cache (~155 lines of glue). The reason: `OhttpClient`'s response shape does not fit `http.BaseClient.send`'s `http.StreamedResponse` contract, and it owns its gateway HTTP plumbing rather than letting the consumer plug in theirs.

Concretely, today's friction:

1. `OhttpClient` is effectively dead weight for any real integration — every consumer bypasses it.
2. `ohttp_dart` hard-depends on `package:http` even for consumers who will use Dio or another client.
3. The package ships no KeyConfig cache, so every consumer reinvents one (and the wallet's hand-rolled cache has a single-flight race past TTL expiry).
4. The custom `OhttpResponse` / `OhttpHeader` types add a translation layer at every consumer boundary.

This task restructures the package into a thin core with zero HTTP-client dependencies plus an opt-in `package:http` adapter, so a wallet (or any other consumer) integrates via ~6 lines of constructor wiring instead of ~155 lines of bespoke wrapper code, and so a future Dio adapter can be added without touching the core.

It is a BLOCKER for every other follow-up: the file layout, public API, and exception/cache/observability surfaces that L-02…L-07 modify only exist after this task lands.

## Technical Details

### New architecture (two libraries in one package)

```
package:ohttp_dart/ohttp_dart.dart   ← core (no HTTP-client dep)
package:ohttp_dart/http.dart         ← opt-in package:http adapter
```

A future Dio adapter would live in `package:ohttp_dart/dio.dart` against the same core, with no changes to core types.

### New file layout

```
lib/
├── ohttp_dart.dart          (core library entry point)
├── http.dart                (http-adapter library entry point)
└── src/
    ├── bhttp.dart                  (unchanged — RFC 9292)
    ├── hpke.dart                   (unchanged — RFC 9180 sender)
    ├── ohttp.dart                  (unchanged — RFC 9458 encap/decap)
    ├── transport.dart              (new)
    ├── key_config_cache.dart       (new)
    ├── ohttp_session.dart          (new)
    └── adapters/
        └── http/
            ├── http_transport.dart      (new)
            └── ohttp_http_client.dart   (new)
```

**Deleted:** `lib/src/ohttp_client.dart` outright. The types it defined (`OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig`) are removed without a deprecation cycle — they have no external consumers and are superseded by the new surface below. No backward-compatibility shims.

### New public API surface

**Core (`package:ohttp_dart/ohttp_dart.dart`):**

- `abstract interface class OhttpTransport` — bytes-in / bytes-out seam between OHTTP orchestration and any HTTP client. Two methods: `Future<Uint8List> fetchKeyConfig()` and `Future<Uint8List> postToGateway(Uint8List body)`. Implementations MUST throw `OhttpGatewayException` on non-2xx so the session can invalidate the cache.
- `class OhttpGatewayException implements Exception` — gateway returned a non-2xx response. Carries `statusCode` and a message. Lives in core (not the adapter) because the invalidation policy that reacts to it lives in `OhttpSession`.
- `class KeyConfigCache` — TTL cache of the gateway's parsed `OhttpKeyConfig` with single-flight de-duplication of concurrent fetches and manual `invalidate()`. Injectable clock for deterministic TTL tests. Default TTL: 1 hour.
- `class OhttpSession` — orchestrator that replaces today's `OhttpClient`. Owns `(transport, cache)`. Per call: cache.get() → BHTTP-serialize the inner request → OHTTP-encapsulate → transport.postToGateway → OHTTP-decapsulate → BHTTP-parse. On `OhttpGatewayException` from the transport, invalidates the cache before rethrowing. Convenience `OhttpSession.withTransport(...)` builds the cache for you.
- `class OhttpRequestData` — HTTP-client-neutral request shape (`method`, `scheme`, `authority`, `path`, `headers`, `body`). `authority` is the **inner target host** the gateway forwards to — NOT the gateway itself; the gateway URL is the transport's concern.
- `class OhttpResponseData` — HTTP-client-neutral response shape. Headers as `List<(String, String)>` to preserve order and duplicates (e.g. `Set-Cookie`); collapse to a `Map` happens only at adapters that force it.
- The existing primitives (`bhttp`, `hpke`, `ohttp` modules) re-exported as today.

**`http` adapter (`package:ohttp_dart/http.dart`):**

- `class HttpClientTransport implements OhttpTransport` — wraps an `http.Client` plus `Uri keysUrl` and `Uri gatewayUrl`. Sets `Content-Type: message/ohttp-req` on the POST. Owns no client lifecycle.
- `class OhttpHttpClient extends http.BaseClient` — drop-in `http.Client` replacement that routes through an `OhttpSession`. Extracts method/scheme/authority/path/body from `request.url` and injects the `host` header (which `dart:io` would normally add at the transport layer, but the inner BHTTP request never reaches that layer because it's encrypted). Optional `closeWith` parameter controls whether `close()` propagates to a raw client (off by default — assume DI owns the raw client).

### Removed API (no shims)

- `lib/src/ohttp_client.dart` (the entire file).
- `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig`.
- `OhttpClient.sendDirect()` and its `effectiveDirectBaseUrl` getter — the bypass concern that L-01 (old) raised is moot in the new architecture: a consumer wanting plaintext HTTP just uses a raw `http.Client` and skips constructing `OhttpHttpClient`. The library no longer exposes a method whose name suggests "use OHTTP" but silently sends plaintext.
- `onLog` string callback from `OhttpClient` — replaced by the typed observer surface introduced in L-05.

### Pubspec

- `cryptography ^2.9.0` — unchanged.
- `http ^1.6.0` — **stays in `dependencies`**. The core library does not `import 'package:http/...'` (verified by the layout above); only `lib/http.dart` and files under `lib/src/adapters/http/` do. Consumers who only want the core pay one unused transitive dep. Splitting into two packages or moving `http` to an opt-in workflow (`dependency_overrides`) is deferred — pragmatically the cost is small.
- No new dependencies.

### Wallet migration (illustrative — not part of this package's work)

The wallet's `ohttp_integration` branch replaces ~155 lines under `lib/data/api/ohttp/` with constructor wiring in its DI module:

```dart
final raw = http.Client();
final transport = HttpClientTransport(
  client: raw,
  keysUrl: Uri.parse(keysUrl),
  gatewayUrl: Uri.parse(gatewayUrl),
);
final session = OhttpSession.withTransport(transport: transport);
final ohttpClient = OhttpHttpClient(session: session, closeWith: raw);
```

Downstream wallet code keeps using `http.Client` interfaces unchanged.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

**Library structure:**

- Two library entry points exist: `lib/ohttp_dart.dart` (core) and `lib/http.dart` (`http` adapter).
- No file under `lib/src/` outside `lib/src/adapters/http/` imports `package:http`. Verified by inspection of `import` lines after the restructure.
- `lib/src/ohttp_client.dart` no longer exists; `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig` are not exported from any library.

**Public API:**

- `OhttpTransport`, `OhttpGatewayException`, `KeyConfigCache`, `OhttpSession`, `OhttpRequestData`, `OhttpResponseData` are exported from `lib/ohttp_dart.dart`.
- `HttpClientTransport` and `OhttpHttpClient` are exported from `lib/http.dart`.
- `OhttpSession.withTransport(...)` convenience constructor builds a default `KeyConfigCache` automatically.
- `KeyConfigCache.get()` shares a single in-flight fetch across concurrent stale calls (single-flight de-duplication).
- `KeyConfigCache.invalidate()` forces a re-fetch on the next `get()`.
- `OhttpSession.send` invalidates the cache when the transport throws `OhttpGatewayException` and **does not** invalidate on any other exception class (network errors, decap failures, etc.).

**Behaviour preserved:**

- The full RFC 9180 / 9292 / 9458 round trip continues to pass the existing `test/hpke_test.dart`, `test/bhttp_test.dart`, and `test/ohttp_test.dart` test files without modification (those test files are not coupled to the public client surface).
- `OhttpHttpClient` honours `host` header injection: it materializes the header from `request.url` if the caller didn't already supply one, and preserves a caller-supplied `host` header verbatim. Default ports (80/443) are excluded from the constructed `authority`.

**Example + docs:**

- `example/ohttp_dart_example.dart` demonstrates two paths in this order: (1) the `OhttpHttpClient` quick-start (what most consumers will copy), (2) the lower-level `OhttpSession.send` path (what a Dio/custom-client adapter would call into).
- `README.md` "Project Structure" and "Usage" sections reflect the new layout and the `OhttpHttpClient` quick-start.
- `CLAUDE.md` "Layering" section enumerates the new files (`transport.dart`, `key_config_cache.dart`, `ohttp_session.dart`, the two adapter files) and notes that core has no HTTP-client dependency.
- `CLAUDE.md` "Cross-cutting gotchas" gains a bullet stating that the `host` header injection lives in the `http` adapter (not core) because the inner BHTTP request is encrypted and never reaches `dart:io`'s transport layer, so other adapters (Dio, custom) must do the same.

**Tests:**

- New `test/key_config_cache_test.dart` covers: cold get, hot get within TTL, TTL expiry with injected clock, `invalidate()` forces re-fetch, concurrent stale gets share one fetch, fetch error propagates without poisoning the cache.
- New `test/ohttp_session_test.dart` verifies orchestration (cache reuse, cache-invalidation-on-gateway-error, cache-preservation-on-non-gateway-error, the bytes that get handed to the transport). Note: full crypto round-trip via `OhttpSession.send` is not testable in this package because no HPKE receiver implementation exists here — `OhttpSession` tests verify orchestration; the crypto round trip in isolation is already covered by `test/ohttp_test.dart`.
- New `test/adapters/http_adapter_test.dart` covers `HttpClientTransport` (GET keysUrl, POST gateway with `message/ohttp-req` content type, `OhttpGatewayException` on non-200) and `OhttpHttpClient` (URL → request-data extraction, host-header policy, default-port handling, `closeWith` behaviour). Uses `MockClient` from `package:http/testing.dart`.
- Full suite (`dart test`) is green. `dart analyze` reports `No issues found!`. Formatter (`dart format --line-length=120 --set-exit-if-changed`) reports no changes.

## Additional

- Implementation guidance: a detailed step-by-step plan exists at `docs/superpowers/plans/2026-05-25-http-client-agnostic-integration.md` (TDD-ordered, 8 tasks ending with example + docs). The design spec is at `docs/superpowers/specs/2026-05-25-http-client-agnostic-integration-design.md`. Both should be consulted by the implementer. Notion will additionally carry the granular sub-task tracking; the link will be attached to the Jira ticket created from this large task.
- This task **must land first**. Every other follow-up (L-02..L-07) targets file paths, type names, and concerns that only exist after the restructure:
  - L-02 input validation lands HTTPS-scheme checks on `HttpClientTransport` and authority validation on `OhttpRequestData`.
  - L-03 exception hierarchy brings `OhttpGatewayException` (introduced here) under the new `OhttpException` base and adds `OhttpHttpException`, `OhttpTimeoutException`, `OhttpDecryptionException`, `OhttpParseException`, `OhttpKeyConfigException`, `OhttpUnsupportedSuiteException`.
  - L-04 hardening adds timeouts to `HttpClientTransport`, OHTTP-layer response size cap to `OhttpSession.send`, BHTTP parser hardening to `bhttp.dart`. The KeyConfig cache and URL normalization concerns from the old L-03 are satisfied by **this** task and are no longer follow-ups.
  - L-05 zeroization in `hpke.dart`/`ohttp.dart` is independent; observability attaches to `OhttpSession`.
  - L-06 negative-path tests retarget `OhttpClient` test references to `OhttpSession` / `HttpClientTransport`.
  - L-07 integration test exercises `OhttpHttpClient` via `MockClient` end-to-end.
- Out of scope for this task: implementing a Dio adapter (deferred — the design supports it without core changes), persisting cached KeyConfigs across app restarts, stale-while-revalidate or background refresh policies, honouring `not_before` / `not_after` metadata (RFC 9458 KeyConfig does not carry these), streaming response bodies, supporting cipher suites other than the current fixed `(0x0020, 0x0001, 0x0001)` triple (multi-suite **parsing** is addressed by L-03; supporting additional suites end-to-end is a separate effort).
- Source: design spec + implementation plan at `docs/superpowers/{specs,plans}/2026-05-25-http-client-agnostic-integration*.md`.
