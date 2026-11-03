# Contract: sweep trigger input

The hourly time trigger passes this JSON object as the run's input
(`guild trigger create --type time --input '<json>'`).

```json
{
  "lfx_review_agent_event": "sweep",
  "repos": ["linuxfoundation/<repo>", "linuxfoundation/<repo>"]
}
```

## Rules for Step 0

- A run is a sweep only when the input's **top level** has
  `lfx_review_agent_event` equal to `"sweep"` **and** has no
  `pull_request`, `comment` or `issue` object. A webhook payload with the
  key anywhere else is a webhook run.
- `repos` entries must match `^linuxfoundation/[A-Za-z0-9._-]+$`. Ignore
  any that don't. With no valid entry, stop and report a configuration
  fault.
- Guild delivers the input as a bare JSON object at the end of the run's
  first message, after its preamble (probe of 2026-11-03, research R1). The
  prompt must not depend on the preamble text, only on the JSON object.

## Producer

`scripts/sweep-trigger.sh` builds `repos` from the workspace's active
`pull_request` webhook triggers for `linux-foundation~lfx-review-agent`,
then creates the time trigger or updates its `--input`. It never writes the
list to a file in the repository.
