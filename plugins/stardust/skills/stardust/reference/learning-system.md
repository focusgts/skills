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

The actor model is **hybrid** (per `design/learning-system.md` Actors
section): canonical material is plugin-side; per-redesign material is
project-side. Stardust reads the union and writes only project-side at
runtime.

```
# Plugin-side (read-only at user runtime)
plugins/stardust/exemplars/    # canonical exemplar corpus
plugins/stardust/captures/     # contributor staging queue (PR-driven)
plugins/stardust/audits/       # plugin-wide histograms

# Project-side (written by stardust during a session)
<user-project>/stardust/exemplars/   # optional local additions (project-only)
<user-project>/stardust/captures/    # optional local captures
<user-project>/stardust/critiques/   # per-redesign critique entries
<user-project>/stardust/audits/      # per-project histogram
```

At read time, exemplars are the **union** of plugin-side and project-side.
Plugin-side is the trusted base; project-side is opt-in additions for the
current session. At write time, stardust writes only project-side.
Promotion of project-side material into the plugin corpus is a manual
contributor action (PR), never automatic.

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

One file per reviewed proposal (or per proposal set, for `comparative`),
schema per `design/learning-system.md` §2, with `verdict` + `why` mandatory
and `moves` populated from the proposal itself. Every critique entry MUST
declare `critique_mode: quick | qualified | comparative | conversation`.

Mode-specific required fields per `design/learning-system.md` §2a:

- `quick`: verdict + 1–3 line `why`. Default.
- `qualified`: adds `per_dimension`, `working_moves`, `failing_moves`,
  `counterfactual`, `anchor_read`, `designer_context`.
- `comparative`: applies to a *set*, not a proposal. Adds `subject`
  (proposal ids), `ranking`, `differentiating_moves`, `designer_context`.
  Stored at `stardust/critiques/<session>/comparative-<n>.yaml`.
- `conversation`: adds `recording`, `transcript`, `extracted_entries[]`,
  `designer_signoff`. The `extracted_entries` are written separately as
  their own `quick` or `qualified` files and listed by id here.

**Candidate moves** (when stardust observes an exemplar that uses a move
not yet in `divergence-toolkit.md`): emit the candidate to
`stardust/captures/candidates/<timestamp>-<id>.yaml` with `id` prefixed
`candidate/`. Do not promote. Do not silently extend the catalog. See
the next section for detection rules and emission protocol.

## Proposed candidates (stardust-observed)

Stardust may emit candidate moves at runtime when it observes a pattern
in an exemplar that is not yet in `divergence-toolkit.md` §1a. This
connects runtime observation to the contribution flow without bypassing
designer judgment — stardust *proposes*, designers (via the curator
pass per `design/curator-pass.md`) *promote*.

### When to emit (detection rules)

The emission gate is **conservative**. False positives pollute the
candidate queue and waste curator time, so detection is restrictive
by design:

1. **Source must be high-verdict.** Only emit when reading an exemplar
   with `verdict: stunning` or `verdict: strong`. Never emit from a
   `competent`, `slop`, or unverdicted source. The signal must be
   strong enough that the observation is worth capturing.
2. **Pattern must be structural.** Only emit on `layout`, `type`,
   `image`, `motion`, or `structural` axes — patterns where shape
   matters. Do **not** emit on `palette` or `tone` axes (palette is
   token-level; tone is content-level; both are too noisy at the
   move-id grain).
3. **Pattern must be unambiguous.** The observation must be specific
   enough to name in one short id. If the only honest description is
   *"something distinctive about how the type works"* without a more
   concrete pattern, do not emit.
4. **Pattern must not match an existing move.** Cross-check against
   §1a. If any existing move id covers the observation (even
   approximately), emit nothing — the existing id is the right
   reference.
5. **Hard cap: at most 2 candidate emissions per session.** If
   stardust would emit a third candidate in a single session, suppress
   the emission and surface a one-line note instead: *"third candidate
   suppressed — file a manual capture if this is meaningful."* The
   cap exists to prevent noisy sessions from flooding the queue.

