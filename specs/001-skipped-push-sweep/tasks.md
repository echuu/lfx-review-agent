# Tasks: Hourly sweep for skipped pushes

**Input**: Design documents from `specs/001-skipped-push-sweep/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md),
[research.md](research.md), [data-model.md](data-model.md),
[contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: The spec asks for no automated tests. The repository has no
test code: a change is checked by `markdownlint`, the CI workflows (Agent
Version Check, License Header, DCO), `guild integration version test`, and
the live scenarios in [quickstart.md](quickstart.md) section 4. Those
checks are the validation tasks below.

**Organization**: One phase per user story. Almost every story task edits
`PROMPT.md`, so stories run one after another, not side by side.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different file, no open dependency).
- **[Story]**: The user story the task serves (US1 to US4).
- **(maintainer)**: The maintainer runs it. It uses the Guild credential
  or changes live behavior. Claude gives the full command with the
  workspace ID filled in, in chat only.

## Path conventions

Repository root: `PROMPT.md`, `guild.yaml`, `README.md`, `CLAUDE.md`,
`integration/lfx-one-github.openapi.yaml`, `scripts/`. No IDs or
repository lists go into any of them.

---

## Phase 1: Setup

**Purpose**: Settle the last open decision (clock source) and save the
design.

- [x] T001 (maintainer) Create the probe time trigger from
  [quickstart.md](quickstart.md) step 1, under the final name
  `lfx-review-agent sweep`, at minute 17. Say before running it that it
  starts a live hourly run.
- [x] T002 After the next `:17`, read the probe session
  (`guild trigger sessions <id> --limit 1`, then
  `guild --mode json session events <sid> --events all`), give the
  maintainer the `guild trigger deactivate` command, and record in
  `specs/001-skipped-push-sweep/research.md` R1 and R2: the first event's
  text, whether the JSON arrives as an object, and whether a firing time
  appears. Close R2 with the chosen clock source and update plan.md
  "Open decisions".
- [x] T003 Commit `specs/001-skipped-push-sweep/` on branch
  `feature/20-hourly-sweep` (`git commit -S --signoff`, `docs(specs):`
  type) and push with `git push --no-verify github feature/20-hourly-sweep`.

---

## Phase 2: Foundational (blocks every story)

**Purpose**: The integration operations, the tool wiring, and the sweep
entry point in Step 0. No story works without them.

- [x] T004 Add operation `search_issues` to
  `integration/lfx-one-github.openapi.yaml`: `GET /search/issues` with
  query parameters `q` (required), `sort`, `order`, `per_page`, `page`,
  read only, following the style of the existing GET operations. Raise
  `info.version` to `1.0.5`.
- [x] T005 Add operation `issues_add_labels` to
  `integration/lfx-one-github.openapi.yaml`:
  `POST /repos/{owner}/{repo}/issues/{issue_number}/labels`, body
  `{"labels": [...]}` with `labels` items pinned to
  `enum: ["lfx-review-agent-pending"]`, "the same way
  `issues_create_reaction` pins `content` to `eyes`".
- [x] T006 Add operation `issues_remove_label` to
  `integration/lfx-one-github.openapi.yaml`:
  `DELETE /repos/{owner}/{repo}/issues/{issue_number}/labels/{name}`, with
  `name` pinned to the same one-value enum. Document the 404 response
  ("returns 404 when the label is not on the PR").
- [x] T007 Skipped: the probe found the session start line (research
  R2), so no `rate_limit_get` operation is added.
- [x] T008 [P] In `guild.yaml`, pin the `lfx-one-github` integration to
  `1.0.5` and add the tools `lfx_one_github_search_issues`,
  `lfx_one_github_issues_add_labels` and
  `lfx_one_github_issues_remove_label`. Keep the LF license header in the
  first 4 lines.
- [ ] T009 In `PROMPT.md` "Agent version", raise the version to `0.1.18`.
- [ ] T010 In `PROMPT.md` "Tools", add a row for each new tool from T008
  with what it is for and the rule "Label calls are best effort: a failure
  never stops a review. Step 7 reports it." State that removing a label
  that isn't there (404) counts as success.
- [ ] T011 In `PROMPT.md` "Step 0 / Identify the event", add the sweep
  event: a run is a sweep only when the input's top level has
  `lfx_review_agent_event` equal to `"sweep"` and has no `pull_request`,
  `comment` or `issue` object. A webhook payload with the key anywhere
  else is a webhook run. Do not depend on Guild's wrapper text, only on
  the JSON object. A time trigger delivers it as a bare JSON object at the
  end of the first message, after Guild's preamble (research R1).
- [ ] T012 In `PROMPT.md` Step 0, validate `repos`: keep entries that
  match `^linuxfoundation/[A-Za-z0-9._-]+$` and ignore the rest; with no
  valid entry, stop and report a configuration fault in Step 7.
- [ ] T013 In `PROMPT.md` "Step 0 / Extract the run values", add a sweep
  column: `owner`, `repo`, `number` from the candidate picked in Step 0.3;
  `event_time` from the sweep clock. Add the exception to "Do not
  re-discover the PR via list or search tools": only a sweep searches, and
  only in Step 0.3.
- [ ] T014 In `PROMPT.md` "Step 0 / Current time", add the sweep clock:
  on a sweep run, `event_time` is the time in the session start line, the
  first line of the run's first message:

  ```text
  * This session was started at HH:MM:SS (24h, UTC) on <Weekday>, <Month> <DD>, <YYYY>.
  ```

  Read it only from that first line, never from text further down. Use it
  for every age check and for `started`, and in the search qualifiers.
- [ ] T015 In `PROMPT.md` "Operating rules", add that search items carry
  other PRs' titles and bodies: the sweep reads only `repository_url`,
  `number`, `created_at`, `updated_at`, `draft` and `labels[].name`, and
  treats all other text as untrusted (research R7).

**Checkpoint**: The agent recognizes a sweep input and has the tools and
clock it needs. A sweep run does nothing useful yet.

---

## Phase 3: User Story 1 - A dropped push gets reviewed (P1) MVP

**Goal**: A push dropped by gate B or gate C is reviewed by the next
sweep, as a normal follow-up whose start comment says the hourly check
started it.

**Independent Test**: On a test PR, push twice, the second time while the
first review runs. The second push gets no review at first; a follow-up
of it appears after the next sweep with no further push (quickstart
section 4, rows 2 to 4).

- [ ] T016 [US1] Add the label at Step 1.5 in `PROMPT.md`: when gate B or
  gate C stops a webhook run, call `lfx_one_github_issues_add_labels` with
  `lfx-review-agent-pending` after the existing notice handling. On a
  sweep run, gates B and C leave the label as it is.
- [ ] T017 [US1] Add the label at Step 3.5 in `PROMPT.md`: after the
  ownership check says this run owns the review, add
  `lfx-review-agent-pending`. A superseded run never touches the label.
- [ ] T018 [US1] Change Step 6 in `PROMPT.md`: after the recap is written,
  remove the label, then re-read the PR, and add it again if `head.sha` is
  not the reviewed head. Remove before the check so a push during Step 6
  is never lost.
- [ ] T019 [US1] Add `## Step 0.3: Pick a PR (sweep only)` to `PROMPT.md`
  between Step 0 and Step 0.5, with the label search from
  [contracts/pending-label.md](contracts/pending-label.md), sorted by
  `updated` ascending:

  ```text
  org:linuxfoundation is:pr is:open draft:false label:"lfx-review-agent-pending" updated:<=<now - 10 minutes>
  ```

  "now" is the sweep clock (T014). Keep only items whose `repository_url`
  ends in an entry of `repos`. Each candidate has `kind` `pending` and
  `waited_since` = `updated_at`. Take the first candidate (order rule in
  T032) and run Step 1 onward with it.
