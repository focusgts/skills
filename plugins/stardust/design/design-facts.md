# `design-facts` — interface spec v1

> Status: **first complete spec.** Refine when implementation begins. This is
> the contract every ingester (URL crawler, Figma reader, vision-on-image,
> PDF parser, video frame sampler) produces, and every consumer (`direct`,
> `prototype`, the exemplar corpus, the moves catalog) reads.

## 1. Purpose

`design-facts` is a normalized description of a single design surface (a web
page, a Figma frame, a screenshot, a PDF page, a video frame range). It has
two simultaneous roles:

- **Descriptive.** A human-readable observation of the design's grammar:
  layout, type, palette, motion, image strategy, tone. Freeform fields.
- **Structured index.** Machine-readable handles into stardust's named
  vocabularies: `observed_moves` into the moves catalog, `brand_axes` into
  the open-tag set.

The descriptive part lets a designer read a `design-facts` block and
recognize the design. The structured part lets stardust filter, cluster, and
reason about it.

## 2. Producers and consumers

**Producers** (anything that fills the shape):

- The `extract` sub-skill, from a URL.
- Future Figma ingester, from a Figma file or frame.
- Future image ingester (vision), from a screenshot.
- Future PDF ingester, from a brand book.
- Future video ingester, from a recorded scroll-through.
- Designers, when authoring an exemplar by hand.

**Consumers** (anything that reads the shape):

- The `direct` sub-skill, to filter exemplars by `brand_axes` and surface
  anchors.
- The `prototype` sub-skill, to verify chosen moves against `observed_moves`
  on cited exemplars.
- The exemplar corpus, as the `source.facts` payload of every entry.
- The house-style audit, as the source of move histograms.

## 3. The shape

```
design-facts:
  schema_version: 1

  # === Descriptive (freeform observation) ===
  layout_grammar:
    primary:    string      # e.g. "asymmetric-grid", "centered-single-column"
    secondary:  string|null # sub-rhythm within sections, if observable
    rhythm:     string      # "dense" | "breathing" | "kinetic" | "varied"
    notes:      string|null # 1 line of nuance, optional

  type_system:
    display:    { family, weight, style, scale }
    body:       { family, weight, style, scale }
    pairing:    string      # "high-contrast" | "harmonic" | "mono-led" | "single-family"
    tone:       string      # "editorial" | "utility" | "poster" | "technical" | "expressive"

  palette:
    roles:      [ { name, value, role } ]   # name as observed, role: "background" | "surface" | "fg" | "accent" | "hierarchy"
    temperature: string     # "warm" | "cool" | "neutral" | "mixed"
    contrast:   string      # "high" | "medium" | "low" | "layered"
    chromaticity: string    # "saturated" | "muted" | "monochrome" | "duotone" | "polychrome"

  motion_personality:
    presence:   string      # "still" | "subtle" | "kinetic" | "noisy" | "unobservable"
    timing:     string|null # "slow" | "snappy" | "varied" | null if presence=still
    signatures: [ string ]  # e.g. ["scroll-driven", "hover-rich", "parallax"]; empty if none

  image_strategy:
    presence:   string      # "none" | "illustrative" | "photographic" | "abstract" | "mixed"
    treatment:  string|null # "edge-to-edge" | "framed" | "masked" | "layered" | null if presence=none
    role:       string|null # "supportive" | "decorative" | "narrative" | "structural"

  tone_signals:
    register:   string      # "declarative" | "poetic" | "dry-witty" | "technical" | "warm" | "mixed"
    density:    string      # "sparse" | "generous" | "dense"
    confidence: string      # "assertive" | "hedged" | "neutral"

  # === Structured index (vocabulary handles) ===
  observed_moves: [ move-id, ... ]   # ids from divergence-toolkit.md
  brand_axes:     [ tag, ... ]       # open tags, e.g. [tactile, niche, editorial]

  # === Provenance ===
  provenance:
    ingester:   string                  # "extract@v0.3.0" | "figma-ingester@v1" | "vision-ingester@v1" | "designer-authored"
    source:     { kind, ref }           # mirrors the exemplar source block
    captured_at: ISO-8601 timestamp
    captured_by: string|null            # designer handle if authored, null if automated
    confidence:  number                 # 0.0–1.0 self-assessed completeness
```

## 4. Per-field semantics

A few fields deserve more than schema text.

**`layout_grammar.primary`** is the load-bearing observation. It names *what
kind of structure* the design uses, before any decoration. A producer should
err toward concrete vocabulary ("asymmetric-grid", "horizontal-scroll",
"full-bleed-cinematic") rather than abstractions ("modern", "clean").

**`type_system.pairing`** captures the relationship between display and body
type — a major driver of perceived design grammar. `mono-led` means the
display uses a monospace; `single-family` means display and body are the
same family with weight/scale variation.

**`palette.roles`** is a list, not a fixed five-token map, because real
designs vary in palette complexity. Each role entry names the color (as
observed: hex, name, or token), and assigns it a structural role.

**`motion_personality.presence = unobservable`** is honest output for
ingesters that can't see motion (still images, PDFs). It is **not** the same
as `still` — `still` is an observed choice, `unobservable` is missing data.

