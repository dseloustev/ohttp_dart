# HTTP-Client-Agnostic Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure `ohttp_dart` so the core has zero HTTP-client dependencies and an optional `package:http` adapter sub-library lets the wallet integrate with ~6 lines of constructor wiring instead of ~155 lines of bespoke wrapper code.

**Architecture:** Two libraries in one package. Core (`package:ohttp_dart/ohttp_dart.dart`) exposes the crypto primitives plus a `OhttpTransport` bytes-in/bytes-out interface, a `KeyConfigCache`, and an `OhttpSession` orchestrator. Optional `package:ohttp_dart/http.dart` adds `HttpClientTransport` and a drop-in `OhttpHttpClient extends http.BaseClient`. Future Dio support is a new `package:ohttp_dart/dio.dart` library against the same core.

**Tech Stack:** Dart `^3.11.1`. Existing deps: `cryptography ^2.9.0`, `http ^1.6.0`, `lints ^3.0.0`, `test ^1.25.6`. No new dependencies.

**Reference spec:** `docs/superpowers/specs/2026-05-25-http-client-agnostic-integration-design.md`.

**Testing note:** This package implements only the HPKE *Sender* side (not the Receiver). That means no test inside this package can produce real OHTTP-encapsulated gateway response bytes — generating them would require implementing the responder. Session tests therefore verify **orchestration** (cache hits/misses, error policies, the bytes that get handed to the transport) rather than a full crypto round trip. The crypto round trip in isolation is already covered by `test/ohttp_test.dart`.

---

## File map

**Create:**
- `lib/src/transport.dart` — `OhttpTransport` interface + `OhttpGatewayException`.
- `lib/src/key_config_cache.dart` — `KeyConfigCache`.
- `lib/src/ohttp_session.dart` — `OhttpRequestData`, `OhttpResponseData`, `OhttpSession`.
- `lib/src/adapters/http/http_transport.dart` — `HttpClientTransport`.
- `lib/src/adapters/http/ohttp_http_client.dart` — `OhttpHttpClient extends http.BaseClient`.
- `lib/http.dart` — `http`-adapter library entry point.
- `test/key_config_cache_test.dart`
- `test/ohttp_session_test.dart`
- `test/adapters/http_adapter_test.dart`

**Modify:**
- `lib/ohttp_dart.dart` — drop `ohttp_client.dart` export, add new exports.
- `example/ohttp_dart_example.dart` — rewrite to use new APIs.
- `README.md` — update Project Structure + Usage sections.
- `CLAUDE.md` — update Layering section, add fourth Cross-cutting gotcha.

**Delete:**
- `lib/src/ohttp_client.dart` — `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig` removed outright.

**Unchanged:**
- `lib/src/bhttp.dart`, `lib/src/hpke.dart`, `lib/src/ohttp.dart`.
- `test/bhttp_test.dart`, `test/hpke_test.dart`, `test/ohttp_test.dart`.
- `pubspec.yaml` (`http ^1.6.0` stays in `dependencies`; only the adapter library imports it).

---

### Task 1: OhttpTransport interface + OhttpGatewayException

**Files:**
- Create: `lib/src/transport.dart`

**What:** Define the bytes-in/bytes-out seam between OHTTP orchestration and whatever HTTP client the consumer uses, plus the exception that signals "gateway error, invalidate the KeyConfig cache."

No test in this task — the file contains only an abstract interface and an exception class. Both are covered indirectly by Task 2 (`KeyConfigCache` uses `OhttpTransport`) and Task 3 (`OhttpSession` catches `OhttpGatewayException`).

- [ ] **Step 1: Create the file**

Write `lib/src/transport.dart`:

```dart
import 'dart:typed_data';

/// Bytes-in/bytes-out seam between OHTTP orchestration and any HTTP client.
///
/// Implementations are responsible for the two raw HTTP calls OHTTP needs:
/// fetching the gateway's KeyConfig and POSTing encapsulated bytes to it.
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

/// Thrown by an [OhttpTransport] when the gateway returns a non-2xx response.
///
/// Lives in core (not in any adapter) because the cache-invalidation policy
/// that responds to it lives in `OhttpSession`.
class OhttpGatewayException implements Exception {
  final String message;
  final int statusCode;

  const OhttpGatewayException(this.message, {required this.statusCode});

  @override
  String toString() => 'OhttpGatewayException: $message';
}
```

- [ ] **Step 2: Verify the file compiles**

Run: `dart analyze lib/src/transport.dart`
Expected: `No issues found!`

- [ ] **Step 3: Commit**

```bash
git add lib/src/transport.dart
git commit -m "feat: Add OhttpTransport interface and OhttpGatewayException

The transport interface is the seam where the core meets the consumer's
HTTP client of choice. OhttpGatewayException signals gateway errors that
should invalidate cached KeyConfigs."
```

---

### Task 2: KeyConfigCache

**Files:**
- Create: `lib/src/key_config_cache.dart`
- Test: `test/key_config_cache_test.dart`

**What:** TTL-based cache with manual invalidation and single-flight de-dup. The single-flight piece fixes a latent race in the wallet's hand-rolled cache where two concurrent stale `get()` calls both issue a fetch.

`OhttpKeyConfig.parse` is already exported from `lib/src/ohttp.dart` (and from `lib/ohttp_dart.dart`).

- [ ] **Step 1: Write the failing tests**

Write `test/key_config_cache_test.dart`:

