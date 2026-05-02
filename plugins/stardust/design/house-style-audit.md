# House-style audit — workflow

> Status: **first complete spec.** Defines how the lead designer runs
> a house-style audit, what the histogram captures, how to interpret
> it, and how the outputs feed back into the curator pass.

The audit is the system's anti-bloat discipline. Move catalogs grow;
without periodic pruning they accrete defaults that make stardust
predictable across unrelated brands — exactly the failure the system
exists to prevent. The audit is how that pruning happens, on a real
signal, not by intuition.

## Purpose

Detect moves that have become accidental defaults — recurring across
unrelated `brand_axes` regardless of intent — and surface them for
**demotion** (flag as default-combo) or **retirement** (archive from
the catalog). Also detect moves that are never chosen (catalog
deadweight) and surface them for retirement.

## Scope

The audit runs at **two scopes**:

### Per-project (runs at user runtime)

- **Trigger.** Anytime the user runs the audit explicitly, or as the
  closing step of a `migrate` run when all pages are migrated.
- **Sample.** The pages of the current redesign, read from the
  `## Committed moves` block of each page-shape brief.
- **Output.** `<user-project>/stardust/audits/<date>-project.md` (and
  `.json`).
- **Detects.** Within-redesign repetition: moves that show up on
  every page regardless of page type, suggesting the redesign has
  collapsed into a single visual idea.
- **Owner.** The project owner (the user running stardust).

### Plugin-wide (runs at curator-pass time)

- **Trigger.** When the corpus crosses meaningful thresholds (~20
  exemplars, periodic intervals, or before a major curator pass).
- **Sample.** All entries in `plugins/stardust/exemplars/`,
  optionally including manually-promoted user critiques and the
  plugin maintainer team's test redesigns. The sample size is
  whatever has accumulated upstream — there is no automatic
  telemetry.
- **Output.** `plugins/stardust/audits/<date>-plugin-wide.md` (and
  `.json`).
- **Detects.** Cross-brand defaults: moves that recur across
  unrelated `brand_axes` regardless of source brand. These are the
  moves that need demotion or retirement.
- **Owner.** A lead designer (per `learning-system.md` §8.4).

## Histogram generation (mechanical)

The data step. Can be a script, an `$stardust audit` invocation, or
a one-off Python notebook — implementation detail. The output shape
is what matters.

### Per-move histogram

For each move id present in the sample, record:

```json
{
  "<move-id>": {
    "count": <int>,                          // total occurrences
    "across_brand_axes": [<tag>, <tag>, ...], // distinct brand_axes tags of the
                                              //   sources that used this move
    "axis_diversity": <int>,                  // |across_brand_axes|
    "co_occurs_with": [                       // moves that frequently appear in
      { "move_id": "...", "count": <int> }   //   the same brief / exemplar
    ]
  }
}
```

### Combination histogram

For each combination from the **default-combinations registry** in
`divergence-toolkit.md` §1a, record:

```json
{
  "<combo-id>": {
    "count": <int>,                          // briefs/exemplars matching all triggers
    "across_brand_axes": [...]
  }
}
```

Plus **emergent combinations** — combinations of ≥3 moves that
co-occur in ≥3 exemplars but aren't yet registered:

```json
{
  "emergent-<n>": {
    "moves": [<move-id>, <move-id>, <move-id>],
    "count": <int>,
    "across_brand_axes": [...]
  }
}
```

### Output JSON

```json
{
  "audit_date": "<ISO-8601>",
  "scope": "per-project | plugin-wide",
  "sample": {
    "exemplars": <int>,
    "test_redesigns": <int>,
    "user_submissions_promoted": <int>,
    "project_pages": <int>                   // per-project only
  },
  "histogram": { ... },
  "default_combinations_observed": { ... },
  "emergent_combinations": { ... }
}
```

This JSON is the *raw signal*. It is **not** the audit. The audit is
the lead designer's interpretation of it.

## Interpretation (manual)

The lead designer reads the JSON and writes a companion markdown
file with **explicit decisions**. The interpretation is what makes
the audit useful — automation can compute counts, but only a human
can tell whether a high count means "the right move for this brand
territory" or "the assistant's accidental default."

### Required sections in the writeup

