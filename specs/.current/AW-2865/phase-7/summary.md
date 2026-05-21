# AW-2865 Phase 7 — Summary

**Phase:** 7 — Compile findings into per-task Markdown files
**Status:** COMPLETE (QA verdict: RELEASE)
**Date:** 2026-05-21
**No source code was changed in this phase.**

---

## What was done

Phase 7 is the final deliverable phase of the AW-2865 investigation. Its sole output is a structured set of independently actionable Markdown task files under `specs/.current/AW-2865/tasks/`, synthesizing all audit findings from Phases 1–6.

The deliverable consists of 35 files in total:

- `specs/.current/AW-2865/tasks/README.md` — executive summary with a direct production-readiness stance, a blockers section, a 34-row task index table, and an out-of-scope section.
- 34 numbered task files (`01-ohttp-bypass-send-direct.md` through `35-test-ohttp-edge-cases.md`, with the number slot `22` intentionally absent).

Each task file is self-contained: it carries a severity label, investigation vector, repo-relative file and line citations, an evidence rationale, a problem description, a proposed change, and bullet-form acceptance criteria. A downstream engineer can scope and begin the work from one file alone without re-reading the audit phases.

The severity breakdown by effective label:

- **1 BLOCKER** — task 01 (`ohttp-bypass-send-direct`)
- **21 HIGH** — tasks 02–21 plus task 32 (escalated)
- **12 IMPROVEMENT** — tasks 23–31, 33–35

All eight investigation vectors are covered across the task files: Cryptographic correctness, Interoperability, Parser robustness, Privacy risks, Network reliability, KeyConfig management, Test suite, Documentation.

---

## Key decisions

### Row 22 merged into task 13

The Consolidated Findings Inventory originally contained 35 rows. Row 22 (`add-structured-observability-hooks`, cross-ref T-X1 note, vector: Interoperability) covered the same structured-observability gap as row 13. Both rows were merged into a single file (`13-add-structured-observability-hooks.md`). The merged file cites cross-references `P6-5, P6-6, P6-8, T-X1 (row 22 note)`. The final deliverable is 34 task files, not 35. The number slot `22` is permanently absent from the filesystem; the README table documents this explicitly with a note below the index.

### Task 32 escalated from IMPROVEMENT to HIGH

Task 32 (`make-effective-direct-base-url-private`) was classified as IMPROVEMENT in the Consolidated Findings Inventory. During Phase 7 authoring it was escalated to HIGH. The rationale recorded in the task file's Evidence field: the `effectiveDirectBaseUrl` public getter silently resolves `null → gatewayBaseUrl` and is reachable via IDE autocomplete from any consumer of `OhttpGatewayConfig`, widening the discoverable surface of the BLOCKER-severity task 01 footgun. The task file documents the original classification, the new classification, and the specific evidence. Task 32 retains its original file number (no renumbering); the README table places it contiguously with the HIGH block for severity-based prioritization.

### Direct-stance README

The README executive summary was required by the PRD to take an explicit production-readiness stance with no neutral or hedged language. The delivered text reads: "the library ... **must not be used in production until the 1 BLOCKER and 21 HIGH items below are resolved**." The IMPROVEMENT count in the summary is 12 (not 13), reflecting the task 32 escalation and the row 22 merge. This wording was verified as satisfying PRD constraint 9.

### Severity-count accounting (two parallel counts)

Two counts coexist after the task 32 escalation. The file-numbering (slot-based) count is 1 BLOCKER, 20 HIGH (numbers 02–21), 13 IMPROVEMENT (numbers 23–35). The effective-severity count used in the README executive summary and the index table's Severity column is 1 BLOCKER, 21 HIGH (02–21 plus escalated 32), 12 IMPROVEMENT (23–31, 33–35). Both counts are consistent and intentional; the task 32 file documents the escalation so the difference is traceable.

### Section 4 logging constraints embedded in task 13

Task 13 (`add-structured-observability-hooks`) merged the observability gap from row 22 and reproduced the Phase 7 section 4 logging constraints verbatim in its Proposed change section. The constraints specify what must never be logged (key material, `enc`, inner request content, `targetAuthority`, response body/headers, `gatewayBaseUrl` at DEBUG level in production) and what is safe to log (KeyConfig fetch status code, gateway POST status code, `sendDirect()` invocation signal). These constraints are the authoritative reference for any future observability implementation.

---

## QA verdict

**RELEASE.** All acceptance criteria from `phase-7/tasks.md` and the PRD Success / Metrics table passed:

| Criterion | Result |
|-----------|--------|
| 34 numbered task files present (gap at 22) | PASS |
| `README.md` exists with 34-row index table | PASS |
| Every task file has all 8 required fields | PASS |
| All 8 investigation vectors covered | PASS |
| At least one BLOCKER task file (task 01) | PASS |
| No duplicate findings across files | PASS |
| README executive summary takes direct production stance | PASS |
| Task 32 Evidence field documents escalation | PASS |
| Task 13 includes section 4 logging constraints verbatim | PASS |
| README IMPROVEMENT count = 12 | PASS |
| No absolute paths in any task file or README | PASS |
| No source code changed (`lib/`, `test/`, `example/`) | PASS |

Three low-severity observations were noted in the QA risk zone (task 32 numbering may jar readers, task 30 "all test files" Files field is coarse, task 13 merged scope may span more than one sprint) — none blocks release.

---

## Overall investigation conclusion

Phases 1–7 of AW-2865 assessed the `ohttp_dart` package across eight investigation vectors. The core RFC implementations (RFC 9180 HPKE, RFC 9292 Binary HTTP, RFC 9458 OHTTP) are correct against test vectors and are interoperable with the Go reference. The library is **not production-ready for a non-custodial crypto wallet** in its current state.

The single BLOCKER (`sendDirect()` OHTTP bypass) can silently expose wallet transactions as plaintext with no runtime signal to the caller. The 21 HIGH items address missing input validation, missing typed errors, missing timeouts and KeyConfig caching, missing key-material zeroization, missing observability, and significant test-coverage gaps. These gaps collectively represent a risk surface that must be closed before the package is used in a wallet context.

The 12 IMPROVEMENT items are quality-of-life follow-ups (documentation clarity, API consistency, code hygiene) that do not block production use but should be addressed alongside the BLOCKER/HIGH work where adjacent.

---

## Entry point for follow-up work

The primary artifact produced by this investigation is:

`specs/.current/AW-2865/tasks/README.md`

All 34 follow-up tasks are indexed there. Engineers picking up implementation work should:

1. Read `specs/.current/AW-2865/tasks/README.md` to select a task.
2. Open the corresponding task file (e.g., `specs/.current/AW-2865/tasks/01-ohttp-bypass-send-direct.md`) for a self-contained scope description.
3. Use the `Severity`, `Vector`, `Files`, and `Acceptance criteria` fields as the source of truth when opening a Jira ticket.
4. Treat the `Files:` line-range citations as hints to the current commit; verify line numbers before coding, as the source may drift.