```dart
import 'dart:async';
import 'dart:typed_data';

import 'package:ohttp_dart/ohttp_dart.dart';
import 'package:test/test.dart';

/// Builds a valid 41-byte KeyConfig with the given key id.
Uint8List _buildKeyConfig(int keyId) {
  final buf = BytesBuilder();
  buf.addByte(keyId);
  buf.add([0x00, 0x20]); // kem_id = X25519
  buf.add(List.filled(32, 0xAB)); // public_key
  buf.add([0x00, 0x04]); // sym_len = 4
  buf.add([0x00, 0x01]); // kdf_id = HKDF-SHA256
  buf.add([0x00, 0x01]); // aead_id = AES-128-GCM
  return Uint8List.fromList(buf.toBytes());
}

class _FakeTransport implements OhttpTransport {
  int fetchCalls = 0;
  int keyId = 0x01;
  Completer<void>? gate; // when non-null, fetch awaits it before returning
  Object? throwOnFetch;

  @override
  Future<Uint8List> fetchKeyConfig() async {
    fetchCalls++;
    if (gate != null) {
      await gate!.future;
    }
    if (throwOnFetch != null) {
      throw throwOnFetch!;
    }
    return _buildKeyConfig(keyId);
  }

  @override
  Future<Uint8List> postToGateway(Uint8List body) {
    throw UnimplementedError('not used in cache tests');
  }
}

void main() {
  group('KeyConfigCache', () {
    test('cold get fetches once and returns the parsed config', () async {
      final transport = _FakeTransport()..keyId = 0x07;
      final cache = KeyConfigCache(transport: transport);

      final config = await cache.get();

      expect(transport.fetchCalls, 1);
      expect(config.keyId, 0x07);
    });

    test('hot get within TTL does not fetch again', () async {
      final transport = _FakeTransport();
      final cache = KeyConfigCache(
        transport: transport,
        ttl: const Duration(hours: 1),
      );

      await cache.get();
      await cache.get();
      await cache.get();

      expect(transport.fetchCalls, 1);
    });

    test('get past TTL fetches again (injected clock)', () async {
      final transport = _FakeTransport();
      var now = DateTime(2026, 1, 1);
      final cache = KeyConfigCache(
        transport: transport,
        ttl: const Duration(minutes: 10),
        now: () => now,
      );

      await cache.get();
      expect(transport.fetchCalls, 1);

      now = now.add(const Duration(minutes: 9));
      await cache.get();
      expect(transport.fetchCalls, 1);

      now = now.add(const Duration(minutes: 2));
      await cache.get();
      expect(transport.fetchCalls, 2);
    });

    test('invalidate() forces re-fetch on next get', () async {
      final transport = _FakeTransport();
      final cache = KeyConfigCache(transport: transport);

      await cache.get();
      cache.invalidate();
      await cache.get();

      expect(transport.fetchCalls, 2);
    });

    test('two concurrent cold gets share one underlying fetch', () async {
      final transport = _FakeTransport()..gate = Completer<void>();
      final cache = KeyConfigCache(transport: transport);

      final f1 = cache.get();
      final f2 = cache.get();
      final f3 = cache.get();

      // All three are awaiting the same in-flight fetch.
      transport.gate!.complete();
      await Future.wait([f1, f2, f3]);

      expect(transport.fetchCalls, 1);
    });

    test('fetch error propagates and does not poison the cache', () async {
      final transport = _FakeTransport()
        ..throwOnFetch = StateError('network down');
      final cache = KeyConfigCache(transport: transport);

      await expectLater(cache.get(), throwsA(isA<StateError>()));

      transport.throwOnFetch = null;
      final config = await cache.get();
      expect(config, isNotNull);
      expect(transport.fetchCalls, 2);
    });
  });
}
```

- [ ] **Step 2: Run the tests and verify they fail**

Run: `dart test test/key_config_cache_test.dart`
Expected: All 6 tests fail with compile error — `KeyConfigCache` not defined.

- [ ] **Step 3: Implement KeyConfigCache**

Write `lib/src/key_config_cache.dart`:

```dart
import 'dart:async';

import 'ohttp.dart';
import 'transport.dart';

/// TTL-based cache of the gateway's [OhttpKeyConfig] with single-flight
/// de-duplication of concurrent fetches.
///
/// Use [invalidate] to force a re-fetch (e.g. when the gateway returns an
/// error suggesting key rotation).
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

- [ ] **Step 4: Wire the new class through the public library export so tests can see it**

Edit `lib/ohttp_dart.dart` — add the new exports next to the existing ones (do not remove `ohttp_client.dart` yet; that happens in Task 6):

```dart
/// Pure Dart OHTTP client (RFC 9458).
///
/// Provides Oblivious HTTP encapsulation/decapsulation with:
/// - HPKE Base Mode Sender (RFC 9180)
/// - Binary HTTP serialization (RFC 9292)
/// - High-level OhttpClient for gateway communication
library;

export 'src/bhttp.dart';
export 'src/hpke.dart';
export 'src/key_config_cache.dart';
export 'src/ohttp.dart';
export 'src/ohttp_client.dart';
export 'src/transport.dart';
```

- [ ] **Step 5: Run the tests and verify they pass**

Run: `dart test test/key_config_cache_test.dart`
Expected: All 6 tests PASS.

- [ ] **Step 6: Run full analyzer + format check**

Run: `dart analyze && dart format --line-length=120 --set-exit-if-changed lib/src/key_config_cache.dart lib/src/transport.dart test/key_config_cache_test.dart lib/ohttp_dart.dart`
Expected: `No issues found!` and no formatter complaints.

- [ ] **Step 7: Commit**

```bash
git add lib/src/key_config_cache.dart lib/ohttp_dart.dart test/key_config_cache_test.dart
git commit -m "feat: Add KeyConfigCache with TTL and single-flight de-dup

Concurrent stale gets now share one in-flight fetch instead of each
issuing a duplicate request. Clock is injectable for deterministic
TTL tests."
```

---

### Task 3: OhttpSession + request/response data types

**Files:**
- Create: `lib/src/ohttp_session.dart`
- Test: `test/ohttp_session_test.dart`

**What:** The orchestrator that replaces today's `OhttpClient`. Owns `(transport, cache)`; the target authority is per-request (in `OhttpRequestData`). On `OhttpGatewayException` from the transport, invalidates the cache before rethrowing.

Tests verify orchestration: cache reuse, error policies, BHTTP bytes that get handed to the transport. The full crypto round trip is not testable here (no receiver in this package); decap of malformed bytes is the closest proxy for "response handling."

- [ ] **Step 1: Write the failing tests**

Write `test/ohttp_session_test.dart`:

```dart
import 'dart:typed_data';

import 'package:ohttp_dart/ohttp_dart.dart';
import 'package:test/test.dart';

Uint8List _buildKeyConfig(int keyId) {
  final buf = BytesBuilder();
  buf.addByte(keyId);
  buf.add([0x00, 0x20]); // kem_id = X25519
  buf.add(List.filled(32, 0xAB));
  buf.add([0x00, 0x04]);
  buf.add([0x00, 0x01]); // kdf
  buf.add([0x00, 0x01]); // aead
  return Uint8List.fromList(buf.toBytes());
}

