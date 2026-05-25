# HTTP-Client-Agnostic Integration — Design

**Status:** Approved (brainstorming)
**Date:** 2026-05-25
**Topic:** Restructure `ohttp_dart` so it integrates trivially into projects using any HTTP client (initially `package:http`, with a clear path for `dio` and others).

## Problem

The wallet's branch `ohttp_integration` adds two files to integrate `ohttp_dart` with its `package:http` stack:

- `lib/data/api/ohttp/ohttp_http_client.dart` (~100 LOC, `http.BaseClient` subclass)
- `lib/data/api/ohttp/ohttp_key_config_cache.dart` (~55 LOC, TTL cache)

That wrapper **does not use** the package's high-level `OhttpClient` at all. It bypasses it and re-orchestrates the primitives directly (`serializeRequest` → `ohttpEncapsulate` → POST → `ohttpDecapsulate` → `parseResponse`). The reason: `OhttpClient` returns a custom `OhttpResponse` type that does not fit `http.BaseClient.send`'s `http.StreamedResponse` contract, and it owns its own gateway HTTP plumbing rather than letting the consumer plug in theirs.

Concretely, today's friction:

1. `OhttpClient` is effectively dead weight for any real integration — every consumer bypasses it.
2. `ohttp_dart` hard-depends on `package:http`, even for consumers who will use Dio or another client.
3. The package ships no KeyConfig cache, so every consumer reinvents one (and the wallet's has a single-flight race past TTL expiry).
4. The custom `OhttpResponse`/`OhttpHeader` types add a translation layer at every consumer boundary.
5. Of the wallet's ~100-line wrapper, only ~15 lines are OHTTP-specific work — the other ~85 are pure adapter glue that should live in the package.

## Goals

- The wallet's two integration files collapse to ~6 lines of constructor wiring.
- A Dio adapter can be added later as a separate sub-library without touching the core.
- Core `ohttp_dart` has zero HTTP-client dependencies.
- KeyConfig caching (TTL + manual invalidation + concurrent de-dup) is provided by the package.

## Non-goals

- Shipping a Dio adapter in this round (design supports it; implementation deferred).
- KeyConfig persistence across app restarts.
- Stale-while-revalidate or background-refresh policies.
- Honoring `not_before`/`not_after` metadata on KeyConfig (RFC 9458 KeyConfig does not carry these).
- Streaming response bodies (OHTTP responses are fully-formed encrypted bodies; not chunked).
- Supporting cipher suites other than `(0x0020, 0x0001, 0x0001)`. Hard-coded constraint of the current implementation.
- Maintaining backward compatibility with the existing `OhttpClient`/`OhttpResponse`/`OhttpHeader`/`OhttpGatewayConfig` types. They have no external users and are removed outright.

## Approach

Restructure into two layers within one package:

- **Core library** — `package:ohttp_dart/ohttp_dart.dart`, zero HTTP-client deps. Exposes the existing primitives plus a small `OhttpTransport` interface (bytes-in/bytes-out), a `KeyConfigCache`, and an `OhttpSession` orchestration class.
- **`http` adapter sub-library** — `package:ohttp_dart/http.dart`, the only place that imports `package:http`. Exposes `HttpClientTransport` (implements `OhttpTransport`) and `OhttpHttpClient extends http.BaseClient` (drop-in replacement for `http.Client`).

A future Dio adapter would live in `package:ohttp_dart/dio.dart` with the same structure (`DioTransport implements OhttpTransport` + `OhttpDioInterceptor`) and require no changes to the core.

## Package layout

```
lib/
├── ohttp_dart.dart                  (core library entry point)
├── http.dart                        (http-adapter library entry point)
└── src/
    ├── bhttp.dart                   (unchanged — RFC 9292 primitives)
    ├── hpke.dart                    (unchanged — RFC 9180 sender)
    ├── ohttp.dart                   (unchanged — RFC 9458 encap/decap)
    ├── transport.dart               (new — OhttpTransport + OhttpGatewayException)
    ├── key_config_cache.dart        (new — KeyConfigCache)
    ├── ohttp_session.dart           (new — OhttpSession + OhttpRequestData + OhttpResponseData)
    └── adapters/
        └── http/
            ├── http_transport.dart       (new — HttpClientTransport)
            └── ohttp_http_client.dart    (new — OhttpHttpClient)
```

**Removed:**
- `lib/src/ohttp_client.dart` — deleted. Types `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig` are removed outright.
- `lib/ohttp_dart.dart` — drops the `ohttp_client.dart` export.

**`pubspec.yaml`:**
- `cryptography: ^2.9.0` — unchanged.
- `http: ^1.6.0` — stays in `dependencies`. The core library does not import it (verified by the package layout above); only `lib/http.dart` and the files under `lib/src/adapters/http/` do. Consumers who only want the core pay one unused transitive dep. Moving `http` to an opt-in workflow (e.g. via `dependency_overrides` or splitting into two packages) is deferred — pragmatically the cost is small.

## Core types

### `OhttpTransport` (`lib/src/transport.dart`)

The seam between OHTTP orchestration and whatever HTTP client the consumer uses. Two methods: fetch raw KeyConfig bytes, POST encapsulated bytes and return raw response bytes.

```dart
abstract interface class OhttpTransport {
  /// Fetch the raw KeyConfig bytes from the gateway's keys URL.
  Future<Uint8List> fetchKeyConfig();

  /// POST encapsulated bytes to the gateway and return its raw response body.
  ///
  /// Implementations MUST set the `Content-Type: message/ohttp-req` header
  /// and MUST throw [OhttpGatewayException] on non-200 responses so the
  /// session can invalidate its KeyConfig cache.
  Future<Uint8List> postToGateway(Uint8List body);
}

class OhttpGatewayException implements Exception {
  final String message;
  final int statusCode;
  const OhttpGatewayException(this.message, {required this.statusCode});

  @override
  String toString() => 'OhttpGatewayException: $message';
}
```

`OhttpGatewayException` lives in core (not in any adapter) because the cache-invalidation policy that responds to it lives in `OhttpSession`.

### `OhttpRequestData` / `OhttpResponseData` (`lib/src/ohttp_session.dart`)

The HTTP-client-neutral request/response shapes consumed and produced by `OhttpSession.send`.

```dart
class OhttpRequestData {
  final String method;
  final String scheme;          // 'https' typically
  final String authority;       // target host[:port] — embedded in BHTTP,
                                // NOT used to address the gateway
  final String path;            // path + query
  final Map<String, String> headers;
  final Uint8List body;

  const OhttpRequestData({
    required this.method,
    required this.scheme,
    required this.authority,
    required this.path,
    this.headers = const {},
    required this.body,
  });
}

class OhttpResponseData {
  final int statusCode;
  final List<(String, String)> headers;   // order-preserving, allows duplicates
  final Uint8List body;

  const OhttpResponseData({
    required this.statusCode,
    required this.headers,
    required this.body,
  });
}
```

**Header representation:** core uses `List<(String, String)>` for responses. BHTTP/HTTP allow duplicate header names and the order can matter (e.g. `Set-Cookie`). The lossy collapse into a `Map<String, String>` happens only at adapter boundaries that force it (e.g. `http.StreamedResponse.headers`); consumers who need lossless access can use `OhttpSession.send` directly.

## `KeyConfigCache` (`lib/src/key_config_cache.dart`)

```dart
class KeyConfigCache {
  static const _defaultTtl = Duration(hours: 1);

  final OhttpTransport _transport;
  final Duration _ttl;
  final DateTime Function() _now;

  OhttpKeyConfig? _cached;
  DateTime? _fetchedAt;
  Future<OhttpKeyConfig>? _inFlight;

  KeyConfigCache({
    required OhttpTransport transport,
    Duration ttl = _defaultTtl,
    DateTime Function()? now,
  }) : _transport = transport,
       _ttl = ttl,
       _now = now ?? DateTime.now;

  Future<OhttpKeyConfig> get() {
    final cached = _cached;
    final at = _fetchedAt;
    if (cached != null && at != null && _now().difference(at) < _ttl) {
      return Future.value(cached);
    }
    return _inFlight ??= _fetch().whenComplete(() => _inFlight = null);
  }

  Future<OhttpKeyConfig> _fetch() async {
    final bytes = await _transport.fetchKeyConfig();
    final config = OhttpKeyConfig.parse(bytes);
    _cached = config;
    _fetchedAt = _now();
    return config;
  }

  void invalidate() {
    _cached = null;
    _fetchedAt = null;
  }
}
```

Notes:

- **Single-flight de-dup** (`_inFlight`) — concurrent stale `get()` calls share one in-flight fetch. Fixes a latent race in the wallet's hand-rolled cache.
- **Injectable clock** (`now`) — enables deterministic TTL tests without `Future.delayed`.
- **No persistence, no `not_before`/`not_after`, no SWR.** Deferred. TTL is purely a client policy; RFC 9458 KeyConfig carries no freshness metadata.

## `OhttpSession` (`lib/src/ohttp_session.dart`)

The orchestrator. Replaces today's `OhttpClient`. Owns the `(transport, cache)` pair; the target authority is per-request, supplied in `OhttpRequestData`.

```dart
class OhttpSession {
  final OhttpTransport _transport;
  final KeyConfigCache _cache;

  OhttpSession({
    required OhttpTransport transport,
    required KeyConfigCache cache,
  }) : _transport = transport,
       _cache = cache;

  /// Convenience constructor: builds a [KeyConfigCache] for you.
  factory OhttpSession.withTransport({
    required OhttpTransport transport,
    Duration keyConfigTtl = const Duration(hours: 1),
  }) {
    return OhttpSession(
      transport: transport,
      cache: KeyConfigCache(transport: transport, ttl: keyConfigTtl),
    );
  }

  Future<OhttpResponseData> send(OhttpRequestData req) async {
    final config = await _cache.get();

    final binaryRequest = bhttp.serializeRequest(
      method: req.method,
      scheme: req.scheme,
      authority: req.authority,
      path: req.path,
      headers: req.headers,
      body: req.body,
    );

    final encResult = await ohttpEncapsulate(config, binaryRequest);

    final Uint8List gatewayResponseBytes;
    try {
      gatewayResponseBytes = await _transport.postToGateway(encResult.encRequest);
    } on OhttpGatewayException {
      _cache.invalidate();    // gateway error => assume keys rotated, force re-fetch
      rethrow;
    }

    final binaryResponse = await ohttpDecapsulate(
      encResult.enc,
      encResult.exportedSecret,
      gatewayResponseBytes,
    );

    final parsed = bhttp.parseResponse(binaryResponse);
    return OhttpResponseData(
      statusCode: parsed.statusCode,
      headers: parsed.headers,    // already List<(String, String)> in bhttp.dart
      body: parsed.body,
    );
  }
}
```

Design notes:

- **No gateway URL or target authority on `OhttpSession`.** Gateway URLs belong to the `Transport` (the thing that addresses the gateway). Target authority is per-request so one session can encapsulate requests to different inner targets — same flexibility the wallet's `http.BaseClient` wrapper already provides implicitly via `request.url.host`.
- **Cache invalidation policy is centralized here.** Only `OhttpGatewayException` triggers invalidation; network errors and decap failures do not (they do not indicate stale keys).
- **No `onLog` callback.** Consumers wrap `session.send()` or instrument their own `OhttpTransport` if they want per-step logging.

## `http` adapter sub-library

### `HttpClientTransport` (`lib/src/adapters/http/http_transport.dart`)

```dart
class HttpClientTransport implements OhttpTransport {
  final http.Client _client;
  final Uri _keysUrl;
  final Uri _gatewayUrl;

  HttpClientTransport({
    required http.Client client,
    required Uri keysUrl,
    required Uri gatewayUrl,
  }) : _client = client,
       _keysUrl = keysUrl,
       _gatewayUrl = gatewayUrl;

  @override
  Future<Uint8List> fetchKeyConfig() async {
    final res = await _client.get(_keysUrl);
    if (res.statusCode != 200) {
      throw OhttpGatewayException(
        'Failed to fetch KeyConfig from $_keysUrl',
        statusCode: res.statusCode,
      );
    }
    return Uint8List.fromList(res.bodyBytes);
  }

  @override
  Future<Uint8List> postToGateway(Uint8List body) async {
    final res = await _client.post(
      _gatewayUrl,
      headers: const {'Content-Type': 'message/ohttp-req'},
      body: body,
    );
    if (res.statusCode != 200) {
      throw OhttpGatewayException(
        'OHTTP gateway returned HTTP ${res.statusCode}',
        statusCode: res.statusCode,
      );
    }
    return Uint8List.fromList(res.bodyBytes);
  }
}
```

### `OhttpHttpClient` (`lib/src/adapters/http/ohttp_http_client.dart`)

```dart
class OhttpHttpClient extends http.BaseClient {
  final OhttpSession _session;
  final http.Client? _innerToClose;

  /// [closeWith] — optional raw client to close when this client is closed.
  /// Omit if the inner client is shared across DI scopes.
  OhttpHttpClient({required OhttpSession session, http.Client? closeWith})
      : _session = session,
        _innerToClose = closeWith;

  @override
  Future<http.StreamedResponse> send(http.BaseRequest request) async {
    final body = request is http.Request
        ? Uint8List.fromList(request.bodyBytes)
        : Uint8List(0);

    final url = request.url;
    final path = url.hasQuery ? '${url.path}?${url.query}' : url.path;
    final authority = (url.hasPort && url.port != 443 && url.port != 80)
        ? '${url.host}:${url.port}'
        : url.host;

    // dart:io adds `host` at transport level, but BHTTP serialization needs it
    // explicit — the inner request is encrypted and never touches dart:io's
    // transport layer.
    final headers = Map<String, String>.of(request.headers)
      ..putIfAbsent('host', () => authority);

    final res = await _session.send(OhttpRequestData(
      method: request.method,
      scheme: url.scheme,
      authority: authority,
      path: path,
      headers: headers,
      body: body,
    ));

    final responseHeaders = <String, String>{
      for (final (name, value) in res.headers) name: value,
    };

    return http.StreamedResponse(
      Stream.value(res.body),
      res.statusCode,
      headers: responseHeaders,
      request: request,
      contentLength: res.body.length,
    );
  }

  @override
  void close() => _innerToClose?.close();
}
```

Notes:

- **`closeWith` is opt-in.** The wallet's wrapper unconditionally closes its inner client, which is a footgun when the raw client is shared. Opt-in matches the typical DI pattern of owning lifetimes at the composition root.
- **Lossy `List<(String,String)> → Map<String,String>` collapse is unavoidable here** — `http.StreamedResponse.headers` only accepts a `Map`. Consumers who need lossless headers use `OhttpSession.send` directly.
- **`host` header injection** mirrors the wallet's current workaround.
- **No logging** — instrument the `OhttpTransport` or wrap `OhttpHttpClient` if logging is desired.

## Wallet migration (illustrative — not part of this package's work)

Wallet deletes both files under `lib/data/api/ohttp/` and replaces them with constructor wiring in the DI module:

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

Downstream code keeps using `http.Client` interfaces unchanged.

## Testing

Existing tests for primitives stay as-is:
- `test/bhttp_test.dart`
- `test/hpke_test.dart`
- `test/ohttp_test.dart`

To remove or migrate (verify during implementation):
- `test/ohttp_client_test.dart` — if it exists, delete; the class it covers is gone.

New tests:

- **`test/key_config_cache_test.dart`** — cold fetch populates; hot read skips fetch; TTL expiry triggers re-fetch (with injected clock); `invalidate()` triggers re-fetch; **two concurrent stale gets share one underlying fetch** (single-flight); transport errors propagate and do not poison the cache.
- **`test/ohttp_session_test.dart`** — full round trip with a fake `OhttpTransport` that returns pre-baked bytes against a known KeyConfig; `OhttpGatewayException` from the transport triggers `invalidate()`; decap failure does *not* trigger invalidation; cache `get()` is called exactly once per round trip.
- **`test/adapters/http_adapter_test.dart`** — uses a fake `OhttpSession` (no real crypto): `send` correctly extracts method/scheme/authority/path/headers/body from an `http.Request`; response headers pass through; `host` header is injected if missing and preserved if present; non-default ports appear in `authority`; query strings preserved; `close()` only closes the inner client when `closeWith` is supplied.

The fake `OhttpTransport` is what makes `OhttpSession` testable without crypto vectors — the payoff of the abstraction.

## Example rewrite

`example/ohttp_dart_example.dart` shows both paths, in this order:

1. **`http` adapter quick-start** (most readers will copy this).
2. **Low-level `OhttpSession.send`** (HTTP-client-neutral, useful for Dio/other consumers and as the documentation of where the Transport seam is).

## Documentation updates

- `CLAUDE.md` — "Layering" section rewrites to reflect the new structure: `bhttp`/`hpke`/`ohttp` primitives → `transport`/`key_config_cache` → `ohttp_session` → optional `http` adapter. Existing "Cross-cutting gotchas" entries remain accurate; add a fourth: *the `host` header injection happens in the `http` adapter (not core) because the core cannot assume what the consumer's HTTP client added at transport level.*
- `README.md` — if it features `OhttpClient`, rewrite to lead with the `OhttpHttpClient` quick-start. Briefly mention the Transport abstraction for advanced/non-`http` users.

## Open questions

None. All decisions taken during brainstorming. The Dio adapter is a future contribution and intentionally out of scope.
