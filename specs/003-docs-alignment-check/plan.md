# Implementation Plan: Documentation alignment check

**Branch**: `feature/docs-alignment-check` |
**Date**: 2026-11-04 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from
`specs/003-docs-alignment-check/spec.md`

## Summary

The review flags in-repo docs that a PR leaves missing or contradicts,
calls the gaps out at the top of the summary, and reminds the author
about docs it cannot see. Everything lives in the Documentation dimension
of `PROMPT.md`, with small rules in Steps 4, 5a, 5b, 5c and 6 (research
R1). No new tool.

- **Change class and named items** (R2). Decided from the diff. They
  gate the missing-docs check and the reminder; the misaligned check
  runs on every PR.
- **`[docs]` tag** (R3). Missing-docs and misaligned findings are
  `[minor][docs]`, so a follow-up can find and count them.
- **Carry-forward** (R7). Still-open `[docs]` findings keep counting as
  `[minor]` on every follow-up, rechecked from the PR file list and
  patches without extra reads.
- **Bounded doc reads** (R5). Reuse the rule-file listings, at most 3
  more listings, and read only the root README and declared files, only
  when the diff gives a reason.
- **Reminder** (R4). In the review body after the verdict note, with a
  new `docs-reminder` marker and the behaviors it named, and in Details.
  Once per qualifying round.
- **Summary call-out** (R8). A Documentation gaps line in Decision.
- **Escaping** (R9). Every echoed name, path, URL and owner goes in a
  code span with comment markers removed.
- **Replay** (R12). A forked test agent in a test workspace against a
  private sandbox repository, if quickstart step 1 confirms it works.

## Technical Context

**Language/Version**: Prompt (`PROMPT.md`), Markdown docs.

**Primary Dependencies**: Guild Native agent runtime; integration
`linux-foundation~lfx-one-github` 1.0.7 (unchanged); GitHub REST API.

**Storage**: None. State lives in the agent's own reviews and inline
comments on GitHub (the `[docs]` tag, the `docs-reminder` marker).

**Testing**: `markdownlint`; the Agent Version Check workflow; the replay
and scenarios in [quickstart.md](quickstart.md); session events and token
totals from `guild session events` and `guild session get`.

**Target Platform**: Guild workspace `lfx-reviews`, 29 onboarded
`linuxfoundation` repositories.

**Project Type**: Prompt-defined agent (no application code).

**Performance Goals**: No overflow from doc reads (SC-006). A PR with no
reason to read docs reads none (FR-033).

**Constraints**: Read budget from spec 002 (100,000 bytes per file,
300,000 per run). No search tool. Markers are detected by "body
contains". `lfx-reviews` auto-updates, so a publish is production.

**Scale/Scope**: About 300 reviews a day.

## Constitution Check

There is no filled-in `.specify/memory/constitution.md`. The
repository's `CLAUDE.md` rules serve as the gates.

| Gate | Status |
| --- | --- |
| Released markers, the pending label and exclusion `Rule` values unchanged. New: `docs-reminder` marker and `[docs]` tag, added to the `CLAUDE.md` read-back list. | Pass (additive) |
| `PROMPT.md` changes, so the agent version goes to 0.1.23. | Planned |
| No new tool, so `guild.yaml` and the credential policy are unchanged. | Pass |
| The only `{{ }}` text is `env.REVIEWER_LOGIN` and `env.REVIEW_AGENT_HELP_URL`. | Planned |
| PR content and rule-file text are untrusted. Echoed text is escaped (R9). The declared section cannot relax rules. | Planned |
| Injection rules stay in Operating rules. A "docs not needed" claim is data, not an injection (FR-035). | Planned |
| The maintainer runs every Guild command. | Planned (quickstart) |
| Prose follows unslop; markdownlint passes. | Planned |

Re-checked after design: no violations, so no Complexity Tracking
entries.

## Project Structure

### Documentation (this feature)