class _RecordingTransport implements OhttpTransport {
  int fetchCalls = 0;
  int postCalls = 0;
  Uint8List? lastPostedBody;
  int keyId = 0x42;

  /// What postToGateway returns when [postException] is null.
  Uint8List postResponse = Uint8List.fromList([0x00]); // intentionally malformed

  /// If set, postToGateway throws this instead of returning [postResponse].
  Object? postException;

  @override
  Future<Uint8List> fetchKeyConfig() async {
    fetchCalls++;
    return _buildKeyConfig(keyId);
  }

  @override
  Future<Uint8List> postToGateway(Uint8List body) async {
    postCalls++;
    lastPostedBody = body;
    if (postException != null) {
      throw postException!;
    }
    return postResponse;
  }
}

OhttpRequestData _req() => OhttpRequestData(
      method: 'GET',
      scheme: 'https',
      authority: 'example.com',
      path: '/hello',
      headers: const {'accept': 'application/json'},
      body: Uint8List(0),
    );

void main() {
  group('OhttpSession.send', () {
    test('fetches KeyConfig once and posts encapsulated bytes to the gateway',
        () async {
      final transport = _RecordingTransport();
      final session = OhttpSession.withTransport(transport: transport);

      // The fake transport returns malformed gateway bytes, so decap will
      // throw — we don't care for this test; we're inspecting what was sent.
      await expectLater(session.send(_req()), throwsA(isA<Object>()));

      expect(transport.fetchCalls, 1);
      expect(transport.postCalls, 1);
      // Encapsulated request starts with the KeyConfig's
      // key_id(1) || kem_id(2) || kdf_id(2) || aead_id(2) header (7 bytes),
      // followed by enc(32) and ciphertext.
      final posted = transport.lastPostedBody!;
      expect(posted[0], transport.keyId); // key_id from KeyConfig
      expect(posted[1], 0x00);
      expect(posted[2], 0x20); // kem
      expect(posted[3], 0x00);
      expect(posted[4], 0x01); // kdf
      expect(posted[5], 0x00);
      expect(posted[6], 0x01); // aead
      expect(posted.length, greaterThan(7 + 32));
    });

    test('reuses cached KeyConfig across calls', () async {
      final transport = _RecordingTransport();
      final session = OhttpSession.withTransport(transport: transport);

      await expectLater(session.send(_req()), throwsA(isA<Object>()));
      await expectLater(session.send(_req()), throwsA(isA<Object>()));

      expect(transport.fetchCalls, 1);
      expect(transport.postCalls, 2);
    });

    test('gateway exception invalidates the cache and rethrows', () async {
      final transport = _RecordingTransport()
        ..postException = const OhttpGatewayException(
          'boom',
          statusCode: 500,
        );
      final session = OhttpSession.withTransport(transport: transport);

      await expectLater(
        session.send(_req()),
        throwsA(isA<OhttpGatewayException>()),
      );

      // Next call must re-fetch the KeyConfig.
      transport.postException = null; // still returns malformed bytes
      await expectLater(session.send(_req()), throwsA(isA<Object>()));

      expect(transport.fetchCalls, 2);
    });

    test('decap failure does NOT invalidate the cache', () async {
      final transport = _RecordingTransport();
      // postResponse is malformed; decap will throw something that is NOT
      // OhttpGatewayException.
      final session = OhttpSession.withTransport(transport: transport);

      await expectLater(session.send(_req()), throwsA(isA<Object>()));
      await expectLater(session.send(_req()), throwsA(isA<Object>()));

      // Cache stays valid: only one fetch.
      expect(transport.fetchCalls, 1);
    });

    test('non-gateway transport error does NOT invalidate the cache',
        () async {
      final transport = _RecordingTransport()
        ..postException = StateError('network down');
      final session = OhttpSession.withTransport(transport: transport);

      // First call: cache populated, post fails with non-gateway error.
      await expectLater(session.send(_req()), throwsA(isA<StateError>()));

      // Switch to a different post error to prove the second call would still
      // use the cached KeyConfig (no second fetchCall).
      transport.postException = StateError('still down');
      await expectLater(session.send(_req()), throwsA(isA<StateError>()));

      expect(transport.fetchCalls, 1);
      expect(transport.postCalls, 2);
    });
  });
}
```

- [ ] **Step 2: Run the tests and verify they fail**

Run: `dart test test/ohttp_session_test.dart`
Expected: All 5 tests fail with compile error — `OhttpSession`, `OhttpRequestData` not defined.

- [ ] **Step 3: Implement OhttpSession + data types**

Write `lib/src/ohttp_session.dart`:

```dart
import 'dart:typed_data';

import 'bhttp.dart' as bhttp;
import 'key_config_cache.dart';
import 'ohttp.dart';
import 'transport.dart';

/// HTTP-client-neutral request shape consumed by [OhttpSession.send].
///
/// [authority] is the inner target host the gateway forwards to — NOT the
/// gateway itself. The gateway URL is the [OhttpTransport]'s concern.
class OhttpRequestData {
  final String method;
  final String scheme;
  final String authority;
  final String path;
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

/// HTTP-client-neutral response shape returned by [OhttpSession.send].
///
/// Headers are kept as `List<(String, String)>` because BHTTP/HTTP allow
/// duplicate header names and the order can matter (e.g. `Set-Cookie`).
/// Adapters that need a `Map` may collapse at their own boundary.
class OhttpResponseData {
  final int statusCode;
  final List<(String, String)> headers;
  final Uint8List body;