### Emission protocol

When the gate passes, stardust:

1. **Names the candidate** deterministically. Form: `candidate/<axis>/<short-name>` where `<short-name>` is the first 3–5
   alphanumeric words describing the pattern, joined by `-`, capped
   at 30 chars. Example: `candidate/layout/full-bleed-with-marginalia`.
2. **Writes the candidate file** to
   `<user-project>/stardust/captures/candidates/<timestamp>-<short-name>.yaml`
   using the schema in `divergence-toolkit.md` §1a:

   ```yaml
   id:               candidate/<axis>/<short-name>
   axis:             <axis>
   summary:          <one line describing the observed pattern>
   when_to_use:      <stardust's best read; explicitly tentative>
   when_not_to_use:  <stardust's best read; explicitly tentative>
   exemplars:        [<exemplar-id>]              # the source
   provenance:
     added_by:       stardust-observed
     from_session:   <session id>
     captures:       []                            # no underlying captures
     observed_in:    [<exemplar-id>]
     confidence:     <0.0–1.0 — stardust's self-assessment>
   ```

   The `added_by: stardust-observed` value is mandatory and tells the
   curator pass this was machine-observed, not contributor-authored.
3. **Notes the candidate** in the design-facts block of the source
   exemplar by appending the candidate id to `observed_moves[]`.
4. **Surfaces a one-line note** to the user:
   *"Observed a pattern in <exemplar-id> not yet named in the catalog;
   saved as candidate/<axis>/<short-name>. Curator pass will validate
   or reject."*

### Curator pass treatment

The curator pass (per `design/curator-pass.md`) treats
`added_by: stardust-observed` candidates as **synthetic capture
clusters** with one observation. They follow the same path as
contributor-driven candidates but with two adjustments:

- They are **not promoted on a single observation.** A
  stardust-observed candidate must be **corroborated by at least one
  contributor capture** (or by a second stardust-observation in a
  different exemplar) before it becomes a candidate-move PR. This
  keeps designers as the gate.
- The originating-designer signoff falls to the **lead designer**
  rather than a single originating contributor (since there is no
  human originator). The lead designer signs off, or the candidate is
  deleted.

### What this path deliberately does NOT do

- **Auto-promote.** Even with multiple observations, candidates land
  via PR after curator review. The runtime never edits §1a.
- **Override designer authorship.** A contributor-authored capture
  for the same idea takes precedence over a stardust-observed
  candidate; the curator merges them with the contributor as the
  signoff owner.
- **Run on low-verdict sources.** Slop and competent entries don't
  emit candidates — they're not signal-rich enough to abstract from.
- **Emit silently.** Every emission produces a user-visible note. No
  background catalog growth.

## Runtime weighting by `critique_mode`

When selecting anchors at `direct` time and when reading the corpus
elsewhere, stardust weights entries by how the judgment was rendered:

- `qualified` > `quick` for **anchor influence**. A `qualified` critique's
  per-dimension assessment and counterfactual carry more useful signal than
  a one-line `why`, so `qualified` entries rank higher in the anchor sort
  when tag overlap is otherwise tied.
- `comparative` entries **never anchor**. They contribute only to move
  metadata at curator-pass time; runtime `direct` ignores them.
- `conversation` entries themselves never anchor. Only their *signed-off
  extracted entries* (which are stored as their own `quick` or `qualified`
  files) participate in anchor selection.
- `quick` entries are always eligible to anchor. They are the floor of the
  system, not a lesser form.

Mode is **not** an override of `verdict`. A `quick stunning` entry can
still anchor when no `qualified stunning` exists in the qualifying set.

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
- Never auto-promote a `quick` critique to `qualified` (or vice versa).
  Mode reflects how judgment was rendered; it is not a quality score the
  curator can adjust.
- Never let `comparative` entries enter the anchor pool. They update move
  metadata only.
- Never write a `conversation` entry's recording/transcript into the
  corpus directly. Only signed-off extracted entries enter; the source
  material stays adjacent for provenance.
