# Contract: the pending label

Name: `lfx-review-agent: pending`. It means "a review of this PR's current
head may be owed". The sweep's label search reads it:

```text
org:linuxfoundation is:pr is:open draft:false label:"lfx-review-agent: pending"
```

sorted by `updated` ascending, `per_page=3`. This is the sweep's first
search. After it, read the clock (research R2), then drop every item whose
`updated_at` is less than 10 minutes before now. Oldest-first order puts
the eligible items first, so the drop only trims the end of the page.

## Who sets it

| Moment | Action |
| --- | --- |
| Step 3.5, after the ownership check says this run owns the review | Add |
| Step 1.5 gate B or C stops a webhook run | Add |
| Step 6, after the recap is written | Remove, then re-read the PR; add again if `head.sha` is not the reviewed head |
| Step 5c closes out the session (merged, closed, excluded, submit failed) | Remove if merged, closed or excluded; leave it after a failed submit |
| A sweep finds the PR closed, a draft, excluded, at the 10-review limit, or with no review due | Remove |
| A sweep stops at gate B or C | Leave |

## Rules

- Never stops a review. A failed add or remove is reported in Step 7.
- Removing a label that isn't there (404) counts as success.
- A superseded run never touches the label.
- The label name is a released marker once 0.1.18 ships. Do not rename it:
  open PRs carry it.
