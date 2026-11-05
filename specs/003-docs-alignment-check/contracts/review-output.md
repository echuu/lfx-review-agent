# Contract: documentation findings and the reminder in the review

## Inline comment title line (Step 5b)

```text
[minor][docs] <short title>

Issue: <what the doc is missing or gets wrong>
Proof: <behavior at path:line>; <doc at path:line, or the surfaces checked>
Why it matters: <who is misled, and how>
Fix: <the doc change; a one-line pointer is enough>
```

- `[docs]` follows the severity, where `[privacy]` goes for privacy
  findings. Only missing-docs and misaligned findings carry it.
- A finding whose location is not in the diff goes under `Findings
  outside the diff` with the same title line and `path:line` in the
  title.
- The 5c 422 retry moves inline findings into the body and keeps the
  tag.
- Anchor: a misaligned finding on a doc the PR changes anchors on the
  doc line. Every other documentation finding anchors on the behavior
  line when it is in the diff.

## Folded item (Step 5c body, under `Findings outside the diff`)

```text
[minor][docs] 2 more documentation gaps

- `RETRY_BACKOFF` (`src/config.ts:41`) is not in the environment table in `README.md`
- `--verbose` (`cmd/root.go:88`) is not in `docs/cli.md`
```

One line per folded gap. Each line counts as one `[minor]`, and a
follow-up classifies each line on its own.

## Documentation section (Step 5c body)

Placed after the 1 to 2 sentence verdict note and before `Findings
outside the diff`. The first line of the body keeps the review and
`review-of` markers, and the second line stays reserved for the
approval-withheld marker.

```text
**Documentation**
<!-- lfx-review-agent:docs-reminder -->
This pull request changes behavior that may also be described outside this repository, such as product pages or an external docs site. I can't read those, so please check them; the author and maintainers own those updates. Changed here:

- `MAX_RETRIES` (new environment variable)
- `/api/v2/projects` (route renamed from `/api/projects`)

Declared: docs site `https://docs.example.org/app` (owner `lfx-docs`)
```

- Present only when the reminder is due (data-model `reminder.due`).
- The `Declared:` line appears only when the repository declares sites.
  Without one, the sentence stays generic.
- The text never says a page is missing, wrong or out of date.
- The behaviors are the ones not named by any earlier reminder on this
  PR (spec FR-025).

## Reminder marker

`<!-- lfx-review-agent:docs-reminder -->`, new. A later run treats an own
review (`user.login` is the reviewer login, body contains the review
marker) whose body contains this marker as a posted reminder, and the
code-span bullets after it as the behaviors it named. A `head_moved`
`COMMENT` review counts. Never rename it once released.

## Escaping (all output)

Every name, path, URL and owner echoed from the PR or the repository:

1. Remove every `<`, `>`, backtick and line break. Removing the angle
   brackets themselves means no comment delimiter can survive or be put
   back together.
2. Cut to 100 characters.
3. Put it in a code span.

This keeps PR text from planting a marker in the agent's own review, and
keeps `@team` from notifying anyone.
