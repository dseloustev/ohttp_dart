# AW-2865 Phase 7 — Plan

Status: PLAN_APPROVED

## Phase Scope

Phase 7 is the deliverable-creation phase of the AW-2865 investigation. The only artefact produced is
a structured set of Markdown files under `specs/.current/AW-2865/tasks/`. No code under `lib/`,
`test/`, or `example/` is changed.

The plan operationalises the Consolidated Findings Inventory in `specs/.current/AW-2865/phase-7/tasks.md §2`
(authoritative source, frozen) into:

- **34 per-task Markdown files**, one per deduplicated finding.
- **1 README.md** index file with an executive summary, blockers section, full task index table, and an
  out-of-scope section.

Row composition after deduplication:

- 1 BLOCKER  (task 01)
- 20 HIGH    (tasks 02–21; task 22 is intentionally absent — row 22 is merged into row 13)
- 13 IMPROVEMENT (tasks 23–35)

Total: **34 files** numbered `01..21, 23..35`. The gap at `22` is deliberate and is preserved in both
the filesystem and the README index. Numbering matches §2 of `phase-7/tasks.md`; no renumbering.

Prior phases (1–6) are prerequisites and are treated as frozen inputs.

---

## Components

The deliverable comprises three logical components, all under `specs/.current/AW-2865/tasks/`.

### C1. The `tasks/` directory

- Path: `specs/.current/AW-2865/tasks/` — a **sibling** of the `phase-N/` folders, not nested inside
  `phase-7/`.
- Contents after Phase 7: `README.md` + 34 task files.
- Naming: `NN-<kebab-slug>.md` where `NN` is the two-digit number from §2 and `<kebab-slug>` matches
  the Slug column.

### C2. Per-task Markdown files (34)

Each file is independently actionable and self-contained. A downstream engineer must be able to scope
the work from one file without re-reading the audit phases.

Required structure (per `phase-7/tasks.md §1`):

```markdown
# Task NN: <Title>

**Severity:** BLOCKER | HIGH | IMPROVEMENT
**Vector:** <one of the eight investigation vectors>
**Files:** `<repo-relative-path:line-range>` (one or more)

**Evidence:** <RFC section, line reference, or concrete malformed-input scenario>

## Description

2–5 sentences explaining the problem.

## Proposed change

Intent and high-level approach (no code).

## Acceptance criteria

- Bullet list of verifiable conditions.
```

Additional required content for selected files:

- **Task 01** (`01-ohttp-bypass-send-direct.md`): must call out the **BLOCKER** classification, link
  to task 32 as the companion API-surface change, and include the missing-`@Deprecated`/missing-warning
  observation at `lib/src/ohttp_client.dart:147` in addition to the body range `:148–171`.

- **Task 02** (`02-enforce-https-scheme.md`): `Files:` should cite
  `lib/src/ohttp_client.dart:43–51, 52, 82, 116`. Adding `:52` is a research-recommended refinement
  (the `effectiveDirectBaseUrl` getter is the companion location).

- **Task 13** (`13-add-structured-observability-hooks.md`): the **merged** file for the original §2
  rows 13 and 22. Must cite cross-references `P6-5, P6-6, P6-8, T-X1 (row 22 note)`. Must include the
  full §4 logging constraints from `phase-7/tasks.md` verbatim (or referenced) as a hard requirement
  on the proposed observability hook.

- **Task 21** (`21-test-integration-gateway-stub.md`): must note that `test/integration/` does not
  exist yet and propose a stub strategy (e.g., local `package:shelf` server or recorded HTTP fixture)
  so CI does not depend on a real gateway.

