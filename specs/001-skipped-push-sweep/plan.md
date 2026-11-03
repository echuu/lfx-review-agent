# Implementation Plan: Hourly sweep for skipped pushes

**Branch**: `feature/20-hourly-sweep` | **Date**: 2026-11-02 |
**Spec**: [spec.md](spec.md)

**Input**: Feature specification from
`specs/001-skipped-push-sweep/spec.md`

## Summary

An hourly Guild time trigger starts the review agent with a sweep input.
The sweep finds at most one PR that is owed a review and runs that review
through the existing steps. To keep idle hours cheap, the agent marks PRs
that may be owed a review with a `lfx-review-agent: pending` label (added
when a review starts or a push is dropped, removed when a review finishes
on the current head), so the sweep's main search is normally empty. A
second, repository-scoped search finds PRs opened in the last 7 days that
the agent never touched (User Story 4).

The maintainer confirmed the label design (research R3, Shape A) on
2026-11-02, knowing it is visible to authors on every onboarded
repository.

## Technical Context

**Language/Version**: Prompt (`PROMPT.md`), Guild manifest (`guild.yaml`),
OpenAPI 3.0.3 integration spec, bash 3.2 script.

**Primary Dependencies**: Guild Native agent runtime; custom integration
`linux-foundation~lfx-one-github` (1.0.4 to 1.0.5); GitHub REST API;
Guild CLI v0.28.0 time triggers.

**Storage**: None. State lives on GitHub (comments, reviews, the pending
label) and in the sweep trigger's input.

**Testing**: `markdownlint`; `guild integration version test --account
linux-foundation` for new operations; the README checklist and
[quickstart.md](quickstart.md) scenarios on live PRs.

**Target Platform**: Guild workspace `lfx-reviews`, 29 onboarded
`linuxfoundation` repositories.

**Project Type**: Prompt-defined agent (no application code).

**Performance Goals**: A dropped push reviewed within 75 minutes (SC-001).
An idle sweep run's tool output within a few KB (SC-004).

**Constraints**: At most one review per sweep run and at most 3
candidates tried. Search queries of 256 characters or less (about six
`repo:` qualifiers). No repository list, workspace ID or credential ID in
the public repository. The agent has no clock (research R2).

**Scale/Scope**: 720 sweep runs a month; at most 24 sweep reviews a day.

## Constitution Check

There is no `.specify/memory/constitution.md`. The repository's
`CLAUDE.md` rules serve as the gates.

| Gate | Status |
| --- | --- |
| Released markers unchanged. New marker text (the label name, the sweep start wording) is additive. | Pass |
| `PROMPT.md` and `guild.yaml` change, so the agent version goes to 0.1.18. | Planned |
| Every new tool is in `guild.yaml`, the prompt's Tools table and the README credential policy. | Planned |
| The integration is published before `guild.yaml` pins it, and the policy exists before the save. | Planned (rollout order) |
| The maintainer runs every Guild command, and is told before any trigger changes live behavior. | Planned (probe and activation steps) |
| A new Step 0 event is documented in the README trigger table and created by a script. | Planned (`scripts/sweep-trigger.sh`) |
| PR content is untrusted. The sweep reads only structured search fields. | Planned (R7) |
| No repository list, workspace ID or credential ID in git. | Planned (trigger input, placeholders) |
| Prose follows unslop. License headers on the new script. | Planned |

Re-checked after design: no violations, so no Complexity Tracking entries.

## Design

### Step 0: sweep event

- Recognize the sweep by the top-level input key only, with no
  `pull_request`, `comment` or `issue` object
  ([contracts/sweep-input.md](contracts/sweep-input.md)).
- Add a sweep column to the run-values table: `owner`, `repo`, `number`
  from the picked candidate; `event_time` from the sweep clock.
- Add the exception to "Do not re-discover the PR via list or search
  tools": only the sweep searches, and only in the new sweep step.

### New step: Step 0.3 "Pick a PR (sweep only)"

1. Run the label search
   ([contracts/pending-label.md](contracts/pending-label.md)), which has
   no time qualifier.
2. Read the clock (R2). Drop label results updated less than 10 minutes
   ago. Then, unless 3 candidates are in hand, run the never-reviewed
   search (R5) in groups of six repositories. Keep only items in `repos`.
   Read only structured fields (R7).
3. Order the candidates (data-model.md) and take the first. Run Step 1
   onward with it.
4. If it stops at Step 1, a gate or Step 3, post nothing, apply the label
   rule for that stop, and try the next candidate, up to 3.
5. With no candidate left, stop and report that none was due.

### Changes to existing steps

- **Step 1.5 gates**: on a sweep run, never post a cap, debounce or skip
  notice. Keep the approval guard. Gates B and C on a webhook run add the
  pending label before stopping.
- **Step 3**: a sweep takes the `synchronize` branch when the PR has a
  prior review, and the `opened` branch when it has none.
- **Step 3.5**: add the label after the ownership check. The start comment
  for a sweep says the hourly check started the review.
- **Step 5c and Step 6**: remove the label, re-read the head, and add it
  back if it moved (pending-label contract).
- **Step 7**: report the sweep outcome and any label failure.
- **Scope**: list the sweep. Stall detection stays "not handled".
- **Tools**: add the new tools
  ([contracts/integration-ops.md](contracts/integration-ops.md)).

### Repository files

- `integration/lfx-one-github.openapi.yaml`: 3 or 4 new operations,
  version 1.0.5.
- `guild.yaml`: pin 1.0.5, add the tools.
- `scripts/sweep-trigger.sh` (new): builds `repos` from the webhook
  triggers and creates or updates the time trigger. Dry run by default.
- `README.md`: trigger table row, credential policy, onboarding step
  (re-run the sweep script), test checklist items.
- `CLAUDE.md`: the label name joins the list of read-back markers.

## Project Structure

### Documentation (this feature)

```text
specs/001-skipped-push-sweep/
├── spec.md
├── plan.md              # This file
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── sweep-input.md
│   ├── integration-ops.md
│   └── pending-label.md
├── checklists/
│   └── requirements.md
└── tasks.md             # /speckit-tasks output (not created yet)
```

### Source Code (repository root)

```text
PROMPT.md                                  # Step 0, new Step 0.3, Steps 1.5, 3, 3.5, 5c, 6, 7
guild.yaml                                 # integration 1.0.5, new tools
integration/lfx-one-github.openapi.yaml    # new operations
scripts/onboard-repo.sh                    # unchanged
scripts/sweep-trigger.sh                   # new
README.md                                  # triggers, policy, checklist
CLAUDE.md                                  # read-back marker list
```

**Structure Decision**: The repository is a prompt plus a manifest. The
feature touches the existing files above and adds one script.

## Rollout

1. Probe the time trigger (quickstart step 1), then record the result in
   `research.md`. This settles whether `rate_limit_get` is needed.
2. Implement on this branch and open the PR.
3. Publish integration 1.0.5, create the ALLOW policy, run the quickstart
   step 2 tests.
4. Merge, then `guild agent save --publish --version-number 0.1.18`.
5. Run `scripts/sweep-trigger.sh`, then activate the sweep trigger.

## Open decisions

- **Visible label** (R3, Shape A): confirmed 2026-11-02.
- **Clock source** (R2): set by the probe.
