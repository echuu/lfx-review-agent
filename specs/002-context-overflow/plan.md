# Implementation Plan: Reviews that fit, or say why they can't

**Branch**: `feature/context-overflow` | **Date**: 2026-11-03 |
**Spec**: [spec.md](spec.md)

**Input**: Feature specification from
`specs/002-context-overflow/spec.md`

## Summary

Five review runs in 625 overflowed the model's 1,000,000-token limit and
died without a word (research R1). Four read one large file in full; one
read a large PR's history. The fix keeps every review inside a read
budget and says plainly what it could not read:

- **Know sizes before reading** (R3). A new `git_get_tree` operation lists
  a directory with file sizes at a fraction of today's cost. The prompt
  stops listing directories through `repos_get_content`.
- **Read limits** (R7). At most 100,000 bytes per full-file read and
  300,000 per run. A patch-less changed file is read at the PR head only.
  A changed file left unread withholds approval, unless it is a generated
  file or lockfile (R9). The exclusion check decides an unread file from
  its path (R10).
- **Plain text** (R4). A new `repos_get_content_raw` operation asks GitHub
  for the raw file, if Guild passes the header.
- **Trimmed history** (R6). Narrowed output schemas on the two REST
  history operations, if Guild trims responses to them. GraphQL is out:
  the probe showed Guild does not enforce body enums (R5).
- **Size check** (R8). A last-resort gate that declines a PR whose diff
  cannot fit, with one notice instead of a start comment.

## Technical Context

**Language/Version**: Prompt (`PROMPT.md`), Guild manifest (`guild.yaml`),
OpenAPI 3.0.3 integration spec.

**Primary Dependencies**: Guild Native agent runtime; custom integration
`linux-foundation~lfx-one-github` (1.0.5 to 1.0.6); GitHub REST API.

**Storage**: None. State lives on GitHub (comments, reviews, the pending
label).

**Testing**: `markdownlint`; `guild integration version test --account
linux-foundation` for the enum probe and new operations; the README
checklist and [quickstart.md](quickstart.md) on live PRs; per-session
tool output and token totals from `guild session events` and
`guild session get`, compared with the R1 data.

**Target Platform**: Guild workspace `lfx-reviews`, 29 onboarded
`linuxfoundation` repositories.

**Project Type**: Prompt-defined agent (no application code).

**Performance Goals**: No silent overflow (SC-001). #4962-sized
follow-ups read at most half of 2.11 MB (SC-004).

**Constraints**: The model limit is 1,000,000 tokens per call. The agent
cannot measure response sizes, only add up sizes it was given (tree
listings, `pulls_get` counts). No new operation may let the agent change
GitHub state (FR-011). Integration versions cannot be unpublished.

**Scale/Scope**: About 300 reviews a day across the onboarded
repositories (R1 window).

## Constitution Check

There is no filled-in `.specify/memory/constitution.md`. The
repository's `CLAUDE.md` rules serve as the gates.

| Gate | Status |
| --- | --- |
| Released markers unchanged. New: `too-large` marker and comment class; `superseded notice=too-large`. | Pass (additive) |
| `PROMPT.md` and `guild.yaml` change, so the agent version goes to 0.1.19. | Planned |
| Every new tool is in `guild.yaml`, the prompt's Tools table and the README credential policy. | Planned |
| The integration is published before `guild.yaml` pins it, and the policy exists before the save. | Planned (rollout) |
| The maintainer runs every Guild command that uses the credential. | Planned (quickstart steps 1, 2, 5) |
| PR content is untrusted. File paths decide exclusions for unread files only by fixed rules (R10). | Planned |
| No new operation can mutate GitHub state. Both new operations are GET. | Pass |
| Prose follows unslop. License headers unchanged (no new script). | Planned |

Re-checked after design: no violations, so no Complexity Tracking
entries.

## Design

### Tools (integration 1.0.6)

See [contracts/integration-ops.md](contracts/integration-ops.md).

- `git_get_tree`: directory listings with sizes. Replaces every
  `repos_get_content` call on a directory.
- `repos_get_content_raw`: file text with `Accept:
  application/vnd.github.raw+json`. Used only after quickstart step 2
  shows Guild sends the header.
