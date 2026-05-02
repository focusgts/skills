# Curator pass — workflow

> Status: **first complete spec.** This is the procedure a curator follows
> when a cluster of captures reaches signal density. It is the human
> workflow that turns raw signal into catalog growth, complementing the
> runtime contract in
> `plugins/stardust/skills/stardust/reference/learning-system.md`.

The curator pass is the load-bearing process in the learning system. It
turns three input streams (captures, critiques, audit output) into
durable catalog changes (new moves, exemplar entries, retirements,
default-combo flags). Every step has a designated actor and produces a
specific artifact.

## When to run

**Queue-driven** (per `learning-system.md` §8.3). Trigger conditions:

- Any cluster in `plugins/stardust/captures/` reaches **≥2–3 captures**
  observing the same underlying idea.
- A critique submitted upstream is tagged for catalog implications by
  the curator (per critique mode rules below).
- A house-style audit has run and surfaced moves to demote or retire.

A pass is **not** triggered on a calendar. The queue's signal density
decides cadence.

## Actors

| role                  | responsibility                              |
|-----------------------|---------------------------------------------|
| **Curator**           | Runs the pass: clustering, drafting candidates, audit interpretation. Typically a lead designer or contributor. |
| **Originating designer** | The contributor who submitted the originating captures. Owns step 4 (signoff). |
| **Plugin maintainer** | Reviews and merges PRs from step 5. Enforces schema and provenance. May overlap with curator. |

A single person can hold multiple roles in the early life of the
system. The doc names roles for accountability, not for headcount.

## Procedure

### Step 1 — Gather inputs

The curator pulls from:

- `plugins/stardust/captures/` — raw captures awaiting clustering.
- `plugins/stardust/captures/candidates/` — already-clustered
  candidates awaiting promotion (carry-over from prior passes).
- Critique entries that contributors PR'd from project-side
  `<user-project>/stardust/critiques/` into the plugin repo (any
  `critique_mode`).
- Recent audit output in `plugins/stardust/audits/<date>-*.md` if a
  histogram run has happened since the last pass.

### Step 2 — Cluster captures

Read each capture's `source` and `sentence`. Group captures by the
underlying idea they describe — not by axis, not by visual similarity,
but by *"what this is doing that's different."*

A **cluster** is a group of ≥2 captures observing the same idea. Lone
captures (single observation, no second voice) **wait**. Do not promote
a one-capture cluster, ever — that's how an individual designer's
idiosyncratic taste becomes canon.

For each cluster, record:

- The captures' ids.
- A one-line synthesis of the underlying idea.
- The proposed `axis` (one of `layout`, `type`, `palette`, `image`,
  `motion`, `tone`, `structural`).

### Step 3 — Draft candidate moves

For each cluster, draft a candidate move per the schema in
`plugins/stardust/skills/stardust/reference/divergence-toolkit.md` §1a,
with `id: candidate/<axis>/<short-name>`. Fields:

- `summary` — one line abstracting what the cluster captures.
- `when_to_use` — short note grounded in brand axes.
- `when_not_to_use` — short note.
- `exemplars` — empty until the move is referenced by an exemplar.
- `provenance` — `{ added_by: <curator handle>, from_session:
  <pass-id>, captures: [capture-id, ...] }`.

Write to `plugins/stardust/captures/candidates/candidate-<axis>-<short-name>.yaml`.

### Step 4 — Originating designer signoff

For each candidate, contact the designer who submitted the originating
captures. Show them:

- The candidate file as drafted.
- The captures it cites.
- The proposed `summary`, `when_to_use`, and `when_not_to_use`.

The designer's response is one of:

- **Sign off.** Abstraction matches what they saw. Proceed to step 5.
- **Revise.** Abstraction misses or distorts. Curator updates the
  candidate file and re-shows. Re-loop until signoff or rejection.
- **Reject.** The cluster doesn't actually share what the curator
  thought. Candidate file is deleted; captures return to the queue
  for re-clustering in a future pass.

Signoff is recorded by adding `provenance.signed_off_by: <designer
handle>` and `provenance.signed_off_at: <ISO-8601>` to the candidate
file.

### Step 5 — Promote to the catalog

For each signed-off candidate, open a PR against
`plugins/stardust/skills/stardust/reference/divergence-toolkit.md`:

- Move the entry into §1a, **removing the `candidate/` prefix** from
  the `id`.
- Preserve provenance (originating designer, captures, session id).
- Bump the toolkit's status header version (patch increment) and add
  a one-line change-note entry.
