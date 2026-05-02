# `plugins/stardust/exemplars/`

The **canonical exemplar corpus** that ships with the stardust plugin.
Read at `direct` time to anchor proposals, per the runtime contract in
`plugins/stardust/skills/stardust/reference/learning-system.md`.

This corpus is **plugin-side and contributor-curated**. User-side
exemplars (optional, project-local) live in
`<user-project>/stardust/exemplars/` and are read additively at
runtime, but never written to from a user session.

## What goes here

One file per exemplar, named `<id>.yaml` where `<id>` matches the
entry's `id` field. Schema is defined in
`plugins/stardust/design/learning-system.md` §2 and uses the
`design-facts` shape from `plugins/stardust/design/design-facts.md`
for the `source.facts` block.

Required fields: `id`, `source`, `verdict`, `moves`, `brand_axes`,
`designer`, `why`. Optional: `anti_pattern`, `critique_mode`
(critique-derived entries only).

## How to contribute

1. Author the entry as a YAML file under this directory.
2. Cite at least one move id from `divergence-toolkit.md` §1a (or use
   `candidate/`-prefixed ids for moves not yet promoted).
3. Open a PR against this repo. The plugin maintainer (or a contributor
   acting as curator) reviews, the originating designer signs off, and
   the entry merges.

See `plugins/stardust/design/learning-system.md` §4 for the full
contribution pipeline (capture → cluster → abstract → promote).

## What this directory does NOT contain

- **Captures** (raw "this is striking" observations) — those live in
  `plugins/stardust/captures/`.
- **Critiques** (per-redesign judgments of stardust's own output) —
  those live project-side in `<user-project>/stardust/critiques/`,
  and only land here if a contributor explicitly promotes one via PR.
- **Move definitions** — those live in
  `plugins/stardust/skills/stardust/reference/divergence-toolkit.md` §1a.
  Exemplars *reference* moves; they don't define them.

## Status

Empty at landing. The corpus grows through the contribution pipeline.
