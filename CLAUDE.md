# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Pure-Dart implementation of Oblivious HTTP (RFC 9458). No native dependencies, no Flutter — runs on plain Dart VM. Package is `ohttp_dart` (Dart SDK `^3.11.1`); `publish_to: none`.

The library is built around a **single fixed cipher suite**: DHKEM(X25519, HKDF-SHA256) + HKDF-SHA256 + AES-128-GCM (KEM `0x0020`, KDF `0x0001`, AEAD `0x0001`). These IDs are hard-coded in `hpke.dart` and validated in `OhttpKeyConfig.validate()`. If a gateway advertises a different suite, the client throws — adding suites means changing constants and key/nonce/hash sizes across `hpke.dart` and `ohttp.dart`, not just adding a branch.

## Commands

```bash
dart pub get                        # install deps
dart test                           # run all tests
dart test test/hpke_test.dart       # run a single test file
dart test -n 'HKDF-Extract'         # run tests by name regex
dart analyze                        # static analysis (strict-casts + strict-raw-types enabled)
dart format --line-length=120 .     # format (analysis_options sets page_width: 120, preserve trailing commas)
dart run example/ohttp_dart_example.dart   # example needs a real gateway URL — edit before running
```

Lints are strict: `lints/recommended` plus a long custom list in `analysis_options.yaml` (e.g. `require_trailing_commas`, `prefer_single_quotes`, `always_declare_return_types`, `unawaited_futures`, `prefer_const_constructors`). `dead_code` and `invalid_assignment` are promoted to errors.

## Layering

Four files in `lib/src/`, layered bottom-up — each depends only on the ones above it in this list:

1. **`bhttp.dart`** — RFC 9292 Binary HTTP. Self-contained: QUIC varints (`encodeVarint`/`decodeVarint`), Known-Length framing (indicator `0` for requests, `1` for responses), `serializeRequest` / `parseResponse`. No crypto.
2. **`hpke.dart`** — RFC 9180 HPKE Base Mode **Sender only** (no receiver). Wraps `package:cryptography` primitives (`X25519`, `Hmac.sha256`, `AesGcm.with128bits`) into the labeled `LabeledExtract` / `LabeledExpand` KDF, KEM Encap, key schedule, and a stateful `HpkeSenderContext` with sequence-number-derived nonces. `setupBaseS` accepts an optional `testKeyPair` so RFC 9180 Appendix A.1 vectors are reproducible.
3. **`ohttp.dart`** — RFC 9458. Parses the gateway's `KeyConfig` (`OhttpKeyConfig.parse`), then `ohttpEncapsulate` builds the HPKE info string (`"message/bhttp request" || 0x00 || header`), seals the BHTTP request with **empty AAD** (matches the Go reference, not the RFC's header-as-AAD wording), and exports a `"message/bhttp response"` secret. `ohttpDecapsulate` runs **plain (unlabeled) HKDF** over `salt = enc || response_nonce` to derive the response AEAD key/nonce.
4. **`ohttp_client.dart`** — High-level `OhttpClient`. Orchestrates: GET `KeyConfig` → BHTTP-serialize the inner request → OHTTP-encapsulate → POST to gateway (`Content-Type: message/ohttp-req`) → decapsulate → BHTTP-parse. `OhttpGatewayConfig` holds gateway base URL + config/request paths + the **target authority** that gets embedded in the inner BHTTP request (this is the privacy-preserving target server, distinct from the gateway host).

The public surface is re-exported from `lib/ohttp_dart.dart`.

## Tests

`test/` mirrors `lib/src/` one-to-one. `hpke_test.dart` validates against RFC 9180 Appendix A.1 vectors using the `testKeyPair` injection hook in `setupBaseS` — keep that hook when refactoring HPKE or the vector tests break silently (they'd start using random ephemerals). `bhttp_test.dart` covers QUIC varint round-trips at the 1/2/4/8-byte boundaries. There are no integration tests against a live gateway.

## Cross-cutting gotchas

- **Empty AAD** in request sealing (`ohttp.dart:149`) is intentional — don't "fix" it to use the OHTTP header even though some RFC readings suggest it. The bundled tests and the Go interop reference both expect empty AAD.
- **Response decap uses plain HKDF**, not the labeled `LabeledExtract`/`LabeledExpand` from HPKE — they're different code paths (`HpkeSender.hkdfExtract` / `hkdfExpand` in `hpke.dart` vs. the labeled variants used internally during `setupBaseS`).
- The KeyConfig parser only knows X25519 (`kemId == 0x0020`); other KEMs require a new `pkLen` branch *and* matching changes in `hpke.dart`.
- Sequence-number nonce computation in `HpkeSenderContext.seal` XORs the counter into `baseNonce`; a single context instance is one-shot in practice (OHTTP only seals once), but the counter machinery is there if it's ever reused.

## Project conventions

- AST-index rules in `.claude/rules/ast-index.md` are loaded automatically — use `ast-index` before `grep` for symbol search.
- `lib/ohttp_dart.dart` is the only public entry point; everything else in `lib/src/` is internal even though there's no `package:meta` enforcement.
- Trailing commas are preserved by the formatter (`trailing_commas: preserve` in `analysis_options.yaml`) — keep them on multi-line argument lists.