- Delete the candidate file from `plugins/stardust/captures/candidates/`
  in the same PR.

The plugin maintainer reviews the PR, validates schema and provenance,
and merges. The candidate is now a catalog move.

### Step 6 — Process critiques

For each critique entry in scope, apply the per-mode rules from the
mining loop (`learning-system.md` §6):

- **`quick`** — may produce exemplar additions (PR new entries to
  `plugins/stardust/exemplars/`), anti-pattern tells (new entries with
  `verdict: slop` and an `anti_pattern` tag), or retirement candidates
  (moves the critique flagged as no-longer-working).
- **`qualified`** — same as `quick`, plus per-dimension annotations
  on existing exemplars (PR an `annotations[]` block to the cited
  exemplar), and new candidate moves seeded from the
  `counterfactual` field (return to step 3 with the counterfactual
  as a synthetic capture).
- **`comparative`** — update `when_to_use` / `when_not_to_use` on
  existing moves in §1a based on the differentiating moves cited.
  Never produces new exemplars.
- **`conversation`** — only the *signed-off extracted entries* enter
  the loop; they re-enter as `quick` or `qualified` and are processed
  as such. The recording/transcript itself is not a corpus entry.

### Step 7 — Audit-driven decisions

If a recent audit flagged moves to demote or retire, the curator
processes each:

- **Demote.** Add the over-used move (or its combination) to the
  default-combinations registry in §1a. PR the addition.
- **Retire.** Move the entry from §1a to a §1a-archive subsection
  (or commented-out block). Record the retirement reason in the
  archive note. **Do not delete** — the history matters for
  understanding why future moves were added or rejected.

### Step 8 — Document the pass

Write a short note at `plugins/stardust/audits/curator-pass-<date>.md`
summarizing:

- Which clusters were processed (ids, captures cited).
- Which candidates were promoted, revised, or rejected (and why).
- Which critiques were applied and how.
- Which moves were demoted or retired (and why).
- The PR links from steps 5–7.

This is the running history of the system's evolution. Future
contributors and curators consult it to understand prior decisions.

## Provenance summary

| artifact                              | provenance fields recorded                  |
|---------------------------------------|---------------------------------------------|
| Candidate move (`captures/candidates/`) | `added_by`, `from_session`, `captures[]`   |
| Promoted move (in §1a)                | `added_by`, `from_session`, `captures[]`, optional `signed_off_by` |
| Exemplar from critique                | `from_critique`, `from_session`             |
| Retirement                            | `retired_by`, `retired_at`, `reason`        |
| Demotion                              | `demoted_by`, `demoted_at`, `reason`, `audit_id` |

Provenance is required, not optional. The curator pass is the only
mechanism that adds catalog state, and every change must be traceable
to the human who made it and the signal that triggered it.

## Hard rules

- Never promote without originating-designer signoff.
- Never silently retire a move. The `archive/` block carries the
  reason.
- Never delete captures. They stay in the queue (or archive) for
  re-clustering and historical reference.
- Never invent abstractions the originating designer hasn't
  validated. The curator's job is to articulate what designers saw,
  not to layer in new opinions.
- Never bypass the PR-and-merge flow. Catalog changes go through code
  review, even when the curator is also a maintainer.
- Never run a pass against a single contributor's captures only. A
  cluster requires ≥2 contributors' observations or it waits.
  Otherwise the system encodes one designer's bias as canon.

## What the curator pass deliberately does NOT do

- **Author exemplars from scratch.** Exemplars come from critique
  signal or contributor authorship, not from the curator's
  imagination.
- **Run the audit.** Audits are a separate workflow, owned by a lead
  designer per `design/learning-system.md` §5. The curator pass
  *consumes* audit output but does not produce it.
- **Modify the runtime contract.** Changes to
  `reference/learning-system.md` are a separate maintainer
  responsibility, not part of any pass.
- **Run on telemetry.** No automatic flow from user runtime to
  curator queue. Promotion is always manual (a contributor opens a PR).

## References

- `plugins/stardust/design/learning-system.md` §4 — the contribution
  pipeline this workflow implements.
- `plugins/stardust/design/learning-system.md` §6 — the mining loop
  by critique mode.
- `plugins/stardust/skills/stardust/reference/divergence-toolkit.md`
  §1a — the move schema candidates are abstracted into.
- `plugins/stardust/skills/stardust/reference/learning-system.md` —
  runtime contract; the curator pass is the manual out-of-band action
  it references.
- `plugins/stardust/captures/README.md`,
  `plugins/stardust/captures/candidates/README.md` — the queues this
  workflow consumes.
