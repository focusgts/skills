# `plugins/stardust/captures/candidates/`

**Candidate moves** — clusters of captures that have been abstracted
into a named move but **not yet promoted** to the catalog in
`divergence-toolkit.md` §1a.

Sitting between raw captures (`../`) and the canonical moves catalog,
candidates are the curator-pass output that's awaiting designer signoff
and the final PR to the divergence toolkit.

## What goes here

One file per candidate move, named `candidate-<axis>-<short-name>.yaml`.
Schema is the move schema from
`plugins/stardust/skills/stardust/reference/divergence-toolkit.md` §1a,
with the `id` field prefixed `candidate/`:

```yaml
id:               candidate/<axis>/<short-name>
axis:             <one of layout|type|palette|image|motion|tone|structural>
summary:          <one line>
when_to_use:      <short note>
when_not_to_use:  <short note>
exemplars:        []                            # populated when promoted
provenance:
  added_by:       <curator handle>
  from_session:   <curator session id>
  captures:       [capture-id, capture-id, ...] # the cluster that produced this
```

The `candidate/` prefix is the signal that this move is observed but
not yet validated by designer signoff or contract review.

## Runtime treatment

At runtime, stardust may emit `candidate/`-prefixed move ids in
`observed_moves` blocks and in proposal commitments — but proposals
that depend exclusively on candidate moves are flagged for review. See
`reference/learning-system.md` Hard rules.

## Promotion

When a candidate move is ready to enter the catalog:

1. Originating designer (the one who submitted the captures) signs off
   that the abstraction matches what they saw.
2. PR against `plugins/stardust/skills/stardust/reference/divergence-toolkit.md`
   adds the entry to §1a *without* the `candidate/` prefix.
3. Move file in this directory is deleted (or archived) since the
   catalog now holds it.
4. Any exemplars or captures that referenced the candidate id are
   updated to use the promoted id.

## What this directory does NOT contain

- **Captures** awaiting clustering — `../`.
- **Promoted moves** — `plugins/stardust/skills/stardust/reference/divergence-toolkit.md` §1a.
- **Anti-default moves** — `divergence-toolkit.md` §1.

## Status

Empty at landing. Populated as the curator pass abstracts capture
clusters into candidate moves.
