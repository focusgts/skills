# Phase 0 — Prerequisites

Goal: confirm the target EDS repository has the **overlay substrate**
in place. Runs once per repository. Subsequent invocations of the
skill see `.snowflake/config.json` and skip this phase silently.

## Why this phase exists

The skill's overlay pattern relies on substrate changes to the EDS
boilerplate (engine in `scripts/scripts.js`, lifecycle CSS, header/
footer block decorators, etc.). Without them, none of the later
phases work. Phase 0 detects whether the substrate is installed and
installs it if not.

## Check first

From the target repo's root:

```bash
cd "$(git rev-parse --show-toplevel)"
if [ -f .snowflake/config.json ]; then
  cat .snowflake/config.json
  # If "substrateVersion" matches the bundled VERSION → skip to Phase 1
fi
```

The bundled version is in `<SKILL_DIR>/assets/substrate/VERSION`. If the
installed version matches, **skip the rest of this phase** — substrate
is current.

## Install (or upgrade)

If `.snowflake/config.json` is absent or its `substrateVersion`
doesn't match the bundled VERSION, the dry-run run during initialization
(see SKILL.md "Initialization") will have reported one of four outcomes.
Act based on which case was found:

### Case A — Clean install (vanilla boilerplate)

The init summary already showed the file list and the user confirmed.
Run the installer now:

```bash
node <SKILL_DIR>/scripts/install-substrate.mjs
```

No additional confirmation needed — the init summary covered it.

### Case B — Custom code detected

The init summary flagged this. Surface the specific files and ask the
user to investigate before proceeding:

> The installer found non-empty content in files it would overwrite.
> This usually means you have a hand-rolled overlay engine. Check:
> `<list from dry-run output>`
>
> Options:
> 1. Let me know if it's safe to overwrite — I'll re-run with `--force`
>    (originals go to `.snowflake/.backup/<timestamp>/`).
> 2. Or leave these files as-is if your overlay engine already works.

Run with `--force` only after the user confirms:

```bash
node <SKILL_DIR>/scripts/install-substrate.mjs --force
```

### Case C — Drift (substrate present but diverged)

Substrate marker is present but files differ from the bundled version.
Surface the drifted files and the version mismatch:

> The installed substrate differs from the bundled v`<VERSION>`.
> Drifted files: `<list from dry-run output>`
>
> Options:
> 1. If the divergence is intentional (your substrate is ahead of the
>    bundled one, or you customized it), keep it as-is and skip Phase 0.
> 2. If the divergence is unintended, I can overwrite with `--force`
>    (originals backed up).

Run with `--force` only after the user confirms:

```bash
node <SKILL_DIR>/scripts/install-substrate.mjs --force
```

### After install

Confirm `.snowflake/config.json` was written with the bundled version
stamped in. The installer also writes the default config keys
(`projectsDir`, `daRoot`, `branchPrefix`, `trunkBranch`, `tagPrefix`)
on a fresh install — user-edited values from an existing config are
preserved.

## What gets installed

See `assets/substrate/MANIFEST.json` for the authoritative list. Summary:

| File | What changes |
|---|---|
| `scripts/scripts.js` | New overlay engine: `applyTemplateOverlay`, `writeSlot` (5 cases), template-name resolution, eager/lazy/delayed branches |
| `scripts/delayed.js` | HEAD-probes per-template animation engine before loading CDN deps |
| `styles/styles.css` | Lifecycle visibility CSS with direct-child selectors |
| `blocks/header/header.js` | Fetches static fragment instead of parsing DA-shape markup |
| `blocks/header/header.css` | Emptied (boilerplate's rules leaked into our fragments) |
| `blocks/footer/footer.js` | Same as header for footer |
| `blocks/footer/footer.css` | Emptied |
| `head.html` | Minimal head — no per-template stylesheet (engine loads dynamically) |
| `.eslintignore` | Patterns added (idempotent merge) |
| `.stylelintignore` | Patterns added (idempotent merge) |
| `.gitignore` | Patterns added (in-progress run state excluded) |

## After install

Phase 0 completes. The `.snowflake/` directory is now seeded:

```
.snowflake/
├── config.json                ← substrateVersion stamped
└── .backup/<timestamp>/       ← originals of files we replaced
```

Continue to Phase 1 (Capture).

## What if substrate is more advanced than bundled

If the user's repo has a NEWER substrate version than the bundled one
(possible if the user is on a snowflake skill release that's behind
their own substrate work), the installer refuses to downgrade. The
user should either update the skill (re-clone the latest) or stay on
the current substrate and skip this phase manually.

## What if the user does NOT want to install substrate

Then this skill cannot help — every later phase assumes the overlay
engine is present. The user has two options:
1. Install substrate, run the conversion.
2. Use a different skill (e.g., `migrate-page` in the same repo) which
   takes a different approach (rewrite into EDS-shape markup instead
   of overlaying).
