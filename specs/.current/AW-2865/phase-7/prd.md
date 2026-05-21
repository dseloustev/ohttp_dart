# AW-2865 Phase 7: Compile Findings into Per-Task Markdown Files

Status: PRD_READY

> Inherits shared context from `specs/.current/AW-2865/` (idea.md, vision.md).
> This PRD covers Phase 7 only: the deliverable creation phase.

---

## Context / Idea

AW-2865 is a read-only investigation of the `ohttp_dart` package to assess its readiness for production use in a non-custodial crypto wallet. Phases 1–6 completed audits across eight investigation vectors (cryptographic correctness, interoperability, parser robustness, privacy risks, network reliability, KeyConfig management, test suite, documentation).

Phase 7 is the **deliverable phase**: its sole output is a structured set of Markdown task files under `specs/.current/AW-2865/tasks/` that downstream engineers can pick up directly, plus a `README.md` index. No code is written or changed in this phase.

The Consolidated Findings Inventory (Phase 7 tasks.md §2) originally contained 35 rows. Rows 13 and 22 cover the same observability gap and are merged into one file, yielding **34 deduplicated findings**: 1 BLOCKER, 21 HIGH, and 12 IMPROVEMENT items (or a redistribution that totals 34), covering all eight investigation vectors. Task files are the final artifact of the AW-2865 investigation.

---

## Goals

1. Produce one Markdown task file per deduplicated finding (**34 files total**) under `specs/.current/AW-2865/tasks/`, each containing a standardized structure (heading, Severity, Vector, Files, Evidence, Description, Proposed change, Acceptance criteria).
2. Produce `specs/.current/AW-2865/tasks/README.md` with an executive summary that takes a **direct stance on production readiness**, a "Blockers for production use" section, an "All follow-up tasks" index table (34 rows), and an "Out of scope" section.
3. Ensure all eight investigation vectors from `idea.md` appear at least once across the task files.
4. Ensure at least one BLOCKER-severity task file exists (task 01: `ohttp-bypass-send-direct`).
5. Ensure no two task files cover the same finding (cross-reference IDs from the Consolidated Findings Inventory are used to detect duplicates).

---

## User Stories

**As a downstream engineer** picking up a follow-up task from the AW-2865 investigation, I want to open a single self-contained Markdown file that tells me exactly what the problem is, where in the code it lives, which RFC section is violated, and what "done" looks like — so I can begin work without needing to read through all six audit phases.

**As a tech lead** reviewing the investigation outcome, I want a `README.md` index that lists all identified issues with their severity and vector in one place — so I can prioritize which tasks to schedule first and identify blockers for wallet integration.

**As a security reviewer**, I want BLOCKER-severity findings to be visually separated and linked directly in the README — so I can immediately see what must be fixed before the package is used in a production wallet.

---

## Main Scenarios

### Scenario 1: Engineer picks up a follow-up task

An engineer opens `tasks/README.md`, reads the "Blockers for production use" section, clicks the link to `01-ohttp-bypass-send-direct.md`, and finds: severity label, the affected file and line range, the RFC reference, a 3-sentence problem description, a proposed change, and a bullet list of acceptance criteria. The engineer has everything needed to scope the work without further investigation.

### Scenario 2: Tech lead prioritizes the backlog

A tech lead opens `tasks/README.md`, reads the executive summary which states directly that the package must not be used in production until the 1 BLOCKER and 21 HIGH items are resolved, scans the "All follow-up tasks" table sorted by severity (BLOCKER → HIGH → IMPROVEMENT), and maps tasks to engineering sprints based on vector groupings.

### Scenario 3: Reviewer audits deliverable completeness

A reviewer runs `ls specs/.current/AW-2865/tasks/` and sees `README.md` plus exactly **34 task files**. The reviewer opens `README.md` and confirms the index table lists all 34 files, all eight vectors appear at least once across the task files, and at least one BLOCKER task file exists. The reviewer marks the acceptance criteria for tasks 7.3–7.6 as satisfied.

### Scenario 4: Duplicate-detection verification

Before finalizing, the author reviews the Consolidated Findings Inventory cross-reference IDs. Findings that share cross-refs are merged into a single file. Row 22 (`add-structured-observability-hooks` duplicate of row 13) is merged into task 13; both cross-reference IDs are cited inside the merged file. The final task file count is 34, not 35.

### Scenario 5: Severity escalation during file creation

While writing a task file, the author finds that an IMPROVEMENT-classified finding carries stronger security implications than the original classification reflects (for example, task 32 / `effectiveDirectBaseUrl` public getter enabling the bypass pattern). The author escalates the severity to HIGH or BLOCKER, documents the specific evidence that justifies the escalation in the task file's Evidence field, and updates the corresponding row in the README index table to reflect the new label. The escalation is self-contained and traceable within the task file itself.

---

## Success / Metrics

