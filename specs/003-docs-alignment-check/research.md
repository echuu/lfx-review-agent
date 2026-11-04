# Research: Documentation alignment check

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

## R1. Where the change lives in `PROMPT.md`

**Decision**: Extend existing steps; add no step and no dimension.

| Place | Change |
| --- | --- |
| Step 4, Gather the material | New "Documentation candidates" item: reuse the root and top-level listings from the rule-file step, at most 3 more listings, the base-doc read rule (R5) |
| Step 4, Review the dimensions, Documentation | Change class (R2), named items, missing-docs and misaligned checks, the `[docs]` tag (R3), the cap (R6), the reminder (R4) |
| Step 4, Follow-up only | The read-free recheck of prior `[docs]` findings (R7) |
| Step 5a | Exception to "prior `[minor]` items outside Security do not count" for still-open `[docs]` items |
| Step 5b | The `[docs]` tag in the inline comment title line |
| Step 5c, `body` | Documentation section with the reminder marker, after the verdict note and before `Findings outside the diff` (R4) |
| Step 6, Decision | Documentation gaps line and blocker sentence (R8) |
| Step 6, tally and issue lines | Prior `[docs]` items under 🟡 with `(prior)` |
| Step 6, Details | Reminder text, just before "Rule files" |
| Read budget | Order: doc reads come after reads for context |

**Rationale**: FR-051 keeps the behavior in the Documentation dimension.
Every other place already exists and only gains a rule.

**Alternatives considered**: A new "Step 4.5: Documentation check".
Rejected: it would split one dimension across two steps and duplicate the
follow-up and placement rules.

## R2. Change class and named items

**Decision**: The Documentation dimension records two values per review:

- `change_class`: `feature`, `behavior-fix`, `behavior-change`, or
  `none`. Decided from the diff (FR-002). Title, prefix, labels and
  commit messages are hints that can make the agent look harder, never
  reasons to skip.
- `named_items`: each flag, environment variable, config key, route,
  endpoint, UI label or message, default value or CLI option the diff
  adds, renames, removes or changes, with its `path:line`, its kind, and
  for a rename or removal its old name.

`none` covers refactors that keep every named item, tests, CI, dependency
bumps, generated files, and docs-only PRs. A PR with an empty
`named_items` list and no new user-facing surface is `none`.

**Rationale**: Named items drive all three checks: the misaligned check
looks for old names in docs, the missing-docs check looks for new names
in the doc that should cover them, and the reminder lists them. Deciding
from the diff closes the gap where a `chore:` title hides a new route.

**Alternatives considered**: Trusting the conventional-commit prefix.
Rejected: PR text is untrusted and easy to get wrong.

## R3. Tagging documentation findings

**Decision**: Missing-docs and misaligned findings carry a `[docs]` tag
after the severity, in the same place as `[privacy]`:
`[minor][docs] <short title>`. The tag appears in the inline comment,
under `Findings outside the diff`, on each line of a folded item, and in
the body the 5c 422 retry builds. Doc-comment and typo nits under the
existing Documentation rules stay untagged.

A follow-up finds earlier documentation findings in the agent's own
inline comments (`user.login` is `{{env.REVIEWER_LOGIN}}` and the title
line has `[docs]`) and in its own review bodies (folded items and
`Findings outside the diff`).

**Rationale**: The 5b title line has no dimension, so without a tag a
follow-up cannot tell a documentation `[minor]` from a Correctness one,
and FR-042 cannot work. A visible tag also tells the author what kind of
fix is wanted.

**Alternatives considered**: A hidden marker in each inline comment.
Rejected: it adds a marker per comment, and the 422 retry would need to
keep it too. The visible tag carries the same information.

## R4. Reminder placement and the reminder marker

**Decision**: The review body gains a Documentation section after the
1 to 2 sentence verdict note and before `Findings outside the diff`. Its
first line is `<!-- lfx-review-agent:docs-reminder -->`, followed by the
reminder sentence and a list of the behaviors it names, one per line,
each in a code span. The second line of the body stays reserved for the
approval-withheld marker. The recap repeats the reminder in Details just
before "Rule files". See
[contracts/review-output.md](contracts/review-output.md).

A later run reads the Step 2 review list (already fetched, no extra
call), keeps own reviews that carry both the review marker and the
reminder marker, and collects the behaviors listed after it. A follow-up
posts a reminder only for named items not in that set (FR-025). New
names go through the R9 escaping before the comparison, so they match
the stored form.

**Rationale**: Submitted reviews cannot be deleted, and Step 2 already
holds every own review body, so the state is free to read and cannot be
lost. Listing the behaviors as visible text keeps PR-derived names out of
the marker itself (R9).

