# Contract: documentation gaps in the Step 6 summary

## Documentation gaps line (Decision)

Order inside `### Decision`:

1. The decision line.
2. The tally line.
3. When the decision is 🔴, the sentence on what blocks approval.
4. **New:** the Documentation gaps line, when the review has a gap.
5. The 5a.1, 5a.2 and 5a.3 notes, failed checks, `head_moved` line, as
   today.

Format:

```text
📝 **Documentation gaps (5):** `MAX_RETRIES` not in the README environment table (`README.md`); stale `--dry-run` flag (`docs/cli.md`, prior); new settings page has no doc (no doc); and 2 more
```

- The count is every gap: this review's documentation findings, folded
  ones included, plus earlier ones still open or partially addressed.
- The first 3 are named, each with its title and doc location in a code
  span, or `no doc`. Earlier ones add `prior`. Then `and N more` when
  there are more.
- Present whatever the decision, including ✅ **Approved with minor
  comments**.
- Absent when there are no gaps.

## Blocker sentence

When the decision is 🔴 and the only thing over the line is the `[minor]`
count, and that count is 2 or fewer without documentation gaps, the
sentence is:

```text
Documentation gaps bring the minor count over 2, which blocks approval.
```

When other blockers exist (a `[blocking]`, Security or privacy finding,
merge conflicts), the sentence names those as today and does not mention
documentation.

## Tally and issue lines

- An earlier `[docs]` item still ❌ open or ⚠️ partially addressed counts
  under 🟡 in the tally and in `❌ N prior item still open`, as earlier
  🔴 items are counted today. Example:
  `🟡 3 minor · ❌ 2 prior items still open`.
- On the 🟡 Minor issue line, it appears with `(prior)` after its title.
- A folded gap appears on the Minor issue line by its own title.

## Reminder in Details

Inside `<details>`, after "Unread files" and before "Rule files" (which
stays last), when a reminder was posted this round:

```text
**Documentation reminder:** changed `MAX_RETRIES`, `/api/v2/projects`. Please check product pages and external docs; the author and maintainers own those updates.
```

It is not a finding and is not counted anywhere.
