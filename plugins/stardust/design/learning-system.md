# Stardust learning system — design v1

> Status: **locked from designer review.** All v0 open questions are
> resolved (see §8). Nothing here is implemented yet — the doc now informs
> the spec and implementation work that follows.

## Why this exists

Two observed weaknesses in stardust today:

1. **Within-brand sameness.** Multiple proposals for one site tend to share a
   skeleton and differ mostly on tokens (palette, spacing).
2. **Cross-brand sameness.** Designs across *different* brands also converge on
   a recurring vocabulary, so a fintech and a children's product can end up
   structurally similar.

Both failures are structural, not chromatic. The fix is a **moves catalog**
that grows under designer guidance, plus a contract that forces every proposal
to commit to a *design grammar* — not a paint job — before color is chosen.

## Scope

In scope for this design:

- Move-combination contract.
- Exemplar / critique corpus (format-agnostic input).
- The shared `design-facts` shape (the interface every ingester produces).
- Designer-driven move contribution pipeline.
- Mining / retirement loop.

Out of scope (separate sessions):

- **Figma-as-extract-source** (Figma file as the *source of truth* for a
  redesign). Depends on `design-facts` being defined here; otherwise its own
  doc.
- Any implementation. This doc is design only; the spec and code follow.

---

## Actors

Three actors, with distinct rights and responsibilities:

1. **Plugin user** (designer or developer running stardust on their own
   project). Reads the canonical corpus and moves catalog at runtime.
   Produces critiques and (optionally) captures during a redesign session
   under the project working directory.
2. **Plugin contributor** (a "great designer" working with the maintainer
   team). Submits captures, abstracts them into named moves with the
   curator, and authors exemplar entries that ship with the plugin. This
   work happens against this repo (PRs, issues, reviews).
3. **Plugin maintainer** (developer of the skill). Owns the schemas, the
   runtime contract, the divergence-toolkit, and the eval suite. Runs the
   curator pass. Does not judge designs themselves.

**This session adopts a hybrid learning model** (per decisions log §8.9):

- The **canonical corpus** (exemplars + the moves catalog) is
  contributor-curated and ships plugin-side.
- **Per-project critiques** are written locally in the user's working
  directory and stay there by default. They never flow upstream
  automatically.
- An **explicit, manual upstream submission path** exists: a user (or
  contributor) can promote a local capture or critique into the plugin
  corpus by opening a PR against this repo. Stardust does not auto-submit.
- This matches stardust's existing project-relative path convention
  (`<user-project>/stardust/...`), so the runtime contract paths in
  `reference/learning-system.md` need no refactor.

Federated learning (telemetry, automatic upstream flow) is a separate
future design session. The current model is "plugin-only by default with a
manual promotion path."

### Where things live

```
# Plugin-side (contributor-curated, ships with the plugin, read-only at user runtime)
plugins/stardust/exemplars/             # canonical exemplar corpus
plugins/stardust/captures/              # contributor staging queue (PR-driven)
plugins/stardust/captures/candidates/   # candidate moves not yet promoted
plugins/stardust/audits/                # plugin-wide histogram audits

# Project-side (local to the user's redesign, not auto-shared)
<user-project>/stardust/exemplars/      # optional local additions (project-only)
<user-project>/stardust/captures/       # optional local captures (PR to promote)
<user-project>/stardust/critiques/      # per-redesign critique entries
<user-project>/stardust/audits/         # per-project histogram (this redesign)
```

