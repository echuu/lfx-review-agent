# Feature Specification: Documentation alignment check

**Feature Branch**: `feature/docs-alignment-check` (proposed, not created)

**Created**: 2026-11-04

**Status**: Draft

**Input**: User description: "Documentation alignment check for the LFX
review agent". The review flags in-repo documentation that a PR leaves
missing or contradicts, and reminds the author about documentation the
agent cannot see. Amended 2026-11-04: "call out documentation gaps in
the summary".

## Context

When a PR adds a feature or fixes a bug, the docs that describe that
behavior often fall behind. These include in-repo knowledge base files,
user docs, in-product help text, and the product pages on the web. The
PR author and the maintainers own those updates, but nothing reminds them
while the PR is still open. The review agent already reviews every PR, so
it can flag documentation that is missing or misaligned and remind the
author what they own.

Today the Documentation dimension in Step 4 ("Review the dimensions")
covers missing or outdated doc comments and README gaps. It defaults
these to `[nit]` and uses `[minor]` only when docs are blatantly wrong or
contradict code. It runs on every PR, with no notion of what kind of
change the PR makes. It does not look at docs the PR did not touch, and
it says nothing about docs outside the repository.

This feature extends that dimension. It does not add a dimension, a
verdict rule (beyond the follow-up carry-forward in FR-042), or a tool.
The directory listings the agent already uses show whether doc files
exist.

Five points in the input conflicted with how the agent works today. This
spec settles them as follows:

- **Contradiction checks are not gated by change class.** The current
  Documentation dimension raises findings on any PR. A refactor that
  renames a documented environment variable must still be caught. Only
  the new missing-docs check and the external-surface reminder depend on
  the change class. Success criterion SC-001 is worded to match.
- **"Exactly one reminder" means one per qualifying round.** A PR gets the
  reminder on its first qualifying review. A later round repeats it only
  when its new commits add a behavior change that no earlier reminder
  named.
- **The finding cap never hides a `[minor]`.** It limits how many
  missing-docs findings get their own comment. Folded findings still count
  one each in the `[minor]` tally, and misaligned findings are never
  folded, so the cap cannot turn a request for changes into an approval.
- **Documentation findings carry forward.** Today a follow-up stops
  counting earlier `[minor]` findings outside Security. Still-open
  documentation findings are the one exception (Clarifications, FR-042),
  so an author cannot get an approval by pushing other fixes and leaving
  the docs alone.
- **Doc reads are bounded, not forbidden.** The agent has no code search
  tool, so finding a stale name in base-branch docs means reading those
  docs. The spec bounds which docs it reads and how much (FR-030 to
  FR-034) instead of "read only to prove a contradiction", which cannot
  be applied before the read.

## Clarifications

### Session 2026-11-04

- Q: Should a missing in-repo doc for a feature or behavior change be
  `[nit]` or `[minor]`? → A: `[minor]`, always. The goal is to make
  authors fix documentation, not ignore it.
- Q: How should an author signal that a change needs no doc updates? →
  A: They can't. There is no label, checkbox or other opt-out. Missing-docs
  findings always stand, and a human reviewer overrides them when docs
  really aren't needed (the override is a dismissal; see the last
  question).
- Q: On a follow-up, should still-open documentation findings from an
  earlier round keep counting toward the `[minor]` tally? → A: Yes, both
  missing and misaligned, until resolved. Doc fixes belong in the same PR
  and should not be deferred.
- Q: When a repository declares owners for its product pages or external
  doc site, how should the reminder show them? → A: As plain text in a
  code span, with no notification. Without a declared section the
  reminder is generic.
- Q: How far may the auto-approve rate on the replay set drop? → A: No
  cap. Every lost approval must trace to a documentation finding that
  names its locations, and a maintainer must judge at least 90% of those
  blocking findings correct.
- Q: When docs really aren't needed, how does a human override carried
  documentation findings, given that approving does not clear the
  agent's review and carry-forward would request changes again? → A: A
  maintainer dismisses the agent's review that requested changes. Later
  runs treat the documentation findings open at that review as resolved.
  New gaps from later commits are still flagged.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Docs that contradict the change are flagged (Priority: P1)

