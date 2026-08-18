# knockaway/github-actions

Shared GitHub Actions for Knock repositories. Each action lives in its own
subfolder and is consumed as a step, pinned to the moving major tag (`@v1`).

| Action | Purpose |
|---|---|
| [`publish-npm`](./publish-npm) | Build and publish a `@knockaway` npm package (stable on GitHub Release, beta on branch pushes). |

## `publish-npm`

Builds a package and publishes it to npmjs.org:

- **Stable** — when invoked from a **GitHub Release**, publishes the release
  tag's version under dist-tag **`latest`** (a leading `v` is stripped).
- **Beta** — otherwise (e.g. a push to a non-`main` branch), publishes
  `<package.json version>-beta.<short-sha>` under dist-tag **`beta`**.

It is safe to re-run: if the exact `name@version` already exists on npm, the
publish step is skipped instead of failing.

### Inputs

| input | required | default | description |
|---|---|---|---|
| `working-directory` | no | `.` | Directory containing the `package.json` to publish (e.g. `sdk`). |
| `node-version` | no | `''` | Explicit Node version. When empty, `node-version-file` is used. |
| `node-version-file` | no | `.nvmrc` | Repo-root file read for the Node version when `node-version` is empty. |
| `npm-token` | **yes** | — | npm automation token with publish rights to the package scope. |
| `dist-tag` | no | *(derived)* | Override the derived dist-tag. |
| `version` | no | *(derived)* | Override the derived version. |
| `access` | no | `public` | Value passed to `pnpm publish --access`. |

### Usage

For a package that lives in a subfolder (e.g. an SDK under `sdk/`):

```yaml
name: Publish SDK
on:
  release:
    types: [published]
  push:
    branches-ignore: [main, 'prod*']
    paths: ['sdk/**']
concurrency:
  group: publish-sdk-${{ github.ref }}
  cancel-in-progress: true
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # reserved for future npm provenance
    steps:
      - uses: actions/checkout@v4
      - uses: knockaway/github-actions/publish-npm@v1
        with:
          working-directory: sdk
          npm-token: ${{ secrets.NPM_TOKEN }}
```

For a package at the repo root, drop `working-directory` and the `paths:` filter.

### Conventions & prerequisites

- **A GitHub Release publishes the package.** On service repos whose published
  artifact is a subpackage (e.g. `temporal-worker`, `jupiter`), the release tag
  is the **SDK's** semver, not the service version.
- The org **`NPM_TOKEN`** secret must have repository access granted to each
  consuming repo, and be an **automation** token with publish rights to the
  package scope.

## Design

See [`docs/2026-08-18-npm-publish-github-actions-design.md`](./docs/2026-08-18-npm-publish-github-actions-design.md).

## Versioning

This repo is tagged `vX.Y.Z` with a moving `vX` major tag. All actions in the
repo share that tag line; consumers pin `@v1`.