At `direct` time, stardust reads the **union** of plugin-side and
project-side exemplars (with plugin-side as the trusted base; project-side
as opt-in additions for that user's session). Captures and critiques are
written project-side. Promotion to plugin-side is a manual contributor
action — never automatic.

---

## 1. Move-combination contract

A **move** is a single, named design decision along one axis. Examples:

- `layout/asymmetric-grid`, `layout/full-bleed-cinematic`, `layout/horizontal-scroll`
- `type/editorial-serif-display`, `type/mono-led`, `type/poster-scale`
- `image/photographic-edge`, `image/abstract-only`, `image/no-imagery`
- `motion/kinetic`, `motion/still-but-precise`
- `tone/dry-witty`, `tone/declarative`, `tone/poetic`

Each move carries:

```
id:           layout/asymmetric-grid
axis:         layout
summary:      one line
when_to_use:  short note grounded in brand axes
when_not_to_use: short note
exemplars:    [exemplar-id, exemplar-id, ...]
provenance:   { added_by, from_session, captures: [...] }
```

**Proposal contract** (enforced at `prototype` time):

- A proposal commits to **≥3 moves spanning ≥3 different axes** *before* color
  is chosen.
- A set of N proposals must **pairwise differ by ≥2 moves** — not by tokens
  alone.
- A **default-combinations registry** flags slop combos (e.g.
  `layout/centered-hero` + `type/sans-default` + `motion/none`). Proposals
  matching one are auto-flagged for divergence.
- **Brand-justification rule.** Stardust has no default move set. Every chosen
  move must be traced to a fact about the extracted brand (audience,
  tactility, formality, scale, etc.). No move may be picked by inertia.

The brand-justification rule is the cross-brand safeguard: if every move is
brand-traced, two different brands can't accidentally land on the same
skeleton.

---

## 2. Exemplar / critique corpus

Paths are defined in the Actors section above. Summary:

- **Exemplars** live in two scopes: plugin-side (canonical, shipped with
  the plugin) and project-side (optional local additions). At runtime
  stardust reads the union; plugin-side is the trusted base.
- **Critiques** live project-side only by default. The user can promote a
  critique into the plugin corpus via PR; nothing flows upstream
  automatically.
- **Captures** live in either scope. Project-side is for designers who
  want to remember something locally; plugin-side is the contributor
  staging queue.

**Entry schema** (one YAML/MD file per exemplar; critique entries add
`critique_mode` plus mode-specific fields per §2a):

```
id:           tactile-chairs-2024
source:
  kind:       url | figma | image | pdf | video
  ref:        <URL or path>
  facts:      <design-facts block — see §3>
verdict:      stunning | strong | competent | slop
moves:        [layout/asymmetric-grid, type/editorial-serif-display, ...]
brand_axes:   [tactile, niche, editorial]
designer:     <handle>
why:          1–3 lines on what makes it work (or fail)
anti_pattern: optional tag (e.g. "gradient-blob-hero")
critique_mode: quick | qualified | comparative | conversation   # critiques only
```

**`brand_axes` are open tags, not a fixed enum.** Designers describe brands
in their own vocabulary; the corpus accepts that. Cost: near-duplicates will
appear (`playful` / `whimsical` / `joyful`) and the curator pass is
responsible for periodic merges, not the contributor.

**Verdict has four levels: `stunning | strong | competent | slop`.**
`competent` is the most load-bearing — it's where most failures hide.

**Anti-patterns live in the same corpus**, as entries with `verdict: slop`
plus an `anti_pattern` tag. Symmetric storage means stardust can use them
without a second read path.

**Read path.** At `direct` time, stardust filters the corpus by the *target
brand's* `brand_axes`, not by visual similarity. Surfaces ~3 `stunning` + 1
`slop` as anchors before generating. At `prototype` time, the chosen moves
are checked against exemplar tags so generated variants point back to real
references.

**Verdict + 1–3 line `why` is the maximum** *for the default (`quick`) mode.*
No essays. Depth in `quick` comes from volume of entries, not length of any
one entry. Other modes — `qualified`, `comparative`, `conversation` — trade
volume for structured depth where it carries information; see §2a.

---

## 2a. Critique modes

A single critique schema can't be both frictionless and qualified — the two
pull in opposite directions. The corpus accepts **four modes**, recorded on
each critique entry as `critique_mode`. The runtime weights them
differently (per `reference/learning-system.md`); the curator pass treats
them differently (per §6).

### `quick` (default, ~30 seconds)

Verdict + 1–3 lines of `why`. The schema in §2. Designed for high cadence
and low friction. Most session critiques are `quick`.

### `qualified` (opt-in, ~5–10 minutes)

For proposals that merit depth. Adds the following fields:

```
critique_mode: qualified
per_dimension:
  layout:  <short note>
  type:    <short note>
  palette: <short note>
  motion:  <short note>
  image:   <short note>
  tone:    <short note>
working_moves:    [ move-id, ... ]    # moves that landed
failing_moves:    [ move-id, ... ]    # moves that didn't land
counterfactual:   "this would be stunning if ..."   # 1–2 lines
anchor_read:      "appropriate" | "lazy" | "wrong-territory" + 1 line   # optional
designer_context: "<perspective; e.g. editorial sensibility, type designer>"
```

Per-dimension assessment forces specificity (vague "feels generic" becomes
locatable "type pairing reads generic; layout grammar is distinctive").
Move attribution connects critique directly to the catalog.
**Counterfactual is the highest-signal field** — most useful design
critique hides in "this would be stunning if".

### `comparative` (~2–5 minutes for a set)

Designer reviews N proposals **together**, not one-by-one. Pairwise
judgment is sharper than absolute and faster than running `qualified` per
proposal.

```
critique_mode: comparative
subject:        [ proposal-id, proposal-id, ... ]
ranking:        [ best → worst, by id ]
differentiating_moves: [ move-id, ... ]   # what actually distinguishes the set
designer_context: "<perspective>"
why:            1–3 lines on what the ranking turns on
```

Comparative critiques **do not anchor**. They update move metadata
(`when_to_use` / `when_not_to_use`) in the moves catalog at curator-pass
time.

### `conversation` (~30 minutes, recorded)

A designer + curator call. Most natural format for senior designers who
prefer talking to writing. Trades writing burden for review burden.

```
critique_mode: conversation
recording:        <ref or path>
transcript:       <ref or path>
extracted_entries: [ entry-id, ... ]   # quick or qualified entries derived from the call
designer_signoff: <handle, ISO-8601>   # designer confirms the extracted entries
```

Stardust transcribes, extracts candidate `quick` or `qualified` entries
from the transcript, and surfaces them for designer signoff before they
land in the corpus.

### When to use which mode

- **Early corpus, small N.** Prefer `qualified` and `conversation`. Depth
  matters more than volume; 5 great qualified critiques outweigh 50 quick
  verdicts.
- **Mature corpus, large N.** Prefer `quick` and `comparative`. Volume
  and pairwise contrast carry the system once depth is established.
- **Always available.** `qualified` and `conversation` opt-in any time.
  `quick` and `comparative` are the default scaffolding of a critique
  session.

The mode of a critique is **never auto-promoted upward** (a `quick`
critique does not become `qualified` by curator-pass enrichment) and
**never auto-demoted downward** (an extracted `quick` entry from a
conversation does not silently inherit conversational caveats). Mode is a
property of how the judgment was rendered, not its conclusion.

Every ingester — URL crawler, Figma reader, vision-on-image, PDF parser,
video frame sampler — produces the same structured shape. This is the only
contract the rest of stardust depends on.

The shape is **specified in [`design-facts.md`](./design-facts.md)** (v1
locked). Summary: a descriptive block (layout_grammar, type_system, palette,
motion_personality, image_strategy, tone_signals) plus a structured index
(`observed_moves`, `brand_axes`) plus provenance.

Filling this shape from a URL is what `extract` already does today (in spirit).
Filling it from Figma, image, PDF, video is new ingestion plumbing — but the
output is identical.

This shape is the **only** dependency that future work (Figma-as-extract,
image-as-direction-reference) takes from this session.

---

## 4. Move contribution pipeline (designer-driven growth)

A fixed catalog just relocates the house-style problem. The catalog must grow
without becoming a curation tax. Four steps:

1. **Capture** — designer (~30 seconds). Sees something striking. Submits
   source (URL / Figma frame / screenshot / etc.) + one line: *"what is this
   doing that's different?"* Stored in `stardust/captures/`.
   - **Frictionless or it dies.** No taxonomy work at this step. No axis
     tag. No required fields beyond `source` and one sentence.
2. **Cluster** — curator pass. **Queue-driven cadence:** the pass triggers
   when any cluster reaches **≥2–3 captures of the same underlying idea**,
   not on a calendar. A lone capture waits. This guards against one
   designer's idiosyncratic taste becoming canon and keeps curation
   reactive to real signal density rather than the clock.
3. **Abstract** — curator + originating designer. Turn the cluster into a
   named move with the §1 schema. Originating designer signs off that the
   abstraction matches what they captured.
4. **Promote** — move enters `divergence-toolkit.md` with full provenance
   (who, which captures, which session). Now eligible for proposals.

Two side paths:

- **Direct authoring.** Senior designers can author a move without going
  through capture, when they can already articulate the abstraction. Rare,
  used for foundational entries.
- **Stardust-proposed candidates.** Stardust scans an extracted exemplar and
  proposes *candidate* moves it observes. Designers validate or reject —
  designers stay the gate.

---

## 5. Retirement and the house-style audit

Growth without pruning recreates the original problem at higher cost. The
audit has two scopes — distinct because critiques don't auto-flow upstream
in the hybrid model:

**Per-project audit** (runs at user runtime, lives in
`<user-project>/stardust/audits/`). Histogram of moves used across pages
of the current redesign. Detects within-redesign defaults that creep in
across pages of the same project.

**Plugin-wide audit** (runs at curator-pass time, lives in
`plugins/stardust/audits/`). Histogram of moves used across exemplars and
across whatever test redesigns + manually-promoted user submissions the
plugin maintainer team has accumulated. Detects cross-brand defaults —
moves the *catalog* over-relies on. Sample size depends on what
contributors and curators have promoted upstream, not on automatic user
telemetry.

For both:

- **Owned by a lead designer (plugin-wide) or the project owner
  (per-project), run manually.** Stardust generates the raw histogram;
  human interprets it. Automation would catch drift mechanically but miss
  qualitative slop, which is the failure the audit is meant to detect.
- Moves that dominate regardless of brand → demote (treat as default-combo
  warning in `divergence-toolkit.md`).
- Moves unused for N sessions → retire (move to archive, not deleted, with
  reason recorded).

---

## 6. Mining loop (how feedback evolves the system)

```
critique session  →  raw critiques  →  curator pass  →  divergence-toolkit.md
                                                     →  new exemplar entries
                                                     →  new anti-patterns
                                                     →  candidate moves queue
                                                     →  move-metadata updates
```

- A **critique session** is lightweight: designer reviews proposals in their
  chosen mode (per §2a). A `quick` session is ~15 minutes for 5 proposals;
  a `qualified` session is ~10 minutes per proposal; `conversation` is ~30
  minutes plus signoff.
- A **curator pass** (human, or stardust-with-review) extracts:
  - From `quick` entries — exemplar additions, anti-pattern tells,
    retirement candidates.
  - From `qualified` entries — same as `quick`, plus per-dimension
    annotations on existing exemplars and new candidate moves seeded by the
    `counterfactual` field.
  - From `comparative` entries — `when_to_use` / `when_not_to_use` updates
    on existing moves; never new exemplars.
  - From `conversation` entries — only the *signed-off* extracted entries
    enter the loop (they re-enter as `quick` or `qualified`); the recording
    itself does not.
- Each pass updates the toolkit and exemplar corpus with provenance (who,
  when, from which session, **what mode**).

---

## 7. Tradeoffs to flag for the designers

- **Frictionless capture is load-bearing.** Any field added to capture step 1
  costs us contributions. Resist the urge to ask for more upfront.
- **Curation discipline is the bottleneck, not capture volume.** A regular
  curator cadence matters more than a big capture queue.
- **Brevity is structural.** Verdict + 1–3 lines per critique. If a thought
  needs more, it should become its own exemplar, not a longer note.
- **Catalog growth needs a deletion discipline.** The histogram audit is the
  mechanism; without it, the catalog bloats and stops being useful.

---

## 8. Decisions log (resolved from designer review)

All v0 open questions, with the resolved answer and a one-line rationale.

1. **`brand_axes` — open tags, not a fixed enum.** Truer to how designers
   describe brands. Curator owns periodic de-duping.
2. **Verdict — keep four levels** (`stunning | strong | competent | slop`).
   Granularity is worth the slightly higher annotation cost; `competent` is
   where the failures hide and we don't want to lose it.
3. **Curator cadence — queue-driven.** Pass triggers when a cluster reaches
   ≥2–3 captures, not on a calendar. Reactive to signal density.
4. **Histogram audit — manual, owned by a lead designer.** Stardust
   generates the raw histogram; interpretation is human. Automation would
   miss qualitative slop, which is the failure the audit is for.
5. **Anti-patterns — one corpus.** Same schema, `verdict: slop` +
   `anti_pattern` tag. Lets stardust use them symmetrically with exemplars.
6. **Move-combination floor — ≥3 moves spanning ≥3 axes.** Confirmed.
7. **Capture step — source + one sentence, no axis tag.** Confirmed.
   Frictionless contribution outweighs cheaper clustering.
8. **Verdict vocabulary — `stunning | strong | competent | slop`.**
   Confirmed. "Slop" is loaded but precise.
9. **Actor model — hybrid (plugin-only by default + manual upstream
   promotion path).** Canonical corpus is contributor-curated and ships
   with the plugin. Per-project critiques and captures live project-side
   and stay local by default; users can promote into the plugin corpus by
   PR. Nothing flows upstream automatically. Aligns with stardust's
   existing project-relative path convention; no runtime path refactor
   required. Federated learning (telemetry, automatic flow) is a separate
   future session. (Resolved mid-session, not part of v0.)
10. **`anchor_read` field on `qualified` critiques — optional.** Designers
    fill it when relevant; the system does not require it. (Resolved
    mid-session.)

---

## What happens next

- ✅ **`design-facts` shape pinned** — see
  [`design-facts.md`](./design-facts.md) (v1).
- Add a `learning-system.md` in `reference/` describing **runtime behavior**
  (what stardust reads at `direct` and `prototype` time). This doc stays as
  the design rationale.
- Extend `divergence-toolkit.md` to formalize the move schema in §1.
- Land empty `stardust/exemplars/`, `stardust/captures/`, `stardust/critiques/`
  scaffolding with READMEs explaining contribution.
- Open a separate session for **Figma-as-extract-source**, dependent on the
  pinned `design-facts` shape.
