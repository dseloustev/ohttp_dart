# Task 28: Expand empty-AAD comment at `ctx.seal` call site

**Severity:** IMPROVEMENT
**Vector:** Documentation
**Files:** `lib/src/ohttp.dart:148-150`

**Evidence:** Cross-ref TASK-O3. Line 148: `// Seal with empty AAD (per reference Go implementation, not RFC header)`. Line 149: `final ct = await ctx.seal(Uint8List(0), binaryRequest);`. Line 150 is blank. The comment mentions the Go reference but does not cite RFC 9458 §4.3 or explain why the deviation is intentional and safe.

## Description

A future maintainer reading this code will reasonably wonder why the OHTTP header is not used as AAD (some readings of RFC 9458 §4.3 suggest it should be). The current one-line comment is insufficient to dissuade a well-intentioned "fix" that would break interoperability with the reference implementation. The comment must explain the decision, cite the RFC section, link to the Go reference, and warn that changing this without coordinated gateway updates breaks interop.

## Proposed change

Expand the comment to:
1. Cite RFC 9458 §4.3 and explain the wording ambiguity.
2. Reference the Go reference implementation (URL or commit identifier) as the source of truth for the empty-AAD choice.
3. State that the bundled tests rely on empty AAD and that changing it without coordinated gateway changes is a wire-format break.
4. Cross-reference task 29 (plain HKDF comment) as the companion maintenance-hazard note.

## Acceptance criteria

- Comment block at lines 148–150 is expanded to at least the four points above.
- A URL or specific commit reference to the Go interop implementation is included.
- Maintainer reading the comment can decide whether a proposed change would break interop without further research.
