# @ahesse/biome-config

Shared [Biome](https://biomejs.dev) house config for the cloudflare-org apps.
One source of truth, so a config change is made once here instead of copied into
every repo (the same idea as `infra/renovate-config`, but Biome resolves
`extends` from `node_modules`, so it ships as a dependency rather than a runtime
`local>` preset).

## Consume it

Add it as a dev dependency (pnpm git dependency, pinned to a tag):

```jsonc
// package.json
{
  "devDependencies": {
    "@ahesse/biome-config": "git+https://git.h.ahesse.dev/infra/biome-config#v1.0.0"
  }
}
```

Then reduce the repo's `biome.jsonc` to the extend (plus any genuinely
unavoidable per-repo override below it):

```jsonc
// biome.jsonc
{
  "$schema": "https://biomejs.dev/schemas/2.5.0/schema.json",
  "extends": ["@ahesse/biome-config/biome"]
}
```

`pnpm install` resolves the git dependency and pins its commit in the lockfile;
CI clones this public repo over https.

## What it sets

The previous copy-pasted house config: Biome formatter (single quotes, no
semicolons, trailing commas), the recommended linter with the org's rule
exceptions, import organizing, the test-file `noNonNullAssertion` override, and
`files.includes` that excludes build output, generated files, CSS (Tailwind owns
it) and **SVG** (Biome 2.5 parses `.svg` and chokes on valid SVG markup).

## Release

Bump `version` in `package.json` and move the matching tag:

```bash
git tag -f v1.0.0 && git push -f origin v1.0.0   # or cut a new vX.Y.Z
```

Consumers pick it up on their next Renovate bump or a manual dependency update.
