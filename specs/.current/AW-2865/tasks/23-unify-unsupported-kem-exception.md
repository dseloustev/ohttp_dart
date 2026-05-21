# Task 23: Unify exception types for unsupported KEM in `parse()` and `validate()`

**Severity:** IMPROVEMENT
**Vector:** Cryptographic correctness
**Files:** `lib/src/ohttp.dart:51, 82-85`

**Evidence:** Cross-refs TASK-O2, T-O5. Line 51 of `lib/src/ohttp.dart`: `parse()` throws `FormatException('Unsupported KEM: ...')`. Lines 82–85: `validate()` throws `UnsupportedError` for the same conceptual condition. Two unrelated exception hierarchies for the same error class force callers to write two `catch` blocks.

## Description

`parse()` and `validate()` disagree on the exception type for the identical "unsupported KEM" error. Callers must either catch broadly (defeating the purpose of typed exceptions) or maintain two parallel catch paths. The mismatch is purely accidental — neither type is preferable in principle — so consolidation is a small, contained improvement.

## Proposed change

Introduce a library-owned `OhttpUnsupportedSuiteException` (or similar) and throw it from both call sites. Re-export from `lib/ohttp_dart.dart`. Coordinate with task 10 if a broader exception hierarchy is being built — `OhttpUnsupportedSuiteException` should slot into that hierarchy.

## Acceptance criteria

- Single library-owned exception type is thrown from both `parse()` (line 51) and `validate()` (lines 82–85) for unsupported KEM.
- New type is re-exported from `lib/ohttp_dart.dart`.
- Unit tests for both `parse()` and `validate()` verify the unified exception type.
- Migration note documents the change for any downstream caller that catches the previous types.