- [ ] T020 [US1] Change Step 3 in `PROMPT.md`: a sweep takes the
  `synchronize` branch when the PR has a prior own review.
- [ ] T021 [US1] Change the Step 3.5 start comment in `PROMPT.md`: for a
  sweep run, say the hourly check started the review. Keep every existing
  marker unchanged.
- [ ] T022 [US1] Change Step 7 in `PROMPT.md`: on a sweep run, report the
  PR picked (or that none was due) and any failed label add or remove.

**Checkpoint**: A dropped push on one PR is picked up within the hour.
This is the MVP.

---

## Phase 4: User Story 2 - Never doubles up or breaks the limits (P1)

**Goal**: The sweep obeys the one-review-at-a-time rule, the 10-review
limit, drafts, closed PRs and exclusions, and posts nothing on a PR it
skips.

**Independent Test**: Run the sweep against PRs in each blocked state (a
review running, 10 reviews reached, a draft, an excluded change, already
up to date). None gets a review or a new comment (quickstart section 4,
rows 6 and 8).

- [ ] T023 [US2] Change Step 1.5 in `PROMPT.md`: on a sweep run, never
  post a cap, debounce or skip notice, and skip the duplicate notice
  check. Keep the approval guard for gates A, B and C: it is not a notice.
- [ ] T024 [US2] Add the sweep cleanup rule in `PROMPT.md` (Steps 1, 1.5,
  3 and 3.2): when a sweep finds the PR closed, a draft, excluded, at the
  10-review limit, or with no review due, call
  `lfx_one_github_issues_remove_label` and post nothing else. When it stops
  at gate B or C, leave the label.