  const OhttpResponseData({
    required this.statusCode,
    required this.headers,
    required this.body,
  });
}

/// Orchestrates one round-trip request through an OHTTP gateway.
///
/// Owns a [KeyConfigCache] and an [OhttpTransport]. Per call: fetches the
/// KeyConfig (cached), serializes the inner request to BHTTP, encapsulates
/// via OHTTP, hands the bytes to the transport, decapsulates the response,
/// and parses it back to BHTTP. On [OhttpGatewayException] from the
/// transport, the cache is invalidated before rethrowing.
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
      _cache.invalidate();
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
      headers: parsed.headers,
      body: parsed.body,
    );
  }
}
```

- [ ] **Step 4: Export from the public library**

Edit `lib/ohttp_dart.dart` — add the `ohttp_session.dart` export alongside the existing ones (keep `ohttp_client.dart` for now):

```dart
/// Pure Dart OHTTP client (RFC 9458).
///
/// Provides Oblivious HTTP encapsulation/decapsulation with:
/// - HPKE Base Mode Sender (RFC 9180)
/// - Binary HTTP serialization (RFC 9292)
/// - High-level OhttpClient for gateway communication
library;

export 'src/bhttp.dart';
export 'src/hpke.dart';
export 'src/key_config_cache.dart';
export 'src/ohttp.dart';
export 'src/ohttp_client.dart';
export 'src/ohttp_session.dart';
export 'src/transport.dart';
```

- [ ] **Step 5: Run the tests and verify they pass**

Run: `dart test test/ohttp_session_test.dart`
Expected: All 5 tests PASS.

- [ ] **Step 6: Run the full suite to confirm nothing else broke**

Run: `dart test`
Expected: All tests PASS (existing `bhttp_test`, `hpke_test`, `ohttp_test`, `key_config_cache_test`, `ohttp_session_test`).

- [ ] **Step 7: Run analyzer + format check**

Run: `dart analyze && dart format --line-length=120 --set-exit-if-changed lib/src/ohttp_session.dart test/ohttp_session_test.dart lib/ohttp_dart.dart`
Expected: `No issues found!` and no formatter complaints.

- [ ] **Step 8: Commit**

```bash
git add lib/src/ohttp_session.dart lib/ohttp_dart.dart test/ohttp_session_test.dart
git commit -m "feat: Add OhttpSession orchestrator and request/response data types

OhttpSession is the HTTP-client-neutral orchestration layer that replaces
the orchestration logic the wallet currently re-implements inline. The
cache-invalidation-on-gateway-error policy is centralized here so no
consumer can forget it."
```

---

### Task 4: HttpClientTransport (http-adapter, half 1)

**Files:**
- Create: `lib/src/adapters/http/http_transport.dart`
- Test: `test/adapters/http_adapter_test.dart` (transport portion; the OhttpHttpClient portion is added in Task 5)

**What:** `OhttpTransport` implementation backed by `package:http`. Uses `MockClient` from `package:http/testing.dart` for tests (already transitively available via `http`).

- [ ] **Step 1: Create the adapter directory**

Run: `mkdir -p lib/src/adapters/http test/adapters`

- [ ] **Step 2: Write the failing tests**

Write `test/adapters/http_adapter_test.dart`:

```dart
import 'dart:typed_data';

import 'package:http/http.dart' as http;
import 'package:http/testing.dart';
import 'package:ohttp_dart/http.dart';
import 'package:ohttp_dart/ohttp_dart.dart';
import 'package:test/test.dart';

void main() {
  group('HttpClientTransport', () {
    test('fetchKeyConfig GETs the keys URL and returns bytes on 200',
        () async {
      Uri? capturedUrl;
      final mock = MockClient((request) async {
        capturedUrl = request.url;
        return http.Response.bytes(
          Uint8List.fromList([1, 2, 3]),
          200,
          request: request,
        );
      });

      final transport = HttpClientTransport(
        client: mock,
        keysUrl: Uri.parse('https://gateway.example.com/ohttp-keys'),
        gatewayUrl: Uri.parse('https://gateway.example.com/ohttp'),
      );

      final bytes = await transport.fetchKeyConfig();

      expect(bytes, Uint8List.fromList([1, 2, 3]));
      expect(capturedUrl?.path, '/ohttp-keys');
    });

    test('fetchKeyConfig throws OhttpGatewayException on non-200', () async {
      final mock = MockClient(
        (request) async => http.Response('nope', 503, request: request),
      );
      final transport = HttpClientTransport(
        client: mock,
        keysUrl: Uri.parse('https://gateway.example.com/k'),
        gatewayUrl: Uri.parse('https://gateway.example.com/g'),
      );

      await expectLater(
        transport.fetchKeyConfig(),
        throwsA(isA<OhttpGatewayException>()
            .having((e) => e.statusCode, 'statusCode', 503)),
      );
    });

    test('postToGateway POSTs body with the correct Content-Type', () async {
      Uri? capturedUrl;
      String? capturedMethod;
      String? capturedContentType;
      List<int>? capturedBody;
      final mock = MockClient((request) async {
        capturedUrl = request.url;
        capturedMethod = request.method;
        capturedContentType = request.headers['content-type'];
        capturedBody = request.bodyBytes;
        return http.Response.bytes(
          Uint8List.fromList([9, 9, 9]),
          200,
          request: request,
        );
      });
      final transport = HttpClientTransport(
        client: mock,
        keysUrl: Uri.parse('https://gateway.example.com/k'),
        gatewayUrl: Uri.parse('https://gateway.example.com/g'),
      );

      final out = await transport.postToGateway(
        Uint8List.fromList([5, 6, 7]),
      );

      expect(out, Uint8List.fromList([9, 9, 9]));
      expect(capturedMethod, 'POST');
      expect(capturedUrl?.path, '/g');
      expect(capturedContentType, 'message/ohttp-req');
      expect(capturedBody, [5, 6, 7]);
    });

    test('postToGateway throws OhttpGatewayException on non-200', () async {
      final mock = MockClient(
        (request) async => http.Response('bad', 502, request: request),
      );
      final transport = HttpClientTransport(
        client: mock,
        keysUrl: Uri.parse('https://gateway.example.com/k'),
        gatewayUrl: Uri.parse('https://gateway.example.com/g'),
      );

      await expectLater(
        transport.postToGateway(Uint8List(0)),
        throwsA(isA<OhttpGatewayException>()
            .having((e) => e.statusCode, 'statusCode', 502)),
      );
    });
  });
}
```

Note: this test imports `package:ohttp_dart/http.dart`, which doesn't exist yet. The compile error in the next step is expected.

- [ ] **Step 3: Run the tests and verify they fail**

Run: `dart test test/adapters/http_adapter_test.dart`
Expected: Compile failure — `package:ohttp_dart/http.dart` not found.

- [ ] **Step 4: Implement HttpClientTransport**

Write `lib/src/adapters/http/http_transport.dart`:

```dart
import 'dart:typed_data';

