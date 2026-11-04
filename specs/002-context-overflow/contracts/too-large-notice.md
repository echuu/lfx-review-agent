# Contract: gate A1 and the too-large notice

Spec FR-001 to FR-005; research R8.

## Gate A1. PR too large for one review

Runs in Step 1.5 after gate A and before gate B, from the Step 1
`lfx_one_github_pulls_get` response. The PR is over the limit when any
of these holds:

- `additions + deletions > max_changed_lines`
- `changed_files > max_changed_files`
- `review_comments > max_review_comments`

A missing field counts as 0, so the check errs toward reviewing.

When over:

1. Run the approval guard in its gate A1 form: no Step 3.2 check (it
   reads every patch of the oversized diff). If there is a live own
   approval, revoke it with a `REQUEST_CHANGES` review, no inline
   comments, whose body starts with `<!-- lfx-review-agent:too-large -->`.
   The approval guard counts that marker; Step 2 does not, so cycle state
   is unchanged.
   Step 5c of a review already running checks for this revoke and never
   turns it back into an approval.
2. If a `too-large-notice` is newer than the PR's most recent session
   comment (any `session-start`, `session-summary` or `abandoned`), stop
   silently. A `re-review` gets a skip notice.
3. Otherwise post the notice below, run the duplicate notice check with
   type `too-large`, and stop.
4. On a sweep, remove the pending label. A webhook run does not add it.

```text
<!-- lfx-review-agent:too-large -->
@<login> This pull request is too large for an automated review (<reason>), so I won't start one. A human review is needed. If the pull request gets smaller, the next push, or a comment that begins with a mention of me followed by `re-review`, starts a review. For help with the review agent, see {{env.REVIEW_AGENT_HELP_URL}}.
```

`<reason>` names the first limit exceeded, for example `21,400 changed
lines; the limit is 20,000`. For a `re-review`, add the `request` line
after the marker, as for the cap notice.

## Other steps

- **Comment classes**: `too-large-notice` sits with the other notices,
  before `abandoned`. Not an attempt.
- **Gate C**: a `too-large-notice` as the most recent own comment does
  not debounce; go to gate D.
- **Step 0.3**: a sweep may post this notice, as it may the stalled
  notice.
- **Skip notice table**: gate A1, "this pull request is too large for an
  automated review (see my earlier note)."
- **Step 7**: report `Review not started: PR too large (<reason>)`.
