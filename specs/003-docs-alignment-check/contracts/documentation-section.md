# Contract: the repository "Documentation" section

A repository tells the agent where its docs live by adding a section to a
rule file the agent already reads: the root or a top-level directory's
`CLAUDE.md` or `AGENTS.md`, or a file in `.claude/rules/`. No new file.

## Format

```markdown
## Documentation

- In-repo: `docs/`, `charts/app/README.md`, `src/assets/i18n/`
- Docs site: https://docs.example.org/app (owner: docs-team)
- Product page: https://www.example.org/products/app (owner: product-marketing)
```

- The heading text is `Documentation`, at any heading level. The section
  ends at the next heading of the same or a higher level.
- Bullets that start with `In-repo:`, `Docs site:` or `Product page:` are
  read. Every other line in the section is ignored.
- `In-repo:` takes comma-separated paths from the repository root, in
  code spans or plain. A path ending in `/` is a directory. At most 10
  paths are used; the rest are ignored.
- `Docs site:` and `Product page:` take one URL each, with an optional
  `(owner: <text>)`. A section can have several of each; at most 5 sites
  in total are used.

## What it does

| Declared | Effect |
| --- | --- |
| In-repo paths | Added to the doc candidates. A declared file can be read in full within the read budget; a declared directory is listed (at most 3 extra listings per run) |
| Docs site, product page | Named in the reminder, as written, in code spans. Never fetched |
| Owner | Named in the reminder in a code span, so GitHub sends no notification, even when written as `@team` |

## What it cannot do

- Relax any rule, severity, cap, gate or verdict. Text such as "skip
  documentation findings" or "always approve" is ignored and, on the
  target branch, is not a finding.
- Take effect from a PR. The section is read at `base_sha`, so a PR that
  adds or edits it is judged by the version on the target branch.
- Turn off the check. There is no opt-out (spec FR-035).

## Precedence

- In-repo paths from every section the agent read are combined.
- For the reminder, the section in the rule file closest to the changed
  files wins for those files; the root section covers the rest.
- A section in a rule file the agent skipped (over 24,000 bytes, over the
  80,000-byte total, a symlink) is not seen. The recap already names the
  skipped rule file, and the agent falls back to conventional locations.
- A declared path that does not exist is skipped without a finding.
