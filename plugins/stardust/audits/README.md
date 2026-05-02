# `plugins/stardust/audits/`

**Plugin-wide histogram audits.** Output of the lead-designer-run
house-style audit per `plugins/stardust/design/learning-system.md` §5.

This directory holds **plugin-wide** audits — over the canonical
exemplar corpus and any test redesigns or manually-promoted user
submissions the maintainer team has accumulated. **Per-project audits**
(over a single redesign's pages) live project-side in
`<user-project>/stardust/audits/` and never land here.

## What goes here

One file per audit run, named `<ISO-8601-date>-<scope>.md` (or `.json`
for the raw histogram alongside a `.md` writeup). Suggested structure:

```
<date>-<scope>.json   # raw histogram
<date>-<scope>.md     # lead designer's interpretation and decisions
```

Histogram shape:

```json
{
  "audit_date": "<ISO-8601>",
  "scope": "plugin-wide",
  "sample": {
    "exemplars":  N,
    "test_redesigns": M,
    "user_submissions_promoted": K
  },
  "histogram": {
    "<move-id>": { "count": N, "across_brand_axes": [...] }
  },
  "default_combinations_observed": [
    { "combo_id": "...", "count": N }
  ]
}
```

The companion `.md` file records the lead designer's read: which moves
look like accidental defaults, which combinations recur across
unrelated `brand_axes`, which moves to demote or retire, which
candidates to promote.

## Why audits live in the repo

Audits are part of the system's evolution. Keeping them in-repo means
the demote/retire decisions are reviewable and the move history is
auditable when contributors return to the catalog months later. See
`plugins/stardust/design/learning-system.md` §5 for the full audit
process.

## What this directory does NOT contain

- **Per-project audits** — those live project-side in
  `<user-project>/stardust/audits/`.
- **The catalog itself** — `divergence-toolkit.md` §1a.
- **Exemplars or captures** — `../exemplars/`, `../captures/`.

## Status

Empty at landing. Populated when the lead designer runs the first
audit (typically once the corpus crosses ~20 exemplars).
