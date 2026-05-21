# QA Report — AW-2865 Phase 7: Compile findings into per-task Markdown files

**Date:** 2026-05-21
**Reviewer:** QA agent
**Phase scope:** Create `specs/.current/AW-2865/tasks/` directory containing `README.md` and 34 per-task Markdown files synthesizing all findings from Phases 1–6. No source code was changed.

---

## Phase Scope

Phase 7 is a documentation-only deliverable. The outputs are the Markdown task files under `specs/.current/AW-2865/tasks/`. There is no executable code to test. All checks below are static (file existence, structural field presence, content correctness, cross-reference accuracy, path hygiene).

---

## Positive Scenarios

### PS-1: Directory and file set is complete and gap-free except for the documented gap at 22

**Expected:** `ls specs/.current/AW-2865/tasks/` returns `README.md` plus exactly the files `01-ohttp-bypass-send-direct.md` through `35-test-ohttp-edge-cases.md`, with file `22` absent.

**Result:** PASS. Glob returns 35 entries total: `README.md` plus files 01–21, 23–35. File 22 is absent. No duplicate filenames. No extra files. Count: 34 numbered task files + 1 README = 35.

---

### PS-2: README index table lists every task file

**Expected:** The "All follow-up tasks" table in `README.md` contains one row per numbered task file (01–21, 23–35) and no row numbered 22.

**Result:** PASS. The table in `README.md` has rows for 01–21, then row 32 (intentionally placed in severity order after the HIGH block), then rows 23–31, 33–35. All 34 numbered files are represented; row 22 is absent. The note at the bottom of the table explicitly states "Number 22 is intentionally absent — row 22 of the Phase 7 inventory was merged into task 13."

---

### PS-3: Every task file contains all required minimum fields

**Expected:** Each of the 34 numbered task files has a `# Task NN:` heading, `**Severity:**`, `**Vector:**`, `**Files:**`, `**Evidence:**`, `## Description`, `## Proposed change`, and `## Acceptance criteria` sections.

**Result:** PASS.
- `**Severity:**` present in all 34 files (confirmed by grep: 34 matches across 34 files).
- `**Vector:**` present in all 34 files (confirmed: 34 matches across 34 files, README excluded).
- `**Files:**` present in all 34 files (confirmed: 34 matches).
- `**Evidence:**` present in all 34 files (confirmed: 34 matches across 34 files).
- `## Description` present in all 34 files (confirmed: 34 matches).
- `## Proposed change` and `## Acceptance criteria` each present in all 34 files (confirmed: 68 total matches across 34 files, 2 per file).

---

### PS-4: At least one BLOCKER task file exists

**Expected:** At least one task file carries `**Severity:** BLOCKER`.

**Result:** PASS. Exactly one file — `01-ohttp-bypass-send-direct.md` — is labelled `BLOCKER`. All other files are labelled `HIGH` (21 files) or `IMPROVEMENT` (12 files).

---

### PS-5: All eight investigation vectors are covered

**Expected:** Every vector from `idea.md` appears in at least one task file's `**Vector:**` field.

**Result:** PASS. Coverage confirmed from grep output:

| Vector | Task files |
|---|---|
| Cryptographic correctness | 06, 23 |
| Interoperability | 07, 26 |
| Parser robustness | 11, 12, 24, 27 |
| Privacy risks | 01, 02, 03, 04, 05, 32 |
| Network reliability | 08, 10, 25 |
| KeyConfig management | 09 |
| Test suite | 14, 15, 16, 17, 18, 19, 20, 21, 30, 33, 34, 35 |
| Documentation | 13, 28, 29, 31 |

All eight vectors have at least one task file.

---

### PS-6: Task 01 is the BLOCKER

**Expected:** `01-ohttp-bypass-send-direct.md` carries `**Severity:** BLOCKER` and identifies `sendDirect()` / `effectiveDirectBaseUrl` as the finding.

**Result:** PASS. The file reads `**Severity:** BLOCKER`, `**Vector:** Privacy risks`, `**Files:** lib/src/ohttp_client.dart:52, 147, 148-171`. Description correctly identifies the silent null-to-gatewayBaseUrl fallback as the mechanism and labels it a privacy footgun with no runtime signal.

---

