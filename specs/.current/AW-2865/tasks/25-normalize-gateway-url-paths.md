# Task 25: Normalize gateway URL path construction

**Severity:** IMPROVEMENT
**Vector:** Network reliability
**Files:** `lib/src/ohttp_client.dart:82, 116, 154`

**Evidence:** Cross-ref C-2. Line 82: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}')`. Line 116: `Uri.parse('${gateway.gatewayBaseUrl}${gateway.requestPath}')`. Line 154: `Uri.parse('${gateway.effectiveDirectBaseUrl}$path')`. Three call sites concatenate strings without normalizing slashes.

## Description

A caller who supplies `gatewayBaseUrl: 'https://gw.example/'` and `configPath: '/key'` produces `https://gw.example//key` — a double slash that most servers tolerate but a strict reverse proxy may reject. The mirror case (no trailing slash on the base, no leading slash on the path) produces a path that misses the separator entirely. Both fail loudly only sometimes; quiet failures (302 redirect to the wrong path) waste developer time.

## Proposed change

Centralize the URL-composition logic in a single helper that normalizes slash handling and unit-test it directly. The three call sites at lines 82, 116, and 154 should delegate to the helper. The helper should also validate that the constructed `Uri` is well-formed before returning.

## Acceptance criteria

- A single helper handles URL composition for KeyConfig GET, gateway POST, and direct-send paths.
- Helper handles all four cases: base with/without trailing slash × path with/without leading slash.
- Unit tests cover the four cases plus the well-formedness assertion.
- Call sites at lines 82, 116, 154 use the helper.
