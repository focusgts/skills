# `plugins/stardust/captures/`

The **contributor staging queue** for raw captures awaiting curator
pass. Plugin-side and PR-driven; designed to stay frictionless.

This is where contributors submit "I saw something striking" observations
without doing taxonomy work. The curator pass (per
`plugins/stardust/design/learning-system.md` §4) groups captures into
clusters, and clusters of ≥2–3 become *candidate moves* (queued under
`candidates/`) before promotion to the catalog.

## What goes here

One file per capture, named `<timestamp>-<short-slug>.yaml`. Minimal
schema (intentionally — frictionless or it dies):

```yaml
source:
  kind: url | figma | image | pdf | video
  ref:  <URL or path>
sentence:     "what is this doing that's different"   # one line
submitted_by: <handle>
submitted_at: <ISO-8601>
```

That is the entire schema. **Do not** add taxonomy fields, axis tags,
verdicts, or move references. The whole point of capture is that it
costs the contributor 30 seconds. Anything that requires longer thought
belongs in an exemplar entry (`../exemplars/`), not here.

## How to contribute

1. Drop a file in this directory matching the schema above.
2. Open a PR. No review gate beyond schema conformance — captures are
   raw signal, not vetted material.
3. The curator pass (out-of-band) groups your capture with similar
   captures from other contributors and decides whether to promote.

## Promotion

Captures are not kept here forever. When the curator pass abstracts a
cluster of captures into a named move:

- The move is added to `divergence-toolkit.md` §1a.
- The move's `provenance.captures[]` list cites the originating capture
  ids.
- The captures themselves may be archived (moved to `archive/`) or
  retained as historical reference. Implementation detail of the
  curator pass — see `plugins/stardust/design/learning-system.md` §4.

## What this directory does NOT contain

- **Curated exemplars** — `../exemplars/`.
- **Candidate moves** (captures clustered but not yet promoted) —
  `./candidates/`.
- **User-runtime captures** — those live project-side in
  `<user-project>/stardust/captures/`. They can be promoted upstream
  by PR'ing them here, but nothing flows automatically.

## Status

Empty at landing. The queue fills as contributors submit captures.