**Alternatives considered**: Storing the behaviors inside the marker
(`items="..."`). Rejected: PR-derived text inside an HTML comment could
close the comment early. Storing it in the session comment: rejected,
because a later run's Step 6 rewrites only its own comment, and session
comments are harder to match than reviews.

## R5. Doc candidates and the read budget

**Decision**:

- **Listings**: reuse the root listing and the top-level directory
  listings the rule-file step already made at `base_sha`. Make at most 3
  more `git_get_tree` listings per run, in this order: declared
  directories, `docs/` (or `doc/`), then the directory of a changed Helm
  `values.yaml` or i18n file when its README or sibling locale files are
  not already visible.
- **Reads**: only the root `README*` and declared in-repo doc files, at
  `base_sha`, each at most `max_file_bytes`, and only when
  `change_class` is not `none` or `named_items` has a rename, removal or
  change (FR-033). They come last in the read order, after reads for
  context, and count toward `max_run_bytes`.
- **Free sources**: rule files already read, and patches of doc files the
  PR changes.
- **Existence-only surfaces**: everything else on the conventional list
  (R10) is checked by listing only (FR-033a).

**Rationale**: Spec 002 exists because of unbounded reads. A dependency
bump reads no docs at all. One root README read is the common case and
fits the budget.

**Alternatives considered**: Reading every file under `docs/`. Rejected:
unbounded, and listings alone can be hundreds of KB (spec 002 saw a
424 KB listing).

## R6. Cap, folding and absence findings

**Decision**:

- At most 3 missing-docs findings get their own comment. The rest fold
  into one `[minor][docs]` item in the review body under `Findings
  outside the diff`, one line per folded gap with its title, behavior
  location and doc location. Each folded gap counts as one `[minor]`.
- At most one "no doc" (absence) finding per review. It covers the whole
  PR and names every new user-facing surface without a doc.
- Misaligned findings are never folded or capped.

**Rationale**: One line per folded gap lets the next round classify each
gap on its own (FR-042). Absence findings are the weakest proof, and with
`[minor]`, carry-forward and no opt-out they are the main risk to
SC-005a, so they are limited to one.

**Alternatives considered**: A title-only folded item. Rejected: the
next round could not count the gaps it holds.

## R7. Follow-up recheck without reads

**Decision**: A follow-up rechecks every earlier `[docs]` finding against
the whole-PR file list from Step 3.2 and its patches:

| Finding | Resolved when | Partially addressed when |
| --- | --- | --- |
| Missing docs, cited doc | The cited doc is in the file list, or the behavior is gone from the PR | Not used |
| Missing docs, "no doc" | A doc of a kind the finding named is in the file list, or the surface is gone | Some surfaces got docs, others did not |
| Misaligned, base doc names an old item | The cited doc's patch removes or replaces the old name, or the code change is gone | The doc changed but still names the old item elsewhere in its patch |
| Misaligned, PR doc contradicts PR code | Either side's current patch makes them agree | Not used |
| Any `[docs]` finding open at a dismissed own review | That review's summary (the session comment matched by `review-of review=<N>`) shows 🔴: resolved, "dismissed by a maintainer" (spec FR-036) | Not used |

Step 2 already lists dismissed own reviews, and Step 1.5 already reads
the session comments, so the dismissal check needs no extra call. A
dismissed own approval does not resolve anything: the "dismiss stale
approvals" setting dismisses approvals on every push, which is not a
decision about docs. With no matching summary, the dismissal counts as
an override; guessing wrong costs at most 2 `[minor]` findings.

**Dedupe**: before raising a documentation finding on a follow-up,
compare it with the earlier `[docs]` findings. A finding with the same
named item (after the R9 escaping) and the same doc path is the earlier
finding: classify it, do not raise it again. The one exception: when
the scope is the compare diff of the new commits and that diff changes
the named item again, it is a new behavior change and gets a new finding.
On a whole-PR fallback (force-push or merge commit) the match always
applies: that diff still holds the original change, and without the
dedupe one gap would count as a new `[minor]` and a prior one.

**Rationale**: FR-043 and FR-040. With carry-forward and no opt-out, a
rule that needs a read could block a PR whose doc no longer fits the
budget, and a duplicate count could request changes on its own. The file
list and patches are already in context.

**Alternatives considered**: Rereading the cited doc at `head_sha`.
Rejected for the reason above.

## R8. Summary call-out

**Decision**: In Step 6 Decision, after the tally line and the 🔴
sentence and before the 5a notes, add a Documentation gaps line whenever
the review has at least one gap (FR-044 to FR-047). The blocker sentence
names the gaps when the `[minor]` count is over 2 only because of them.
Prior `[docs]` items still open are counted under 🟡 in the tally and on
the Minor issue line with `(prior)`, and in `❌ N prior item still open`,
the same way prior 🔴 items are counted today. See
[contracts/summary-call-out.md](contracts/summary-call-out.md).

