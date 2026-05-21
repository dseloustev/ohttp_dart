# Task 12: Add size caps on BHTTP header section and response body

**Severity:** HIGH
**Vector:** Parser robustness
**Files:** `lib/src/bhttp.dart:164-187; lib/src/ohttp_client.dart:129-133`

**Evidence:** Cross-refs F2.4a, F2.4b, F2.4c, C-9. `parseResponse` at lines 164–187 reads `headersLen` and `contentLen` as raw varints; an adversarial gateway can declare `headersLen = 2^30`, forcing a ~1 GiB allocation in `data.sublist(...)` at line 173 or 178. Lines 129–133 of `ohttp_client.dart` pass `gatewayResponse.bodyBytes` to `ohttpDecapsulate` without inspecting its length. RFC 9292 §3.5.3 governs the framing.

## Description

A malicious or compromised gateway can return a payload that declares enormous header or body lengths to exhaust the wallet's memory or trigger an OOM crash before any parsing happens. On a mobile wallet, even a sub-OOM allocation in the hundreds of MiB is enough to terminate the app. The library currently provides no defensive bound at either the BHTTP layer (`parseResponse`) or the OHTTP layer (`OhttpClient.send` before handing bytes to `ohttpDecapsulate`).

## Proposed change

Apply two layered caps:

1. At the OHTTP client layer (lines 129–133 of `ohttp_client.dart`), check `gatewayResponse.bodyBytes.length` against a configurable upper bound and throw a typed library exception if exceeded. Coordinate with the typed-error hierarchy from task 10.
2. At the BHTTP layer (lines 164–187 of `bhttp.dart`), check `headersLen` and `contentLen` against configurable upper bounds before allocating. Reject oversized declarations with `FormatException` (coordinate with task 11's helper).

Both caps must be configurable (via `OhttpGatewayConfig` or equivalent) and documented with sensible defaults appropriate for wallet RPC payloads (single-digit MiB at most).

## Acceptance criteria

- `parseResponse` rejects header sections larger than the configured cap with a `FormatException`.
- `parseResponse` rejects content larger than the configured cap with a `FormatException`.
- `OhttpClient.send` rejects gateway responses larger than the configured cap before parsing.
- Defaults are documented and conservatively low (suitable for wallet payloads).
- Unit tests cover: oversized headers, oversized body, oversized response payload at the client layer.