A PR renames an environment variable, changes a default, removes a CLI
flag, or moves an API route. The README or a doc page on the target
branch still names the old value, or a doc the PR edits says something
the code in the PR does not do. Today the agent only notices this when
the doc is in the diff and the contradiction is obvious. Instead, the
agent compares the names and values the PR changes against the docs it
can read, and raises a `[minor]` finding that names both places: the
code and the doc.

**Why this priority**: Wrong docs mislead readers, and a contradiction is
the only documentation problem that can be proven from the repository
alone. It also extends what the Documentation dimension already does, so
it carries the least new risk.

**Independent Test**: On a test PR that renames an environment variable
named in the root README, without touching the README, confirm one
`[minor]` Documentation finding that cites the code line and the README
line. Confirm that it counts in the `[minor]` tally.

**Acceptance Scenarios**:

1. **Given** a PR that renames a configuration key and a README at the
   target branch that names the old key, **When** the agent reviews,
   **Then** it raises a `[minor]` Documentation finding whose proof cites
   the code location and the README location.
2. **Given** a PR that edits a doc page to describe a default of 30 while
   the code in the same PR sets 60, **When** the agent reviews, **Then**
   it raises a `[minor]` finding anchored on the doc line, citing the code
   line.
3. **Given** a docs-only PR (which does not qualify) whose README edit
   names a flag the code at the head commit does not have, **When** the
   agent reviews, **Then** it still raises the misaligned finding. The
   change class does not gate this check.
4. **Given** a PR that renames a key and updates every doc that named
   it, **When** the agent reviews, **Then** it raises no misaligned
   finding.
5. **Given** three misaligned findings, **When** the agent decides the
   verdict, **Then** they count as three `[minor]` issues and the existing
   more-than-2 rule requests changes.

---

### User Story 2 - A feature without docs is flagged (Priority: P2)

A PR adds a feature, fixes a bug in documented behavior, or changes
behavior, configuration or an API, and the PR changes no docs that cover
it. The agent decides from the diff that the PR qualifies and finds the
doc surface that should have changed. It raises a missing-docs finding
that names the new or changed behavior and either the doc that covers
that area or the absence of any doc for a new user-facing surface.

**Why this priority**: This is the main gap the feature closes. It ranks
below Story 1 because "this doc should say more" is a judgment, while a
contradiction can be proven. Its findings are `[minor]`, so a PR that
leaves several gaps gets a request for changes.

**Independent Test**: On a test PR with a `feat:` title that adds an
environment variable and no README change, in a repository whose README
has an environment variable table, confirm one `[minor]` missing-docs
finding that cites the new variable and the table. On a `chore:` PR that
bumps a dependency, confirm no missing-docs finding.

**Acceptance Scenarios**:

1. **Given** a PR that adds a new environment variable and a README with
   an environment variable table that the PR leaves unchanged, **When**
   the agent reviews, **Then** it raises a `[minor]` missing-docs finding
   citing the code line and the table, and it counts in the `[minor]`
   tally.
2. **Given** a PR that adds a new user-facing page and a repository with
   no doc that covers that area, **When** the agent reviews, **Then** the
   finding cites the new page and states that no doc covers it.
3. **Given** a PR that adds a feature and also updates the doc that
   covers it, **When** the agent reviews, **Then** it raises no
   missing-docs finding.
4. **Given** a PR that only refactors code, changes tests or CI, or bumps
   dependencies, **When** the agent reviews, **Then** it raises no
   missing-docs finding.
5. **Given** a PR titled `chore:` whose diff adds a new API route,
   **When** the agent reviews, **Then** the PR qualifies: the diff
   decides, not the title.
6. **Given** a PR whose diff shows five missing-docs gaps, **When** the
   agent reviews, **Then** it reports three as separate findings, folds
   the other two into one summary item, and counts all five in the
   `[minor]` tally.
7. **Given** a PR with three missing-docs findings and no other findings,
   **When** the agent decides the verdict, **Then** the existing
   more-than-2 rule requests changes.
