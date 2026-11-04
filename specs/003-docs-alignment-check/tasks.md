# Tasks: Documentation alignment check

**Input**: Design documents from `specs/003-docs-alignment-check/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md),
[research.md](research.md), [data-model.md](data-model.md),
[contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: The spec asks for no automated tests. The repository has no
test code: a change is checked by `markdownlint`, the CI workflows (Agent
Version Check, License Header, DCO), and the replay and scenarios in
[quickstart.md](quickstart.md) sections 3 and 4. Those checks are the
validation tasks below.

**Organization**: Foundational tasks build what every story shares: doc
candidates, change class and named items, the `[docs]` tag, escaping,
carry-forward, and the summary call-out. Then one phase per user story,
in priority order: US1 (P1), US2 and US3 (P2), US4 (P3). Almost every
task edits `PROMPT.md`, so tasks run one after another, not side by side.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different file, no open dependency).
- **[Story]**: The user story the task serves (US1 to US4).
- **(maintainer)**: The maintainer runs it. It uses the Guild credential
  or changes live behavior. Claude gives the full command, with IDs
  filled in, in chat only.

## Path conventions

Repository root: `PROMPT.md`, `README.md`, `CLAUDE.md`. `guild.yaml`,
`integration/` and `scripts/` do not change. Line numbers below are from
`main` at c485311 and move as tasks land; find each place by its heading
or quoted text.

---

## Phase 1: Setup

- [ ] T001 Create branch `feature/docs-alignment-check` from an updated
  `main`, and commit the `specs/003-docs-alignment-check/` artifacts as
  the first commit (`git commit -S --signoff`, type `docs(spec)`).
- [ ] T002 [P] (maintainer) Run quickstart section 1 (sandbox
  repository, forked test agent in its own directory, test workspace,
  triggers) and record the result in
  `specs/003-docs-alignment-check/research.md` R12: "harness confirmed"
  or "fallback: monitoring after publish".
- [ ] T003 [P] Pick the replay set. Get the onboarded repositories the
  way `scripts/sweep-trigger.sh` does: the `repo` of each active
  `pull_request synchronize` trigger for the agent in `guild --mode json
  trigger list --workspace <lfx-reviews ID>`. Then find about 20 PRs
  merged in the last 30 days with `env -u GH_TOKEN -u GITHUB_TOKEN gh pr
  list --repo <repo> --state merged`: at least 5 `feat`, 5 `fix`, and 5
  chore, refactor, test, CI or dependency PRs, at least 3 that rename or
  remove a documented item, 2 multi-round cycles. Write it to
  `replay-set.md` in the session scratchpad or another directory outside
  this repository, never under `specs/`: the repository is public, some
  onboarded repositories are private, and `scripts/sweep-trigger.sh`
  forbids writing the repository list to a file here. One row per PR:
  repo, number, class by title, why it was picked.

---

## Phase 2: Foundational (blocks every story)

- [ ] T004 In `PROMPT.md` "Read budget" (line 244, "Order: Step 3.2 reads
  first, ..."), add documentation reads (Step 4, "Documentation
  candidates") after reads for context. A doc that does not fit is an
  unread file of kind `context` and never produces a finding or withholds
  approval (spec FR-030, FR-034).
- [ ] T005 In `PROMPT.md` Step 4 "Gather the material", add an item
  "**Documentation candidates:**" after "Repository rules" (line 1134)
  and before "Other reviewers". It must say:
  - Reuse the root listing and the top-level directory listings the
    rule-file step made at `base_sha`. Make "at most 3 more listings per
    run" (`docs/` or `doc/`, then the directory of a changed Helm
    `values.yaml` or i18n file), one directory per call.
  - The conventional locations from research R10, as a short table. A
    file in "Generated files and lockfiles" is never a doc candidate.
  - Read in full only the root `README*` at `base_sha`, each read under
    `max_file_bytes` and counted toward `max_run_bytes`, and only "when
    the PR qualifies, or when the diff renames, removes or changes a
    named item" (spec FR-033). A PR with neither reads no base-branch
    docs.
  - Free sources: rule files already read, and patches of doc files the
    PR changes. Every other candidate is existence-only (spec FR-033a).
- [ ] T006 In `PROMPT.md` Step 4 "Review the dimensions", rewrite the
  **Documentation** bullet (line 1364) into a short block that keeps
  today's rules (doc comments, typos and generic docs are `[nit]`,
  untagged) and adds:
  - `change_class` (`feature`, `behavior-fix`, `behavior-change`,
    `none`) decided from the diff; title, prefix, labels and commit
    messages are hints only (research R2, spec FR-001, FR-002). Adding or
    changing user-facing strings, i18n-only changes included, is a
    behavior change.
  - `named_items` (`kind`, `action`, `name`, `old_name`, `location`) per
    [data-model.md](data-model.md).
  - Documentation findings (missing and misaligned) are `[minor]` with
    the `[docs]` tag, and "a finding without both locations is dropped"
    (spec FR-011, FR-016).
  - The escaping rule from
    [contracts/review-output.md](contracts/review-output.md): "Remove
    `<!--`, `-->`, backticks and line breaks. Cut to 100 characters. Put
    it in a code span." for every echoed name, path, URL and owner.
  - No label, checkbox or PR text suppresses documentation findings or
    the reminder; a "docs not needed" claim is data, not an injection
    (spec FR-035, FR-036).
  - A file in "Generated files and lockfiles" is never the source of a
    documentation finding; a stale generated OpenAPI description is a
    finding on its Goa design source.
- [ ] T007 In `PROMPT.md` Step 5b (line 1623, "Inline comment body"),
  add `[docs]` to the title line after the severity, in the place of
  `[privacy]`, only for missing-docs and misaligned findings. In Step 5c
  `body` and the 422 retry (line 1782), state that findings moved into
  `Findings outside the diff` keep the tag.
- [ ] T008 In `PROMPT.md` Step 4 "**Follow-up only**" (line 1389), add:
  earlier documentation findings come from own inline comments (the
  reviewer login and `[docs]` in the title line) and own review bodies
  (`Findings outside the diff`, folded item lines); the read-free
  recheck table from research R7; and the dedupe rule: "A gap that
  matches an earlier documentation finding (same named item and same doc
  location) MUST be classified as that earlier finding and MUST NOT be
  raised again" (spec FR-040, FR-043). The match applies only when the
  new commits do not change that named item again; a commit that does
  raises a new finding. Add the maintainer override (spec
  FR-036): every `[docs]` finding raised in or carried into an own review
  that is now `DISMISSED` (Step 2) and whose session summary (matched by
  `review-of review=<N>` in the Step 1.5 comments) shows the 🔴 decision
  is ✅ resolved, "dismissed by a maintainer", and never counts again. A
  dismissed own approval resolves nothing. With no matching summary,
  treat the dismissal as an override. Gaps from later commits are
  new findings.
- [ ] T009 In `PROMPT.md` Step 5a (line 1423, "Prior `[minor]` items
  outside Security do not count toward the `[minor]` total"), add the
  exception: an earlier `[docs]` item still ❌ open or ⚠️ partially
  addressed counts as one `[minor]` on every follow-up until resolved
  (spec FR-042). Keep the five Approve bullets unchanged.
- [ ] T010 In `PROMPT.md` Step 6 "Recap contents", per
  [contracts/summary-call-out.md](contracts/summary-call-out.md):
  - Decision (line 1888): the Documentation gaps line after the 🔴
    sentence and before the 5a notes, whenever the review has a gap; the
    first 3 named with doc location or `no doc`, `prior` on earlier
    ones, then `and N more`; absent with no gaps (spec FR-044, FR-045,
    FR-047).
  - The blocker sentence "Documentation gaps bring the minor count over
    2, which blocks approval." only when the `[minor]` count is 2 or
    fewer without the gaps and nothing else blocks (FR-046).
  - Tally (line 1904) and Minor issue line (line 1964): earlier `[docs]`
    items still open count under 🟡 and in `❌ N prior item still open`,
    with `(prior)` on the Minor line.

**Checkpoint**: The shared rules are in place. No check raises a
documentation finding yet.

---

## Phase 3: User Story 1 - Contradicting docs are flagged (Priority: P1) MVP

**Goal**: A doc that names something the PR renames, removes or changes,
or a doc in the PR that contradicts the PR's code, gets a
`[minor][docs]` finding naming both locations, on every PR.

**Independent Test**: On a sandbox PR that renames an environment
variable named in the root README, README unchanged, confirm one
`[minor][docs]` finding citing the code line and the README line,
counted in the `[minor]` tally and on the Documentation gaps line.

- [ ] T011 [US1] In the `PROMPT.md` Documentation block (T006), add the
  misaligned check (spec FR-015 to FR-017, FR-004):
  - It runs on every reviewed PR, whatever `change_class` is, docs-only
    PRs included.
  - Base-branch docs: for each `named_items` entry with `action`
    `renamed`, `removed` or `changed`, look for its `old_name` or old
    value in the docs this run has: the root README read in T005, rule
    files already read, and patches of docs the PR changes. A match in a
    doc the PR does not update is a finding.
  - PR docs: a doc line the PR adds or changes that the PR's code does
    not match is a finding, anchored on the doc line, citing the code
    line.
  - Misaligned findings are never folded or capped.
- [ ] T012 [US1] In the same `PROMPT.md` Documentation block, add the
  US1 edge cases and anchoring: a docs-only PR is still checked against
  the code at `head_sha`; a renamed item that the PR updates in every doc
  that named it yields no finding; a finding on a doc line in the diff
  anchors inline, anything else goes under `Findings outside the diff`
  (spec FR-050).

**Checkpoint**: US1 works on its own: misaligned findings appear,
count, carry forward (T008, T009) and show in the gaps line (T010).

---

## Phase 4: User Story 2 - A feature without docs is flagged (Priority: P2)

**Goal**: A qualifying PR that changes no doc covering its new or
changed behavior gets a `[minor][docs]` missing-docs finding.

**Independent Test**: On a sandbox `feat:` PR that adds an environment
variable, in a repository whose README has an environment variable
table, README unchanged, confirm one missing-docs finding. On a `chore:`
dependency bump, confirm none and no base-doc read in the session
events.

- [ ] T013 [US2] In the `PROMPT.md` Documentation block, add the
  missing-docs check (spec FR-010 to FR-013, FR-033a):
  - Only when `change_class` is not `none`.
  - The four surfaces from spec FR-005, matched against the doc
    candidates from T005.
  - No finding when the whole-PR file list from Step 3.2 already updates
    a doc that covers the change, on follow-ups too (FR-012).
  - Proof: the behavior `path:line` and the doc `path:line`, or for an
    existence-only surface the doc path, only when the PR adds a new
    item of the same kind (for example a new Helm value with a values
    README beside it).
  - A hard-coded user-facing string in a repository that keeps strings
    in i18n files is a missing-docs finding on the help surface.
  - Anchoring: on the behavior line when it is in the diff, otherwise
    under `Findings outside the diff` (spec FR-050).
- [ ] T014 [US2] In the same block, add the cap from research R6 and
  data-model: "At most 3 `missing` and `missing-none` findings with
  `folded: false`, and at most one `missing-none`". The rest fold into
  one item. In `PROMPT.md` Step 5c `body`, define the folded item format
  from [contracts/review-output.md](contracts/review-output.md): title
  `[minor][docs] N more documentation gaps`, then one line per gap with
  its title, behavior location and doc location. "Every `doc_findings`
  entry, folded or not, counts as one `[minor]`" (spec FR-014).

**Checkpoint**: US2 adds missing-docs findings; with T008 to T010 they
carry forward and appear in the gaps line.

---

## Phase 5: User Story 3 - Reminder for docs it can't see (Priority: P2)

**Goal**: A qualifying round gets one reminder naming the changed
behaviors, never claiming an external page is wrong.

**Independent Test**: On a qualifying sandbox PR, confirm the first
review has the reminder in the review body and in Details. Push a
typo-only commit: no reminder. Push a commit that adds a config key: a
reminder naming only that key.

- [ ] T015 [US3] In the `PROMPT.md` Documentation block, add the reminder
  rule (spec FR-020 to FR-025, research R4):
  - Due when `change_class` is not `none` and at least one named item or
    new surface is not in an earlier reminder.
  - Earlier reminders: own reviews in the Step 2 list whose body contains
    both `<!-- lfx-review-agent:review -->` and
    `<!-- lfx-review-agent:docs-reminder -->`; their behaviors are the
    code-span bullets after the marker. Escape new names (T006) before
    comparing. If the reviews can't be read, post the reminder.
  - The reminder never says a page is missing, wrong or out of date, and
    the agent never fetches external pages. It is not a finding and is
    not counted.
- [ ] T016 [US3] In `PROMPT.md` Step 5c `body` (line 1764), add the
  Documentation section after the 1 to 2 sentence verdict note and
  before `Findings outside the diff`, in the format from
  [contracts/review-output.md](contracts/review-output.md): `**Documentation**`,
  then `<!-- lfx-review-agent:docs-reminder -->` on its own line, the
  reminder sentence, and the behaviors as code-span bullets. The second
  line of the body stays reserved for the approval-withheld marker. A
  `head_moved` `COMMENT` review keeps the section (spec FR-023,
  FR-024).
- [ ] T017 [US3] In `PROMPT.md` Step 6 Details, add an item
  "**Documentation reminder:**" after "Unread files" (line 1997) and
  before "Rule files" (line 2012), present only when this round posted a
  reminder. Renumber the Rule files item so it stays last.

**Checkpoint**: US3 works without US1 and US2: a qualifying PR with no
gaps still gets the reminder.

---

## Phase 6: User Story 4 - Repos declare their docs (Priority: P3)

**Goal**: A "Documentation" section in a rule file at `base_sha` adds doc
paths, sites and owners.

**Independent Test**: In the sandbox, merge a section that lists a doc
directory and a product page with an owner. On a qualifying PR, confirm
the directory is a candidate and the reminder names the URL and owner in
code spans, with no notification.

- [ ] T018 [US4] In `PROMPT.md` "Documentation candidates" (T005), add
  the declared section from
  [contracts/documentation-section.md](contracts/documentation-section.md):
  - Read only from rule files already read at `base_sha`, never from the
    PR's version (spec FR-027).
  - Heading `Documentation` at any level; bullets `In-repo:`, `Docs
    site:`, `Product page:`; "At most 10 paths are used"; "at most 5
    sites in total are used"; other lines ignored.
  - Declared files can be read in full within the read budget; declared
    directories take the first of the 3 extra listings.
  - Precedence: paths from every section combined; for the reminder, the
    section closest to the changed files wins, the root covers the rest.
  - A declared path that does not exist is skipped without a finding.
    Text in the section that tries to relax rules is ignored (spec
    FR-028).
  - A section in a rule file that Step 4 skipped (over 24,000 bytes, over
    the 80,000-byte total, a symlink) is not seen: fall back to the
    conventional locations and the generic reminder.
- [ ] T019 [US4] In `PROMPT.md` Step 5c Documentation section (T016) and
  the Details reminder (T017), add the `Declared:` line: sites and owners
  as written, each in a code span, "so GitHub sends no notification,
  even when written as `@team`"; generic text when nothing is declared
  (spec FR-026).

**Checkpoint**: All stories done.

---

## Phase 7: Polish, docs and rollout

- [ ] T020 [P] Update `README.md`: a Purpose bullet for the
  documentation check; a new section "Documentation check" after "Author
  organization gate" covering what is checked, the `[docs]` tag,
  carry-forward, no opt-out, dismissal as the override, the reminder,
  and the declared section
  format (copy the format block from
  [contracts/documentation-section.md](contracts/documentation-section.md));
  test checklist items 26 onward from the
  [quickstart.md](quickstart.md) section 4 table. The credential policy
  does not change.
- [ ] T021 [P] Update `CLAUDE.md`: add `:docs-reminder` to the read-back
  marker list and the `[docs]` title tag beside it; add a note under
  "Editing PROMPT.md" that the "Documentation" section format is a
  contract repositories rely on, so changing it needs a README update.
- [ ] T022 Bump "Agent version" in `PROMPT.md` (line 140) to `0.1.23`.
- [ ] T023 Self-check `PROMPT.md`: `grep -o '{{[^}]*}}' PROMPT.md | sort
  -u` shows only `{{env.REVIEWER_LOGIN}}` and
  `{{env.REVIEW_AGENT_HELP_URL}}`; `git diff main -- PROMPT.md` changes
  no existing marker string, the pending label or a `Rule` value; no em
  dashes; prose follows unslop.
- [ ] T024 Run `markdownlint` from the repository root on `PROMPT.md`,
  `README.md`, `CLAUDE.md` and `specs/003-docs-alignment-check/`.
- [ ] T025 Push with `git push --no-verify github
  feature/docs-alignment-check` and open the PR with `env -u GH_TOKEN -u
  GITHUB_TOKEN gh pr create`, label `ai-assisted`, reviewers from
  CODEOWNERS.
- [ ] T026 (maintainer) Replay per quickstart section 3 on the test
  agent: 0.1.22 first, then the branch version, on separate sandbox
  copies of the T003 set. Load each version into the test agent from its
  own directory only (quickstart section 2), never from this checkout,
  whose `guild.json` points at the production agent. Skip when T002
  chose the fallback.
- [ ] T027 Compile the T026 results into `replay-results.md` next to
  `replay-set.md`, outside this repository: one row per PR with
  verdicts, `[docs]` findings and the maintainer's correct or not mark,
  then SC-001 to SC-008 pass or fail. Put only the pass or fail counts,
  with no repository names or PR numbers, in the PR description. Fix the
  prompt and rerun for any failure before merging.
- [ ] T028 (maintainer) Merge the PR, then from an updated `main`:
  `guild agent save --publish --version-number 0.1.23 -m
  "Documentation alignment check (spec 003)"`.
- [ ] T029 After publish, check the first 50 reviews under 0.1.23
  (`guild session list` and the summaries): Documentation gaps line
  present exactly when there is a gap (SC-008), no duplicate reminders
  (SC-004), no overflow (SC-006), and how many PRs reach the 10-review
  cap compared with the week before. When T002 chose the fallback, this is
  the replay: run all of quickstart section 3's criteria on the first 48
  hours, and restore 0.1.22 with `guild agent unpublish
  linux-foundation~lfx-review-agent` on a failure.

---

## Dependencies and execution order

### Phases

- **Setup (Phase 1)**: T001 first. T002 and T003 any time; T026 needs
  both.
- **Foundational (Phase 2)**: T004 to T010 edit `PROMPT.md`, in order.
- **User stories (Phases 3 to 6)**: all wait for Phase 2 and edit
  `PROMPT.md`, so do them in order: US1, US2, US3, US4. T019 needs T016
  and T017.
- **Polish (Phase 7)**: T020 and T021 can start after Phase 2. T022 to
  T025 after the stories. T026 to T029 in order.

### Stories

- **US1 (P1)**: needs only Phase 2. The MVP.
- **US2 (P2)**: needs Phase 2. Independent of US1 in the prompt; both
  feed the same gaps line.
- **US3 (P2)**: needs Phase 2 (escaping, change class). Independent of
  US1 and US2.
- **US4 (P3)**: extends T005 (candidates) and T016, T017 (reminder).

### Parallel opportunities

- T002 and T003 beside everything in Phases 1 to 6.
- T020 (`README.md`) and T021 (`CLAUDE.md`) beside any `PROMPT.md`
  task after Phase 2.

```text
After T001:
  PROMPT.md:   T004 → ... → T010 → T011 → T012 → T013 → T014 → T015 → ... → T019 → T022
  side tasks:  T002 | T003 | T020 | T021
Then:          T023 → T024 → T025 → T026 → T027 → T028 → T029
```

---

## Implementation strategy

### MVP (US1)

Phase 2 plus US1 flags provably wrong docs, carries them forward and
calls them out at the top of the summary. It is the lowest-risk slice:
every finding is a contradiction with two locations.

### Incremental delivery

1. Phases 1 and 2, then US1.
2. US2 (missing docs and the cap).
3. US3 (reminder and marker).
4. US4 (declared section).
5. Polish and rollout.

All of it ships in one PR and one agent version (0.1.23), since the
version check counts one bump per PR and `docs-reminder` becomes a
released marker at 0.1.23.
