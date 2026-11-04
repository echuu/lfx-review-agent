# Quickstart: Reviews that fit, or say why they can't

Validation steps for the maintainer. Every Guild command here uses the
`lfx-one` credential, so the maintainer runs it.

## 1. Enum probe (before implementation, no publish)

**Done 2026-11-03.** Result: HTTP 422 from GitHub, so Guild does not
enforce body enums and `graphql_read` is out (research R5, R12).

Decided whether `graphql_read` ships. The target is an
`lfx-one` comment on merged PR #35. The `content` value is invalid, so
GitHub creates nothing whatever Guild does. Path values must be strings.

```bash
guild integration version test linux-foundation~lfx-one-github \
  --version-number 1.0.5 \
  --operation issues_create_reaction \
  --account linux-foundation \
  --input-path '{"owner":"linuxfoundation","repo":"lfx-review-agent","comment_id":"6618074477"}' \
  --input-body '{"content":"lfx-probe-invalid"}'
```

| Result | Meaning |
| --- | --- |
| A Guild validation error before any request (names the `enum`) | Guild enforces body enums. `graphql_read` goes into 1.0.6. |
| HTTP 422 from GitHub ("Validation Failed") | Guild passes values through. No `graphql_read`; R6 waits for a read-only token (#19). |

## 2. Test the new operations (after integration 1.0.6 is published)

Create the ALLOW policy first. Run with `--account linux-foundation`.

- `git_get_tree` with `tree_sha` set to
  `<a lfx-self-serve commit SHA>:packages/shared/src/utils`: expect 200,
  `tree` entries with `size`, and a response near 78 KB. Repeat with the
  commit SHA alone for the root.
- `repos_get_content` with `Accept: application/vnd.github.raw+json` on
  `AGENTS.md` in `lfx-self-serve` at that commit: expect plain Markdown
  (about 22 KB), not JSON with base64. If it returns JSON, Guild dropped
  the header: remove the header from the prompt before merging (plan,
  Rollout step 3).
- `issues_remove_label` on a test PR in `linuxfoundation/lfx-review-agent`
  that carries the pending label and one other label: expect 200 and only
  the pending label gone, then 404 on a second call. Confirm the
  existing ALLOW policy still allows it (no policy denial).
- `pulls_list_reviews` and `pulls_list_review_comments` on #4962, which
  has outdated comments (`line: null`): expect 200 for both. A Guild
  validation error means the narrowed schema broke the read; fix it
  before merging. Record whether each response shrank to the narrowed
  schema (research R6), and send the result to Guild with the question
  either way.

## 3. Rollout order

1. Step 1 above, recorded.
2. From this branch's `integration/lfx-one-github.openapi.yaml`, publish
   1.0.6: `guild integration version create`, `operation create
   --openapi`, `version build`, `version publish`. Check that the
   published version lists every operation (20 plus the new ones). A
   published version can't be removed.
3. Create the ALLOW policy for `git_get_tree`.
4. Step 2 above.
5. Merge, then `guild agent save --publish --version-number 0.1.20 -m
   "..."` from an updated `main`.

## 4. Validation scenarios

| Scenario | Expected |
| --- | --- |
| PR over 20,000 changed lines | One `too-large` notice, no start comment, no review. A second push posts nothing. |
| PR with a `too-large` notice that drops under the limits | The next push starts a review without a debounce. |
| Changed lockfile without a patch (for example `pnpm-lock.yaml`) | Not read; named as "generated or lockfile, not reviewed line by line"; approval not withheld for it. |
| Changed patch-less file over 100,000 bytes, not on the list | Not read; named at the top of the summary; `COMMENT` with the approval-withheld marker instead of `APPROVE`. |
| Changed file with a patch, over 100,000 bytes | Patch reviewed; whole file not read; named in the recap; no withholding. |
| Context read over the limit (like #4989's `campaigns.component.ts`) | Skipped; named in the recap. |
| Rule-file discovery | `git_get_tree` listings, no `repos_get_content` call on a directory in the session events. |
| Unread patch-less file with `calendar` in its path | Step 3.2 exclusion notice (path rule). |

## 5. Measure

Repeat the R1 scan (per-session tool output from `guild session events
--events all`, token totals from `guild session get`) on the first 200
reviews under 0.1.20:

- No `prompt is too long` errors without a notice (SC-001).
- No PR declined by gate A1 that a 0.1.17 run reviewed (SC-003).
- Input tokens per model call on a follow-up review of a large PR,
  against R1's figures. If plain text holds, compare base64 and raw on
  the same file: the per-call difference after one rule-file read gives
  the real ratio for research R4, and the read limits in research R7 can
  rise in a patch release.
