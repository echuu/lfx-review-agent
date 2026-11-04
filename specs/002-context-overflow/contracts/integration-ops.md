# Contract: `lfx-one-github` integration 1.0.6

Added to `integration/lfx-one-github.openapi.yaml`, to `guild.yaml`, and
to the README credential policy. The existing 20 operations keep their
paths and parameters.

## New operations

| Operation id | Tool | Request | Used by |
| --- | --- | --- | --- |
| `git_get_tree` | `lfx_one_github_git_get_tree` | `GET /repos/{owner}/{repo}/git/trees/{tree_sha}`, where `tree_sha` is `<commit>:<directory>` or a commit SHA for the root | Every directory listing; sizes for the read budget (research R3) |
| `repos_get_content_raw` | `lfx_one_github_repos_get_content_raw` | `GET /repos/{owner}/{repo}/contents/{path}` with `ref`, and header `Accept` pinned by `enum` to `application/vnd.github.raw+json` | Full-file reads as plain text (R4) |
| `graphql_read` (only if the enum probe passes) | `lfx_one_github_graphql_read` | `POST /graphql`, body `{query, variables}`, `query` pinned by `enum` to the fixed queries below | Reviews and review threads (R6) |

`git_get_tree` never takes `recursive`: a recursive tree of a large
repository is megabytes. One directory per call.

## `graphql_read` queries

Variables: `owner`, `repo` (strings), `number` (int), `after` (cursor or
null). The schema allows no other variable and no other `query`.

Reviews:

```graphql
query($owner:String!,$repo:String!,$number:Int!,$after:String){repository(owner:$owner,name:$repo){pullRequest(number:$number){reviews(first:100,after:$after){pageInfo{hasNextPage endCursor} nodes{databaseId author{login} state body submittedAt commit{oid} url}}}}}
```

Review threads:

```graphql
query($owner:String!,$repo:String!,$number:Int!,$after:String){repository(owner:$owner,name:$repo){pullRequest(number:$number){reviewThreads(first:100,after:$after){pageInfo{hasNextPage endCursor} nodes{isResolved isOutdated path line originalLine comments(first:50){nodes{databaseId author{login} body createdAt url}}}}}}}
```

A thread with more than 50 comments shows its first 50; the recap says
so. Both queries are read only. A mutation cannot be expressed through
them.

## Narrowed output schemas

`pulls_list_reviews` and `pulls_list_review_comments` get output schemas
that list only the fields the prompt reads. If Guild trims responses to
the schema, the reads shrink (research R6); if it doesn't, nothing
changes. Because a trim drops every field not listed, each schema keeps
every field any step reads:

- Reviews: `id`, `state`, `body`, `html_url`, `user.login`, `commit_id`,
  `submitted_at`.
- Inline review comments: `id`, `pull_request_review_id`,
  `in_reply_to_id`, `user.login`, `body`, `path`, `line`, `original_line`,
  `side`, `commit_id`, `html_url`, `created_at`, `updated_at`.

The implementation re-checks these lists against every use in
`PROMPT.md` before publishing.

## Constraints

- Every new operation is read only (spec FR-011).
- `repos_get_content_raw` returns files up to 100 MB, so the prompt calls
  it only after a tree listing gave a size within the read budget.
- If quickstart step 2 shows Guild drops the `Accept` header, the
  operation returns the JSON form. The prompt then keeps
  `repos_get_content` for file reads, and the operation stays unused.

## Credential policy

One new ALLOW policy on the `lfx-one` credential for the new operations,
limited to this workspace and agent, like the existing ones.