import 'package:http/http.dart' as http;

import '../../transport.dart';

/// [OhttpTransport] backed by `package:http`.
///
/// Owns no client lifecycle: pass in any [http.Client] you already manage
/// (raw clients, instrumented clients, etc.). Closing this transport does
/// not close the underlying client.
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

- [ ] **Step 5: Create the http-adapter library entry point**

Write `lib/http.dart`:

```dart
/// `package:http` adapter for `ohttp_dart`.
///
/// Importing this library transitively brings in `package:http`. The core
/// `package:ohttp_dart/ohttp_dart.dart` library does not.
library;

export 'src/adapters/http/http_transport.dart';
// OhttpHttpClient export added in Task 5.
```

- [ ] **Step 6: Run the tests and verify they pass**

Run: `dart test test/adapters/http_adapter_test.dart`
Expected: All 4 tests PASS.

- [ ] **Step 7: Run the full suite + analyzer + format**

Run: `dart test && dart analyze && dart format --line-length=120 --set-exit-if-changed lib/src/adapters/http/http_transport.dart lib/http.dart test/adapters/http_adapter_test.dart`
Expected: Everything passes; no formatter changes.

- [ ] **Step 8: Commit**

```bash
git add lib/src/adapters/http/http_transport.dart lib/http.dart test/adapters/http_adapter_test.dart
git commit -m "feat: Add HttpClientTransport and http-adapter library entry

HttpClientTransport implements OhttpTransport on top of package:http.
Lives in the opt-in lib/http.dart sub-library so the core stays free of
any HTTP-client dependency at the import level."
```

---

### Task 5: OhttpHttpClient adapter (http-adapter, half 2)

**Files:**
- Create: `lib/src/adapters/http/ohttp_http_client.dart`
- Modify: `lib/http.dart`
- Modify: `test/adapters/http_adapter_test.dart` — add a new `group` for `OhttpHttpClient` tests.

**What:** The drop-in `http.Client` replacement that turns this whole package into "import `lib/http.dart` and you're done." Tests use a test-double subclass of `OhttpSession` that overrides `send` — simpler than mocking the entire crypto pipeline.

- [ ] **Step 1: Add failing OhttpHttpClient tests**

Edit `test/adapters/http_adapter_test.dart`. Add the following imports if they aren't already present (`http/testing.dart` should already be imported from Task 4):

```dart
import 'dart:convert';
```

Then append the following content **before the closing `}` of `void main()`**:

```dart
  group('OhttpHttpClient', () {
    test('extracts method/scheme/authority/path/body and forwards headers',
        () async {
      late OhttpRequestData captured;
      final session = _RecordingSession((req) {
        captured = req;
        return OhttpResponseData(
          statusCode: 200,
          headers: const [('content-type', 'text/plain')],
          body: Uint8List.fromList(utf8.encode('hi')),
        );
      });
      final client = OhttpHttpClient(session: session);

      final body = Uint8List.fromList(utf8.encode('payload'));
      final res = await client.post(
        Uri.parse('https://inner.example.com/api/v1/things?id=42'),
        headers: {'accept': 'text/plain'},
        body: body,
      );

      expect(captured.method, 'POST');
      expect(captured.scheme, 'https');
      expect(captured.authority, 'inner.example.com');
      expect(captured.path, '/api/v1/things?id=42');
      expect(captured.headers['accept'], 'text/plain');
      expect(captured.headers['host'], 'inner.example.com');
      expect(captured.body, body);

      expect(res.statusCode, 200);
      expect(res.body, utf8.encode('hi'));
      expect(res.headers['content-type'], 'text/plain');
    });

    test('preserves user-supplied host header instead of overwriting it',
        () async {
      late OhttpRequestData captured;
      final session = _RecordingSession((req) {
        captured = req;
        return OhttpResponseData(
          statusCode: 204,
          headers: const [],
          body: Uint8List(0),
        );
      });
      final client = OhttpHttpClient(session: session);

      await client.get(
        Uri.parse('https://inner.example.com/x'),
        headers: {'host': 'overridden.example.com'},
      );

      expect(captured.headers['host'], 'overridden.example.com');
    });

    test('includes non-default port in authority', () async {
      late OhttpRequestData captured;
      final session = _RecordingSession((req) {
        captured = req;
        return OhttpResponseData(
          statusCode: 200,
          headers: const [],
          body: Uint8List(0),
        );
      });
      final client = OhttpHttpClient(session: session);

      await client.get(Uri.parse('https://inner.example.com:8443/p'));

      expect(captured.authority, 'inner.example.com:8443');
    });

    test('omits default ports (443/80) from authority', () async {
      late OhttpRequestData captured;
      final session = _RecordingSession((req) {
        captured = req;
        return OhttpResponseData(
          statusCode: 200,
          headers: const [],
          body: Uint8List(0),
        );
      });
      final client = OhttpHttpClient(session: session);

      await client.get(Uri.parse('https://inner.example.com:443/p'));
      expect(captured.authority, 'inner.example.com');

      await client.get(Uri.parse('http://inner.example.com:80/p'));
      expect(captured.authority, 'inner.example.com');
    });

    test('close() is a no-op when closeWith is omitted', () async {
      final session = _RecordingSession(
        (_) => OhttpResponseData(
          statusCode: 200,
          headers: const [],
          body: Uint8List(0),
        ),
      );
      final client = OhttpHttpClient(session: session);

      // No inner client to track; just confirm close() doesn't throw and the
      // client remains constructed (negative assertion is implicit — no
      // accessible side effect).
      expect(() => client.close(), returnsNormally);
    });

    test('close() closes the closeWith client when supplied', () async {
      final inner = _ClosableMock();
      final session = _RecordingSession(
        (_) => OhttpResponseData(
          statusCode: 200,
          headers: const [],
          body: Uint8List(0),
        ),
      );
      final client = OhttpHttpClient(session: session, closeWith: inner);

      client.close();

      expect(inner.closed, isTrue);
    });
  });
}

/// Test double — bypasses the real OhttpSession's crypto by overriding [send].
class _RecordingSession extends OhttpSession {
  final OhttpResponseData Function(OhttpRequestData) _handler;

  _RecordingSession(this._handler)
      : super(
          transport: _NeverCalledTransport(),
          cache: KeyConfigCache(transport: _NeverCalledTransport()),
        );

  @override
  Future<OhttpResponseData> send(OhttpRequestData req) async => _handler(req);
}

class _NeverCalledTransport implements OhttpTransport {
  @override
  Future<Uint8List> fetchKeyConfig() =>
      throw StateError('transport should never be called in adapter tests');

  @override
  Future<Uint8List> postToGateway(Uint8List body) =>
      throw StateError('transport should never be called in adapter tests');
}

class _ClosableMock extends http.BaseClient {
  bool closed = false;

  @override
  Future<http.StreamedResponse> send(http.BaseRequest request) =>
      throw UnimplementedError();

  @override
  void close() {
    closed = true;
    super.close();
  }
}
```