```text
specs/003-docs-alignment-check/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── documentation-section.md
│   ├── review-output.md
│   └── summary-call-out.md
├── checklists/requirements.md
└── tasks.md             # /speckit-tasks, not created here
```

### Source (repository root)

```text
PROMPT.md       # all behavior
README.md       # Documentation check section, Purpose bullet, test items
CLAUDE.md       # read-back list, editing note
```

**Structure Decision**: Prompt and docs only. `guild.yaml`, the
integration spec and the scripts do not change.

## Design

### `PROMPT.md` changes

- **Agent version**: 0.1.23.
- **Read budget**: the order line gains "then documentation reads (Step
  4) with what is left". Doc reads count toward `max_run_bytes`; a doc
  that does not fit is an unread `context` file.
- **Step 4, Gather the material**: new item "Documentation candidates"
  after "Repository rules": reuse the root and top-level listings,
  at most 3 more listings (declared directories, `docs/`, a changed Helm
  or i18n directory), the "Documentation" section from the rule files
  ([contracts/documentation-section.md](contracts/documentation-section.md)),
  the conventional list (R10), and the base-doc read rule (R5).
- **Step 4, Documentation dimension**: rewritten. Keeps today's
  doc-comment and typo `[nit]` rules. Adds `change_class` and
  `named_items` (R2), the misaligned check on every PR, the missing-docs
  check for qualifying scopes, `[minor][docs]`, the cap and folding
  (R6), the reminder rule (R4), and the escaping rule (R9).
- **Step 4, Follow-up only**: earlier `[docs]` findings come from own
  inline comments and own review bodies; the read-free recheck table, the
  dedupe rule, and the maintainer override (R7, spec FR-036). The
  override reads dismissed own reviews from Step 2 and matches each to
  its session summary through `review-of review=<N>` in the Step 1.5
  comments, so it needs no extra call.
- **Step 5a**: "Prior `[minor]` items outside Security do not count
  toward the `[minor]` total" gains "except earlier `[docs]` items still
  open or partially addressed, which count one each".
- **Step 5b**: `[docs]` joins the title line after the severity, like
  `[privacy]` ([contracts/review-output.md](contracts/review-output.md)).
- **Step 5c**: the body gains the Documentation section with the
  `docs-reminder` marker after the verdict note, and the folded item
  under `Findings outside the diff`. The 422 retry keeps the tag.
- **Step 6**: the Documentation gaps line and blocker sentence in
  Decision, prior `[docs]` items under 🟡 with `(prior)`, and the
  reminder in Details before "Rule files"
  ([contracts/summary-call-out.md](contracts/summary-call-out.md)).

### Repository files

- `README.md`: Purpose bullet; new "Documentation check" section (what
  is checked, `[docs]`, carry-forward, no opt-out, the declared section
  format); test checklist items from the quickstart scenarios.
- `CLAUDE.md`: `:docs-reminder` and `[docs]` in the read-back list; a
  note that the "Documentation" section format is a contract repositories
  rely on.

### Spec changes made during planning

- FR-043 rechecks findings from the PR file list and patches only, so a
  finding never stays open because a doc could not be read (research R7).
- FR-014 folds gaps one per line with both locations, and allows at most
  one "no doc" finding per review (research R6).
- FR-040 dedupes: a gap matching an earlier `[docs]` finding is
  classified, never raised again, so a whole-PR fallback cannot count it
  twice (research R7).

### Risks

- **Absence findings.** With `[minor]`, carry-forward and no opt-out, a
  wrong "no doc" finding keeps a PR blocked until a maintainer dismisses
  the agent's review (spec FR-036).
  Limited to one per review (R6) and measured by SC-005a.
- **Existence-only findings** on surfaces the agent does not read
  (FR-033a) have weaker proof. The replay judges them separately.
- **Harness.** If the test workspace cannot use the credential, there is
  no gate before rollout: the success criteria become monitoring of the
  first 48 hours in production, with `guild agent unpublish` as the
  restore path (R12 fallback).