| Criterion | Verification |
|-----------|-------------|
| `tasks/` directory exists | `ls specs/.current/AW-2865/tasks/` returns non-empty listing |
| Exactly **34** task files present | File count matches deduplicated row count in Consolidated Findings Inventory §2 (rows 13 and 22 merged) |
| `README.md` exists | File present at `specs/.current/AW-2865/tasks/README.md` |
| Every task file has all required fields | Heading, Severity, Vector, Files, Evidence, Description, Proposed change, Acceptance criteria present in each file |
| All eight investigation vectors covered | Vector column in README index table contains each of the eight vectors at least once |
| At least one BLOCKER task file | Task 01 (`01-ohttp-bypass-send-direct.md`) is present and labeled `BLOCKER` |
| No duplicate findings across files | Cross-reference IDs in §2 are reconciled; rows 13 and 22 produce exactly one file; no two files describe the same finding |
| README index lists every task file | Row count in "All follow-up tasks" table equals **34** (file count in `tasks/` minus `README.md`) |
| README executive summary takes a direct production-readiness stance | Summary explicitly states the package must not be used in production until the 1 BLOCKER and 21 HIGH items are resolved |
| No code changed | `git diff lib/ test/ example/` is empty after Phase 7 work |
| BLOCKER tasks linked in README | "Blockers for production use" section contains at minimum a link to `01-ohttp-bypass-send-direct.md` |
| Severity escalations are justified in-file | Any task where severity differs from the Consolidated Findings Inventory classification includes an explicit Evidence rationale for the change |

---

## Constraints and Assumptions

1. **No code changes.** Phase 7 produces only Markdown files under `specs/.current/AW-2865/tasks/`. No edits to `lib/`, `test/`, or `example/`.
2. **No Jira ticket creation.** Task files are the final deliverable; Jira tickets are not created automatically from this phase.
3. **Findings are frozen.** The Consolidated Findings Inventory in `phase-7/tasks.md §2` is the authoritative source. No new findings are added in Phase 7; findings must trace back to Phases 1–6.
4. **Deduplication required — rows 13 and 22 are merged.** Rows 13 and 22 in the Consolidated Findings Inventory cover the same structured-observability gap. They produce exactly one task file (numbered 13); the file cites both cross-reference IDs. The final deliverable is **34 task files**, not 35.
5. **Ordering convention.** Task files are numbered sequentially with BLOCKER tasks first (01), HIGH tasks second (02–22), IMPROVEMENT tasks last (23–34). The numbering matches the Consolidated Findings Inventory §2 after the row-22 merge is applied.
6. **File naming.** Files are named `NN-<kebab-slug>.md` where `NN` is the two-digit sequence number and `<kebab-slug>` matches the `Slug` column in §2.
7. **Severity vocabulary.** Only three labels are valid: `BLOCKER`, `HIGH`, `IMPROVEMENT` — as defined in `vision.md §2`.
8. **Severity re-classification is permitted with justification.** The Phase 7 author may escalate (or downgrade) a severity label if evidence clearly warrants it. The task file must document the original classification from the Consolidated Findings Inventory, the new classification, and the specific evidence that justifies the change. Task 32 (`effectiveDirectBaseUrl` public getter) is a candidate for escalation.
9. **README executive summary is direct.** The summary must explicitly state that the package must not be used in production until the 1 BLOCKER and 21 HIGH items are resolved. Neutral or hedged language is not acceptable.
10. **Vector vocabulary.** Only the eight investigation vectors from `idea.md` are valid: Cryptographic correctness, Interoperability, Parser robustness, Privacy risks, Network reliability, KeyConfig management, Test suite, Documentation.
11. **Logging constraints from vision §7** apply to the description of task 13: any future logging must never log key material, inner request content, `targetAuthority`, response body/headers, or `gatewayBaseUrl` at DEBUG level in production.
12. **Phases 1–6 must be complete** before Phase 7 begins — this phase depends on all prior audit outputs.

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Findings from different phases partially overlap, creating near-duplicate task files | Medium | Downstream engineers waste time discovering they are working on the same fix | Use cross-reference ID column in Consolidated Findings Inventory to merge overlapping findings before creating files; tasks 7.4 verification step |
| A task file is missing a required field, making it ambiguous to act on | Low | Engineer must re-investigate to understand scope | Task 7.3 verification step: check all required fields are present before marking Phase 7 done |
| Not all eight investigation vectors appear in task files | Low | Investigation acceptance criteria not met; AW-2865 cannot be closed | Vector coverage map in §3 of tasks.md is pre-verified; task 7.5 verification step |
| README index table gets out of sync with actual task files | Medium | Tech lead makes incorrect prioritization decisions | Task 7.9 verification: README index row count must equal 34 |
| A severity escalation is made without adequate evidence, causing prioritization disputes | Low | Downstream team challenges the classification; re-work required | Constraint 8 requires explicit evidence rationale in-file for any classification change; reviewer checks this during verification |

---

## Resolved Questions

The following questions were raised during drafting and have been answered by the product owner.

1. **Row 22 merge decision** — RESOLVED: Rows 13 and 22 are merged into one task file. The final deliverable is 34 task files, not 35.

2. **Final task file count** — RESOLVED: 34 task entries in the README index table.

3. **README framing** — RESOLVED: The README executive summary takes a direct stance. It must state clearly that the package must not be used in production until the 1 BLOCKER and 21 HIGH items are resolved. Neutral or hedged language is not acceptable.

4. **Severity re-classification authority** — RESOLVED: Re-classification is allowed when justified. The Phase 7 author may escalate (or downgrade) severity if evidence clearly warrants it (for example, task 32 could be escalated). The task file must document the original classification and the specific evidence that justifies the change.
