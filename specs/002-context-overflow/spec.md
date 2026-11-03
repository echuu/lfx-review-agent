# Feature Specification: Reviews that fit, or say why they can't

**Feature Branch**: `feature/context-overflow`

**Created**: 2026-11-03

**Status**: Draft

**Input**: Phases B and C of the context-overflow fix, from the
investigation of lfx-self-serve#4962 review 5, lfx-v2-campaign-service#472
and lf-dbt#3177. Phase A (the stalled notice and the Step 5c tail read)
shipped in 0.1.18 (PR #35).

## Context

The model behind the review agent accepts at most 1,000,000 tokens per
call. A run that goes over gets a provider error, and the run ends at
once: it cannot post a summary, a review, or a notice. Guild does not
retry it. The start comment stays up with no report, and the author sees
nothing for at least 45 minutes.

Between the 0.1.16 release (2026-11-02) and 2026-11-03, 3 of 299 webhook
runs ended this way:

| PR | Prompt size | What filled it |
| --- | --- | --- |
| lfx-self-serve#4962, review 5 | 1,041,163 tokens | PR history: changed files with patches (813 KB), inline review comments (435 KB), reviews read twice (177 KB each), commits (106 KB) |
| lfx-v2-campaign-service#472, review 3 | 1,456,008 tokens | One read of a generated 1.3 MB file whose patch GitHub omitted |
| lf-dbt#3177, review 2 | 1,936,920 tokens | One file whose patch GitHub omitted, read in full at two commits (1.0 MB and 0.9 MB) |

Two causes, two fixes:

- **One file too large.** When GitHub omits a file's patch, the prompt
  reads the whole file, at both ends of the diff, with no size limit. One
  file can fill the context. Even when it fits, the model has to find the
  changes by comparing two large copies, which it does poorly.
- **Too much history.** On a large PR with many review rounds, the
  history the agent reads adds up. Most of it is not the history itself:
  GitHub's responses wrap it in fields the review never uses. On #4962,
  comment text was 8% of the inline comment response; the diff excerpt
  repeated on every comment was 55% and user records 14%. Review text was
  46% of the review response. File contents arrive base64-encoded, which
  is larger than the source and harder for the model to read.

## Clarifications

### Session 2026-11-03

- Q: When a changed file is too large to read in full, should the agent
  fail the review, or review the rest? → A: Review the rest, name the
  file at the top of the summary, and withhold approval so a human looks
  at it.
- Q: Should the size check before a review be a last resort, with a
  visible notice? → A: Yes. Failing with no word is the worst outcome.
- Q: Should an unread generated file or lockfile withhold approval? → A:
  No, for a fixed list of known patterns. Name it as not reviewed line by
  line; its source is still reviewed. Any other unread file withholds
  approval.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A PR too large to review gets a clear notice (Priority: P1)

A PR author pushes to a PR that is too large for one review run. Today
the agent posts "I'm starting review N", fails, and leaves that comment
for 45 minutes or more. Instead, the agent sees before it starts that the
PR is over the limit, posts no start comment, and posts one notice: the
PR is too large for automated review, and a human review is needed.

**Why this priority**: This is the fallback for every other story. It
turns a silent failure into a clear next step, and it works even before
the cheaper reads in Stories 2 and 3 ship.

**Independent Test**: On a test PR over the limit, push a commit. Confirm
one notice, no start comment, no review, and that a second push posts no
second notice.

**Acceptance Scenarios**:

1. **Given** a PR over the size limit, **When** a push or the hourly
   sweep starts a run, **Then** the agent posts one notice, no start
   comment and no review.
2. **Given** a PR that already has that notice and is still over the
   limit, **When** another run starts, **Then** it posts nothing.
3. **Given** a PR with a notice that later drops under the limit (for
   example, files were removed), **When** a push starts a run, **Then**
   the agent reviews it as usual.
4. **Given** a PR under the limit, **When** a run starts, **Then** the
   size check changes nothing about the review.
5. **Given** a PR with a live approval from the agent that goes over the
   limit, **When** a run starts, **Then** the approval guard still runs
   as for the other gates, so an exclusion match still revokes it.

---

### User Story 2 - One huge file does not sink the review (Priority: P1)

A PR changes a file so large that GitHub leaves out its patch, such as a
generated API description or a large test fixture. Today the agent reads
the whole file at both commits and can overflow. Instead, it reviews
every other file, names the large file at the top of the summary as not
reviewed, and withholds approval, so a human must look at that file.

**Why this priority**: Two of the three failures were this case, and a
single file triggers it on an otherwise small PR.

**Independent Test**: On a test PR that changes one file too large for
GitHub to show a patch and one normal file, confirm the review covers the
normal file, names the large file, and is not an approval.

**Acceptance Scenarios**:

1. **Given** a PR where GitHub omits one file's patch and that file is
   over the read limit, **When** the agent reviews, **Then** it does not
   read that file, reviews the rest, names it at the top of the summary,
   and posts a COMMENT review with the approval-withheld marker instead
   of an approval.
2. **Given** the same PR with blocking findings elsewhere, **When** the
   agent reviews, **Then** it requests changes as usual and still names
   the unread file.
3. **Given** a file whose patch GitHub omitted but which is under the
   read limit, **When** the agent reviews, **Then** it reads the file and
   reviews it as today.
4. **Given** a later push that no longer changes the large file, **When**
   the agent reviews, **Then** approval is no longer withheld for it.

---

### User Story 3 - Large, busy PRs fit in one review run (Priority: P2)

A PR with many files and many review rounds keeps getting reviewed. The
agent reads the same history it reads today, but in a form that costs
far less of the model's context, so fewer PRs reach the size limit in
Story 1.

**Why this priority**: Stories 1 and 2 stop the silent failures. This
story keeps more PRs reviewable instead of handing them to a human, and
it costs less per run.

**Independent Test**: Run a follow-up review on a PR with history like
that of #4962. Compare the total tool output with the 2.11 MB that
review 5 read, and confirm the review still reconciles every prior finding and
every bot comment.

**Acceptance Scenarios**:

1. **Given** a follow-up review, **When** the agent reads prior reviews
   and inline comments, **Then** it gets every one of them, including
   their text, author, file and line, and nothing it does not use.
2. **Given** a rule file or context file read, **When** the agent reads
   it, **Then** it receives the file as plain text.
3. **Given** a follow-up review, **When** the agent reconciles prior
   feedback and bot comments, **Then** the result matches what it would
   have produced from the full responses.

---

### Edge Cases

- A PR is just under the size limit and a run still overflows: the run
  dies as today, and Phase A's sweep notice covers it. The limits get
  tuned from the run data.
- A PR changes several large files: each is named, and approval is
  withheld once, listing all of them.
- A large file is deleted by the PR: there is nothing new to review in
  it. It is named as deleted, and approval is not withheld for it.
- A large file is generated (for example Goa's `gen/http/openapi3.json`)
  or a lockfile (`package-lock.json`, `yarn.lock`, `go.sum`): if it
  matches the fixed list of known patterns, it is named as not reviewed
  line by line, and approval is not withheld for it. Its source (the Goa
  design files, `package.json`, `go.mod`) is still reviewed. A large file
  outside that list withholds approval (FR-013).
- The size check reads counts that GitHub reports late or as zero: the
  check uses only what the PR record already holds and errs toward
  reviewing.
- The PR is excluded or at the 10-review limit: those gates run first, as
  today, and the size notice is not posted.
- A re-review request on an over-limit PR (if the comment trigger is
  turned back on): it gets the existing skip reply, naming the size limit.
- Guild cannot trim responses and no safe alternative exists: Story 3's
  trimmed history reads are deferred; the plain-text file reads can still
  ship on their own.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Before posting a start comment, a run MUST check the PR's
  size against fixed limits, using data the run already reads.
- **FR-002**: A run that finds the PR over a limit MUST post one notice
  that says the PR is too large for automated review and a human review
  is needed, and MUST NOT post a start comment or a review.
- **FR-003**: The size notice MUST be posted at most once while the PR
  stays over the limit, and MUST NOT count toward the 10-review limit.
- **FR-004**: A PR that drops back under the limits MUST be reviewed as
  usual on its next event.
- **FR-005**: The limits MUST be set so that none of the successful runs
  since 0.1.16 would have been stopped by them, and every run that
  overflowed on PR history would have been.
- **FR-006**: The agent MUST NOT read a changed file in full when that
  file is over a per-file read limit. It MUST review the rest of the PR.
- **FR-007**: A review that skipped a changed file under FR-006 MUST name
  each such file at the top of the summary and MUST NOT approve, except
  as FR-013 allows. When it
  would otherwise approve, it MUST post a COMMENT review marked as
  approval withheld, like a failed required check.
- **FR-008**: A file whose patch GitHub omitted MUST be read at the PR's
  head only, not at both ends of the diff.
- **FR-009**: File contents the agent reads MUST arrive as plain text,
  not base64, if the integration can request that.
- **FR-010**: Prior reviews and inline comments MUST be read in a form
  that keeps their text, author, state, file, line and time, and drops
  fields the review does not use, if Guild offers a safe way to do this.
- **FR-011**: Any new operation MUST give the agent no ability to change
  GitHub state beyond what its current operations allow. A free-form
  query operation is not acceptable.
- **FR-012**: Each new operation MUST be added to the integration spec,
  the Guild manifest, and the README credential policy.
- **FR-013**: An unread file that matches a fixed list of generated-file
  and lockfile patterns MUST be named as not reviewed line by line and
  MUST NOT withhold approval on its own. The list MUST live in the prompt
  so maintainers can extend it, like the review exclusions.

### Key Entities

- **Size limits**: the thresholds the pre-review check compares against
  (changed files, changed lines, and the amount of review history).
- **Size notice**: a comment with its own marker that tells the author
  the PR is too large for automated review. Read back by later runs.
- **Unread file**: a changed file over the per-file read limit, named in
  the summary and the reason approval was withheld.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In the 30 days after release, no run ends with the
  provider's "prompt is too long" error without the author getting a
  notice that says what to do.
- **SC-002**: Every PR the agent declines for size gets its notice within
  5 minutes of the event, with no start comment left behind.
- **SC-003**: None of the 296 successful runs since 0.1.16 would have been
  declined by the size limits.
- **SC-004**: A follow-up review on a PR with #4962's history reads at
  most half of the 2.11 MB that review 5 read.
- **SC-005**: No approval is posted on a PR with a changed file the agent
  did not read, unless that file is on the generated-file and lockfile
  list.

## Assumptions

- The model's limit stays at 1,000,000 tokens per call. Guild picks the
  model, and this feature does not change it.
- GitHub's PR record already gives changed files, added and deleted
  lines, and comment counts, so the size check costs no extra read.
- GitHub can return file contents as plain text when asked. Whether a
  Guild integration operation can send that request is not known yet; a
  probe settles it.
- Whether Guild can trim fields from a response is not known yet. It is a
  question for Guild (Rod Brathwaite).
- The 5c re-read of conversation comments is left as is: comments can be
  deleted, so reading only the newer pages is not safe there.
- PRs stuck before release (#4962 and #472) are handled by hand.
- Phase A's stalled notice stays as the backstop for runs that still
  overflow.