- **Task 32** (`32-make-effective-direct-base-url-private.md`): **escalated from IMPROVEMENT to HIGH**.
  The file must record:
  - Original classification: IMPROVEMENT
  - New classification: HIGH
  - Evidence rationale: public getter widens the discoverable surface of the BLOCKER task 01
    footgun; getter is reachable from any consumer via `OhttpGatewayConfig`; companion (not duplicate)
    to task 01 because it changes API surface rather than adding a runtime warning.
  - The file remains numbered `32` (no renumbering after escalation).

  Note: Even though task 32 is escalated to HIGH, **it stays in the IMPROVEMENT-numbered block (32)**
  per the "no renumbering" rule in the resolved research questions. The README index reflects the new
  severity (HIGH) but the file number is preserved.

### C3. `README.md` index

Required sections (per `phase-7/tasks.md §7.7–7.10`):

1. **Top-level heading** — `# AW-2865: ohttp_dart investigation — follow-up tasks`.

2. **Executive summary (3–5 sentences)** — must take a **direct stance on production readiness**.
   Required phrasing in spirit: the package must not be used in production until the 1 BLOCKER and
   the HIGH items are resolved. (After the task-32 escalation, the HIGH count in the index is 21
   despite only 20 HIGH-numbered file slots — see "Severity-count accounting" below.) Neutral or
   hedged language is **not** acceptable.

3. **Blockers for production use** — bullet list. Currently contains exactly one entry linking to
   `01-ohttp-bypass-send-direct.md` with a one-line summary.

4. **All follow-up tasks** — a table with columns `# | Task title | Vector | Severity | File link`.
   The table contains **34 rows**, one per task file. Sort order: BLOCKER first, then HIGH (by file
   number ascending, including escalated row 32), then IMPROVEMENT (by file number ascending). Task
   numbers in the `#` column match the file numbers (01–21, 23–35); the table does not include a row
   for the missing number 22.

5. **Out of scope** — verbatim copy of the six bullets from `vision.md §Out of scope` (also reproduced
   in `phase-7/tasks.md §5`).

6. **Deliverable note** — short paragraph stating that these `.md` files are the final deliverable;
   no Jira tickets are created automatically; engineers picking up follow-up work should reference
   these files directly.

#### Severity-count accounting in the README

Two counts coexist after the task-32 escalation:

- **By file numbering** (slot-based, used for filenames and table sort): 1 BLOCKER, 20 HIGH
  (numbers 02–21), 13 IMPROVEMENT (numbers 23–35).
- **By effective severity label** (used for the executive summary and "blockers" copy): 1 BLOCKER,
  **21 HIGH** (numbers 02–21 plus escalated 32), 12 IMPROVEMENT (numbers 23–31, 33–35).

The README executive summary cites the **effective-severity** counts ("1 BLOCKER and 21 HIGH"),
because that is what tech leads use for prioritisation. The "All follow-up tasks" table's `Severity`
column also uses the effective label (HIGH for task 32). Both counts and the rationale are documented
inside the task 32 file itself; the README does not need to repeat the escalation rationale beyond
showing HIGH in the table.

---

## API Contract

This phase exposes a **filesystem contract**, not a code API. Downstream tooling and reviewers depend
on the following invariants.

### Filesystem contract

- Directory `specs/.current/AW-2865/tasks/` exists and contains exactly:
  - `README.md`
  - 34 files matching the glob `[0-9][0-9]-*.md`
- File numbers present: `01, 02, 03, 04, 05, 06, 07, 08, 09, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19,
  20, 21, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35`. The number `22` is **not** used.
- Every task file ends in `.md` and is UTF-8 plain text.

### Per-file content contract

Every task file contains these eight required elements (verified during task 7.3):

1. `# Task NN: <Title>` heading where `NN` matches the filename prefix.
2. `**Severity:**` line with exactly one of `BLOCKER`, `HIGH`, `IMPROVEMENT`.
3. `**Vector:**` line with exactly one of the eight investigation vectors verbatim:
   `Cryptographic correctness`, `Interoperability`, `Parser robustness`, `Privacy risks`,
   `Network reliability`, `KeyConfig management`, `Test suite`, `Documentation`.
4. `**Files:**` line with at least one repo-relative path (no `/Users/...`, no `C:\...`). Paths cite
   line ranges where the §2 inventory provides them.
