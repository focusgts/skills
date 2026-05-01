# Learning system — runtime contract

This file defines what stardust **does at runtime** with the moves catalog
and the exemplar/critique corpus. The design rationale lives in
`plugins/stardust/design/learning-system.md`; this file is the contract the
implementation honors. It is loaded by `direct` and `prototype` (and any
sub-skill that produces or consumes `design-facts`).

The shape every ingester and entry uses is `design-facts`, specified in
`plugins/stardust/design/design-facts.md`. The named-move vocabulary lives
in `divergence-toolkit.md`.

---

## Where things live

```
stardust/exemplars/    # plugin-shipped or org-curated reference set; read-only at runtime
stardust/captures/     # designer-submitted raw captures (one file per capture)
stardust/critiques/    # per-redesign critique entries (one file per critique)
stardust/audits/       # output of the house-style audit (timestamped)
```

Exemplars and critiques use the entry schema in
`design/learning-system.md` §2. Captures use the minimal capture schema in
`design/learning-system.md` §4 (source + one sentence; no taxonomy).

## What stardust reads at `direct` time

After resolving the user's intent (per `intent-reasoning.md`), but **before**
proposing target tokens or layout commitments:

1. Compute the **target brand_axes** from the resolved intent and the
   extracted current state (`stardust/current/DESIGN.md`,
   `stardust/current/PRODUCT.md`). Open tags; no enum.
2. Filter the corpus (`stardust/exemplars/` ∪ `stardust/critiques/`) by
   `brand_axes` overlap. An entry qualifies when at least two of its tags
   overlap the target tags, or one tag overlaps and the entry is `stunning`.
3. From the qualifying set, surface **anchors**:
   - Up to **three `stunning`** entries, ordered by tag-overlap richness.
   - **One `slop`** entry, ordered by tag-overlap richness, to anchor the
     anti-pattern side.
4. Cite the anchors in `stardust/direction.md` under a `# Anchors` section,
   each with: `id`, `verdict`, `brand_axes`, the entry's `why`, and the
   moves it exhibits.

Anchors are reference, not template. The proposed direction is **not bound**
to copy moves from anchors. They constrain the *space* the proposal lives
in, not its specific choices.

## What stardust enforces at `prototype` time

Before generating any HTML or asking impeccable to render:

1. Read the chosen moves for each proposal from
   `stardust/proposals/<page>/proposal-<n>.yaml`.
2. **Move-combination contract.** For each proposal:
   - Reject if fewer than **3 moves** are committed.
   - Reject if those moves span fewer than **3 distinct axes**.
   - Reject if every move lacks a `brand_justification` field tracing it to
     a fact in the extracted brand or the resolved direction.
   - Flag (not reject) if the move set matches a `default-combinations`
     entry in `divergence-toolkit.md`. Surface the warning to the user and
     ask them to confirm or revise before continuing.
3. **Pairwise variance.** For a set of N proposals on the same page, every
   pair must differ by **at least 2 moves**. Token-only differences
   (palette, spacing) do not count.
4. **Anchor coherence.** Each proposal must cite at least one anchor from
   the `direct`-time anchor set (`stardust/direction.md` `# Anchors`). Cited
   anchors must share at least one move with the proposal, OR the proposal
   must explain in one line why it diverges from the anchor.

A proposal that fails any of 2–4 is returned to the agent with the failing
rule and the offending field, not silently corrected.

## What stardust writes

**Captures** (when the user submits one mid-session):

```
stardust/captures/<timestamp>-<slug>.yaml
---
source:    { kind, ref }
sentence:  "<one line: what this is doing that's different>"
submitted_by: <handle>
submitted_at: <ISO-8601>
```

That is the entire schema. Stardust does not auto-tag, auto-classify, or
add fields. The curator pass (out of band) handles abstraction.

**Critiques** (per-redesign, when designer reviews proposals):

One file per reviewed proposal, schema per `design/learning-system.md` §2,
with `verdict` + `why` mandatory and `moves` populated from the proposal
itself.

**Candidate moves** (when stardust observes an exemplar that uses a move
not yet in `divergence-toolkit.md`): emit the candidate to
`stardust/captures/candidates/<timestamp>-<id>.yaml` with `id` prefixed
`candidate/`. Do not promote. Do not silently extend the catalog.

## Empty-corpus and small-corpus behavior

The corpus will be empty or thin for the early life of the system. Stardust
must degrade gracefully:

- **Empty corpus.** Skip the anchor step at `direct` time. Note in
  `stardust/direction.md` that no anchors were available. Proceed to
  proposals using only `divergence-toolkit.md` moves.
- **Thin corpus** (<3 qualifying entries after filter). Surface what's
  there, even if it's a single entry. Note the thinness in
  `stardust/direction.md` so the user knows the anchor signal is weak.
- **No `slop` entry available** for the target axes. Skip the slop anchor;
  do not substitute from a different brand-axis territory just to fill the
  slot.

Move-combination enforcement at `prototype` time is unchanged regardless of
corpus size. The catalog is the floor; the corpus is the ceiling.

## Cross-references

- `divergence-toolkit.md` — the moves catalog and the
  `default-combinations` registry. Authoritative for move ids and axes.
- `plugins/stardust/design/design-facts.md` — the shape of every entry's
  `source.facts` block and every ingester's output.
- `plugins/stardust/design/learning-system.md` — the design rationale,
  decisions log, and contribution pipeline.
- `state-machine.md` — page lifecycle and stale rules. Anchor changes
  that would meaningfully alter direction count as a direction change and
  mark prototyped/migrated pages stale.

---

## Hard rules

- Never invent moves at runtime. Every move id stardust uses must already
  exist in `divergence-toolkit.md`, or be prefixed `candidate/` to mark a
  not-yet-promoted observation.
- Never silently filter out the corpus when it's thin. A thin corpus is
  surfaced to the user, not hidden.
- Never auto-promote a candidate move into the catalog. Promotion is a
  curator-pass action, not a runtime action.
- Never extend or rewrite the capture schema. Frictionless contribution is
  load-bearing; runtime additions defeat the purpose.
- Never bypass the move-combination contract on grounds of "the user said
  go." A user override is permitted only by an explicit
  `--bypass-contract` flag passed to the prototype sub-command, recorded in
  the proposal's provenance.