**`observed_moves`** is the bridge to the moves catalog. Producers populate
this by matching the design against known moves (or by leaving it empty if
they can't). The `prototype` sub-skill reads this to verify that a cited
exemplar actually exhibits the moves a proposal is borrowing.

**`brand_axes`** are open tags (per learning-system v1 §8.1). Producers
contribute new tags freely; the curator pass periodically de-duplicates.

**`provenance.confidence`** is a self-assessment by the producer of how
complete this `design-facts` is. A vision ingester reading a small thumbnail
might emit 0.5; a careful designer-authored block might emit 0.9. Consumers
can filter on this.

## 5. Source-kind observability

Not every ingester can fill every field. This matrix names what each source
*can* observe vs. what it must mark `null` or `unobservable`.

| field                       | url | figma | image | pdf | video |
|-----------------------------|-----|-------|-------|-----|-------|
| layout_grammar              | yes | yes   | yes   | yes | yes   |
| type_system.family/weight   | yes | yes   | partial (vision) | partial | partial |
| type_system.pairing/tone    | yes | yes   | yes   | yes | yes   |
| palette                     | yes | yes   | yes   | yes | yes   |
| motion_personality          | yes | partial (prototype only) | unobservable | unobservable | yes |
| image_strategy              | yes | yes   | yes   | yes | yes   |
| tone_signals                | yes | yes   | partial | yes | partial |
| observed_moves              | yes | yes   | yes   | yes | yes   |
| brand_axes                  | yes | yes   | yes   | yes | yes   |

Producers MUST emit `unobservable` (not `null`) for fields the source
inherently cannot see. `null` is reserved for "observable in principle but
not present in this design" (e.g. `motion_personality.timing = null` when
`presence = still`).

## 6. Versioning

`schema_version: 1` is the v1 lock. Future revisions:

- **Additive changes** (new optional field) bump the minor: `1.1`, `1.2`.
  Consumers ignore unknown fields.
- **Breaking changes** (renamed/removed/required-field) bump major: `2`. A
  migration note documents how to upgrade existing entries.
- The schema lives at this doc. Producers MUST emit `schema_version`;
  consumers MUST refuse blocks they don't understand rather than silently
  misinterpret.

## 7. Conformance

A `design-facts` block is valid iff:

1. `schema_version` is present and recognized.
2. All top-level keys in §3 are present (descriptive, structured, provenance).
3. Required nested fields per §3 are present; optional ones may be `null`.
4. `unobservable` and `null` are used per §5.
5. `observed_moves` ids exist in the moves catalog at read time, OR are
   prefixed `candidate/` to mark not-yet-promoted observations.
6. `provenance.ingester` is a recognized producer name.

A consumer that reads an invalid block MUST either reject it or downgrade to
"unknown" rather than guess.

## 8. Examples

**(a) URL ingester output, abridged:**

```yaml
schema_version: 1
layout_grammar:
  primary: full-bleed-cinematic
  secondary: two-up-asymmetric
  rhythm: breathing
type_system:
  display: { family: "GT Sectra", weight: 700, style: italic, scale: poster }
  body:    { family: "Inter", weight: 400, style: normal, scale: editorial }
  pairing: high-contrast
  tone:    editorial
palette:
  roles:
    - { name: "#0B0B0B", value: "#0B0B0B", role: background }
    - { name: "off-white", value: "#F4EFE6", role: fg }
    - { name: "ember", value: "#D9421B", role: accent }
  temperature: warm
  contrast: high
  chromaticity: duotone
motion_personality:
  presence: subtle
  timing: slow
  signatures: [scroll-driven]
image_strategy:
  presence: photographic
  treatment: edge-to-edge
  role: narrative
tone_signals:
  register: poetic
  density: sparse
  confidence: assertive
observed_moves:
  - layout/full-bleed-cinematic
  - type/editorial-serif-display
  - image/photographic-edge
brand_axes: [editorial, niche, tactile]
provenance:
  ingester: extract@v0.3.0
  source:   { kind: url, ref: "https://example.com/about" }
  captured_at: 2026-04-22T10:14:00Z
  captured_by: null
  confidence: 0.85
```

**(b) Image ingester output (a screenshot), abridged:**

```yaml
schema_version: 1
layout_grammar:
  primary: asymmetric-grid
  secondary: null
  rhythm: dense
type_system:
  display: { family: "unknown-grotesk", weight: 800, style: normal, scale: poster }
  body:    { family: "unknown-sans", weight: 400, style: normal, scale: utility }
  pairing: high-contrast
  tone:    poster
# ...
motion_personality:
  presence: unobservable
  timing: null
  signatures: []
provenance:
  ingester: vision-ingester@v1
  source: { kind: image, ref: "captures/2026-04-22-poster.png" }
  confidence: 0.55
```

Note `motion_personality.presence: unobservable` — the screenshot can't see
motion. And `display.family: unknown-grotesk` — the vision model couldn't
identify the exact font; the producer picked a category placeholder.

## 9. What this spec deliberately does NOT do

- **It does not enumerate the moves catalog.** Moves live in
  `divergence-toolkit.md`. `design-facts` only references move ids.
- **It does not enumerate `brand_axes` values.** Open tags by design.
- **It does not specify storage format.** YAML is shown; JSON is fine. The
  shape is the contract, not the encoding.
- **It does not specify ingester behavior.** Each ingester gets its own spec
  (URL ingester is the existing `extract`; Figma/image/PDF/video are future
  sessions). They share only this output shape.
