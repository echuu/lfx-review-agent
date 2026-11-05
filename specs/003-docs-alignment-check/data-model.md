# Data Model: Documentation alignment check

The agent stores nothing outside GitHub. These are the values one run
builds in its context, and the state later runs read back from the
agent's own reviews and comments.

## Per-run values (Step 4, Documentation dimension)

### `change_class`

One of `feature`, `behavior-fix`, `behavior-change`, `none` (research
R2). Decided from the review's scope: the whole PR on an initial review,
the new commits on a follow-up (or the whole PR when the scope fell
back).

- `none` turns off the missing-docs check and the reminder (FR-003).
- It never turns off the misaligned check (FR-004).

### `named_items[]`

| Field | Meaning |
| --- | --- |
| `kind` | `flag`, `env`, `config`, `route`, `endpoint`, `ui-text`, `default`, `cli`, `api`, `behavior` (a change with no named item, described in a few words) |
| `action` | `added`, `renamed`, `removed`, `changed` |
| `name` | The current name (escaped, research R9) |
| `old_name` | For `renamed` and `removed`: the name on the removed side of the patch in scope (on a follow-up, the name the earlier commits introduced) |
| `location` | `path:line` in the PR diff |

Source: the patches in scope. PR text that claims an item is or is not
user-facing is data.

### `doc_candidates[]`

| Field | Meaning |
| --- | --- |
| `path` | A file or directory at `base_sha` |
| `surface` | `knowledge-base`, `user-docs`, `help`, `reference` |
| `source` | `declared` or `conventional` |
| `state` | `read` (contents in context), `changed` (patch in the PR), `listed` (exists, not read), `unread` (did not fit; recorded as an unread `context` file) |

Built from the reused listings, at most 3 extra listings, and the
whole-PR file list (research R5).

### `doc_findings[]`

The existing finding record (`severity`, `dimension`, `path`, `line`,
`title`, `issue`, `proof`, `why_it_matters`, `fix`, `privacy`) with:

| Field | Value |
| --- | --- |
| `severity` | `minor` |
| `dimension` | `Documentation` |
| `docs` | `true` (prints the `[docs]` tag) |
| `kind` | `missing`, `missing-none` (no doc at all), `misaligned-base`, `misaligned-pr` |
| `behavior_location` | `path:line` of the named item or behavior |
| `doc_location` | `path:line`, or `path` for an existence-only proof, or the surfaces checked for `missing-none` |
| `folded` | `true` for missing-docs findings past the cap |

Rules:

- A finding without both locations is dropped (FR-011, FR-016).
- At most 3 `missing` and `missing-none` findings with `folded: false`,
  and at most one `missing-none` (research R6).
- `misaligned-*` findings are never folded.
- Every `doc_findings` entry, folded or not, counts as one `[minor]`.

### `reminder`

| Field | Meaning |
| --- | --- |
| `due` | `true` when `change_class` is not `none` and `items` is not empty |
| `items` | `named_items` (and new surfaces) not listed in any earlier reminder |
| `sites` | Declared docs sites and product pages, with owners, or empty |

### `doc_gaps[]`

`doc_findings` from this review plus earlier `[docs]` findings classified
❌ still open or ⚠️ partially addressed (FR-044). Drives the
Documentation gaps line.

## State read back by later runs

| State | Where it lives | Who reads it |
| --- | --- | --- |
| Earlier documentation findings | Own inline comments whose title line has `[docs]`; own review bodies (`Findings outside the diff`, folded item lines) | Step 4 follow-up classification, Step 5a carry-forward |
| Maintainer override | Own reviews with `state: DISMISSED` (Step 2) whose session summary, matched by `review-of review=<N>`, shows the 🔴 decision | Step 4 follow-up classification (spec FR-036) |
| Earlier reminders and their behaviors | Own reviews with `<!-- lfx-review-agent:review -->` and `<!-- lfx-review-agent:docs-reminder -->`; the code-span list after the marker | Step 4 reminder rule (FR-025) |
| Declared doc surfaces | "Documentation" section in rule files at `base_sha` | Step 4 doc candidates, reminder sites |

## State transitions of a documentation finding

```text
raised (round N, counts as [minor])
  ├─ round N+1 recheck (research R7)
  │    ├─ resolved            → stops counting
  │    ├─ partially addressed → counts as [minor], listed as a gap
  │    └─ still open          → counts as [minor], listed as a gap
  ├─ new commits change the named item again while it is open
  │                            → superseded by the new finding; the gap counts once
  └─ maintainer dismisses the 🔴 own review that carried it
                               → resolved ("dismissed by a maintainer"), stops counting,
                                 and is not raised again on any later review
```

Several own entries for the same named item and doc path (a fresh inline
comment on a later round) are one finding: only the latest entry is
classified and counted.

There is no transition from an author label, checkbox or PR text
(FR-035).