Place the new `group('OhttpHttpClient', ...)` block inside the existing `void main()` body (after the `group('HttpClientTransport', ...)` block from Task 4), and place the helper classes (`_RecordingSession`, `_NeverCalledTransport`, `_ClosableMock`) at the file's top level after `main`. Final file structure: `void main() { group('HttpClientTransport', ...); group('OhttpHttpClient', ...); }` followed by the three helper classes.

- [ ] **Step 2: Run the tests and verify they fail**

Run: `dart test test/adapters/http_adapter_test.dart`
Expected: All `OhttpHttpClient` tests fail with compile error — `OhttpHttpClient` not defined.

- [ ] **Step 3: Implement OhttpHttpClient**

Write `lib/src/adapters/http/ohttp_http_client.dart`:

```dart
import 'dart:typed_data';

import 'package:http/http.dart' as http;

import '../../ohttp_session.dart';

/// Drop-in [http.BaseClient] replacement that routes every request through
/// an [OhttpSession].
///
/// Existing call sites that already speak `package:http` need no changes
/// beyond constructing this client and using it where they'd previously
/// have used [http.Client]. WebSocket connections bypass [http.Client]
/// entirely and are unaffected.
class OhttpHttpClient extends http.BaseClient {
  final OhttpSession _session;
  final http.Client? _innerToClose;

  /// [closeWith] is the optional raw client to close when this client is
  /// closed. Omit it if the inner client is shared across DI scopes (the
  /// common case in app DI containers).
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

    // dart:io adds `host` at transport level, but BHTTP serialization needs
    // it explicit — the inner request is encrypted and never touches
    // dart:io's transport layer.
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

    // List<(String, String)> → Map<String, String> is unavoidable here:
    // http.StreamedResponse.headers only accepts a Map. Consumers needing
    // lossless headers (e.g. duplicate Set-Cookie) should use
    // OhttpSession.send directly.
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
  void close() {
    _innerToClose?.close();
    super.close();
  }
}
```

- [ ] **Step 4: Export from the http-adapter library**

Replace the contents of `lib/http.dart`:

```dart
/// `package:http` adapter for `ohttp_dart`.
///
/// Importing this library transitively brings in `package:http`. The core
/// `package:ohttp_dart/ohttp_dart.dart` library does not.
library;

export 'src/adapters/http/http_transport.dart';
export 'src/adapters/http/ohttp_http_client.dart';
```

- [ ] **Step 5: Run the tests and verify they pass**

Run: `dart test test/adapters/http_adapter_test.dart`
Expected: All tests PASS (4 from Task 4 + 6 new).

- [ ] **Step 6: Run the full suite + analyzer + format**

Run: `dart test && dart analyze && dart format --line-length=120 --set-exit-if-changed lib/src/adapters/http/ohttp_http_client.dart lib/http.dart test/adapters/http_adapter_test.dart`
Expected: Everything passes; no formatter changes.

- [ ] **Step 7: Commit**

```bash
git add lib/src/adapters/http/ohttp_http_client.dart lib/http.dart test/adapters/http_adapter_test.dart
git commit -m "feat: Add OhttpHttpClient drop-in http.BaseClient adapter

Consumers using package:http now get OHTTP routing by constructing
OhttpHttpClient and using it anywhere they'd use http.Client. The
closeWith parameter is opt-in to avoid the footgun of auto-closing a
shared raw client."
```

---

### Task 6: Delete the old OhttpClient and update core exports

**Files:**
- Delete: `lib/src/ohttp_client.dart`
- Modify: `lib/ohttp_dart.dart`

**What:** Remove the dead `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig` types from the core. The new types replace them.

- [ ] **Step 1: Verify no test references the old class**

