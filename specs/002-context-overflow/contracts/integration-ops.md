# Contract: `lfx-one-github` integration 1.0.6

Added to `integration/lfx-one-github.openapi.yaml`, to `guild.yaml`, and
to the README credential policy. The existing 20 operations keep their
paths and parameters.

## New operations

| Operation id | Tool | Request | Used by |
| --- | --- | --- | --- |
| `git_get_tree` | `lfx_one_github_git_get_tree` | `GET /repos/{owner}/{repo}/git/trees/{tree_sha}`, where `tree_sha` is `<commit>:<directory>` or a commit SHA for the root | Every directory listing; sizes for the read budget (research R3) |

`git_get_tree` never takes `recursive`: a recursive tree of a large
repository is megabytes. One directory per call.

## Changed operations

| Operation id | 1.0.5 | 1.0.6 |
| --- | --- | --- |
| `repos_get_content` | No header parameter | Optional `Accept` header (below) |
| `issues_remove_label` | `DELETE /repos/{owner}/{repo}/issues/{issue_number}/labels/{name}`, `name` pinned by an `enum` that Guild is not known to enforce (body enums are not, research R5) | `DELETE /repos/{owner}/{repo}/issues/{issue_number}/labels/lfx-review-agent-pending`, no `name` parameter |

`repos_get_content` gains an optional header parameter `Accept`,
documented with `enum: ["application/vnd.github.raw+json"]`. Guild does
not enforce the enum (research R5); any `Accept` value on this GET is
still a read. OpenAPI allows one GET per path, so this cannot be a
separate operation.

Both operations keep their ids and tool names, so the ALLOW policies
that name them should still apply; quickstart step 2 checks that. The
prompt stops passing `name` to `issues_remove_label`. The live agent
(0.1.19) pins 1.0.5, so publishing 1.0.6 does not change it.

OpenAPI 3.0.3 ignores a header parameter named `Accept`, so the
`repos_get_content` 200 response also lists the
`application/vnd.github.raw+json` media type next to `application/json`.
The narrowed history schemas mark nullable fields `nullable: true`.

## No GraphQL operation

Guild does not enforce body `enum` values (research R5), so a
`POST /graphql` operation could carry any query, mutations included. It
waits for a read-only token (#19).

## Narrowed output schemas (dropped)

1.0.6 gave `pulls_list_reviews` and `pulls_list_review_comments` output
schemas listing only the fields the prompt reads. 1.0.7 drops them to stay
under Guild's generator size limit (research R11). Both operations return
`OkList` again.

## Constraints

- Every new operation is read only (spec FR-011).
- A raw read returns files up to 100 MB, so the prompt reads a file
  only after a tree listing gave a size within the read budget.
- If quickstart step 2 shows Guild drops the `Accept` header, the read
  returns the JSON form, and the prompt stops passing the header.

## Credential policy

One new ALLOW policy on the `lfx-one` credential for `git_get_tree`,
limited to this workspace and agent, like the existing ones.
