# Task 10: Add a typed error hierarchy for `OhttpClient`

**Severity:** HIGH
**Vector:** Network reliability
**Files:** `lib/src/ohttp_client.dart:84-87, 120-122`

**Evidence:** Cross-refs C-7, C-8. Lines 84–87 contain `if (configResponse.statusCode != 200) { throw Exception('...'); }`. Lines 120–122 contain the same `throw Exception('...')` pattern for the gateway POST. Both use the bare `Exception` type and only embed status code information in the message string.

## Description

Bare `Exception` instances force callers to match on string contents or fall back to broad `catch (e)` blocks. That makes meaningful error handling — distinguishing a gateway 5xx from a 4xx, a timeout from a parse error, an AEAD authentication failure from a transport failure — impossible without parsing exception messages. For a wallet, this prevents sensible UI flows like "retry on transient failure, abort on misconfiguration."

## Proposed change

Introduce a typed library exception hierarchy under a common base (e.g., `OhttpException`) with concrete subclasses for: HTTP status errors (with `statusCode` field), timeouts, parse errors, key config errors, and decryption errors (coordinates with task 07). Replace bare `throw Exception(...)` calls in `OhttpClient.send()` and `OhttpClient.sendDirect()` with the typed variants. Re-export the hierarchy from `lib/ohttp_dart.dart`.

## Acceptance criteria

- Library-owned `OhttpException` base class is exported with at least the subtypes listed above.
- All `throw Exception(...)` sites in `lib/src/ohttp_client.dart` are replaced with typed subclasses.
- Doc comments on `send()` and `sendDirect()` enumerate the exceptions they may throw.
- Unit tests cover at least: gateway 4xx → typed error with `statusCode == 4xx`; gateway 5xx → typed error with `statusCode == 5xx`; timeout → typed error.