8. **Given** a review that requested changes for three missing-docs
   findings, **When** the author pushes commits that fix other things but
   leave the docs alone, **Then** the follow-up classifies the three as
   still open, counts them as three `[minor]` issues, and requests changes
   again.
9. **Given** the same PR, **When** the author's next push updates the
   docs, **Then** the follow-up classifies the findings as resolved and
   they stop counting.
10. **Given** a review with any documentation gap, new or carried
    forward, **When** the author reads the summary comment, **Then** the
    visible top of the summary names the gaps, without opening Details.
11. **Given** a review whose request for changes comes from documentation
    findings, **When** the author reads the Decision, **Then** the
    sentence on what blocks approval says that documentation gaps do.
12. **Given** a review that requested changes only for documentation
    gaps, **When** a maintainer dismisses it and the author pushes again
    without doc changes, **Then** the follow-up classifies those gaps as
    resolved ("dismissed by a maintainer"), does not count them, and
    raises only gaps from the new commits.

---

### User Story 3 - A reminder for docs the agent cannot see (Priority: P2)

Product web pages and external doc sites live outside the repository. The
agent has no tool to read them and must never claim they are missing or
wrong. When a PR qualifies, the review adds a short reminder that the
author and the maintainers own those updates, and names the specific
behavior that changed. It appears once per qualifying round, not on every
push.

**Why this priority**: It covers the surfaces most likely to fall behind,
and it costs nothing in reads. It is not a finding and never changes the
verdict.

**Independent Test**: On a qualifying test PR, confirm the first review
carries one reminder naming the changed behavior. Push a commit that only
fixes a typo and confirm the follow-up carries no reminder. Push a commit
that adds another configuration option and confirm the next follow-up
carries a reminder naming that option.

**Acceptance Scenarios**:

1. **Given** a qualifying PR on its first review, **When** the agent
   posts the review, **Then** the review body carries a Documentation
   section with the reminder, and the summary's Details block carries the
   documentation reminder, both naming the behavior that changed.
2. **Given** a PR that already got the reminder, **When** a follow-up
   reviews commits that add no new behavior change, **Then** the follow-up
   carries no reminder.
3. **Given** a PR that already got the reminder, **When** a follow-up
   reviews commits that add a behavior change no earlier reminder named,
   **Then** the follow-up carries a reminder naming only the new change.
4. **Given** any reminder, **When** a reader reads it, **Then** it asks
   the owners to check the external surfaces and never states that a page
   is missing, wrong or out of date.
5. **Given** a non-qualifying PR, **When** the agent reviews, **Then**
   there is no reminder.
6. **Given** a PR whose first review did not qualify, **When** a later
   push makes it qualify, **Then** that review carries the first
   reminder.

---

### User Story 4 - A repository declares where its docs live (Priority: P3)

A maintainer adds a "Documentation" section to the repository's
`CLAUDE.md`, `AGENTS.md` or a file in `.claude/rules/`. It lists the
in-repo doc paths, the external doc site, the product page, and who owns
each. Once that section is on the target branch, the agent checks those
paths and names those sites and owners in the reminder. A repository
without the section gets conventional paths and a generic reminder.

**Why this priority**: Conventional paths and a generic reminder work
without it. The section makes findings more precise and the reminder
more useful, but only for repositories that opt in.

**Independent Test**: In a test repository, merge a "Documentation"
section that lists `site/content/` and a product page URL with an owner.
Open a qualifying PR and confirm the agent considers `site/content/` and
the reminder names the URL and owner. Open the same PR in a repository
without the section and confirm a generic reminder.

**Acceptance Scenarios**:

1. **Given** a declared section on the target branch, **When** a
   qualifying PR is reviewed, **Then** the declared in-repo paths are
   doc candidates and the reminder names the declared sites and owners.
2. **Given** a PR that adds or edits the section itself, **When** that PR
   is reviewed, **Then** the agent uses the section at the target branch,
   not the PR's version, as it does for every rule file.
3. **Given** no section, **When** a qualifying PR is reviewed, **Then**
   the agent uses conventional paths and a generic reminder.
