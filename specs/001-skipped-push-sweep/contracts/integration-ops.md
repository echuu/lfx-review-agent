# Contract: new `lfx-one-github` operations (integration 1.0.5)

Added to `integration/lfx-one-github.openapi.yaml`, to `guild.yaml`, and
to the README credential policy. The existing 17 operations are unchanged.

| Operation id | Tool | Request | Used by |
| --- | --- | --- | --- |
| `search_issues` | `lfx_one_github_search_issues` | `GET /search/issues` with `q`, `sort`, `order`, `per_page`, `page` | Sweep candidate searches (R3, R5) |
| `issues_add_labels` | `lfx_one_github_issues_add_labels` | `POST /repos/{owner}/{repo}/issues/{issue_number}/labels`, body `{"labels": ["lfx-review-agent-pending"]}` | Step 3.5, gates B and C, Step 6 re-add |
| `issues_remove_label` | `lfx_one_github_issues_remove_label` | `DELETE /repos/{owner}/{repo}/issues/{issue_number}/labels/{name}` | Step 6, sweep cleanup |

## Constraints

- `issues_add_labels` pins `labels` to an `enum` of the one label name, the
  same way `issues_create_reaction` pins `content` to `eyes`.
- `issues_remove_label` returns 404 when the label is not on the PR. The
  prompt treats that as success.
- Label calls are best effort: a failure never stops a review. Step 7
  reports it.
- `search_issues` is read only. The sweep uses at most 1 label search and 5
  never-reviewed searches per run, each with `per_page=3`.

## Credential policy

One new ALLOW policy on the `lfx-one` credential for the new operations,
limited to this workspace and agent, like the existing three.
