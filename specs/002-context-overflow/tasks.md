# Tasks: Reviews that fit, or say why they can't

**Input**: Design documents from `specs/002-context-overflow/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md),
[research.md](research.md), [data-model.md](data-model.md),
[contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: The spec asks for no automated tests. The repository has no
test code: a change is checked by `markdownlint`, the CI workflows (Agent
Version Check, License Header, DCO), `guild integration version test`,
and the live scenarios in [quickstart.md](quickstart.md) sections 2, 4
and 5. Those checks are the validation tasks below.

**Organization**: One phase per user story, in priority order: US2 (P1),
then US1 and US3 (P2). Almost every story task edits `PROMPT.md`, so
stories run one after another, not side by side.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different file, no open dependency).
- **[Story]**: The user story the task serves (US1 to US3).
- **(maintainer)**: The maintainer runs it. It uses the Guild credential
  or changes live behavior. Claude gives the full command, with IDs
  filled in, in chat only.

## Path conventions

Repository root: `PROMPT.md`, `guild.yaml`, `README.md`, `CLAUDE.md`,
`integration/lfx-one-github.openapi.yaml`. No IDs or repository lists
go into any of them.

---

## Phase 1: Setup

- [x] T001 (maintainer) Run the enum probe in quickstart step 1 and
  record it in `specs/002-context-overflow/research.md` R5. Done
  2026-11-03: GitHub answered 422, so Guild does not enforce body enums
  and `graphql_read` is out.
- [ ] T002 [P] (maintainer) Ask Guild (Rod Brathwaite) two questions from
  `specs/002-context-overflow/research.md` R6 and R12: does Guild trim an
  operation's response to its output schema, and can a credential policy
  or Guild itself enforce request parameter values. Record the answers in
  research R6 and R12.

---

## Phase 2: Foundational (blocks every story)

Integration 1.0.6 and the shared prompt sections.

- [x] T003 Add operation `git_get_tree` to
  `integration/lfx-one-github.openapi.yaml`: `GET
  /repos/{owner}/{repo}/git/trees/{tree_sha}`, parameters `owner`,
  `repo` and `tree_sha` (path, string; described as `<commit>:<directory>`
  or a commit SHA for the root), response `Ok`. No `recursive` parameter
  ([contracts/integration-ops.md](contracts/integration-ops.md)).
- [x] T004 Add an optional header parameter `Accept` (`in: header`,
  `required: false`, `schema: { type: string, enum:
  ["application/vnd.github.raw+json"] }`) to `repos_get_content` in
  `integration/lfx-one-github.openapi.yaml`. Update its summary to say
  the header returns the file's text. No new operation: OpenAPI allows one
  GET per path (research R4).
- [x] T005 Change `issues_remove_label` in
  `integration/lfx-one-github.openapi.yaml` to the path
  `/repos/{owner}/{repo}/issues/{issue_number}/labels/lfx-review-agent-pending`
  and remove the `name` parameter. Keep the operation id, summary and
  responses (200 `OkList`, 404 `Ok`) (research R12).
- [x] T006 Add output schemas to `pulls_list_reviews` and
  `pulls_list_review_comments` in
  `integration/lfx-one-github.openapi.yaml` with exactly the fields in
  [contracts/integration-ops.md](contracts/integration-ops.md),
  "Narrowed output schemas". Before writing them, grep `PROMPT.md` for
  every field read from these two responses and add any the contract
  misses.
- [x] T007 Update the header comment in
  `integration/lfx-one-github.openapi.yaml` (operation count and what was
  added) and set `info.version` to `1.0.6`.
- [x] T008 [P] In `guild.yaml`, pin the integration to 1.0.6 and add the
  tool `lfx_one_github_git_get_tree`.
- [x] T009 In `PROMPT.md` "Tools", add rows for `git_get_tree` ("A
  directory's entries at a commit: `path`, `type`, `mode`, `size`, `sha`;
  `tree_sha` is `<commit>:<directory>`, or the commit SHA for the root"),
  change the file-content row to show the `Accept` header, and change the
  remove-label row to drop `name`.
  Add one sentence: never call `repos_get_content` on a directory.
- [x] T010 In `PROMPT.md`, add a section "Read budget" after "Pending
  label", from [contracts/read-budget.md](contracts/read-budget.md) and
  [data-model.md](data-model.md): the limits table with the values
  verbatim (`max_file_bytes` 100,000; `max_run_bytes` 300,000; rule-file
  caps 24,000 and 80,000 count toward `max_run_bytes`), the "Before any
  full-file read" rule, the unread-file kinds `changed-no-patch`,
  `changed-with-patch` and `context`, and the reasons `over the
  100,000-byte limit`, `over the 300,000-byte run total`, `size unknown`.
  Set the order: patch-less changed files (Step 3.2) may use up to
  220,000 bytes, leaving 80,000 for rule files; context reads use what is
  left. A file read in Step 3.2 is reused in Step 4 and counted once. A
  failed tree listing means `size unknown`, which withholds approval for
  a patch-less changed file.
- [x] T011 In `PROMPT.md`, add a section "Generated files and lockfiles"
  after "Review exclusions", with the research R9 table verbatim and the
  rule: a listed file that is not read is named "generated or lockfile,
  not reviewed line by line" and never withholds approval. Say that
  maintainers add a row to extend it.

**Checkpoint**: integration spec and shared prompt sections ready.

---

## Phase 3: User Story 2 - One huge file doesn't sink it (Priority: P1)

**Goal**: No full-file read without a size, no read over the limits, and
an unread patch-less changed file withholds approval (spec FR-006 to
FR-008, FR-013 to FR-015).

**Independent Test**: quickstart section 4 rows for the lockfile, the
patch-less file over 100,000 bytes, the file with a patch, the context
read and rule-file discovery.

- [x] T012 [US2] In `PROMPT.md` Step 4 "Repository rules", replace the
  `repos_get_content` directory listings with `git_get_tree` at
  `base_sha`: the root as `tree_sha: <base_sha>`, `.claude/rules/` as
  `<base_sha>:.claude/rules`, and each top-level directory as
  `<base_sha>:<dir>`. Map the entry fields: read only `type: "blob"`
  entries whose `mode` is not `120000`; mode `120000` is a symlink and
  keeps the existing skip-and-name rule. Drop the `path: "."` note.
  Rule-file reads add their sizes to `bytes_read`.
- [x] T013 [US2] In `PROMPT.md` Step 4, rewrite the "Large files" and
  "Context" bullets to follow the "Read budget" section: list the
  file's directory with `git_get_tree` at the same ref (reuse listings),
  check the limits, then read; otherwise record the unread file with its
  kind and reason. A changed file with a patch is reviewed from its
  patch; only its whole-file read is skipped.
- [x] T014 [US2] In `PROMPT.md` Step 3.2, replace "read it with
  `lfx_one_github_repos_get_content` at both the merge base and
  `head_sha`" with: read it at `head_sha` only, within the read budget.
  Add the research R10 rule for a file left unread: a file on the
  generated-file and lockfile table is never a match; any other is a
  match only if its path points at the excluded area (for
  `meeting-ics`: a name part equals `ics`, `ical` or `calendar`,
  case-insensitive, where name parts are the path split on `/`, `.`, `-`
  and `_`; so `ics.helper.ts` matches and `analytics.ts` does not).
  Keep "when unsure, treat it as a match" for files that were read.
- [x] T015 [US2] In `PROMPT.md`, add "### 5a.3. Unread changed files"
  after 5a.2: run when the verdict would be `APPROVE`. If any unread file
  has kind `changed-no-patch` and is not on the generated-file table,
  withhold approval with the approval-withheld marker and the note from
  [contracts/read-budget.md](contracts/read-budget.md) verbatim. Check
  the whole-PR file list from Step 3.2 on every review, follow-ups
  included: a file the PR still changes stays unread on later rounds, so
  the PR keeps a withheld approval until the file leaves the diff or a
  human approves. A `REQUEST_CHANGES` verdict stands. A deleted file is
  never `changed-no-patch`.
- [x] T016 [US2] In `PROMPT.md`, extend every place that lists the
  approval-withholding gates to include 5a.3: Step 2 terminal reviews
  ("posted by Step 5a.1 or 5a.2"), Step 5c `event` (`COMMENT` only when
  5a.1, 5a.2 or 5a.3 withheld approval) and the review body's second
  line notes, and the Step 7 "Approval withheld" report line.
- [x] T017 [US2] In `PROMPT.md` Step 6, add the "Unread files" recap line
  from [contracts/read-budget.md](contracts/read-budget.md) before the
  rule-files line (deleted large files are listed as `deleted`), and
  name withholding files at the top of the summary when 5a.3 withheld
  approval. Update the recap example.
- [x] T018 [US2] In `PROMPT.md` "Error handling", add `git_get_tree` and
  full-file reads to the list of reads that never stop a run: a failed
  listing means the size is unknown, so the file is not read (reason
  `size unknown`).

**Checkpoint**: Story 2 complete. The four single-file overflows in
research R1 cannot recur.

---

## Phase 4: User Story 1 - Too-large PR gets a notice (Priority: P2)

**Goal**: Gate A1 declines a PR over the limits with one notice and no
start comment (spec FR-001 to FR-005).

**Independent Test**: quickstart section 4 rows for a PR over 20,000
changed lines and one that drops back under.

- [x] T019 [US1] In `PROMPT.md` "Comment classes", add class
  `too-large-notice` (marker `<!-- lfx-review-agent:too-large -->`)
  after `stalled-notice` and before `abandoned`; renumber the later
  classes and fix the gate C reference to the summary class number.
  Add it to the list of notices that do not count as attempts.
- [x] T020 [US1] In `PROMPT.md` "Apply gates", add gate A1 after gate A
  from [contracts/too-large-notice.md](contracts/too-large-notice.md):
  limits `max_changed_lines` 20,000 (`additions + deletions`),
  `max_changed_files` 300, `max_review_comments` 150, missing fields
  count as 0; the notice text verbatim; once per PR while a
  `too-large-notice` is newer than the most recent `session-start`,
  `session-summary` or `abandoned` comment. Add the limits to the "Read
  budget" table. Add gate A1 to the approval guard's gate list.
- [x] T021 [US1] In `PROMPT.md`, gate C: a `too-large-notice` does not
  debounce (go to gate D). "Duplicate notice check": add type
  `too-large` (every earlier `too-large-notice` newer than the most
  recent session comment is a duplicate) and the superseded marker
  `<!-- lfx-review-agent:superseded notice=too-large -->`. "Skip notice":
  add a gate A1 row.
- [x] T022 [US1] In `PROMPT.md` "Pending label" and Step 0.3: a sweep
  removes the label at gate A1; Step 0.3 item 4 allows the too-large
  notice beside the stalled notice; the "A sweep follows every later
  step" paragraph says a sweep may post these two notices.
- [x] T023 [US1] In `PROMPT.md` Step 7, add `Review not started: PR too
  large (<reason>)` with a link to the notice.

**Checkpoint**: Story 1 complete.

---

## Phase 5: User Story 3 - Busy PRs fit in one run (Priority: P2)

**Goal**: File contents as plain text; history trimmed if Guild trims
to output schemas (spec FR-009, FR-010).

**Independent Test**: quickstart step 2 (the header returns Markdown) and
step 5 (tool output and tokens on a large follow-up).

- [x] T024 [US3] In `PROMPT.md`, make every full-file read (rule files,
  patch-less changed files, context) pass `Accept:
  "application/vnd.github.raw+json"` to `repos_get_content` (T004), with
  sizes still taken from `git_get_tree`. Say the response is the file's
  text, not JSON.
- [x] T025 [US3] In `PROMPT.md` "Tools", note that the history reads
  may return only the fields listed in the table (output schema). The
  field check itself is part of T006.

**Checkpoint**: Story 3 complete, pending quickstart step 2.

---

## Phase 6: Polish, docs and rollout

- [x] T026 [P] Update `README.md`: credential policy (the full ALLOW
  list gains `git_get_tree`; a new `policy create` example for it; note
  that `issues_remove_label` now removes only the pending label), and test
  checklist items for
  quickstart section 4.
- [x] T027 [P] Update `CLAUDE.md`: add `:too-large` to the read-back
  markers, and next to the review-exclusions note, say how to extend the
  generated-file and lockfile table.
- [x] T028 Bump "Agent version" in `PROMPT.md` to `0.1.20` (main took
  0.1.19 for PR #36).
- [x] T029 Run `markdownlint` from the repository root on every changed
  `.md` file. Re-read the `PROMPT.md` diff for unslop rules and for `{{ }}`
  text other than `{{env.KEY}}`.
- [ ] T030 Open the PR from `feature/context-overflow` (label
  `ai-assisted`, assignee `dealako`, body ends with the Claude Code line).
  Confirm Agent Version Check, License Header and DCO pass.
- [ ] T031 (maintainer) Publish integration 1.0.6 from this branch's spec
  (quickstart section 3) and check the published version lists every
  operation.
- [ ] T032 (maintainer) Create the ALLOW policy for `git_get_tree`.
- [ ] T033 (maintainer) Run quickstart step 2. If the header read returns
  JSON, revert T024 (stop passing the header) before merging. Record results
  in research R4, R6 and R12.
- [ ] T034 (maintainer) Merge the PR, then run `guild agent save
  --publish --version-number 0.1.20 -m "..."` from an updated `main`.
- [ ] T035 Run quickstart step 5 on the first 200 reviews under 0.1.20,
  record the results in research R7, and update the memory file with the
  live state (agent 0.1.20, integration 1.0.6). Compare one follow-up's
  prior-feedback and bot reconciliation on the same PR before and after
  1.0.6 (spec US3 scenario 3). Time one too-large notice against its
  event (SC-002). If T002 confirms trimming, raise
  `max_review_comments` per research R8.

---

## Dependencies and execution order

### Phases

- **Setup (Phase 1)**: T001 done. T002 runs any time; its answers feed
  T025 and T033.
- **Foundational (Phase 2)**: T003 to T007 edit one file, in order. T008
  beside them. T009 to T011 edit `PROMPT.md`, in order.
- **User stories (Phases 3 to 5)**: all wait for Phase 2 and edit
  `PROMPT.md`, so do them in order: US2, US1, US3.
- **Polish (Phase 6)**: T026 and T027 can start after Phase 2. T028 and
  T029 after the stories. T030 to T035 in order.

### Stories

- **US2 (P1)**: needs only Phase 2. Fixes 4 of the 5 observed overflows.
- **US1 (P2)**: independent of US2. A backstop; caught none of the 5.
- **US3 (P2)**: builds on the read budget from US2 (sizes before reads).

### Parallel opportunities

- T008 (`guild.yaml`) beside T003 to T007.
- T026 (`README.md`) and T027 (`CLAUDE.md`) beside any `PROMPT.md` task.
- T002 beside everything.

```text
After Phase 2:
  PROMPT.md:   T012 → ... → T018 → T019 → ... → T025 → T028
  side tasks:  T026 | T027
```

---

## Implementation strategy

### MVP (US2)

US2 alone stops the single-file overflows, the most common cause. It
needs `git_get_tree`, so it ships with integration 1.0.6.

### Incremental delivery

1. Phases 1 and 2, then US2.
2. US1 (gate A1).
3. US3, confirmed by quickstart step 2 before merge.
4. Polish and rollout: one PR, one agent version (0.1.20), one
   integration version (1.0.6).

All of it ships in one PR, since the version check counts one bump per
PR and `too-large` becomes a released marker at 0.1.20.
