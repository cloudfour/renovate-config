# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) presets for Cloud Four repositories.

Centralizing here means a change we want everywhere is a one-file edit instead of a hand-edit across two dozen repos.

## Presets

### `default`

The baseline for every repo. It extends `config:recommended`, auto-merges patch and minor updates, pins exact versions, and waits three days after a release before opening a PR.

It also groups `eslint` with `@cloudfour/eslint-config`, and `stylelint` with our stylelint configs, because those packages pin each other's plugins and a split PR lands in a broken state. Both groups are no-ops in repos that don't use those packages.

One rule opts out of pinning: `engines` is left as a range. It describes which Node versions a project works on, so an exact version there contradicts the `.nvmrc` and CI `node-version` the project is tested on. Raising a supported-Node floor stays a deliberate edit. Node updates to `.nvmrc` and to workflow files come from different managers and are unaffected.

Those two Node updates arrive as a single PR. Renovate normally learns about a release from nodejs.org for `.nvmrc` and a day or two later from `actions/node-versions` for `setup-node`, so with the three-day wait they would come due on different days, as two PRs. Until both merge, local development and CI would run different Node versions. The preset reads both from nodejs.org so they come due together. A Node image in a job's `container:` is still checked against Docker Hub, since a tag can't be used until it's published there.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>cloudfour/renovate-config"]
}
```

### `library`

For repos that **publish to npm**. Everything in `default`, plus one rule that keeps `peerDependencies` as ranges rather than pinning them — a pinned peer range forces every downstream consumer onto one exact version.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>cloudfour/renovate-config:library"]
}
```

If a repo publishes to npm, use `library` even if it has no `peerDependencies` field today. The rule costs nothing while that field is absent, and it's the difference between a silent breaking release and a correct one on the day someone adds it.

## Versioning

Repos extend the default branch and pick changes up automatically — the presets are deliberately **not** pinned to a tag. Pinning would mean editing every consuming repo to bump the tag, which is the chore this repo exists to eliminate.

The trade-off is a wide blast radius on a bad commit, so the validator workflow runs on every PR and is a required check.

## Local overrides

A repo may still add its own rules on top of a preset. When it does, **every local rule must carry a `description`** explaining why it overrides or extends the shared config.

That description is the only remaining record of the rule's intent, since centralizing the config means the per-repo git history no longer tells that story.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>cloudfour/renovate-config"],
  "packageRules": [
    {
      "description": "Storybook majors have broken our visual regression suite twice. Review these by hand.",
      "matchPackageNames": ["storybook"],
      "automerge": false
    }
  ]
}
```

Rules from `extends` are concatenated *ahead of* a repo's local rules, and for any given package the last matching rule wins — so a local rule overrides the preset.

## Onboarding new repos

Renovate looks for `<org>/renovate-config` with a `default.json` when it onboards a repo, so new Cloud Four repos pick up `default` on their own. Publishing to npm is the one case that needs a manual edit, to point at `library` instead.

## Validating a change

```sh
npx --yes --package renovate -- renovate-config-validator --strict default.json library.json
```