### PS-7: Task 32 documents IMPROVEMENT-to-HIGH escalation in Evidence field

**Expected:** `32-make-effective-direct-base-url-private.md` states an original classification of IMPROVEMENT and a new classification of HIGH with a rationale in the Evidence field.

**Result:** PASS. The Evidence field of task 32 contains:
- "Original classification: IMPROVEMENT."
- "New classification: HIGH."
- A multi-sentence escalation rationale referencing the public getter, IDE autocomplete discovery risk, and companion relationship to task 01.

---

### PS-8: Task 13 includes the §4 logging constraints verbatim

**Expected:** `13-add-structured-observability-hooks.md` reproduces the §4 logging constraints verbatim (both "never log" and "safe to log" blocks) as required by `tasks.md §4`.

**Result:** PASS. The Proposed change section of task 13 contains the heading "§4 Logging constraints — verbatim:" followed by the six "never log" items and three "safe to log" items matching `tasks.md §4` exactly. The Acceptance criteria also states "Doc comment on the observability interface reproduces the §4 constraints verbatim."

---

### PS-9: Gap at file 22 is intentional and documented in README

**Expected:** README explicitly acknowledges the absent file 22 and explains it was merged into task 13.

**Result:** PASS. A note immediately below the "All follow-up tasks" table reads: "Number 22 is intentionally absent — row 22 of the Phase 7 inventory was merged into task 13."

---

### PS-10: README executive summary takes a direct production stance

**Expected:** README states "must not be used in production" or equivalent direct stance, not hedged language.

**Result:** PASS. The executive summary line reads: "the library ... **must not be used in production until the 1 BLOCKER and 21 HIGH items below are resolved**." The stance is unambiguous and unhedged.

---

### PS-11: README IMPROVEMENT count is 12 (not 13)

**Expected:** The executive summary in README says "12 IMPROVEMENT items", consistent with the merged-22 deduplication.

**Result:** PASS. The summary reads "The 12 IMPROVEMENT items are quality-of-life follow-ups..." The inventory count is correct: files 23–31, 33, 34, 35 = 12 IMPROVEMENT items (row 32 was escalated to HIGH and counted there).

---

### PS-12: All task files use repo-relative paths; no absolute paths present

