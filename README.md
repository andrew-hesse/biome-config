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
  recommended a11y rules on.
- **`@ahesse/biome-config/hono`** is `base` plus the linter exceptions that only
  make sense for Hono + Alpine Workers (the cloudflare org). Use it only for
  Hono repos; it switches off a handful of a11y/style rules that false-positive
  on Hono's JSX and Alpine attributes.

`@ahesse/biome-config/biome` is a deprecated alias of `/hono`, kept so the
cloudflare repos already pinned to it keep resolving. New repos should name the
preset they actually use.

## Consume it

Add it as a dev dependency (pnpm git dependency, pinned to a tag):

```jsonc
// package.json
{
  "devDependencies": {
    "@ahesse/biome-config": "git+https://git.h.ahesse.dev/infra/biome-config#v1.1.0"
  }
}
```

Then reduce the repo's Biome config to the extend (plus any genuinely
unavoidable per-repo override below it):

```jsonc
// biome.jsonc
{
  "$schema": "https://biomejs.dev/schemas/2.5.0/schema.json",
  "extends": ["@ahesse/biome-config/base"]
}
```

### Reachability

`git.h.ahesse.dev` is a private host (LAN/VPN only), so the git dependency
resolves only where that host is reachable: developer machines on the network
and the self-hosted Forgejo runner. Repos whose CI runs on a public cloud
(gitlab.com or github.com runners) cannot fetch it during `pnpm install`, so
those repos keep an inlined config instead of this dependency.

### A repo with its own files.includes

Biome **replaces** (does not merge) the `files.includes` array when a consumer
declares its own. A repo that needs custom includes therefore loses the base
excludes and must re-list the ones it still wants, in particular `"!**/*.svg"`
(Biome 2.5 parses `.svg` and chokes on valid SVG markup) and `"!**/*.css"`:

```jsonc
{
  "extends": ["@ahesse/biome-config/base"],
  "files": { "includes": ["src/**", "!**/generated", "!**/*.css", "!**/*.svg"] }
}
```

Linter rules, by contrast, do merge, so a repo can add or override individual
rules without restating the baseline.

## What it sets

`base`: the Biome formatter (single quotes, no semicolons, trailing commas,
2-space indent, 100 columns), the recommended linter with
`useImportType`/`useExportType` as errors, import organizing, a test-file
`noNonNullAssertion` override, and `files.includes` that excludes build output,
generated files, CSS (Tailwind owns it), and SVG.

`hono`: everything in `base`, plus `useShorthandFunctionType`,
`noSvgWithoutTitle`, `useSemanticElements`, and `useFocusableInteractive`
switched off for Hono + Alpine.

## Release

Bump `version` in `package.json` and tag:

```bash
git tag v1.1.0 && git push origin v1.1.0
```

Consumers pick it up on their next Renovate bump or a manual dependency update.