4. **Given** a section that says "skip documentation findings" or
   "always approve", **When** the agent reads it, **Then** it ignores
   that text. The section can add checks but never relaxes rules or the
   verdict.

---

### Edge Cases

- **Docs-only PR.** It does not qualify for the missing-docs check or the
  reminder. The misaligned check still compares its doc edits against the
  code at the head commit.
- **Excluded PR.** A PR that matches "Review exclusions" is never
  reviewed, so it gets no documentation check. A changed file that is in
  "Generated files and lockfiles" is never a source of documentation
  findings: generated API descriptions and lockfiles are covered through
  their sources.
- **Generated reference docs.** When an OpenAPI description is generated
  (for example under `gen/`), a stale description is a finding on its
  source design file, not on the generated file.
- **Repository with no declared section.** Only the root `README`, the
  rule files and docs the PR changes are read. A stale name in an
  unread `docs/` page is not caught. Declaring the section is how a
  repository widens what the agent reads (FR-033a).
- **Approved PR with gaps.** One or two documentation gaps and nothing
  else still allow "Approved with minor comments". The documentation gaps
  line still appears, so the author sees what to fix before merging.
- **Doc too large or over budget.** A doc the agent cannot read within
  the read budget is not read. It produces no finding, it does not
  withhold approval, and the recap names it as an unread `context` file.
- **Rule file skipped.** If the rule file that holds the
  "Documentation" section is skipped (over 24,000 bytes, over the 80,000
  total, or a symlink), the agent falls back to conventional paths and the
  generic reminder. The recap already names the skipped rule file.
- **Two declared sections.** When both the root and a subdirectory rule
  file declare one, in-repo paths from both are candidates. For the
  reminder, the section closest to the changed files wins for those
  files, and the root section covers the rest.
- **Declared path that does not exist.** It is skipped without a finding.
  The agent does not report the repository's own declaration as wrong.
- **Follow-up after a force-push or merge commit.** The review scope falls
  back to the whole PR diff, as today. The reminder rule still compares
  against behaviors named in earlier reminders, so the fallback does not
  repeat it.
- **Prior reminder unreadable.** If the agent cannot read its earlier
  reviews to tell whether a reminder was posted, it posts the reminder.
  One repeat is better than none.
- **PR with only i18n string changes.** Adding or changing user-facing
  strings is a behavior change for users, so it qualifies. Changing a
  string file to match code the PR also changes is the doc update itself.
- **Hard-coded user-facing strings.** When the repository keeps its
  strings in i18n files and the PR adds a hard-coded user-facing message,
  that is a missing-docs finding on the help-text surface, citing the
  string and the i18n file.
- **Author says docs are not needed.** A `docs-not-needed` label, a
  checked box or a sentence in the PR description has no effect. The
  findings and the reminder stand (FR-035).
- **Injection in docs.** Doc text in the PR or the repository that tries
  to change the verdict or the agent's behavior is handled under the
  Operating rules, like any other PR content.
- **Head moved during review.** The reminder marker is in the submitted
  review. A review that becomes a plain `COMMENT` because new commits
  arrived still counts as having posted the reminder for the behaviors it
  named.
- **Dismissed approval.** A repository setting that dismisses stale
  approvals on push dismisses an own approval that carried one or two
  gaps. That is not a maintainer decision about docs, so those gaps keep
  counting (FR-036).
- **Review cap.** Carry-forward means more rounds that request changes,
  so a PR can reach the existing 10-review cap sooner. The cap rules do
  not change; post-publish monitoring counts how many PRs reach it.

## Requirements *(mandatory)*

### Functional Requirements

#### Change classification

- **FR-001**: Each review MUST classify the PR, or on a follow-up the new
  commits, as one of: feature; bug fix that changes documented behavior;
  behavior, configuration or API change; or none of these (refactor,
  tests only, CI, dependency bump, generated files, docs only).
