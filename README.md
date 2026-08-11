# @ahesse/biome-config

Shared [Biome](https://biomejs.dev) house config, used across every org's repos
(not just cloudflare). One source of truth for the formatter style and linter
baseline, so a change is made once here instead of copied into every repo (the
same idea as `infra/renovate-config`, but Biome resolves `extends` from
`node_modules`, so it ships as a dependency rather than a runtime `local>`
preset).

## Presets

Two entry points, picked by stack:

- **`@ahesse/biome-config/base`** is the universal style and linter baseline for
  any stack (plain TS, CLIs, cron jobs, Astro, Next, ...). It keeps Biome's
  recommended a11y rules on. Most repos use just this.
- **`@ahesse/biome-config/hono`** is a small **delta overlay** for Hono + Alpine
  Workers (the cloudflare org). It is NOT a standalone config: it only switches
  off a handful of a11y/style rules that false-positive on Hono's JSX and Alpine
  attributes, and must be layered **on top of** `base` (see below).

Biome does not resolve a nested `extends` when a config is pulled in through a
package export, so the overlay cannot extend `base` itself — composition happens
in the consuming repo by listing both presets.

## Consume it

Add it as a dev dependency (pnpm git dependency, pinned to a tag):

```jsonc
// package.json
{
  "devDependencies": {
    "@ahesse/biome-config": "github:andrew-hesse/biome-config#v1.5.0"
  }
}
```

Then reduce the repo's Biome config to the extend (plus any genuinely
unavoidable per-repo override below it). Non-Hono repos use `base` alone:

```jsonc
// biome.jsonc
{
  "$schema": "https://biomejs.dev/schemas/2.5.0/schema.json",
  "extends": ["@ahesse/biome-config/base"]
}
```

Hono/cloudflare repos layer the overlay on top — order matters (base first):

```jsonc
// biome.jsonc
{
  "$schema": "https://biomejs.dev/schemas/2.5.0/schema.json",
  "extends": ["@ahesse/biome-config/base", "@ahesse/biome-config/hono"]
}
```

### Reachability

The repo is public, so the dependency resolves anywhere with internet access:
developer machines, self-hosted runners, and cloud CI (github.com, gitlab.com)
alike. Nothing needs a token, and no consumer needs an inlined copy of the config.

Use the `github:` shorthand rather than a `git+https://` URL. pnpm resolves
`github:` specs to a **codeload tarball** over its own HTTPS client, so a Docker
build stage running `pnpm install` needs neither `git` on `PATH` nor a system CA
bundle. A `git+https://` spec makes pnpm shell out to `git` instead, which fails
`spawn git ENOENT` on a slim base and `server certificate verification failed
(CAfile: none)` without `ca-certificates`.

The lockfile records the resolved commit SHA, not the tag, so a consumer's install
stays reproducible even if a tag is moved.

### A repo with its own files.includes

Biome **concatenates** the `files.includes` arrays on extend (base's patterns
first, then the consumer's) and evaluates them in order, last match winning.
Two consequences:

- Base's `"**"` stays in effect, so a consumer's positive globs (e.g.
  `"src/**"`) do **not** narrow what is linted. To keep something out, add a
  negative pattern; do not rely on an allowlist of positive globs.
- A broad positive pattern listed after base's excludes re-includes what they
  excluded. `"**"` (or `"src/**"`) placed after base's `"!**/*.svg"` pulls SVGs
  back in, so a consumer that wants them kept out must repeat `"!**/*.svg"`
  after its own broad patterns (Biome 2.5 parses `.svg` and chokes on valid SVG
  markup).

```jsonc
{
  "extends": ["@ahesse/biome-config/base"],
  "files": { "includes": ["**", "!docs/vendor", "!**/*.svg"] }
}
```

Linter rules also merge (deep, last match wins), so a repo can add or override
individual rules without restating the baseline.

### Claude Code worktrees

The base preset excludes Claude Code's own committed configuration, including
direct `.claude/*.json` files and the `.claude/agents`, `.claude/commands`,
`.claude/hooks`, `.claude/rules`, and `.claude/skills` directories. It does not
exclude `.claude` itself.

This distinction keeps native worktrees under `.claude/worktrees/<name>`
lintable. A whole-directory `!**/.claude` exclusion also matches the worktree's
absolute path when `biome check .` runs inside it, causing Biome to process zero
files and fail the command.

Because those exclusions are now file-level rather than a whole directory, the
concatenation rule above applies to them: a consumer whose own `files.includes`
lists a broad positive like `"**/*.json"` re-includes `.claude/settings.json` and
starts failing `biome check .` on a file Claude Code owns and formats. Repeat the
negative after the consumer's positive globs:

```jsonc
{
  "extends": ["@ahesse/biome-config/base"],
  "files": { "includes": ["**/*.ts", "**/*.json", "!**/.claude/*.json"] }
}
```

### Comments belong in biome.jsonc, not biome.json

Biome parses `biome.json` as strict JSON. A `//` comment there is a parse error
and Biome falls back to its **default** configuration rather than stopping, so
the visible symptom is a large unexplained formatting diff (tabs, different line
widths) rather than a config complaint. The parse diagnostic is printed, but at
the top of the output, above the flood of format errors it causes. Rename the
file to `biome.jsonc` if a comment is worth having.

## What it sets

`base`: the Biome formatter (single quotes, no semicolons, trailing commas,
2-space indent, 100 columns), the recommended linter plus a stricter "high bar"
(`useImportType`/`useExportType`, `noUnusedImports`/`noUnusedVariables`,
`useNodejsImportProtocol`, and `noExplicitAny` all as errors), import organizing,
a test-file `noNonNullAssertion` override, and `files.includes` that excludes
build output, generated files, CSS (Tailwind owns it), and SVG. A repo that
legitimately needs `any` overrides `noExplicitAny` to `off` in its own config.

`hono`: a delta overlay (layered after `base`) that switches off
`useShorthandFunctionType`, `noSvgWithoutTitle`, `useSemanticElements`, and
`useFocusableInteractive` for Hono + Alpine. Nothing else - it relies on `base`
for the formatter and linter baseline.

## Release

Bump `version` in `package.json` and tag:

```bash
git tag v1.5.0 && git push origin v1.5.0
```

Consumers pick it up on their next Renovate bump or a manual dependency update.
