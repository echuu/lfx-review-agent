# Contract: read budget and unread files

The rules the prompt adds for every full-file read (spec FR-006 to
FR-008, FR-013, FR-014; research R7, R9, R10). Field names are in
[data-model.md](../data-model.md).

## Before any full-file read

1. Get the file's `size` from a `git_get_tree` listing of its directory
   at the same ref. Reuse a listing already read in this run.
2. Read the file only if `size <= max_file_bytes` and
   `bytes_read + size <= max_run_bytes`. Then add `size` to `bytes_read`.
3. Otherwise record an unread file and go on with the review.

Never call `repos_get_content` on a directory. Directory listings come
from `git_get_tree` only.

## Changed files without a patch

- Read at `head_sha` only, within the budget. The base side is not read.
- Not read: an unread file of kind `changed-no-patch`.

## Exclusion check (Step 3.2) on an unread file

Decide from the path alone:

- On the generated-file and lockfile table: never a match.
- Otherwise a match only if the path points at an excluded area. For
  `meeting-ics`: the path contains `ics`, `ical` or `calendar`
  (case-insensitive) as a name part.

## Verdict (new Step 5a.3)

Run after 5a.2 when the verdict would be `APPROVE`. If any unread file
has `withholds` true, submit `COMMENT` with the approval-withheld marker
and this note in the review body and the Step 6 summary:

```text
I couldn't read these changed files in full, so I can't approve this pull request: <paths>. GitHub doesn't show their diffs, and each is over the review agent's read limit. A maintainer needs to review them before merge.
```

A `REQUEST_CHANGES` verdict stands; the summary still names the files.

## Recap (Step 6)

One line under Details, after the rule files line:

```text
**Unread files:** `<path>` (<reason>), ...
```

Generated files and lockfiles say `generated or lockfile, not reviewed
line by line`. Context-only skips are listed too. Omit the line when
nothing was skipped.