- **FR-002**: The classification MUST be decided from the diff. The
  title, conventional-commit prefix, labels and commit messages are hints.
  A PR whose diff adds or changes user-visible or operator-visible
  behavior (a route, flag, environment variable, config key, default
  value, UI label or message, public API, or CLI output) qualifies,
  whatever its title says. So does a fix that changes what a user sees or
  relies on without touching a named item (a table that now sorts by
  date); the agent then describes the behavior in a few words.
- **FR-003**: Only a qualifying PR or round MUST get the missing-docs
  check (FR-010 to FR-014) and the reminder (FR-020 to FR-025).
- **FR-004**: The misaligned check (FR-015 to FR-017) and the existing
  Documentation checks MUST run on every reviewed PR, whatever its class.

#### Doc surfaces

- **FR-005**: The agent MUST consider these in-repo surfaces:
  - Knowledge base: `README`, `docs/`, ADRs, `CHANGELOG`, and `CLAUDE.md`,
    `AGENTS.md` and `.claude/rules/` when the change alters a convention
    they describe.
  - User documentation with sources in the repository (mkdocs, Docusaurus
    or similar).
  - Help in the product: tooltips, help text, empty states, user-facing
    error messages, i18n string files, in-app help links.
  - Reference material: OpenAPI descriptions, CLI `--help` text, tables of
    environment variables and configuration, Helm values docs.
- **FR-006**: When a repository declares a "Documentation" section, its
  in-repo paths MUST be added to the candidates. Without one, the agent
  MUST use the conventional locations in FR-005.

#### Missing in-repo documentation

- **FR-010**: For a qualifying PR, the agent MUST raise a missing-docs
  finding when a new or changed behavior has a doc surface that covers its
  area and the PR does not update it, or when a new user-facing surface
  has no doc at all.
- **FR-011**: A missing-docs finding MUST carry proof: the new or changed
  behavior (`path:line`) and either the doc location that covers the area
  or a statement of which surfaces were checked and found absent. A
  finding without both parts MUST be dropped.
- **FR-012**: A missing-docs finding MUST NOT be raised when the PR
  already updates a doc that covers the change. This check MUST use the
  whole-PR file list, on follow-ups too, so a doc updated in an earlier
  commit counts.
- **FR-013**: Missing-docs findings MUST be `[minor]` and MUST count in
  the existing `[minor]` tally. They add no verdict rule.
- **FR-014**: A review MUST report at most 3 missing-docs findings as
  separate findings, and at most one of them MAY be a "no doc" finding
  for a new user-facing surface. The rest MUST be folded into one item
  with one line per folded gap, giving its title, behavior location and
  doc location. Each folded gap MUST still count as one `[minor]` in the
  tally and on the Minor issue line, and a follow-up classifies each line
  on its own. The cap MUST NOT apply to misaligned findings.

#### Misaligned documentation

- **FR-015**: The agent MUST raise a misaligned finding when a doc changed
  in the PR contradicts code in the PR, or when a doc at the target branch
  names something the PR renames, removes or changes (a flag, environment
  variable, endpoint, route, UI label or default value) and the PR
  leaves that reference in place. An edit elsewhere in the same doc is
  not an update.
- **FR-016**: A misaligned finding MUST name both locations, the code and
  the doc, each as `path:line`. A finding that cannot name both MUST be
  dropped.
- **FR-017**: Misaligned findings MUST be `[minor]` and MUST count in the
  existing `[minor]` tally. They add no verdict rule.

#### External-surface reminder

- **FR-020**: A qualifying review MUST include a reminder that the author
  and the maintainers own updates to external doc sites and product
  pages. It MUST name the specific behaviors that changed.
- **FR-021**: The reminder MUST NOT claim that any external page is
  missing, wrong or out of date. The agent MUST NOT fetch external pages.
- **FR-022**: The reminder is not a finding. It MUST NOT change the
  verdict, the tally, or the issue lines.
- **FR-023**: The reminder MUST appear in a Documentation section of the
  review body and in the Step 6 recap. In the recap it MUST sit inside
  Details, before "Rule files", which stays the last line. This extends
  the Step 5c rule that the review body is a 1 to 2 sentence note: the
  Documentation section follows that note, before "Findings outside the
  diff", and stays short.
