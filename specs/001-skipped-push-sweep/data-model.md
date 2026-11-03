# Data model: Hourly sweep for skipped pushes

The agent keeps no store. Every value below is read from GitHub or from
the sweep trigger's input at the start of a run.

## Sweep input

The time trigger's `--input`, written by `scripts/sweep-trigger.sh`. See
[contracts/sweep-input.md](contracts/sweep-input.md).

| Field | Type | Rule |
| --- | --- | --- |
| `lfx_review_agent_event` | string | Exactly `"sweep"`. |
| `repos` | array of strings | `linuxfoundation/<name>` entries, at least one. Entries that don't match the pattern are ignored. |

## Sweep clock

| Field | Source | Rule |
| --- | --- | --- |
| `now` | The session start line, the first line of the run's first message: `* This session was started at HH:MM:SS (24h, UTC) on <Weekday>, <Month> <DD>, <YYYY>.` (R2). | Used as `event_time` for the rest of the run. Read only from the first line. |

## Candidate PR

Built only from structured search fields.

| Field | Source |
| --- | --- |
| `owner`, `repo` | `repository_url`, last two path segments. Must be in `repos`. |
| `number` | `number` |
| `kind` | `pending` (found by the label search) or `never-reviewed` (found by the R5 search). |
| `waited_since` | `updated_at` for `pending`, `created_at` for `never-reviewed`. |

Eligibility, checked by the existing steps:

- Open and not a draft (Step 1).
- No exclusion notice, under the 10-review limit, no review running, last
  own comment older than 5 minutes (Step 1.5 gates A0, A, B, C).
- A review is due (Step 3).
- Updated at least 10 minutes before `now` (search qualifier).

Order: oldest `waited_since` first. `pending` candidates come before
`never-reviewed` ones.

## Pending label

Name: `lfx-review-agent: pending`. See
[contracts/pending-label.md](contracts/pending-label.md).

```text
            Step 3.5 start, or gate B/C drop
  (none) ───────────────────────────────────────▶ pending
    ▲                                               │
    │  Step 6 finish on current head, or the sweep  │
    └────── finds the PR can't be reviewed ─────────┘
```

- Gate B or C on a sweep run leaves the label as it is.
- A superseded run (Step 3.5 ownership check) does not touch it: the
  owning run does.
- A closed PR keeps its label. The search only reads open PRs.

## Sweep run

| Field | Rule |
| --- | --- |
| `candidates_tried` | At most 3 per run. |
| `candidates_tried` stops | At the first candidate that reaches Step 3.5. |
| `reviews_started` | 0 or 1. |
| Outcome | One review, or nothing posted. Step 7 reports which PR it picked, or that none was due. |