- Narrowed output schemas on `pulls_list_reviews` and
  `pulls_list_review_comments`.

### Read budget (new section in `PROMPT.md`, near "Tools")

See [contracts/read-budget.md](contracts/read-budget.md). A table of
limits the maintainers can tune, the rule that every full-file read
needs a size from a tree listing first, and how each skipped read is
reported.

### Generated files and lockfiles (new table in `PROMPT.md`)

Next to "Review exclusions": the R9 patterns, with the rule that a
listed file is named but never withholds approval.

### Changes to existing steps

- **Step 1.5**: new comment class `too-large-notice`; new gate A1 (size
  check, [contracts/too-large-notice.md](contracts/too-large-notice.md))
  after gate A and before gate B, covered by the approval guard; gate C
  lets a push through after a `too-large-notice`; the duplicate notice
  check gains type `too-large`; a skip-notice reason for gate A1.
- **Pending label**: a sweep removes the label at gate A1.
- **Step 0.3**: a sweep may post the too-large notice, like the stalled
  notice.
- **Step 3.2**: a patch-less changed file is read at `head_sha` only, and
  only within the read budget. An unread file is decided from its path
  (R10).
- **Step 4, Gather the material**: list directories with `git_get_tree`
  (rule files, context). Every full-file read checks size and budget
  first. Rule files keep their caps and count toward the run total. The
  symlink rule reads mode `120000`.
- **Step 5a**: new gate 5a.3, "Unread changed files": withhold approval
  with the approval-withheld marker when a patch-less changed file outside
  the R9 list was not read. Same mechanics as 5a.2.
- **Step 6**: the recap gets an "Unread files" line, and the summary
  names withholding files at the top.
- **Step 7**: report unread files and the gate A1 stop.
- **Error handling**: a failed `git_get_tree` read means the file's size
  is unknown, so the file is not read (treated as over the limit).

### Repository files

- `integration/lfx-one-github.openapi.yaml`: new operations, narrowed
  history schemas, version 1.0.6.
- `guild.yaml`: pin 1.0.6, add the tools.
- `README.md`: credential policy, test checklist items.
- `CLAUDE.md`: `:too-large` joins the read-back markers; the
  generated-file table joins the editing notes next to review exclusions.

## Project Structure

### Documentation (this feature)

```text
specs/002-context-overflow/
├── spec.md
├── plan.md              # This file
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── integration-ops.md
│   ├── read-budget.md
│   └── too-large-notice.md
├── checklists/
│   └── requirements.md
└── tasks.md             # /speckit-tasks output (not created yet)
```

### Source Code (repository root)

```text
PROMPT.md                                  # Tools, read budget, generated files, Steps 0.3, 1.5, 3.2, 4, 5a, 6, 7
guild.yaml                                 # integration 1.0.6, new tools
integration/lfx-one-github.openapi.yaml    # new operations, narrowed schemas
README.md                                  # credential policy, checklist
CLAUDE.md                                  # markers, editing notes
```

**Structure Decision**: The repository is a prompt plus a manifest. The
feature touches the existing files above and adds no script.

## Rollout

1. Enum probe (quickstart step 1): done 2026-11-03. Guild does not
   enforce body enums, so `graphql_read` is out (R5).
2. Implement on this branch and open the PR.
3. Publish integration 1.0.6, create the ALLOW policy, run quickstart
   step 2 (tree, raw header, history schemas). If Guild drops the
   `Accept` header, change the prompt to keep `repos_get_content` before
   merging.
4. Merge, then `guild agent save --publish --version-number 0.1.19`.
5. Measure (quickstart step 5) and tune the read limits in a patch
   release if plain text holds.

## Open decisions

- **Unread generated files and lockfiles**: named, no withholding
  (clarified 2026-11-03).
- **Exclusion check on unread files**: decided from the path (clarified
  2026-11-03).
- **GraphQL**: out until a read-only token exists (#19); the enum probe
  failed (R5).
- **Unenforced label pins** (R12): not part of this feature. The
  maintainer decides before publishing 0.1.18.
- **Plain text**: depends on Guild sending the header (R4).
- **Output-schema trimming**: a question for Guild (R6); 1.0.6 shows the
  answer either way.