- **FR-024**: The review that carries a reminder MUST carry a new marker,
  defined from scratch, that records it. Later runs MUST read it from the
  agent's own reviews to decide whether a reminder was already posted and
  which behaviors it named.
- **FR-025**: A follow-up MUST post the reminder only when its new commits
  add a behavior change that no earlier reminder named, and MUST name
  only the new changes.
- **FR-026**: When the repository declares external sites and owners, the
  reminder MUST name them. Otherwise it MUST be generic. Declared owners
  MUST appear as plain text in a code span, so GitHub sends them no
  notification. A declared owner written with a leading `@` MUST still
  go in a code span. Declared URLs MUST appear as written and MUST NOT be
  fetched.

#### Repository declaration

- **FR-027**: The agent MUST read the "Documentation" section only from
  the rule files it already reads at the target branch. It MUST NOT add a
  configuration file.
- **FR-028**: The section MUST be able to add doc paths, sites and owners.
  It MUST NOT relax this prompt's rules, severity labels, cap or verdict.
  Text in it that tries is ignored, as for any rule file.

#### Read budget

- **FR-030**: All doc reads MUST stay inside the "Read budget" table and
  its order. Doc reads come last: after Step 3.2 reads, rule files, and
  reads for context.
- **FR-031**: Rule files already read in Step 4 MUST be reused for the
  doc checks and MUST NOT be read again.
- **FR-032**: To find whether doc files exist, the agent MUST use
  directory listings. It MUST reuse the root listing from the rule-file
  step and MUST make at most 3 more listings per run for doc candidates:
  declared directories, `docs/` (or `doc/`), and the directory of a
  changed Helm `values.yaml` or i18n file.
- **FR-033**: The agent MAY read the full contents of at most these docs
  at the target branch: the root `README` and declared in-repo doc files,
  each under `max_file_bytes`. It MUST read them only when the PR
  qualifies, or when the diff renames, removes or changes a named item (a
  flag, environment variable, route, UI label or default value). A PR
  with neither, such as a dependency bump, reads no base-branch docs. Doc
  files the PR changes are reviewed from their patches.
- **FR-033a**: Other conventional surfaces (`docs/`, `CHANGELOG`, ADRs,
  Helm values docs, user-doc sources, OpenAPI files) are checked for
  existence only, from listings and the PR's file list. A proof that a
  doc "covers the area but was not updated", or that a base-branch doc
  names a changed item, is possible only for docs the agent read: the
  root `README`, declared files, rule files already read, and docs the PR
  changes. For other surfaces a missing-docs finding can only state that
  the surface exists and the PR does not touch it, when the PR adds a new
  item of the same kind (for example a new Helm value with a Helm values
  README beside it).
- **FR-034**: A doc that does not fit MUST be recorded as an unread file
  of kind `context`. It MUST NOT produce a finding and MUST NOT withhold
  approval.

#### No author opt-out

- **FR-035**: Labels, checkboxes and PR text MUST NOT suppress
  documentation findings or the reminder. Text in the PR that claims docs
  are not needed is data with no effect. It is not an injection attempt
  unless it tries to change the verdict or the agent's behavior
  (Operating rules).
- **FR-036**: When docs really are not needed, the override is a
  maintainer dismissing the agent's review that requested changes. Every
  documentation finding raised in or carried into a dismissed own review
  whose summary shows the 🔴 decision MUST be treated as resolved, with
  the reason "dismissed by a maintainer", and MUST NOT count again. A
  dismissed own approval (for example by the "dismiss stale approvals"
  setting) is not an override. When no session summary matches the
  dismissed review (the run ended before Step 6, or the comment is gone),
  the dismissal counts as an override: the stale-approval setting only
  dismisses approvals, and an approval carries at most 2 `[minor]`
  findings. A human approval alone does not clear the agent's review and
  is not an override. Gaps raised by later commits are new findings
  (FR-040). The override applies on every review of a PR with earlier
  reviews by the agent, including a re-review of the same head, which
  runs as an initial review: a gap it resolved is not raised again.