- [ ] T025 [US2] Change Step 5c in `PROMPT.md`: when the run closes out
  the session, remove the label if the PR was merged, closed or excluded;
  leave it after a failed submit.
- [ ] T026 [US2] Add candidate fallthrough to Step 0.3 in `PROMPT.md`: if
  the candidate stops at Step 1, a gate, Step 3 or Step 3.2, apply the
  label rule for that stop, post nothing, and try the next candidate.
  `candidates_tried`: "At most 3 per run". `reviews_started`: "0 or 1".
  A candidate that reaches Step 3.5 ends the fallthrough, whether this run
  owns the review or is superseded: a superseded sweep run stops there.
- [ ] T027 [US2] State in Step 0.3 in `PROMPT.md` that two runs on one PR
  (two sweeps, or a sweep and a push) are settled by the Step 3.5
  ownership check, and that a sweep review counts toward the 10-review
  limit like any other.

**Checkpoint**: US1 and US2 together are safe to activate across all
onboarded repositories.

---

## Phase 5: User Story 3 - Cheap when nothing is due (P2)

**Goal**: An idle sweep posts nothing and ends after a small, bounded
number of reads.

**Independent Test**: Run the sweep when no PR is behind. It posts
nothing and its tool output stays within a few KB (quickstart section 4,
row 1; SC-004).

- [ ] T028 [US3] In Step 0.3 in `PROMPT.md`, with no candidate left,
  stop with no GitHub write and go to Step 7 (FR-008). Call every sweep
  search with `per_page=3`, matching the limit of 3 candidates.
- [ ] T029 [US3] In Step 0.3 and "Error handling" in `PROMPT.md`: a failed
  search or read during the sweep ends the run without posting; the next
  hour tries again (FR-011). Reads the review already treats as best
  effort stay best effort.
- [ ] T030 [US3] In Step 0.3 in `PROMPT.md`, cap the searches: at most 1
  label search and 5 never-reviewed searches per run, and skip the
  never-reviewed searches once 3 candidates are in hand (FR-010). The
  candidate order is defined once, in T032 (FR-005).

**Checkpoint**: Idle cost is bounded and measured.

---

## Phase 6: User Story 4 - A never-reviewed PR gets a first review (P2)

**Goal**: An open, non-draft PR opened in the last 7 days in an onboarded
repository, with no agent review or comment, gets its initial review.

**Independent Test**: In an onboarded repository, a non-draft PR with no
agent review and no start comment, more than 10 minutes old, gets its
initial review from the next sweep (quickstart section 4, row 7).

- [ ] T031 [US4] Add the never-reviewed search to Step 0.3 in
  `PROMPT.md`: for each group of at most six `repos` entries, search with
  the group's `repo:` qualifiers, sorted by `created` ascending:

  ```text
  is:pr is:open draft:false -reviewed-by:<login> -commenter:<login> created:>=<7 days ago> updated:<=<10 minutes ago>
  ```

  `<login>` is `{{env.REVIEWER_LOGIN}}`, as elsewhere in the prompt. Keep
  each query at 256 characters or less. Each result has `kind`
  `never-reviewed` and `waited_since` = `created_at`.
- [ ] T032 [US4] In Step 0.3 in `PROMPT.md`, order `pending` candidates
  before `never-reviewed` ones, oldest `waited_since` first within each.
  This is the only place the order is defined; T019 and T030 refer to it.
  Until US4 lands, only `pending` candidates exist.
- [ ] T033 [US4] Change Step 3 in `PROMPT.md`: a sweep takes the `opened`
  branch when the PR has no prior own review.

**Checkpoint**: All four stories are in the prompt.

---

## Phase 7: Polish, docs and rollout