**Expected:** No task file or README contains `/Users/`, `/home/`, or `C:\` path prefixes.

**Result:** PASS. Grep across the entire `specs/.current/AW-2865/tasks/` directory for `/Users/`, `/home/`, and `C:\` returned no matches. All file citations use `lib/src/...`, `test/...`, or `example/...` repo-relative forms.

---

### PS-13: Severity ordering in directory matches tasks.md specification

**Expected:** BLOCKER tasks come first (01), then HIGH (02–21), then IMPROVEMENT (23–35).

**Result:** PASS with one intentional ordering note. Files 01–21 are BLOCKER/HIGH; files 23–35 are IMPROVEMENT. Task 32 is numbered 32 (originally an IMPROVEMENT slot) but carries `**Severity:** HIGH` — its number reflects its origin slot in the Phase 7 inventory before escalation, and the README table places it contiguously with the HIGH block. The tasks.md inventory documents this escalation explicitly. This is consistent and documented, not a defect.

---

### PS-14: No two task files cover the same finding (deduplication)

**Expected:** The merge of rows 13 and 22 into a single file (task 13) is the only deduplication event; no other duplicate finding exists.

**Result:** PASS. Cross-reference IDs across all 34 task files were verified against the `tasks.md §2` inventory. Each cross-ref ID (e.g., P6-1, C-4, TASK-O1) appears in at most one task file. The merged row-22 finding (T-X1 note / interoperability observability) is absorbed into task 13's Evidence field, which reads "Cross-refs P6-5, P6-6, P6-8, T-X1 (row 22 note)."

---

### PS-15: README "Blockers for production use" section lists only BLOCKER-severity tasks

**Expected:** The blockers section contains exactly one entry pointing to task 01.

**Result:** PASS. The section contains a single bullet for "Task 01 — OHTTP bypass via silent `effectiveDirectBaseUrl` fallback" with a relative link to `./01-ohttp-bypass-send-direct.md`.

---

### PS-16: README "Out of scope" section matches vision.md verbatim

**Expected:** The out-of-scope section reproduces the six items from `vision.md §Out of scope` with the attribution note.

**Result:** PASS. The section opens with "Verbatim from `vision.md §Out of scope`:" and lists all six items unchanged, matching `tasks.md §5`.

---

### PS-17: No source files were modified

**Expected:** No files under `lib/`, `test/`, or `example/` were created or modified.

**Result:** PASS. Git status shows only `specs/.current/AW-2865/tasks/` as untracked; no changes to `lib/`, `test/`, or `example/`.

---

## Negative and Edge Cases

### NC-1: File 22 does not exist (gap is intentional, not a missing file)

**Check:** Confirm that `22-add-structured-observability-hooks.md` does not exist and that no other file covers the row-22 scope independently of task 13.

**Result:** PASS. No file matching `22-*.md` exists. Row 22's scope (T-X1 note + interoperability observability) is documented in task 13's Evidence field and merged scope description.

---

### NC-2: README table does not claim 22 rows of BLOCKER + HIGH (arithmetic correctness)

**Check:** 1 BLOCKER + 21 HIGH = 22 entries before IMPROVEMENT; table should list tasks 01–21 plus task 32 in the HIGH block = 22 HIGH-or-BLOCKER rows, then 12 IMPROVEMENT rows.

**Result:** PASS. Counting the README table: rows 01–21 (21 rows) + row 32 = 22 BLOCKER/HIGH entries; rows 23–31, 33, 34, 35 = 12 IMPROVEMENT entries. Total = 34. Matches the 34 numbered task files present on disk.

---

### NC-3: Task 04 vector is Privacy risks, not Cryptographic correctness

**Check:** Zeroization of HPKE fields is classified as Privacy risks (not Cryptographic correctness), consistent with `tasks.md §2` and the eight-vector coverage map.

**Result:** PASS. `04-zeroize-hpke-key-material.md` carries `**Vector:** Privacy risks`. The `tasks.md §3` vector map lists tasks 04 and 05 under "Privacy risks" and tasks 06 and 23 under "Cryptographic correctness" — no cross-contamination.

---

### NC-4: Task 30 (fuzz tests) cites "all test files" — acceptable for Files field

**Check:** The template requires at least one file and one line range or RFC section. Task 30 uses "all test files" with no line range. Verify Evidence provides a concrete cross-ref and the description explains why no specific line is cited.

**Result:** ACCEPTABLE. The Evidence field cites cross-ref T-X2, and the description explains the coverage gap motivating the task. "All test files" is acceptable because the task adds new tests that span the entire suite rather than modifying a specific existing location. The Evidence field satisfies the concrete-scenario requirement.

---

### NC-5: Task 21 cites a non-existent directory as Files value

**Check:** `**Files:** new test/integration/ directory` does not exist on disk. This is expected behaviour for a task that proposes creating new infrastructure, but the QA should confirm it is not a stale reference to an existing location.

**Result:** ACCEPTABLE. The current repo has no `test/integration/` directory. The word "new" in the Files field is intentional; it signals a creation task, not a modification task. The description and acceptance criteria make clear this is a proposal to create the directory. No false reference.

---

### NC-6: README severity count in executive summary matches actual file counts

**Check:** Executive summary claims "1 BLOCKER", "21 HIGH", "12 IMPROVEMENT". Actual file counts: 1 BLOCKER (`01`), task 32 is in HIGH giving 21 HIGH tasks (02–21 + 32), 12 IMPROVEMENT (23–31, 33–35). Total = 34 files.

**Result:** PASS. Counts verified:
- BLOCKER: file 01 = 1.
- HIGH: files 02–21 (20 files) + file 32 (1 file) = 21.
- IMPROVEMENT: files 23–31 (9 files) + files 33–35 (3 files) = 12.
- Total: 1 + 21 + 12 = 34. Matches disk.

---

### NC-7: Task 05 (ohttp.dart) and task 04 (hpke.dart) do not duplicate each other

**Check:** Both address zeroization but at different layers; verify they cite different files and different cross-refs.

**Result:** PASS. Task 04 cites `lib/src/hpke.dart` (HpkeSenderContext fields, lines 258-263), cross-refs Phase 1 F-2, P6-3. Task 05 cites `lib/src/ohttp.dart:105-120, 196-207`, cross-refs TASK-O6, P6-4. Distinct files, distinct lines, distinct cross-refs.

---

## Automated Tests Coverage

Phase 7 produces no executable code; there is nothing to unit-test or integrate-test in this phase. All verification is static document analysis. No automated test suite applies.

The deliverable itself (the task files) is the input for future automated regression — specifically:

- The acceptance criteria fields in each task file are written as verifiable conditions suitable for future CI checks once the corresponding implementation tasks are picked up.
- Task 30 (`30-add-fuzz-property-based-tests.md`) explicitly specifies a CI-seed reproducibility requirement for future automated runs.
- Task 21 (`21-test-integration-gateway-stub.md`) requires future integration tests to be hermetic (no live network in CI).

---

## Manual Checks Needed

All checks for this phase are manual (file inspection). The following were performed and are confirmed:

1. Full file listing of `specs/.current/AW-2865/tasks/` — checked (35 files, correct names, no duplicates, gap at 22).
2. Field completeness scan across all 34 task files — checked (all 8 required fields present in every file).
3. README table row-count vs. file count cross-check — checked (34 rows = 34 files).
4. Eight-vector coverage map from `tasks.md §3` vs. actual `**Vector:**` values — checked (all 8 covered).
5. BLOCKER count and identity — checked (exactly 1, task 01).
6. IMPROVEMENT count in executive summary vs. actual count — checked (12 both ways).
7. Task 32 escalation Evidence content — read and confirmed.
8. Task 13 §4 logging constraints verbatim presence — read and confirmed.
9. README direct-stance language — read and confirmed ("must not be used in production").
10. README gap-22 note — read and confirmed.
11. Absolute path check across entire `tasks/` directory — grep confirmed 0 matches.
12. Out-of-scope section vs. `vision.md §Out of scope` / `tasks.md §5` — checked, matches.
13. No files created or modified in `lib/`, `test/`, `example/` — confirmed via git status.

---

## Phase-Specific Risk Zone

**Risk 1 (LOW): Task 32 numbering may confuse readers.** Task 32 is numbered at its original IMPROVEMENT slot but carries HIGH severity. The README table does not place it contiguously with tasks 01–21 in numerical order — it appears at position 22 of the table (after task 21, before task 23), which is correct by severity but jarring by number. Mitigation: the Evidence field explains the escalation; the gap-22 note explains the numbering. Engineers reading the file will see the HIGH label immediately. Risk is low; no action required before production use.

**Risk 2 (LOW): Task 30 "Files: all test files" is less actionable than a specific location.** An engineer picking up task 30 must infer which test files to add the fuzz layer to. The description and acceptance criteria are specific enough to compensate. Risk is low.

**Risk 3 (LOW): Task 13 merged scope (rows 13 + 22) may be larger than a single sprint.** The merged task covers both Documentation and Interoperability vectors and proposes a breaking API change (replacing `onLog`). Engineers should consider splitting implementation into two PRs. This is a planning risk, not a deliverable defect.

**Risk 4 (NONE): No source code risk.** No `lib/`, `test/`, or `example/` file was modified. There is no regression risk to the running library.

---

## Final Verdict

**RELEASE**

All acceptance criteria from `specs/.current/AW-2865/phase-7/tasks.md` are met:

- (a) README index table lists all 34 task files — PASS.
- (b) Every task file has a severity label and a file/line citation — PASS.
- (c) All eight investigation vectors appear at least once — PASS.
- (d) At least one BLOCKER task file exists (task 01) — PASS.
- (e) `specs/.current/AW-2865/tasks/` contains `README.md` plus one file per task with no duplicates — PASS.

All additional checks specified in the QA brief are met:

- Task 01 is BLOCKER — PASS.
- Task 32 documents IMPROVEMENT-to-HIGH escalation in Evidence — PASS.
- Task 13 includes §4 logging constraints verbatim — PASS.
- Gap at file 22 is intentional and documented in README — PASS.
- README executive summary takes direct production stance — PASS.
- README IMPROVEMENT count is "12" (not "13") — PASS.
- No absolute paths in any task file or README — PASS.

Three low-severity observations are noted in the Risk Zone section; none blocks release of the Phase 7 deliverable.