1. **Sample read.** A paragraph: how big was the sample, how
   representative is it, what are the known biases (e.g. "weighted
   toward editorial brands because most early exemplars came from
   that territory").
2. **Demote candidates.** Per move flagged for demotion: the
   `move_id`, the count, the `brand_axes` it spans, and a one-line
   rationale ("appears in 14 of 18 exemplars across 8 distinct
   `brand_axes` — this is the assistant's editorial default, not a
   move the brand chose"). The proposed action: add a
   default-combinations entry, OR add the move itself to §1
   anti-defaults if it's load-bearing across the catalog, OR ban
   without justification.
3. **Retire candidates.** Per move flagged for retirement: the
   `move_id`, count of zero or near-zero, last referenced
   exemplar/session, and a one-line rationale ("not chosen by any
   designer in 6 months; the abstraction we landed on doesn't
   match how designers describe what's distinctive in this axis").
   The proposed action: archive in §1a-archive with reason.
4. **Promote candidates** (optional). Sometimes the audit reveals
   the catalog is *missing* a move — a recurring pattern in
   exemplars that doesn't have a name. Per identified gap: a
   sketch of the candidate move and a note to feed it back into
   the curator pass.
5. **No-action notes.** Moves that *look* like defaults on the
   histogram but aren't — high count concentrated in a single
   `brand_axes` is a brand-appropriate staple, not a slop default.
   Document why each near-miss isn't actionable.

The writeup ends with a **decisions list** — a clean, scannable
summary of every demote/retire/promote action proposed. This list is
what the curator pass picks up at step 7.

## Outputs and how they feed the curator pass

Each audit produces:

- `<scope>-<date>.json` — raw histogram (machine-generated).
- `<scope>-<date>.md` — lead-designer interpretation with the
  decisions list.

The curator pass (per `curator-pass.md` step 7) reads the decisions
list and:

- For each **demote** decision: PR an entry into the
  default-combinations registry in §1a.
- For each **retire** decision: PR the move from §1a into a §1a
  archive subsection with the retirement reason recorded.
- For each **promote** decision: feeds the proposed move back into
  step 3 of the curator pass as a synthetic capture cluster (with
  the audit as provenance).

The audit does **not** make catalog changes directly. It produces
recommendations that the curator pass enacts via PR. This separation
keeps every change reviewable.

## Cadence

The audit is **not on a calendar**. Run when one of these holds:

- Corpus crossed a meaningful threshold (first 20 exemplars, then
  every ~30–50, then ad hoc).
- A curator is about to do a large pass and wants signal on what to
  prune.
- A maintainer has noticed convergence anecdotally and wants
  confirmation.
- A contributor flagged a move as suspect and the lead designer is
  evaluating whether to retire it.

Per-project audits are cheaper to run and can land routinely at
`migrate` completion. Plugin-wide audits are heavier and rare.

## Hard rules

- Never make catalog changes from the audit alone. The audit
  *recommends*; the curator pass *enacts*. This separation keeps
  changes reviewable.
- Never automate the interpretation step. The histogram is data;
  the read is judgment. Anyone reviewing the writeup must see the
  human read explicitly, not just the numbers.
- Never delete the JSON or markdown after a curator pass acts on
  them. The audit history is what lets future curators understand
  why a move was retired or demoted; deleting it would erase the
  reasoning.
- Never run a plugin-wide audit on a corpus the lead designer
  hasn't reviewed. The sample's biases must be named in the
  writeup; running blind produces misleading recommendations.

## What the audit deliberately does NOT do

- **Replace designer judgment.** The histogram is a starting point
  for a conversation, not a ranking.
- **Enforce decisions.** Audit recommendations don't auto-apply.
  Every change goes through the curator pass and a PR.
- **Operate on telemetry.** Per-user runtime data does not flow
  upstream automatically. The plugin-wide audit's sample is what
  contributors and maintainers have manually accumulated.
- **Score moves.** There is no "this move is good" / "this move is
  bad" rating. Moves are correct or incorrect *for a given brand
  context*, and the audit only flags the ones that have stopped
  having a context.

## References

- `plugins/stardust/design/learning-system.md` §5 — the audit's
  role in retirement and house-style detection.
- `plugins/stardust/design/curator-pass.md` step 7 — how the
  curator pass consumes audit output.
- `plugins/stardust/skills/stardust/reference/divergence-toolkit.md`
  §1a — the default-combinations registry and the move catalog the
  audit operates on.
- `plugins/stardust/audits/README.md` — file naming conventions and
  the histogram JSON schema (mirrored here in §Histogram generation).
