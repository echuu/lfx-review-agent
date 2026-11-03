# Feature Specification: Hourly sweep for skipped pushes

**Feature Branch**: `feature/20-hourly-sweep`

**Created**: 2026-11-02

**Status**: Draft

**Input**: GitHub issue #20, "Review pushes skipped while a review runs or
during the debounce". The maintainer chose one org-wide hourly sweep.

## Context

The review agent starts a review from a GitHub webhook. A push is dropped
without a review in two cases:

- A review is already running on the PR (Step 1.5 gate B).
- The agent commented on the PR less than 5 minutes ago (gate C).

Nothing picks those commits up later. They are reviewed only when the
author pushes again. A PR whose last push was dropped can sit with an
outdated review, or with no approval, until someone notices. Raised in the
CDP and Insights feedback (#17, item O7).

## Clarifications

### Session 2026-11-02

- Q: Should the hourly sweep also review open PRs that the agent never
  reviewed at all? → A: Yes, never-reviewed PRs opened in the last 7 days,
  in onboarded repositories.
- Q: Should the sweep run every hour of every day, or only during working
  hours on weekdays? → A: Every hour, every day (720 runs a month).
- Q: Should this feature also cover stall detection (a Slack alert when a
  review cycle stays unresolved)? → A: No. Out of scope; it gets its own
  issue.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A dropped push gets reviewed (Priority: P1)

A PR author pushes a fix while the agent is still reviewing the previous
commit. The running review finishes on the old commit, and the push that
arrived during it is dropped. Within about an hour, the agent reviews the
new commits on its own, as a normal follow-up review.

**Why this priority**: This is the gap the issue reports. Without it the
author has to push an empty or trivial commit to get a review.

**Independent Test**: On a test PR, push twice, the second time while the
first review is running. Confirm the second push gets no review at first,
and that a follow-up review of it appears after the next sweep, without
any further push.

**Acceptance Scenarios**:

1. **Given** a PR whose latest commit is newer than the agent's last
   review, and no review is running, **When** the sweep runs, **Then** the
   agent starts a follow-up review of the commits after the last reviewed
   one.
2. **Given** a push dropped by the 5-minute debounce, **When** the sweep
   runs more than 5 minutes after the agent's last comment, **Then** the
   push is reviewed as in scenario 1.
3. **Given** the follow-up started by the sweep, **When** the author reads
   it, **Then** it looks like any other follow-up, and its start comment
   says the hourly check started it.

---

### User Story 2 - The sweep never doubles up or breaks the limits (Priority: P1)

A maintainer relies on the existing safety rules: one review at a time per
PR, at most 10 auto-reviews per PR, no reviews of drafts, closed PRs or
excluded areas. The sweep follows all of them.

**Why this priority**: A sweep that ignores these rules would post
duplicate reviews or loop on stuck PRs, which costs money and annoys
authors.

**Independent Test**: Run the sweep against PRs in each blocked state (a
review running, 10 reviews reached, a draft, an excluded change, already up
to date) and confirm none of them gets a review or a new comment.

**Acceptance Scenarios**:

1. **Given** a PR with a review in progress, **When** the sweep runs,
   **Then** it skips that PR.
2. **Given** a PR that already had 10 auto-reviews, **When** the sweep
   runs, **Then** it skips that PR and posts nothing, including no new cap
   notice.
3. **Given** a PR whose latest commit was already reviewed, **When** the
   sweep runs, **Then** it skips that PR.
4. **Given** a sweep review, **When** the reviews are counted, **Then** it
   counts toward the 10-review limit like any other.

---

### User Story 3 - The sweep costs little when nothing is due (Priority: P2)

The workspace has a monthly budget, and most hours nothing is waiting. A
sweep that finds no PR to review ends quickly and posts nothing.

**Why this priority**: The sweep runs every hour. Its idle cost is paid
about 720 times a month.

**Independent Test**: Run the sweep when no PR is behind. Confirm it posts
nothing and ends after a small, bounded number of reads.

**Acceptance Scenarios**:

1. **Given** no PR is behind its last review, **When** the sweep runs,
   **Then** it posts nothing on GitHub and ends.
2. **Given** several PRs are behind, **When** the sweep runs, **Then** it
   reviews only one of them, the one that has waited longest. The others
   wait for later sweeps.

---

### User Story 4 - A PR that never got a first review gets one (Priority: P2)

A PR opened while the agent could not run, for example during the October
30 workspace budget stop, has no review at all. A later push doesn't help:
a push to a PR with no prior review starts nothing. The sweep finds such a
PR if it was opened in the last 7 days in an onboarded repository, and
runs its initial review.

**Why this priority**: Outages are rare, but each one leaves PRs that are
never reviewed unless someone notices.

**Independent Test**: In an onboarded repository, find or open a non-draft
PR that has no agent review and no start comment, and is more than 10
minutes old. Confirm the next sweep runs its initial review.

**Acceptance Scenarios**:

1. **Given** an open, non-draft PR opened in the last 7 days in an
   onboarded repository, with no agent review, **When** the sweep runs and
   the PR is the one that has waited longest, **Then** the agent runs its
   initial review.
2. **Given** a never-reviewed PR opened more than 7 days ago, **When** the
   sweep runs, **Then** it skips that PR.
3. **Given** a never-reviewed PR in a repository the agent doesn't review,
   **When** the sweep runs, **Then** it skips that PR.

### Edge Cases

- A PR was pushed to again after the sweep picked it: the review covers the
  newest commit, as any follow-up does.
- A PR's last own review was withdrawn or dismissed: the sweep treats the
  commit of the latest own review as the last reviewed one, as Step 2 does.
- A PR was never reviewed by the agent, for example because the workspace
  budget stopped all runs when it opened: the sweep runs its initial review
  if it was opened in the last 7 days in an onboarded repository (User
  Story 4). Older ones are left for a human.
- A never-reviewed PR has an exclusion notice, or a start comment from a
  run that is still going: the sweep skips it, as the existing gates do.
- A review run crashed and left a start comment older than 45 minutes: the
  sweep treats it as abandoned, the same way a push does, and can start a
  new review.
- The sweep's search or a read fails: the sweep ends without posting.
  The next hour tries again.
- Many PRs are behind at once (for example after an outage): one is
  reviewed per hour, oldest first.
- A push lands seconds before the sweep, while its own webhook run is
  starting: the sweep picks a PR only when the PR has had no activity for
  at least 10 minutes, so it doesn't race the webhook run. GitHub doesn't
  record when a commit was pushed (a commit's date is when it was made),
  so the PR's last-updated time stands in for it.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The agent MUST run a sweep once an hour, every day including
  weekends, started by a schedule, across every repository it reviews.
- **FR-002**: The sweep MUST consider only open, non-draft PRs in
  onboarded repositories that either have a push the agent dropped or a
  review that did not finish, or were opened in the last 7 days and have
  no agent review or comment.
- **FR-002a**: For a never-reviewed PR, the sweep MUST pick it only when
  the PR was last updated at least 10 minutes before the sweep, and MUST
  run the initial review, not a follow-up.
- **FR-003**: The sweep MUST pick a reviewed PR only when its latest
  commit is not the commit of the agent's latest review, and the PR was
  last updated at least 10 minutes before the sweep.
- **FR-004**: The sweep MUST skip a PR with a review running, a PR at the
  10-review limit, and a PR that the review exclusions cover.
- **FR-005**: The sweep MUST start at most one review per run, on the
  eligible PR that has waited longest.
- **FR-006**: A review started by the sweep MUST follow the same steps,
  gates and output as a review started by a push, and MUST count toward the
  10-review limit.
- **FR-007**: The start comment of a sweep review MUST say the hourly check
  started it.
- **FR-008**: When nothing is eligible, the sweep MUST post nothing on
  GitHub.
- **FR-009**: The sweep MUST NOT post cap, debounce or skip notices, or
  any other comment, on a PR it skips. It may clear the mark that says a
  review is owed.
- **FR-010**: The sweep MUST bound its reads: it checks a fixed maximum
  number of candidate PRs per run.
- **FR-011**: A failed read during the sweep MUST end the run without
  posting, except for reads that the review itself already treats as best
  effort.
- **FR-012**: The new schedule and the tool the sweep needs MUST be
  documented in the README with the trigger setup and credential policy,
  like the existing webhook triggers.

### Key Entities

- **Candidate PR**: an open, non-draft PR in an onboarded repository that
  the agent marked as owed a review (a dropped push or an unfinished
  review), or that was opened in the last 7 days and has no agent review
  or comment. Its state is its latest commit and when the PR was last
  updated.
- **Onboarded repository**: a repository with the agent's webhook
  triggers, which the sweep must be able to recognize.
- **Last reviewed commit**: the commit of the agent's latest marked review
  on the PR, as Step 2 derives it.
- **Sweep run**: one scheduled run. It picks zero or one PR and reviews it.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A push dropped by gate B or gate C is reviewed within 75
  minutes, without another push, when no more than one PR is waiting.
- **SC-001a**: A never-reviewed PR opened in the last 7 days in an
  onboarded repository gets its initial review within 75 minutes of the
  agent running again, when no more than one PR is waiting.
- **SC-002**: No PR gets two reviews of the same commit because of the
  sweep.
- **SC-003**: No sweep posts on a PR that is closed, a draft, excluded, at
  the 10-review limit, or up to date.
- **SC-004**: An idle sweep run costs no more than one tenth of a typical
  follow-up review.
- **SC-005**: The sweep adds at most 24 reviews a day.

## Assumptions

- Guild's scheduled trigger can start this agent with a fixed JSON input,
  and the run has no webhook payload.
- GitHub search can find open PRs in the organization by label, and, per
  repository, recent open PRs that `lfx-one` has not reviewed or commented
  on, with the existing token.
- Pushes dropped before the sweep ships are not picked up. Those PRs get a
  review on their next push, as today.
- A failed attempt to mark a PR as owed a review is reported in the run's
  summary and not retried. That PR waits for its next push.
- One review per hour is enough to clear the usual backlog. Bursts after an
  outage take longer and are accepted.
- Ten minutes without PR activity is long enough for a push's
  own webhook run to start and post.
- Never-reviewed PRs opened more than 7 days ago, and PRs in repositories
  that are not onboarded, are out of scope.
- Stall detection and Slack alerts for unresolved review cycles are out of
  scope and tracked separately. The Scope section of the prompt keeps
  listing them as not handled.
