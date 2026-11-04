# Data Model: Reviews that fit, or say why they can't

All state lives on GitHub or inside one run. Nothing is stored between
runs except comments, reviews and the pending label.

## Size limits (prompt table)

Fixed values in `PROMPT.md`, tuned by maintainers (research R7, R8).

| Field | Value | Used by |
| --- | --- | --- |
| `max_changed_lines` | 20,000 (`additions + deletions`) | Gate A1 |
| `max_changed_files` | 300 (`changed_files`) | Gate A1 |
| `max_review_comments` | 150 (`review_comments`) | Gate A1 |
| `max_file_bytes` | 100,000 | Every full-file read |
| `max_run_bytes` | 300,000 | Sum of all full-file reads in one run |
| Rule file caps (existing) | 24,000 per file, 80,000 total | Step 4 rule files; count toward `max_run_bytes` |

## Read budget (per run)

- `bytes_read`: the sum of the tree-listed `size` of every file read in
  full so far, starting at 0. Rule files count.
- A read is allowed only when the file has a listed `size`, `size <=
  max_file_bytes`, and `bytes_read + size <= max_run_bytes`.
- A file without a listed size (tree read failed, path not found in its
  directory's listing) is not read.

## Unread file (per run)

| Field | Values |
| --- | --- |
| `path` | Repository path |
| `kind` | `changed-no-patch` (changed file whose patch GitHub omitted), `changed-with-patch` (patch reviewed, whole file not read), `context` (wanted for context only) |
| `reason` | `over the 100,000-byte limit`, `over the run total`, `size unknown`, `read failed`, `symlink` |
| `generated` | true when the path matches the generated-file and lockfile table |
| `withholds` | true only for `changed-no-patch` with `generated` false |

A deleted file is never `changed-no-patch`: there is nothing new to
review. It is named in the recap as deleted.

## Too-large notice (GitHub comment)

- Marker: `<!-- lfx-review-agent:too-large -->`.
- Comment class `too-large-notice` (Step 1.5), placed with the other
  notices. Not an attempt; not counted toward the 10-review limit.
- State per PR:
  - **Over, no notice since the last session comment** → gate A1 posts
    one.
  - **Over, notice newer than the last session comment** → gate A1 stops
    silently.
  - **Under** → no gate A1 effect; gate C does not debounce a push after
    a `too-large-notice`.
- A notice superseded in a race is rewritten to
  `<!-- lfx-review-agent:superseded notice=too-large -->`.

## Approval withheld for unread files

Reuses the existing approval-withheld review: `COMMENTED`, body line 2
`<!-- lfx-review-agent:approval-withheld -->`, then the 5a.3 note. Step 2
already treats it as a terminal review, so the next push gets a
follow-up.