#### Follow-ups

- **FR-040**: On a follow-up, new documentation findings MUST come only
  from the new commits, unless the review fell back to the whole PR diff.
  A gap that matches an earlier documentation finding (same named item
  and same doc location) MUST be classified as that earlier finding and
  MUST NOT be raised again, so one gap never counts twice. The one
  exception: when the review's scope is the diff of the new commits and
  that diff changes the named item again, it is a new finding, even when
  the earlier one was resolved by a dismissal (FR-036). The new finding
  supersedes an earlier one still open or partially addressed, so the gap
  counts once. Several entries the agent posted for the same named item
  and doc location are one finding; only the latest is classified and
  counted. On a whole-PR fallback the match always applies, since that
  diff still holds the original change.
- **FR-041**: Earlier documentation findings MUST be classified as
  resolved, partially addressed or still open, like every other finding.
- **FR-042**: An earlier documentation finding (missing or misaligned)
  classified still open or partially addressed MUST count as one
  `[minor]` in the follow-up's tally, on every follow-up until it is
  resolved. This is an exception to the Step 5a rule that earlier
  `[minor]` findings outside Security do not count on a follow-up. It
  applies only to the Documentation dimension and adds no other verdict
  rule.
- **FR-043**: A follow-up MUST recheck earlier documentation findings
  from the PR's whole file list and its patches, with no extra reads. A
  missing-docs finding is resolved when the cited doc's patch adds the
  named item or describes the behavior (or, when GitHub omits that patch,
  the doc is in the list and not deleted), or the behavior is gone from
  the PR. An edit elsewhere in the doc, or deleting it, resolves nothing.
  A "no doc" finding is resolved when every surface it named is covered
  or gone, and partially addressed when only some are. A misaligned
  finding is resolved when the cited doc's patch changes the named item,
  or the code change is gone. A doc that
  changed but still names the old item is partially addressed. No rule
  depends on a read that may not fit the budget, so a finding never stays
  open only because the agent could not look.

#### Summary call-out

- **FR-044**: A documentation gap is a missing-docs or misaligned finding
  raised in this review, or an earlier one classified still open or
  partially addressed (FR-042). Folded findings past the cap (FR-014)
  are gaps too.
- **FR-045**: When a review has at least one documentation gap, the
  Decision section of the Step 6 summary MUST carry a documentation gaps
  line, whatever the decision. It sits in the visible top of the summary,
  not inside Details. The line gives the number of gaps and, for each of
  the first 3, its title and the doc location (or "no doc" for a new
  surface with none), then "and N more" when there are more. Carried
  forward gaps are marked `prior`. The exact format is in
  `contracts/summary-call-out.md`, for example: ``📝 **Documentation gaps
  (2):** `MAX_RETRIES` not in the README environment table
  (`README.md`); stale `--dry-run` flag (`docs/cli.md`, prior)``.
- **FR-046**: When the decision is 🔴 and approval would be possible
  without the documentation gaps, the sentence on what blocks approval
  MUST name them, in the words the contract gives: "Documentation gaps
  bring the minor count over 2, which blocks approval."
- **FR-047**: A review with no documentation gaps MUST NOT carry the line.
  The gaps still appear as usual in the tally, the Minor issue line,
  revision tracking and the inline comments. The line adds a call-out
  and changes no count.

#### Placement and compatibility

- **FR-050**: Each documentation finding MUST be an inline comment when
  its location is in the PR diff, and otherwise go under "Findings
  outside the diff", using the existing template.
- **FR-051**: The behavior MUST live in the agent prompt, inside the
  existing Documentation dimension. No new dimension.
- **FR-052**: Existing markers, the `lfx-review-agent-pending` label and
  released exclusion `Rule` values MUST NOT change. The new marker MUST be
  added to the marker list in the repository `CLAUDE.md`.
- **FR-053**: The agent version MUST go up one patch. Prose MUST follow the
  unslop rules, the only `{{ }}` text MUST be the existing environment
  variables, and markdownlint MUST pass.