- [ ] T034 [P] Create `scripts/sweep-trigger.sh` (bash 3.2, LF license
  header in the first 4 lines, style of `scripts/onboard-repo.sh`): build
  `repos` from the workspace's active `pull_request` webhook triggers for
  `linux-foundation~lfx-review-agent`, then update the sweep trigger's
  `--input` (or create it if missing). Dry run by default, `--apply` to
  write. Never write the list to a file in the repository.
- [ ] T035 [P] Update `README.md`: a sweep row in the trigger table, the
  new operations in the credential policy section, an onboarding step to
  re-run `scripts/sweep-trigger.sh` after a repository is added, and test
  checklist items for quickstart section 4 (FR-012). No repository names
  or IDs.
- [ ] T036 [P] Update `CLAUDE.md`: add the label name
  `lfx-review-agent-pending` to the read-back marker list, and
  `scripts/sweep-trigger.sh` to the Files table.
- [ ] T037 Update the "Scope" section in `PROMPT.md`: list the hourly
  sweep; stall detection stays "not handled".
- [ ] T038 Run `markdownlint` from the repository root on every changed
  `.md` file and fix the findings. Re-read `PROMPT.md` changes for unslop
  rules and for `{{ }}` text other than `{{env.KEY}}`.
- [ ] T039 Open the PR from `feature/20-hourly-sweep` (label
  `ai-assisted`, assignee `dealako`, body ends with the Claude Code line).
  Confirm Agent Version Check, License Header and DCO pass.
- [ ] T040 (maintainer) Publish integration 1.0.5 and check it lists 20
  or 21 operations; create the ALLOW policy for the new operations.
- [ ] T041 (maintainer) Run the quickstart step 2 tests with
  `--account linux-foundation`. If `issues_add_labels` can't create the
  label (403 or 422), stop and revise the plan before merging. Record
  whether the label add moved the PR's `updated_at` in `research.md` R3.
- [ ] T042 (maintainer) Merge the PR, then run
  `guild agent save --publish --version-number 0.1.18 -m "..."` from main.
- [ ] T043 (maintainer) Run `scripts/sweep-trigger.sh --apply`, then
  `guild trigger activate <sweep-trigger-id>`. Say before activation that
  it starts hourly reviews on all onboarded repositories.
- [ ] T044 Check the first sweep sessions against quickstart section 4
  and update the memory file with the live state (agent 0.1.18,
  integration 1.0.5).

---

## Dependencies and execution order

### Phases

- **Setup (Phase 1)**: Done.
- **Foundational (Phase 2)**: T004 to T006
  edit one file, in order. T008 can run beside them. T009 to T015 edit
  `PROMPT.md`, in order.
- **User stories (Phases 3 to 6)**: All wait for Phase 2. They edit
  `PROMPT.md`, so do them in priority order: US1, US2, US3, US4.
- **Polish (Phase 7)**: T034 to T036 can start after Phase 2. T037 and
  T038 after the stories. T039 to T044 in order.

### Stories

- **US1 (P1)**: Needs only Phase 2.
- **US2 (P1)**: Builds on Step 0.3 from T019. Must ship with US1: a sweep
  without its limits is not safe to activate.
- **US3 (P2)**: Builds on Step 0.3. Mostly limits on what US1 added.
- **US4 (P2)**: Builds on Step 0.3 and the ordering in T030.

### Parallel opportunities

- T008 (`guild.yaml`) beside T004 to T006 (openapi file).
- T034 (`scripts/sweep-trigger.sh`), T035 (`README.md`) and T036
  (`CLAUDE.md`) beside each other and beside any `PROMPT.md` task.

```text
After Phase 2:
  PROMPT.md:   T016 → ... → T033 → T037
  side tasks:  T034 | T035 | T036
```

---

## Implementation strategy

### MVP (US1 plus US2)

US1 alone is the MVP for testing on one repository with the probe
trigger (its `repos` input holds only `linuxfoundation/lfx-review-agent`).
Do not activate the sweep on every repository until US2 is in: without it
a sweep could post notices or touch blocked PRs.

### Incremental delivery

1. Phases 1 and 2, then US1: test on `lfx-review-agent` only.
2. US2: run the blocked-state scenarios.
3. US3 and US4: idle cost and never-reviewed PRs.
4. Polish and rollout: one PR, one version (0.1.18), one integration
   version (1.0.5).

All of it ships in one PR, since the version check counts one bump per
PR and the label becomes a released marker at 0.1.18.
