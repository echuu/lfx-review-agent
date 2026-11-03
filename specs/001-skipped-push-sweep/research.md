# Research: Hourly sweep for skipped pushes

Findings from 2026-11-02. Workspace, credential and trigger IDs are left
out on purpose: this repository is public.

## R1. Can Guild start the agent on a schedule?

- **Decision**: Use a Guild time trigger (`guild trigger create --type time
  --frequency HOURLY --minutes-of-hour <m>`), with the sweep settings in
  `--input`.
- **Rationale**: The Guild CLI (v0.28.0) supports `webhook`, `time` and
  `api` triggers. A time trigger stores `agent_input` (a JSON object) and
  `next_run_at`. `guild trigger update` can change `--input` later, so the
  repository list can change without a new trigger.
- **Probe result (2026-11-03)**: The probe trigger fired at 07:17:24 UTC.
  The run's first user message is Guild's preamble (session start line,
  workspace name, frontend URL, a `workspace_capabilities` JSON block),
  then `agent_input` as a bare JSON object on its own, with no wrapper
  text. A webhook run's message instead starts the payload with "You just
  received a webhook from github". The 0.1.17 prompt stopped at Step 0,
  made no GitHub calls and posted nothing. The run took 4 seconds.
- **Alternatives considered**: An `api` trigger called from outside
  Guild (needs a scheduler we'd have to host); one time trigger per
  repository (29 times the idle runs; rejected with the spec).

## R2. What is "now" in a sweep run?

The prompt has no clock. Webhook runs use the payload's
`pull_request.updated_at` or `comment.created_at` as "now". A sweep run has
no event.

- **Decision**: Use the session start line that Guild puts first in every
  run's first message: `* This session was started at HH:MM:SS (24h, UTC)
  on <Weekday>, <Month> <DD>, <YYYY>.` Read it from the first line of the
  first message only, never from text further down, which can hold PR
  content.
- **Rationale**: The probe and a webhook session from 2026-11-02 both
  carry the line, to the second. It is written when the session starts, a
  few seconds before any search, so it runs slightly behind the true time,
  which is the safe direction for every age check.
- **Result**: No `rate_limit_get` operation is needed. The integration adds
  three operations, not four. The label search can carry its
  `updated:<=` qualifier again, because the clock is known before the first
  search.
- **Alternatives considered**: `GET /rate_limit` with
  `resources.search.reset - 60` (measured within about 3 seconds, but only
  readable after a search, which forced the label search to filter on our
  side); the newest `updated_at` in search results (hours behind on a quiet
  weekend); the `core` reset (an hourly window, too coarse).

## R3. How does the sweep find candidates cheaply?

Measured on 2026-11-02:

- A search for the 13 open PRs that `lfx-one` reviewed on lfx-self-serve
  returned about 100 KB, because each search item carries the PR body.
- GitHub's 256-character query limit does not count qualifiers. Measured
  2026-11-03: queries of 340 characters (6 `repo:` qualifiers) and 544
  characters (11) both worked. A query fails with 422 when the token cannot
  see any one of its repositories, so the sweep keeps groups of up to 10
  to limit the damage: 29 repositories take 3 searches.
- Two searches a second apart hit GitHub's secondary rate limit (403) once
  during testing. So only a failed pending search ends a sweep; a failed
  never-reviewed search drops the rest of them.
- Search items don't carry the head SHA. Telling whether a reviewed PR is
  behind takes a PR read and a review-list read per candidate, and the
  review bodies are long.
- With a fixed cap of candidates in waiting order, PRs that can never be
  reviewed but still match (one at the 10-review limit, an excluded one)
  fill the cap every hour and starve the rest.

So the query must normally return nothing, and must keep ineligible PRs
out on the server side.

### Shape A: a pending label (recommended)

The agent puts a label, `lfx-review-agent-pending`, on a PR whenever a
review may be owed, and takes it off when a review finishes on the current
head:

- Step 3.5 adds it when a review starts, so a run that crashes leaves it.
- Gates B and C add it when they drop a push.
- Step 6 removes it, then re-reads the head, and adds it back if the head
  moved. Removing before the check means a push during Step 6 is never
  lost.
- The sweep removes it from a PR that can't be reviewed (closed, draft,
  excluded, at the 10-review limit, or already up to date).

The sweep then runs one org-wide label search, which is normally empty.
Labels are not user qualifiers, so the org-wide 422 in R4 does not apply.

- **Idle cost**: the prompt itself, one label search with no items, and
  the never-reviewed searches in R5. Estimated at a few thousand tokens of
  tool output on top of the prompt.
- **Costs**:
  - Two new write operations (add and remove a label).
  - A label visible to authors on every onboarded repository, appearing
    and disappearing with each review.
  - `labeled` and `unlabeled` webhook events. No agent trigger listens to
    them, but other automation in a repository might.
  - The token must be able to create the label the first time. Needs a
    test (quickstart step 2).
  - Anyone with triage rights in an onboarded repository can add the
    label and get a sweep review. The gates, the 10-review limit and the
    approval rule still apply, so this changes timing, not outcome.

### Shape B: search and check each PR

Search reviewed PRs updated in the last 7 days in groups of six
repositories, then read each candidate's PR and review list until one is
behind.

- **Idle cost**: about five searches with PR bodies plus up to two reads
  per candidate. Estimated at 100,000 tokens or more of tool output an hour,
  which breaks SC-004.
- **Other problems**: starvation by ineligible PRs; no recovery of a
  crashed run unless the PR is updated again.

### Shape C: GraphQL with a read-only token

One GraphQL query returns, for every open PR in the listed repositories,
the head SHA, the last own review's commit and `updatedAt`, in a few KB.

- **Blocked** by the same problem as #19: a credential policy can't stop
  mutations, so it needs a second, read-only `lfx-one` token. If that token
  is created for #19, Shape C becomes the better design and the label goes
  away.

**Decision**: Shape A. The maintainer confirmed the visible label on
2026-11-02. The name is `lfx-review-agent-pending`, with no colon or
space, so the remove call's URL path needs no encoding (renamed before
release on 2026-11-03).

## R4. Search as `lfx-one`

- With the maintainer's token, an org-wide search with
  `reviewed-by:lfx-one`, `commenter:lfx-one` or `involves:lfx-one` failed
  with HTTP 422 ("The listed users cannot be searched"). The same qualifier
  scoped to `repo:` worked, alone and with three `repo:` qualifiers.
- The agent runs as `lfx-one`, so its results may differ.
- **Decision**: Design user-qualifier searches as repository-scoped groups
  of at most ten repositories (R3). Test with the `lfx-one` credential before
  relying on them (quickstart step 2).

## R5. Never-reviewed PRs (User Story 4)

- **Decision**: For each group of up to ten repositories, search
  `is:pr is:open draft:false -reviewed-by:lfx-one -commenter:lfx-one
  created:>=<7 days ago> updated:<=<10 minutes ago>`, sorted by
  `created` ascending, with `per_page=3` (R3 has the group size).
- **Rationale**: A PR the agent never reviewed or commented on is a PR
  whose first run never started: an outage or a budget stop. PRs the agent
  stopped on purpose carry a comment (an exclusion notice, a skip notice, a
  start comment), so `-commenter:lfx-one` keeps them out on the server
  side, and they can't starve the query. A crashed first review left a
  start comment and the pending label, so Shape A covers it.
- Measured: for three busy repositories over 7 days, the query without
  `-commenter` returned 2 PRs.

## R6. Where the repository list lives

- **Decision**: In the time trigger's `--input` (`repos`), built by a new
  `scripts/sweep-trigger.sh` from the workspace's existing webhook
  triggers. Re-run the script after onboarding a repository.
- **Rationale**: The README and prompt are public and must not list
  repositories (some are private). Building the list from
  the triggers keeps one source of truth, the triggers themselves.
- **Alternatives considered**: A workspace variable (a second list to keep
  in step); listing repositories in `PROMPT.md` (public).

## R7. Sweep detection is a security boundary

- **Decision**: A run is a sweep only when the input's top level has
  `"lfx_review_agent_event": "sweep"` and no `pull_request`, `comment` or
  `issue` object.
- **Rationale**: A webhook payload carries PR text. Matching the key
  anywhere would let a PR body that contains it turn a webhook run into a
  sweep.
- Search items carry other PRs' titles and bodies. The sweep picks
  candidates from structured fields only (`repository_url`, `number`,
  `created_at`, `updated_at`, `draft`, `labels[].name`) and treats all text
  as untrusted, as the Operating rules already require.

## R8. How a sweep review maps onto the existing steps

- **Decision**: The sweep fills the run values (`owner`, `repo`,
  `number`, `event_time` = "now") and runs Step 1 onward. Step 3 takes the
  `synchronize` branch when the PR has a prior review and the `opened`
  branch when it has none.
- Gates and notices: the sweep posts no cap, debounce or skip notice
  (FR-009). It still runs the approval guard for gates A, B and C, which is
  not a notice: skipping it would leave a live approval on an excluded
  change.
- Two runs on one PR (two sweeps, or a sweep and a push): the Step 3.5
  ownership check settles it. No new logic.
- Candidate fallthrough: when the first candidate stops at a gate, the
  sweep may try the next, up to three, so one blocked PR doesn't waste the
  hour.
- Loops: abandoned reviews don't count toward the 10-review limit, so a PR
  whose review keeps crashing or failing to submit could get a full review
  every hour. A failed submit removes the label, and a sweep stops on a PR
  with 2 or more abandoned reviews since its last summary. One crash still
  gets one retry from the sweep.