**Rationale**: The top of the summary is what most readers see. Counting
prior documentation items under 🟡 keeps the tally equal to the counts
the verdict used.

**Alternatives considered**: Putting the gaps in `### Summary`. Rejected:
that section is 1 to 3 plain sentences with no lists.

## R9. Escaping echoed text

**Decision**: Every name, path, URL or owner the agent echoes from the PR
or the repository (named items, doc paths, declared sites and owners)
goes in a code span, after removing `<!--`, `-->`, backticks and line
breaks and cutting it to 100 characters. This applies to the review
body, inline comments and the recap.

**Rationale**: Later runs detect markers with "body contains" on the
agent's own reviews and comments. A flag name such as
`<!-- lfx-review-agent:approval-withheld -->` echoed raw would plant a
marker in the agent's own review. A code span also stops `@team` from
notifying anyone (FR-026).

**Alternatives considered**: HTML-escaping. Rejected: the text is
Markdown, and GitHub renders entities inconsistently in code spans.

## R10. Conventional doc locations

**Decision**: Without a declared section, the candidates are:

| Surface | Locations |
| --- | --- |
| Knowledge base | Root `README*`, `CHANGELOG*`, `docs/`, `doc/`, `adr/`, `docs/adr/`, plus `CLAUDE.md`, `AGENTS.md`, `.claude/rules/` (already read) |
| User docs | `mkdocs.yml` with its `docs/`, `website/docs/`, `docusaurus.config.*` with `docs/` |
| Help in the product | `i18n/`, `locales/`, `src/assets/i18n/` and similar locale directories |
| Reference | `openapi*.{yaml,yml,json}`, `swagger*` outside "Generated files and lockfiles", `charts/*/README.md` beside `values.yaml`, CLI help text in the code the diff changes |

A generated OpenAPI description under `gen/**` is never a doc candidate;
a stale description is a finding on its Goa design source.

**Rationale**: These cover the LFX stacks the agent reviews (Angular,
Go and Goa, Helm, dbt, mkdocs/techdocs).

## R11. Declared "Documentation" section format

**Decision**: A heading named `Documentation` (any level) in a rule file
at `base_sha`, with bullets that start `In-repo:`, `Docs site:` or
`Product page:`. See
[contracts/documentation-section.md](contracts/documentation-section.md).

**Rationale**: Rule files are already read with caps, so the section
costs nothing extra. Fixed bullet prefixes are easy to write and to
parse, and anything else in the section is ignored.

**Alternatives considered**: A YAML file (`.lfx-review.yaml`). Rejected
by the input: no new configuration file.

## R12. Replay harness

**Decision**: Run the replay in a separate Guild workspace with a forked
test agent, against a private sandbox repository, before publishing to
`lfx-reviews`.

1. `guild agent fork` the live agent into a test agent, and add it to a
   new workspace with the same variables. Its triggers point only at the
   sandbox repository.
2. For each replay PR, push the original base commit and head commit (in
   commit order, for multi-round cycles) to branches in the sandbox and
   open a PR as an org member. The base branch carries the original
   rule files and README.
3. Baseline: run the test agent at 0.1.22 on the same copies first, then
   at the candidate version. Compare the Step 5a verdict, before the 5a.1
   and 5a.2 gates, which differ in the sandbox.

Open points that quickstart step 1 confirms before any replay: whether
the forked agent can use the `lfx-one` credential (the ALLOW policy may
be per agent), and whether a test workspace can run triggers on a
sandbox repository without touching `lfx-reviews`. If either fails,
there is no replay before rollout: 0.1.23 is published, the success
criteria become monitoring of the first 48 hours rather than a gate, and
`guild agent unpublish` restores 0.1.22.

**Rationale**: `lfx-reviews` auto-updates, so a publish there is
production. Step 1 stops on PRs that are not open, so merged PRs must be
reopened as copies. Running 0.1.22 on the same copies keeps model and
version drift out of the SC-005 comparison.

**Alternatives considered**: Comparing with the production verdicts on
the original PRs. Rejected as the primary baseline: those ran on 0.1.16
to 0.1.22 against other heads. It stays as a sanity check.

## R13. Repository documentation this change needs

**Decision**: The feature's own PR updates:

- `README.md`: a "Documentation check" section (what the agent checks,
  the `[docs]` tag, carry-forward, no opt-out, the declared section
  format), a Purpose bullet, and test checklist items. The credential
  policy does not change (no new tool).
- `CLAUDE.md`: `:docs-reminder` and the `[docs]` tag join the read-back
  list; a note on the Documentation section contract.
- `mkdocs.yml` builds techdocs from the README; no `docs/` directory
  exists, so there is no other page to update.

**Rationale**: The PR is judged by its own check. A new marker and a new
repo-facing format are exactly what FR-010 flags when undocumented.
