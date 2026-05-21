## Unreleased

- AW-2865 Phase 1: Read-only RFC 9180 audit of `lib/src/hpke.dart`. All labeled-KDF, KEM Encap, key-schedule, nonce-derivation, and export sections verified correct. Seven findings produced (2 HIGH, 5 IMPROVEMENT, 0 BLOCKER); draft Jira tasks TASK-1 through TASK-7 recorded. No code changed.

## 0.1.0

- OHTTP client (RFC 9458) — encapsulate/decapsulate HTTP requests via gateway
- HPKE Base Mode Sender (RFC 9180) — pure Dart, tested against RFC test vectors
- Binary HTTP (RFC 9292) — serialize/parse HTTP messages
- High-level `OhttpClient` with configurable gateway
- Tested on iOS, macOS, Android, Windows
