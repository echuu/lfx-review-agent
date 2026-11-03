# Contract: the pending label

Name: `lfx-review-agent-pending`. It means "a review of this PR's current
head may be owed". The sweep's label search reads it:

```text
org:linuxfoundation is:pr is:open draft:false label:"lfx-review-agent-pending" updated:<=<now - 10 minutes>
```

sorted by `updated` ascending, `per_page=10`. It is org-wide; candidates
not in `repos` are dropped and their label is removed, so labeled PRs in a
repository outside the sweep cannot fill the page every hour. "now" is
the sweep clock (research R2), known before the first search.

## Who sets it

| Moment | Action |
| --- | --- |
| Step 3.5, after the ownership check says this run owns the review, or when a comment read there fails | Add |
| Step 1.5 gate B or C stops a webhook run | Add |
| Step 6, after the recap is written | Remove, then re-read the PR; add again if `head.sha` is not the reviewed head or the read fails |
| Step 5c closes out the session (merged, closed, excluded, submit failed) | Remove |
| A sweep finds 2 or more abandoned reviews since the last summary | Approval guard, then remove (the PR waits for its next push) |
| A sweep finds the PR closed, a draft, excluded, or at the 10-review limit | Remove |
| A sweep finds no review due (Step 3) | Remove, then re-read the PR as in Step 6 |
| A pending search result is in a repository not in `repos` | Remove |
| A sweep stops at gate B or C | Leave |

## Rules

- Never stops a review. A failed add or remove is reported in Step 7.
- Removing a label that isn't there (404) counts as success.
- A superseded run never touches the label.
- The label name is a released marker once 0.1.18 ships. Do not rename it:
  open PRs carry it.