Run: `grep -rn 'OhttpClient\b\|OhttpResponse\b\|OhttpHeader\b\|OhttpGatewayConfig\b' test/ lib/ example/`
Expected: only matches in `lib/src/ohttp_client.dart` itself, plus `example/ohttp_dart_example.dart` (will be rewritten in Task 7) and `lib/ohttp_dart.dart` (export line we're about to drop). No matches in `test/`.

If anything in `test/` still references these names, stop and update the failing test to use the new types before continuing.

- [ ] **Step 2: Delete the old file**

Run: `git rm lib/src/ohttp_client.dart`
Expected: file removed from working tree and staged for deletion.

- [ ] **Step 3: Update the public library exports**

Write `lib/ohttp_dart.dart`:

```dart
/// Pure Dart OHTTP client (RFC 9458).
///
/// Provides Oblivious HTTP encapsulation/decapsulation with:
/// - HPKE Base Mode Sender (RFC 9180)
/// - Binary HTTP serialization (RFC 9292)
/// - Pluggable transport (OhttpTransport) and orchestration (OhttpSession)
///
/// HTTP-client adapters live in separate libraries:
/// - `package:ohttp_dart/http.dart` for `package:http`
library;

export 'src/bhttp.dart';
export 'src/hpke.dart';
export 'src/key_config_cache.dart';
export 'src/ohttp.dart';
export 'src/ohttp_session.dart';
export 'src/transport.dart';
```

- [ ] **Step 4: Verify the example still compiles (it will not — that's expected for now)**

Run: `dart analyze example/ohttp_dart_example.dart`
Expected: Errors about `OhttpClient`, `OhttpGatewayConfig` being undefined. **This is expected** — the example is rewritten in Task 7.

- [ ] **Step 5: Verify the rest of the project still compiles**

Run: `dart analyze lib/ test/`
Expected: `No issues found!`

- [ ] **Step 6: Run the full test suite**

Run: `dart test`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git add lib/ohttp_dart.dart
git commit -m "refactor: Remove legacy OhttpClient and update core exports

OhttpClient/OhttpResponse/OhttpHeader/OhttpGatewayConfig are removed
outright. They had no external consumers and have been superseded by
the OhttpSession + OhttpRequestData + OhttpResponseData triple plus
the optional OhttpHttpClient adapter in package:ohttp_dart/http.dart."
```

---

### Task 7: Rewrite the example

**Files:**
- Modify: `example/ohttp_dart_example.dart`

**What:** Show both integration paths — the `http`-adapter quick-start (which most readers will copy) and the lower-level `OhttpSession.send` for `http`-agnostic consumers.

- [ ] **Step 1: Rewrite the example**

Write `example/ohttp_dart_example.dart`:

```dart
// ignore_for_file: avoid_print

import 'dart:convert';
import 'dart:typed_data';

import 'package:http/http.dart' as http;
import 'package:ohttp_dart/http.dart';
import 'package:ohttp_dart/ohttp_dart.dart';

/// Edit these for your gateway before running.
const _keysUrl = 'https://your-gateway.example.com/ohttp/config';
const _gatewayUrl = 'https://your-gateway.example.com/ohttp/gateway';
const _targetHost = 'your-target.example.com';

/// Path 1: `package:http` drop-in — recommended for apps already using `http`.
Future<void> httpAdapterDemo() async {
  final raw = http.Client();
  final transport = HttpClientTransport(
    client: raw,
    keysUrl: Uri.parse(_keysUrl),
    gatewayUrl: Uri.parse(_gatewayUrl),
  );
  final session = OhttpSession.withTransport(transport: transport);
  final ohttpClient = OhttpHttpClient(session: session, closeWith: raw);

  try {
    final res = await ohttpClient.get(
      Uri.parse('https://$_targetHost/get'),
      headers: {'accept': 'application/json'},
    );
    print('GET ${res.statusCode}: ${res.body}');

    final post = await ohttpClient.post(
      Uri.parse('https://$_targetHost/post'),
      headers: const {
        'content-type': 'application/json',
        'accept': 'application/json',
      },
      body: utf8.encode('{"hello":"ohttp"}'),
    );
    print('POST ${post.statusCode}: ${post.body}');
  } finally {
    ohttpClient.close();
  }
}

/// Path 2: HTTP-client-neutral — use OhttpSession.send with any transport.
///
/// This is what a Dio or custom-client adapter would use under the hood.
Future<void> sessionDemo() async {
  final raw = http.Client();
  final transport = HttpClientTransport(
    client: raw,
    keysUrl: Uri.parse(_keysUrl),
    gatewayUrl: Uri.parse(_gatewayUrl),
  );
  final session = OhttpSession.withTransport(transport: transport);

  try {
    final res = await session.send(OhttpRequestData(
      method: 'GET',
      scheme: 'https',
      authority: _targetHost,
      path: '/get',
      headers: const {'accept': 'application/json'},
      body: Uint8List(0),
    ));
    print('Session GET ${res.statusCode}: ${utf8.decode(res.body)}');
  } finally {
    raw.close();
  }
}

void main() async {
  await httpAdapterDemo();
  await sessionDemo();
}
```

- [ ] **Step 2: Verify the example compiles cleanly**

Run: `dart analyze example/ohttp_dart_example.dart && dart format --line-length=120 --set-exit-if-changed example/ohttp_dart_example.dart`
Expected: `No issues found!` and no formatter changes.

- [ ] **Step 3: Verify the full project still passes**

Run: `dart test && dart analyze`
Expected: Everything green.

- [ ] **Step 4: Commit**

```bash
git add example/ohttp_dart_example.dart
git commit -m "docs: Rewrite example to show http-adapter and session APIs

Quick-start path uses OhttpHttpClient (most consumers will copy this).
Second path uses OhttpSession.send directly so a Dio or custom adapter
can be sketched from the same starting point."
```

---

### Task 8: Update CLAUDE.md and README.md

**Files:**
- Modify: `CLAUDE.md`
- Modify: `README.md`

**What:** Bring the docs in line with the new layering and surface area.

- [ ] **Step 1: Update `CLAUDE.md` Layering section**

In `CLAUDE.md`, locate the section beginning `## Layering` and replace the entire enumerated list (currently four numbered items 1–4, with the file list ending at `ohttp_client.dart`). Replace with:

```markdown
## Layering

Two libraries, one package. Core (`lib/ohttp_dart.dart`) has no HTTP-client deps. The `package:http` adapter is opt-in via `lib/http.dart`.

**Core — `lib/src/`:**

1. **`bhttp.dart`** — RFC 9292 Binary HTTP. Self-contained: QUIC varints (`encodeVarint`/`decodeVarint`), Known-Length framing (indicator `0` for requests, `1` for responses), `serializeRequest` / `parseResponse`. No crypto.
2. **`hpke.dart`** — RFC 9180 HPKE Base Mode **Sender only** (no receiver). Wraps `package:cryptography` primitives (`X25519`, `Hmac.sha256`, `AesGcm.with128bits`) into the labeled `LabeledExtract` / `LabeledExpand` KDF, KEM Encap, key schedule, and a stateful `HpkeSenderContext` with sequence-number-derived nonces. `setupBaseS` accepts an optional `testKeyPair` so RFC 9180 Appendix A.1 vectors are reproducible.
3. **`ohttp.dart`** — RFC 9458. Parses the gateway's `KeyConfig` (`OhttpKeyConfig.parse`), then `ohttpEncapsulate` builds the HPKE info string (`"message/bhttp request" || 0x00 || header`), seals the BHTTP request with **empty AAD** (matches the Go reference, not the RFC's header-as-AAD wording), and exports a `"message/bhttp response"` secret. `ohttpDecapsulate` runs **plain (unlabeled) HKDF** over `salt = enc || response_nonce` to derive the response AEAD key/nonce.
4. **`transport.dart`** — `OhttpTransport` interface (bytes-in/bytes-out: fetch KeyConfig, POST encapsulated bytes) and `OhttpGatewayException`. The seam where consumers plug in their HTTP client. Lives in core so the cache-invalidation policy in `OhttpSession` can see the exception type.
5. **`key_config_cache.dart`** — `KeyConfigCache`. TTL-based, single-flight de-dup so concurrent stale `get()` calls share one fetch. Clock is injectable for deterministic TTL tests.
6. **`ohttp_session.dart`** — `OhttpSession` orchestration plus `OhttpRequestData` / `OhttpResponseData`. Owns `(transport, cache)`; target authority is per-request. On `OhttpGatewayException` from the transport, invalidates the cache then rethrows. Non-gateway errors do not invalidate.

**`package:http` adapter — `lib/src/adapters/http/` (exported via `lib/http.dart`):**

7. **`http_transport.dart`** — `HttpClientTransport implements OhttpTransport`. Wraps an `http.Client` + keys URL + gateway URL.
8. **`ohttp_http_client.dart`** — `OhttpHttpClient extends http.BaseClient`. Drop-in `http.Client` replacement; owns an `OhttpSession`. The opt-in `closeWith` parameter controls whether `close()` propagates to a raw client (off by default — assume the raw client is shared in DI).
```

- [ ] **Step 2: Update the "Cross-cutting gotchas" section in `CLAUDE.md`**

In the existing `## Cross-cutting gotchas` section, append this new bullet after the existing fourth bullet (the one about KeyConfig parser KEM support):

```markdown
- The `host` header injection lives in the `http` adapter (`OhttpHttpClient.send`), not in core. `dart:io` normally adds `host` at the transport layer, but the inner BHTTP request is encrypted and never reaches that layer — so the adapter materializes the header before handing the request to `OhttpSession`. Other adapters (Dio, custom) must do the same.
```

- [ ] **Step 3: Update `README.md` — Features bullet list**

In `README.md`, replace this line:

```markdown
- **High-level OhttpClient** — fetch KeyConfig + send requests via gateway
```

with these two:

```markdown
- **OhttpSession** — HTTP-client-neutral orchestrator + `OhttpTransport` interface
- **`http` adapter** (`package:ohttp_dart/http.dart`) — drop-in `OhttpHttpClient extends http.BaseClient`
```

- [ ] **Step 4: Update `README.md` — Project Structure block**

Replace the `## Project Structure` code block with:

```markdown
## Project Structure

```
lib/
├── ohttp_dart.dart          # Core library exports (no HTTP-client dep)
├── http.dart                # Opt-in package:http adapter exports
└── src/
    ├── bhttp.dart           # Binary HTTP serialize/parse (RFC 9292)
    ├── hpke.dart            # HPKE Base Mode Sender (RFC 9180)
    ├── ohttp.dart           # OHTTP encap/decap + KeyConfig parser (RFC 9458)
    ├── transport.dart       # OhttpTransport interface + OhttpGatewayException
    ├── key_config_cache.dart  # TTL + single-flight KeyConfig cache
    ├── ohttp_session.dart   # Orchestrator + OhttpRequestData/OhttpResponseData
    └── adapters/
        └── http/
            ├── http_transport.dart      # HttpClientTransport
            └── ohttp_http_client.dart   # OhttpHttpClient extends http.BaseClient
test/
├── bhttp_test.dart          # Varint, serialization, parsing
├── hpke_test.dart           # RFC 9180 test vectors
├── ohttp_test.dart          # Encapsulate/decapsulate, KeyConfig
├── key_config_cache_test.dart   # TTL, invalidation, single-flight
├── ohttp_session_test.dart      # Orchestration, error policies
└── adapters/
    └── http_adapter_test.dart   # HttpClientTransport + OhttpHttpClient
```
```

- [ ] **Step 5: Update `README.md` — Usage section**

Replace the `## Usage` section's code block with:

```markdown
## Usage

```dart
import 'package:http/http.dart' as http;
import 'package:ohttp_dart/http.dart';
import 'package:ohttp_dart/ohttp_dart.dart';

final raw = http.Client();
final transport = HttpClientTransport(
  client: raw,
  keysUrl: Uri.parse('https://gateway.example.com/ohttp/config'),
  gatewayUrl: Uri.parse('https://gateway.example.com/ohttp/gateway'),
);
final session = OhttpSession.withTransport(transport: transport);
final client = OhttpHttpClient(session: session, closeWith: raw);

final res = await client.get(Uri.parse('https://target.example.com/api/data'));
print(res.statusCode);
print(res.body);
```

For HTTP-client-agnostic use (Dio, custom clients), use `OhttpSession.send` directly with your own `OhttpTransport` implementation. See `example/ohttp_dart_example.dart`.
```

- [ ] **Step 6: Verify markdown didn't break anything**

Run: `dart test && dart analyze`
Expected: Everything green (these doc files don't affect compilation, but run anyway as a sanity check).

- [ ] **Step 7: Commit**

```bash
git add CLAUDE.md README.md
git commit -m "docs: Update CLAUDE.md and README.md for new layering

Layering grows from four files to six (core) plus two (adapter). Adds
a fourth Cross-cutting gotcha about the host-header injection living
in the adapter rather than core. README Usage block shows the new
quick-start with OhttpHttpClient."
```

---

## Final verification

- [ ] **Run the whole suite one more time, with the strict formatter check**

Run:
```bash
dart test && \
dart analyze && \
dart format --line-length=120 --set-exit-if-changed lib/ test/ example/
```
Expected: All tests pass, no analyzer issues, no formatter changes.

- [ ] **Verify the wallet can do the swap as described**

Confirm by inspection that the wallet's planned replacement code from the spec compiles against the new public surface (no need to actually wire it into the wallet branch — just re-read the snippet in `docs/superpowers/specs/2026-05-25-http-client-agnostic-integration-design.md` and verify every identifier exists in the package now):

- `http.Client` — from `package:http`
- `HttpClientTransport` — exported by `lib/http.dart`
- `OhttpSession.withTransport` — exported by `lib/ohttp_dart.dart`
- `OhttpHttpClient` — exported by `lib/http.dart`

If any identifier is missing, fix it before declaring done.
