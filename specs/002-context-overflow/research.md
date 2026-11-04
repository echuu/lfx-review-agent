# Research: Reviews that fit, or say why they can't

**Feature**: [spec.md](spec.md) | **Date**: 2026-11-03

## R1. What overflowed, and what fit

**Data**: every webhook session in workspace `lfx-reviews` from
2026-11-02 18:00 to 2026-11-04 00:02 UTC (agent 0.1.16 and 0.1.17), 625
runs, read with `guild session events --events all`. For each run: the
byte length of every tool result, the PR's size fields from its
`pulls_get` result, and whether it submitted a review. Guild records
token counts per session only, not per model call, so the tool output a
run read is the proxy for its prompt size.

- 392 runs posted a review. 5 ended with the provider's
  `prompt is too long` error (spec Context table).
- An earlier count of "3 of 299 runs since 0.1.16" was wrong: that scan
  covered only 2026-11-03 05:16 to 21:48 UTC.
- 4 of the 5 overflows came from reading single large files: a lockfile
  and a dbt data file each read at both commits (#449, #3177), one
  generated file (#472), and large source files read for context
  (#4989: `campaigns.component.ts` had a patch, so the 593 KB read was
  for context). 1 came from PR history (#4962).

Largest successful reviews:

| PR | Changed lines | Files | Inline review comments | Tool output |
| --- | --- | --- | --- | --- |
| lfx-v2-newsletter-service#103 | 13,153 | 118 | 35 | 1.59 MB |
| lfx-self-serve#4962 (review 4) | 11,979 | 82 | 55 | 1.85 MB |
| lfx-self-serve#4899 | 1,217 | 13 | 73 | 1.44 MB |
| lf-dbt#3196 | 2,047 | 14 | 65 | 1.94 MB |

Tool output alone does not predict an overflow: lf-dbt#3196 succeeded at
1.94 MB and #4989 failed at 1.17 MB. The difference is what the bytes
were (R4).

**Watch**: lf-dbt#3196 (open). Its inline review comment read grew from
172 KB to 1.01 MB over four reviews. It is the most likely next history
overflow. lfx-crowdfunding#449 and lf-dbt#3177 are merged; #4989
recovered on later runs.

## R2. Where the bytes go

| Read | Example | Useful part |
| --- | --- | --- |
| Inline review comments (`pulls_list_review_comments`) | #4962: 408 KB | Comment text 8%; `diff_hunk` 55%; user records 14%; links 8% |
| Reviews (`pulls_list_reviews`) | #4962: 169 KB | Review text 46%; user records 32% |
| Changed files (`pulls_list_files`) | #4962: 783 KB | Patches 89% |
| Directory listing (`repos_get_content` on a directory) | `packages/shared/src/utils`: 344 KB for 316 entries | Name, type, size: under 10% |
| File read (`repos_get_content` on a file) | AGENTS.md: 31 KB response for a 22 KB file | Base64 text; see R4 |

The patches are real review material. The rest is mostly envelope.

## R3. Knowing a file's size before reading it

The prompt already takes rule-file sizes from directory listings, but a
listing through `repos_get_content` can be hundreds of KB (R2), and
`pulls_list_files` carries no size.

| Option | `packages/shared/src/utils` | Notes |
| --- | --- | --- |
| `repos_get_content` on the directory (today) | 344 KB | Links and URLs on every entry |
| REST `GET /repos/{owner}/{repo}/git/trees/{tree_sha}` with `tree_sha` = `<commit>:<dir>` | 78 KB | `path`, `mode`, `type`, `sha`, `size`. Mode `120000` is a symlink. Works with the path URL-encoded too. Read only. |
| GraphQL `object(expression: "<commit>:<dir>") { ... on Tree { entries { name type object { ... on Blob { byteSize } } } } }` | 25 KB | Needs a safe GraphQL operation (R5) |

**Decision**: add `git_get_tree` (REST) in integration 1.0.6. Every
directory listing uses it, including rule-file discovery; the prompt
stops calling `repos_get_content` on directories. The tree for the
repository root is `git_get_tree` with the commit SHA alone.

**Alternatives**: GraphQL tree (smaller, but gated on R5); keep
`repos_get_content` listings (the 424 KB problem).

## R4. Plain text instead of base64

`repos_get_content` returns JSON with base64 `content`. That is GitHub's
default for `Accept: application/json`; it was never chosen.

**Cost estimate** (not measured per call, R1): in #4989's failed run,
about 850 KB of base64 file reads plus about 450 KB of other text went
over 1,000,000 tokens. That puts base64 near 1 byte per token. JSON text
in #4962's run averaged about 2 bytes per token, and plain source is
usually 3 to 4. So a file read as base64 likely costs 3 to 5 times the
tokens of the same file as plain text. Quickstart step 5 measures this.

Checked against GitHub directly:

- `Accept: application/vnd.github.raw+json` returns the file as text
  (AGENTS.md: 22,181 bytes instead of a 31,415-byte JSON response).
- The same header on a directory still returns the JSON listing.
- Raw returns files up to 100 MB, while the JSON form returns empty
  content above 1 MB. A size check (R3) must come first.
- GraphQL `Blob.text` returns plain text, truncated at about 512 KB
  (`isTruncated`), with `byteSize` and `isBinary`.

**Decision**: add `repos_get_content_raw` in integration 1.0.6: the same
path as `repos_get_content`, plus an `Accept` header parameter pinned to
one enum value. Whether Guild sends header parameters is unknown, so
quickstart step 2 tests it after publish. If Guild drops the header, the
operation returns the JSON form and the prompt keeps using
`repos_get_content`; GraphQL `Blob.text` (R5) is the fallback.

**Alternatives**: GraphQL `Blob.text` first (gated on R5);
`raw.githubusercontent.com` (another host, and private repositories need
the token there too).

## R5. A safe GraphQL operation

GraphQL would trim history (R6) and give plain text and small listings.
The blocker, recorded for #19 and the sweep (001 research R3, Shape C):
a credential policy allows or denies a whole operation, so a GraphQL
operation that takes any query also lets the agent send mutations as
`lfx-one`.

A GraphQL operation is safe only if its `query` can hold nothing but
fixed, read-only strings. OpenAPI allows one operation per
`POST /graphql`, so it would be one operation, `graphql_read`, whose
`query` is an `enum` of fixed queries, with variables for owner, repo,
number, expression and cursor only.

That depends on Guild rejecting a body value outside the enum. The
existing operations pin values the same way (`issues_create_reaction`
pins `content` to `eyes`; the label operations pin the label name), but
no test has shown that Guild enforces it.

**Probe** (quickstart step 1, no publish needed): call
`issues_create_reaction` with `content` set to an invalid value. A Guild
validation error means the enum is enforced. A GitHub 422 means Guild
passed the value through; nothing is created either way.

**Result** (2026-11-03, integration 1.0.5, `lfx-one` credential): Guild
sent the request to
`/repos/linuxfoundation/lfx-review-agent/issues/comments/6618074477/reactions`
and GitHub answered 422: "lfx-probe-invalid is not a member of [...]".
Guild does not enforce body `enum` values.

**Decision**: no `graphql_read`. A GraphQL operation waits for a
read-only token (#19). Every GraphQL option in R3, R4 and R6 is out.
See R12 for what this means for the existing pins.

## R6. Trimming history

Fixed GraphQL queries, measured on #4962 with the maintainer's token:

| Read | REST today | GraphQL |
| --- | --- | --- |
| Reviews: `databaseId author state body submittedAt commit.oid url` | 169 KB | 96 KB |
| Inline comments, as review threads: `isResolved isOutdated path line` plus each comment's `databaseId author body createdAt url` | 408 KB | 75 KB |

The thread query also gives resolution status, which #19 asks for.

Other options:

- Guild may trim a response to the operation's output schema
  (`guild integration operation create` takes `--output-body-schema`).
  Unknown; a question for Guild (Rod Brathwaite). Narrowing the output
  schema of the two history operations in 1.0.6 costs nothing and shows
  the answer in the first review after publish.
- REST with less data: read inline comments `since` the last own review,
  plus the last own review's comments through
  `/pulls/{n}/reviews/{id}/comments`. Smaller, but it still carries
  `diff_hunk`, and older bot threads would drop out of reconciliation.

**Decision**: GraphQL is out (R5). Narrow the output schemas of the two
history operations in 1.0.6 and ask Guild whether it trims to them. If
it does, the history reads shrink without GraphQL. If it doesn't,
history stays as large as today, and the Phase A tail read and gate A1's
`review_comments` limit (R8) are the only protection; SC-004 is then not
met in 0.1.19. The REST `since` read is rejected: it loses history the
reconciliation needs.

## R7. Read limits

Limits are on file size, taken from tree listings (R3), so the run can
add them up before reading.

| Limit | Value | Reason |
| --- | --- | --- |
| Per file, full read | 100,000 bytes | 51 of 3,726 file reads in successful reviews returned more than 140 KB (a file of about 100 KB). #4989, #449, #472 and #3177 each read 600 KB to 1.3 MB. |
| Per run, all full-file reads | 300,000 bytes | #4989 failed at 593 KB plus 215 KB of base64, and succeeded at 600 KB alone. |
| Patch-less changed file | Read at head only | Reading both commits doubled #449 and #3177. The base side adds little: the model cannot diff two large copies well. |

These values assume base64 reads. When plain-text reads are confirmed
(R4), the limits can rise; quickstart step 5 gives the numbers. Rule
files keep their own limits (24,000 bytes each, 80,000 in total) and
count toward the run total.

What a skipped read means:

- Patch-less changed file over the limit: unread. Named at the top of
  the summary. Approval withheld, unless the path is on the generated
  and lockfile list (R9).
- Changed file with a patch, too large to read whole for context: the
  patch is reviewed; the file is named in the recap; no withholding.
- File wanted only for context: skipped and named in the recap.

## R8. Size check before a review

No limit separates a PR that fits from one that overflows by a few
comments: #4962 review 4 fit and review 5 failed with nearly the same
PR. The check is a backstop for a diff that cannot fit at all.

`pulls_get` (already read in Step 1) gives `changed_files`, `additions`,
`deletions`, `comments`, `review_comments` and `commits`. Changed files
cost about 50 to 65 bytes per changed line in `pulls_list_files` (#4962:
783 KB for 12,110 lines; #103: 667 KB for 13,153).

| Limit | Value | Largest reviewed | Reason |
| --- | --- | --- | --- |
| Changed lines (`additions + deletions`) | 20,000 | 13,153 | About 1.3 MB of patches, near 650,000 tokens before any history or rule file |
| Changed files | 300 | 118 | Three pages of file list; patches for that many files rarely fit |
| Inline review comments | 150 | 73 | At about 5.7 KB each in REST (#4962), 150 is about 850 KB. Raise after R6 ships. |

None of the 5 overflows would have been stopped by these limits. They
are fixed by R3, R4, R6 and R7; the Phase A stalled notice remains the
backstop for the rest.

**Notice**: one per PR while it stays over, with marker
`<!-- lfx-review-agent:too-large -->`. A later run posts no second
notice while one is newer than the PR's most recent session comment.
Gate C lets the next push through after a `too-large-notice`, as it does
after a stalled notice. Two runs at once settle a duplicate with the
existing duplicate notice check (new type `too-large`).

## R9. Generated files and lockfiles

A fixed list in the prompt, like the review exclusions. Files on it are
named as "not reviewed line by line" when unread and never withhold
approval (spec FR-013). First list:

| Pattern | Kind |
| --- | --- |
| `package-lock.json`, `npm-shrinkwrap.json`, `yarn.lock`, `pnpm-lock.yaml` | npm, Yarn and pnpm lockfiles |
| `go.sum` | Go checksums |
| `Cargo.lock`, `poetry.lock`, `Pipfile.lock`, `uv.lock`, `composer.lock`, `Gemfile.lock` | Other lockfiles |
| `gen/**`, `**/kodata/gen/**` | Goa generated code and its embedded copies |
| `*.min.js`, `*.min.css` | Minified assets |

Matching is on the file name or path only, at any depth unless the
pattern has a directory.

## R10. Exclusions and unread files

Today Step 3.2 reads a patch-less file at both commits to decide an
exclusion, and treats "unsure" as a match. With the read limit, a large
patch-less file is not read. The maintainer chose (2026-11-03): decide
such a file from its path. A file on the R9 list is never a match. Any
other unread file is a match only if its path points at the excluded
area (for `meeting-ics`: `ics`, `ical`, `calendar` in the path). The
unread file still withholds approval in the review (R7), so a human
looks at it either way.

## R11. Versions and rollout

- Agent 0.1.19: all prompt changes (Phase B, plus Phase C prompt use of
  whatever 1.0.6 confirms).
- Integration 1.0.6: `git_get_tree` and `repos_get_content_raw`;
  narrowed output schemas on the two history operations. One new ALLOW
  policy for the two new operations.
- Publish order as before: integration, policy, merge, `guild agent
  save --publish --version-number 0.1.19`.

## R12. The existing pins are not enforced

The R5 probe shows that an `enum` in an operation's request body is
documentation only: Guild sends any value. The integration relies on
such pins in three places:

| Operation | Pin | What the agent could do instead |
| --- | --- | --- |
| `issues_create_reaction` | `content: eyes` | Any of GitHub's eight reactions. Low impact. |
| `issues_add_labels` (1.0.5) | `labels: [lfx-review-agent-pending]` | Add any label that exists in the repository, or create one. Some labels start workflows: lfx-self-serve's `deploy-preview` deploys the branch to the dev cluster. |
| `issues_remove_label` (1.0.5) | path `name` pinned to the label | Remove any label, such as one a maintainer set to block a merge. |

Today only the prompt keeps the agent to the pending label. The prompt
treats PR content as untrusted, but an injected instruction that
gets past that could add or remove labels as `lfx-one`. The exposure
starts when an agent version that lists the label tools is published
(0.1.18); 0.1.17 does not list them in `guild.yaml`.

Options, outside this feature's scope, for the maintainer to decide
before publishing 0.1.18:

- Pin the remove in the path itself:
  `DELETE /repos/{owner}/{repo}/issues/{issue_number}/labels/lfx-review-agent-pending`
  with no `name` parameter. That operation can remove only that label.
  GitHub has no add-one-label path, so the add stays a body value.
- Ask Guild (Rod Brathwaite) whether a credential policy can restrict
  parameter values, or whether Guild can enforce request schemas.
- Accept the add risk for now, since outside contributors' PRs are the
  main injection source and the agent never approves them.