5. `**Evidence:**` field with an RFC section, a concrete line reference, or a malformed-input
   scenario.
6. `## Description` section with 2–5 sentences.
7. `## Proposed change` section describing intent (no code).
8. `## Acceptance criteria` section with a bullet list of verifiable conditions.

For task 32, the Evidence field additionally contains the escalation rationale (original and new
classification + evidence).

### README contract

- Contains the four required sections (Executive summary, Blockers, All follow-up tasks, Out of
  scope) plus the deliverable note.
- Index table has exactly 34 data rows.
- Every row's `File link` column is a relative markdown link (e.g. `[01-ohttp-bypass-send-direct.md](01-ohttp-bypass-send-direct.md)`)
  to a file that exists in the same directory.
- All eight investigation vectors appear at least once in the `Vector` column.
- At least one row has `Severity = BLOCKER`.
- The "Blockers for production use" section contains a link to `01-ohttp-bypass-send-direct.md`.

---

## Data Flows

### Authoring flow

```
phase-7/tasks.md §2  ─┐
phase-7/research.md  ─┼─►  per-task .md files (34)  ─►  tasks/README.md
idea.md, vision.md   ─┘
```

1. Read `phase-7/tasks.md §2` (authoritative findings inventory) row by row.
2. For each row, read the corresponding draft-finding source from prior phases (P6-*, C-*, F2.*, T-*,
   TASK-O*) for narrative context. The cross-reference column in §2 lists the source IDs.
3. For each row that requires it, consult `phase-7/research.md` for:
   - Verified line numbers (Citation Verification section).
   - Task 32 escalation evidence.
   - Logging constraints for task 13.
   - Open implementation notes (e.g. task 21 `package:shelf` suggestion).
4. Emit one task file per row using the template in `phase-7/tasks.md §1`.
5. After all 34 files exist, emit `README.md` with the index table populated from the same row data.

### Verification flow

After authoring, perform the checks specified in `phase-7/tasks.md §Acceptance Criteria` and the
PRD `Success / Metrics` table:

1. `ls specs/.current/AW-2865/tasks/` → returns `README.md` plus 34 task files (no file at slot 22).
2. For each task file, confirm the eight required content elements are present.
3. Confirm every vector in the eight-vector list appears at least once in the README's `Vector`
   column.
4. Confirm at least one row in the README has `Severity = BLOCKER` and that the blockers section
   links to it.
5. Confirm the README index has exactly 34 data rows.
6. Confirm no two task files cover the same finding (the §2 cross-reference column was the input;
   row 22 was merged into row 13 during authoring).
7. Confirm `git diff lib/ test/ example/` is empty.
8. For task 32 specifically, confirm the Evidence field documents the escalation.

---

## NFR (Non-Functional Requirements)

