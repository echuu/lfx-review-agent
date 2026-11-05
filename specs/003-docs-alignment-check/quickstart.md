# Quickstart: Documentation alignment check

Validation steps for the maintainer. Every Guild command here uses the
`lfx-one` credential or the `linux-foundation` account, so the maintainer
runs it.

## 1. Confirm the replay harness (before implementation)

Research R12. Decide whether the replay can run outside `lfx-reviews`.

1. **Sandbox repository.** Create a private repository in the
   `linuxfoundation` org for the replay. Turn off GitHub Actions (copied
   workflows would run) and give no LFX team access, so copied
   `CODEOWNERS` files request no reviews.
2. **Test agent in its own directory.** The main checkout's local
   `guild.json` points at the production agent, so never save the test
   agent from it. Fork into a separate directory and run every test-agent
   command from there:

   ```bash
   guild agent fork linux-foundation~lfx-review-agent --name lfx-review-agent-test --owner linux-foundation --directory ~/projects/linuxfoundation/lfx-review-agent-test
   guild workspace create lfx-reviews-test
   ```

   Add the test agent to the test workspace (`guild workspace agent`
   group) and set `REVIEWER_LOGIN` and `REVIEW_AGENT_HELP_URL` there.
3. **Triggers and credential.** Create the webhook triggers for the
   sandbox repository only, in the test workspace.
   `scripts/onboard-repo.sh` hard-codes
   `AGENT=linux-foundation~lfx-review-agent`, so run a local copy with
   `AGENT` set to the test agent (this feature does not change the
   script). Open a throwaway PR in the sandbox, then confirm:
   - a session appears in the test workspace and none in `lfx-reviews`;
   - `guild session events <id>` shows 200 responses for the GitHub
     tools, with no policy denial, so the forked agent can use the
     `lfx-one` credential.

| Result | Meaning |
| --- | --- |
| Step 3 passes | Run the replay (section 3) as a gate before publishing to `lfx-reviews` |
| Credential or trigger check fails | No replay before rollout. Publish 0.1.23 to `lfx-reviews` and treat the success criteria as monitoring of the first 48 hours, not a gate. The restore path is `guild agent unpublish linux-foundation~lfx-review-agent`; confirm with `guild agent versions` that `lfx-reviews` is back on 0.1.22 |

## 2. Rollout order

1. Step 1, recorded in research R12.
2. Implement on a branch; `markdownlint` passes; the Agent Version Check
   passes at 0.1.23.
3. Replay (section 3) on the test agent at 0.1.22, then at the branch
   version. Load each version from the test agent's directory only:

   ```bash
   cd ~/projects/linuxfoundation/lfx-review-agent-test
   cp ~/projects/linuxfoundation/lfx-review-agent/PROMPT.md PROMPT.md   # check out the version to test first
   guild agent save --publish -m "replay: <version>"
   ```

   Never run `guild agent save` for the test agent from the main
   checkout: its `guild.json` points at the production agent. Before
   each save, confirm `guild.json` in the current directory names the
   test agent.
4. Merge, then from an updated `main`:

   ```bash
   guild agent save --publish --version-number 0.1.23 -m "Documentation alignment check (spec 003)"
   ```

## 3. Replay

Keep the replay set and results outside this repository (tasks T003,
T027): it is public, and some onboarded repositories are private.

Pick about 20 PRs merged in the last 30 days from onboarded repositories:
at least 5 `feat`, 5 `fix`, and 5 chore, refactor, test, CI or dependency
PRs. Include at least 3 that rename or remove a documented item, 2
multi-round cycles, and one whose base branch gets a "Documentation"
section ([contracts/documentation-section.md](contracts/documentation-section.md)).

For each one, push its base commit and its head commits (in order, for a
cycle) to branches in the sandbox repository and open a PR as an org
member. Copy the PR title; copy the body with every `@mention` removed.
Run the test agent at 0.1.22, then at the candidate, on separate copies.

Record per PR: change class, verdict, every `[docs]` finding with its two
locations, the reminder, and the Documentation gaps line. A maintainer
marks each `[docs]` finding correct or not.

Read the verdict from the Decision label, so the sandbox's missing CI and
org gates do not matter:

| Decision label | Verdict |
| --- | --- |
| ✅ Approved, ✅ Approved with minor comments, any ⚠️ "Review passed" | APPROVE |
| 🔴 Needs changes before approval | REQUEST_CHANGES |
| ⏸️ Not approved, new commits pushed | Rerun that copy |

Model runs vary. When a verdict changes between 0.1.22 and the candidate
and no `[docs]` finding explains it, rerun the candidate once on a fresh
copy. A change that does not repeat is noise; one that repeats fails
SC-005.

| Criterion | Pass when |
| --- | --- |
| SC-001 | Chore, refactor, test, CI and dependency PRs: no `missing` finding and no reminder; any `[docs]` finding is misaligned with both locations |
| SC-002 | Every `[docs]` finding names both locations, or the behavior and the surfaces checked |
| SC-003 | No reminder says a page is missing, wrong or out of date |
| SC-004 | Each qualifying PR gets one reminder on its first qualifying review; a later one only names new behaviors; no duplicates |
| SC-005 | Every verdict that is APPROVE at 0.1.22 and not at 0.1.23 traces to a `[docs]` finding |
| SC-005a | At least 90% of `[docs]` findings on those lost approvals are judged correct |
| SC-006 | No session overflows; doc reads stay inside the read budget (`guild session events`) |
| SC-007 | At least 80% of all `[docs]` findings are judged correct |
| SC-008 | Every summary with a gap has the Documentation gaps line in Decision; none without a gap has it |

## 4. Validation scenarios

These become README test checklist items.

| Scenario | Expected |
| --- | --- |
| PR renames an env var named in the root README, README unchanged | `[minor][docs]` misaligned finding citing the code line and the README line |
| `feat:` PR adds an env var, README has an env table, README unchanged | `[minor][docs]` missing-docs finding; Documentation gaps line in Decision; reminder in the review body and Details |
| `chore:` dependency bump | No `[docs]` finding, no reminder, no base-doc read in the session events |
| `chore:` PR that adds an API route | Treated as qualifying: reminder present |
| Follow-up that fixes other things and leaves the docs alone | Prior `[docs]` findings still open, counted under 🟡 with `(prior)`, verdict stays 🔴 when over 2 |
| Follow-up that updates the cited doc | Findings resolved; no gaps line |
| Follow-up whose commits add no new named item | No second reminder |
| Follow-up that adds a new config key | Reminder naming only the new key |
| PR with five missing-docs gaps | Three findings, one folded item with two lines, five counted |
| Repository with a declared section | Declared paths used; reminder names the declared site and owner in code spans, no notification |
| PR description says "docs not needed" or has a `docs-not-needed` label | No effect |
| A maintainer dismisses a 🔴 review that requested changes only for `[docs]` gaps; the author pushes again without doc changes | Those gaps are resolved ("dismissed by a maintainer") and not counted; only new gaps are raised |
| The stale-approval setting dismisses an approval that carried one `[docs]` gap | The gap still counts on the next follow-up |
| A maintainer dismisses an own review whose session summary was never written (abandoned run) | Its `[docs]` gaps are resolved ("dismissed by a maintainer") |
| After a dismissal resolved a gap on `MAX_RETRIES`, a new commit renames `MAX_RETRIES` again | A new `[docs]` finding is raised |
| Force-push to a PR with an open `[docs]` gap (whole-PR fallback) | The gap is classified as prior and counted once, not raised again |
| Named item containing `<!--` | Echoed in a code span with the comment markers removed; no extra marker in the review body |
