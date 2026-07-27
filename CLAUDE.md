# CLAUDE.md - `andrew-hesse/biome-config`

The shared Biome preset for every JS/TS repo, consumed as a tag-pinned `github:`
dependency (`@ahesse/biome-config`). Two entry points: `base.jsonc`
(the universal formatter + linter baseline) and `hono.jsonc` (a small delta
overlay for Hono/Alpine JSX). `README.md` is the full consumer guide; this file
covers editing the presets.

## Editing the presets

- `hono.jsonc` is a DELTA overlay layered AFTER `base` at the consumer
  (`extends: [base, hono]`); it must NOT `extends` base itself. Biome does not
  resolve a nested `extends` reached through a package export, so a self-extending
  overlay silently drops base's rules.
- Put stack-specific exceptions in `hono.jsonc` or the consuming repo, never in
  `base`. `base` is the universal high bar; widening it loosens every repo.
- `files.includes` arrays CONCATENATE on extend (base first, last match wins), so
  a consumer's positive glob does not narrow what base lints, and a broad positive
  glob re-includes what base excluded (repeat negatives like `!**/*.svg`).

## Releasing

Any rule or format change is a breaking surface for every consuming repo, so it
ships as a version bump + tag: bump `version` in `package.json`, `git tag vX.Y.Z`,
push the tag. Consumers pick it up via Renovate or a manual dep bump. See
`README.md` Release.

## Conventions

- No app code, build, tests, or CI here; the artifact is the JSONC config itself.
- Land changes via a `gh` PR; `main` is not branch-protected. Describe changes as
  a human dev (no AI attribution, no em-dashes).