| NFR | Target | Verification |
|-----|--------|--------------|
| Self-containedness | Every task file scopable without reading other phases | Acceptance-criteria checklist |
| Path correctness | All paths in task files and README are repo-relative | Grep for absolute-path prefixes (`/Users/`, `/home/`, `C:\`) |
| Vocabulary correctness | Severity ∈ {BLOCKER, HIGH, IMPROVEMENT}; Vector ∈ the 8 listed | Manual review during 7.3 |
| Citation accuracy | Line numbers in `Files:` fields match current source | `phase-7/research.md` Citation Verification table pre-validates all citations |
| Determinism | Re-running the authoring process yields the same 34 files with the same names | Filename derived from §2 Slug column |
| Cross-vector coverage | All 8 vectors represented in the README index | Vector coverage map in `phase-7/tasks.md §3` |
| Production-readiness clarity | README executive summary takes a direct stance, not hedged | Manual review against PRD constraint 9 |
| Escalation traceability | Severity changes are justified in-file | Task 32 file Evidence field includes the rationale |
| No-code-change invariant | `lib/`, `test/`, `example/` are untouched | `git diff` is empty for these paths |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| File count drifts from 34 (e.g. an authoring slip duplicates or omits a finding) | Low | README and filesystem disagree; reviewer rejects deliverable | Final verification step (7.3 / 7.9) counts files and matches against §2 row count after the row-22 merge |
| Row 22 is accidentally produced as a separate file | Low | Duplicate observability task; engineer wastes effort | Authoring rule documented in this plan: merge row 22 into row 13; both cross-references cited in the single merged file. Verified at 7.4 |
| Numbering re-base shifts file numbers after the gap at 22 | Medium | Cross-references in §2 become stale | Resolved research question 3 fixes the rule: keep original numbers; gap at 22 stays |
| Task 32 file lacks rationale despite escalation | Low | Reviewer disputes the classification change; rework required | PRD constraint 8 and this plan require the rationale block inside the Evidence field |
| Vector typos cause the 8-vector verification to fail silently | Low | Acceptance criterion (c) misreported as passing | Use the exact verbatim strings from `idea.md`; the verification step compares against the list, not against a tolerant pattern |
| Line numbers cited in task files go stale before the file is actioned | Medium | Engineer chases a moved line | `phase-7/research.md` Citation Verification table records line numbers as of the current commit; task files can also cite the symbol name (`HpkeSenderContext.key`, `OhttpGatewayConfig.effectiveDirectBaseUrl`) as a stable fallback |
| README executive summary uses neutral/hedged language | Low | PRD constraint 9 violated; reviewer requests rewrite | Explicit instruction in this plan that the summary must state the package must not be used in production until 1 BLOCKER and 21 HIGH items are resolved |
| Severity counts in README (1 BLOCKER, 21 HIGH, 12 IMPROVEMENT after task 32 escalation) contradict the file-number block (1 BLOCKER, 20 HIGH, 13 IMPROVEMENT) | Medium | Reader confusion | Explicit accounting section in this plan (see C3 above): file-number counts are slot-based; README copy uses effective severity |
| Authoring agent regenerates a flat `plan.md` or `tasks/` somewhere unexpected | Low | Artefacts in wrong location | This plan and the ticket-parsing rules specify `specs/.current/AW-2865/tasks/` as a sibling of `phase-N/` folders |

---

## Dependencies

### Required upstream artefacts (must exist and be complete)

- `specs/.current/AW-2865/idea.md` — vector list, severity vocabulary.
- `specs/.current/AW-2865/vision.md` — severity definitions, out-of-scope list (used verbatim in
  README §Out of scope).
- `specs/.current/AW-2865/phase-7/prd.md` — Phase 7 requirements (this plan implements it).
- `specs/.current/AW-2865/phase-7/tasks.md` §2 — authoritative findings inventory, §3 vector
  coverage map, §4 logging constraints, §5 out-of-scope copy source.
- `specs/.current/AW-2865/phase-7/research.md` — citation verification, task 32 escalation evidence,
  task 21 integration-test scaffolding notes.
- Phase 1–6 task drafts (`phase-1/tasks.md` through `phase-6/tasks.md`) — narrative context for
  individual findings via the cross-reference IDs in §2.

### Required source files (read-only)

- `lib/src/ohttp_client.dart`
- `lib/src/ohttp.dart`
- `lib/src/hpke.dart`
- `lib/src/bhttp.dart`
- `test/hpke_test.dart`, `test/bhttp_test.dart`, `test/ohttp_test.dart`
- `example/ohttp_dart_example.dart`

These are read only to confirm citations and quote brief evidence; they are never edited.

### No downstream dependencies during this phase

Jira ticket creation, code changes, and CI changes are explicitly out of scope. Downstream
consumption of the `.md` files happens after Phase 7 closes.

---

## Open Questions

None. All product-level questions from `phase-7/prd.md §Resolved Questions` are resolved, and the
research-level questions in `phase-7/research.md §Resolved Questions` are answered. The remaining
"New Technical Questions" in `phase-7/research.md` are forward-looking notes for the downstream
implementation phase (when the actual fixes are picked up), not blockers for Phase 7 authoring.
