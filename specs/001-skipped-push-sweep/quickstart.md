# Quickstart: validate the hourly sweep

The maintainer runs every Guild command here. `<workspace-id>` and other
IDs are placeholders: this repository is public.

## 1. Probe the time trigger (done 2026-11-03)

Settles R1 and R2: the message a time-triggered run receives, and whether
it carries the firing time. Guild has no trigger delete, so the probe is
created once under the sweep trigger's final name and reused at rollout.

This creates a live trigger. Today's prompt stops at Step 0 on any input
it doesn't recognize, so the run posts nothing and costs one short run.

```bash
guild trigger create --workspace <workspace-id> --type time \
  --agent linux-foundation~lfx-review-agent --name "lfx-review-agent sweep" \
  --frequency HOURLY --minutes-of-hour 17 \
  --input '{"lfx_review_agent_event":"sweep","repos":["linuxfoundation/lfx-review-agent"]}'
```

After the next `:17`:

```bash
guild trigger sessions <sweep-trigger-id> --limit 1
guild --mode json session events <session-id> --events all
guild trigger deactivate <sweep-trigger-id>
```

Record in `research.md`: the first event's text, whether the JSON appears
as an object, and whether a timestamp appears.

## 2. Test the new operations with the `lfx-one` credential

After integration 1.0.5 is published and the policy is created. Run with
`--account linux-foundation`.

- `search_issues`, repository-scoped, with `-reviewed-by:lfx-one
  -commenter:lfx-one`: expect HTTP 200. Repeat org-wide with
  `reviewed-by:lfx-one` to record whether `lfx-one` can search itself.
- `search_issues` with the label query from
  [contracts/pending-label.md](contracts/pending-label.md): expect 200 and
  `total_count: 0`.
- `issues_add_labels` on a test PR in `linuxfoundation/lfx-review-agent`:
  expect the label to be created and added. If it returns 403 or 422, the
  label must be created by hand in each repository, and the plan changes.
- After the add, read the PR and record whether `updated_at` moved to the
  time of the label call, in research R3. If it moved, the 10-minute wait
  restarts at the label add, which pushes the SC-001 worst case out by up
  to 10 minutes.
- `issues_remove_label` on the same PR, twice: expect 200, then 404. After
  the first call, read the PR and confirm the label is gone, so a 404 can
  never hide a remove that doesn't work.

## 3. Rollout order

1. Publish integration 1.0.5 and check that it lists 20 operations.
2. Create the ALLOW policy for the new operations.
3. Merge the PR and run `guild agent save --publish --version-number
   0.1.18 -m "..."`.
4. Run `scripts/sweep-trigger.sh` to set the full `repos` input, then
   `guild trigger activate <sweep-trigger-id>`.

Skipping step 1 or 2 makes the save fail, as it did for 0.1.15.

## 4. Validation scenarios

| Scenario | Expected |
| --- | --- |
| Idle hour (no pending label, no never-reviewed PR) | Session posts nothing; tool output stays within a few KB (SC-004). |
| Push while a review runs | Gate B adds the label; the next sweep runs a follow-up whose start comment says the hourly check started it. |
| Two pushes within 5 minutes of a summary | Gate C posts its debounce notice and adds the label; the next sweep reviews the second push. |
| Push during Step 6 | Step 6 removes the label, sees the head moved, adds it back; the next sweep reviews. |
| Run crashes after Step 3.5 | The label stays; a sweep at least 45 minutes later closes out the start and reviews. |
| PR at the 10-review limit with the label | The sweep removes the label and posts nothing. |
| Review crashes again after the sweep's retry | The sweep sees 2 abandoned reviews, removes the label and posts nothing; the next push starts a review. |
| One never-reviewed search fails (403 or 422) | The sweep keeps the pending candidates, skips the other never-reviewed searches, and reports the failure. |
| PR opened during an outage, no agent comment | The sweep runs its initial review within the hour, if it's under 7 days old. |
| Same PR picked by a sweep and a push | One review; the other run is superseded (Step 3.5). |