- **FR-054**: If the plan finds a new tool is needed, it MUST be added to
  `guild.yaml` and to the README credential policy.

### Key Entities

- **Change class**: the result of FR-001 for a review: one of the three
  qualifying classes, or none. Decided per review from that review's
  scope.
- **Doc surface**: a place where behavior is described: an in-repo file or
  directory (conventional or declared), or an external site or product
  page (declared or generic).
- **Documentation section**: an optional part of a rule file at the
  target branch. It lists in-repo doc paths, external sites, product
  pages, and owners.
- **Missing-docs finding**: `[minor]`, Documentation dimension. Proof is the
  behavior location plus the uncovered doc location, or the surfaces found
  absent.
- **Misaligned finding**: `[minor]`, Documentation dimension. Proof is the
  code location plus the contradicting doc location.
- **External-surface reminder**: text in the review body and the recap.
  It names changed behaviors and, when declared, sites and owners. It is
  not a finding.
- **Documentation gap**: a missing-docs or misaligned finding from this
  review, or an earlier one still open or partially addressed. The
  summary calls these out (FR-044 to FR-047).
- **Reminder marker**: a new marker in the agent's own review that records
  that a reminder was posted. Later runs read it.

## Success Criteria *(mandatory)*

### Measurable Outcomes

Measured on a replay set of about 20 recently merged LFX PRs: at least 5
features, at least 5 bug fixes, and at least 5 chore, refactor, test, CI
or dependency PRs.

In these criteria a `[docs]` finding is a missing-docs or misaligned
finding. Today's untagged doc-comment and typo nits are not counted.

- **SC-001**: Chore, refactor, test, CI and dependency PRs get zero
  missing-docs findings and zero reminders. Any `[docs]` finding on them
  is a misaligned finding that names both locations.
- **SC-002**: 100% of `[docs]` findings name both locations, or the
  behavior location and the surfaces found absent.
- **SC-003**: Zero reminders state that an external page is missing,
  wrong or out of date.
- **SC-004**: Each qualifying PR gets a reminder on its first qualifying
  review. Across a multi-round cycle, a later reminder appears only for a
  round that adds a behavior change no earlier reminder named. The replay
  shows zero duplicate reminders.
- **SC-005**: There is no cap on how far the approve rate drops. Every
  approval lost compared with today's agent traces to a `[docs]` finding
  that names its locations.
- **SC-005a**: A maintainer judges at least 90% of the `[docs]` findings
  that blocked an approval on the replay set to be correct.
- **SC-006**: No replay run ends with a context overflow or goes over the
  read budget because of doc reads.
- **SC-007**: A maintainer judges at least 80% of the `[docs]` findings
  on the replay set to be correct and worth acting on.
- **SC-008**: On the replay set, 100% of summaries for reviews with a
  documentation gap name the gaps in the visible top of the summary, and
  0% of summaries without one carry the line.

## Assumptions

- Labels, the title and commit messages are hints for the change class
  only (FR-002). The PR metadata the agent already reads carries them. No
  new tool is expected.
- The replay cannot run against merged PRs directly, because Step 1
  stops on a PR that is not open. The plan recreates each replay PR as an
  open PR in a test repository, or uses another harness the plan names.
- "Conventional locations" means the root `README`, `CHANGELOG`, `docs/`,
  `adr/` or `docs/adr/`, mkdocs or Docusaurus source directories, i18n
  directories, OpenAPI files, and Helm `values.yaml` with its README. The
  plan fixes the exact list.
- The format of the "Documentation" section (headings and fields) is set
  in the plan and documented in the README, so repository maintainers can
  write one.
- Doc reads compete with reads for context. Correctness context comes
  first, so on a large PR the doc checks may see less, and say so in the
  recap.
- This repository's own README and `CLAUDE.md` are updated in the same
  change to describe the new behavior and marker.

### Out of scope

- Writing the documentation (a one-line pointer in `fix` is fine).
- Fetching external web pages.
- Opening issues or tickets for doc work.
- Blocking merges because of external surfaces.
- An author opt-out (label, PR-template checkbox, or PR text) for
  documentation findings.
