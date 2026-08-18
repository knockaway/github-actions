# Automated npm SDK publishing via a shared GitHub Actions composite action

**Date:** 2026-08-18
**Status:** Draft for review
**Author:** rsteele (with Claude)
**First consumers:** `temporal-worker` (`@knockaway/workflow-client`), `jupiter` (`@knockaway/jupiter-sdk`)

## 1. Problem

Several `@knockaway/*` npm packages are published **manually**: bump the version, build, `pnpm publish`,
then repeat for beta iterations. This is slow and error-prone, and the cycle time dominates when
iterating on a package and its consumers. We want branch-driven, automated publishing so a release is
a single deliberate action and betas fall out of normal branch pushes.

The two lead packages are **subpackage SDKs**, not repo-root packages:

| Repo | Package (published) | Path | Current version |
|---|---|---|---|
| `temporal-worker` (service, root `temporal-workers` v1.1.0) | `@knockaway/workflow-client` | `sdk/` | 0.8.0 |
| `jupiter` (service, root `jupiter` v1.0.0, private) | `@knockaway/jupiter-sdk` | `sdk/` | 1.9.0 |

## 2. Goals / non-goals

**Goals**
- Automate stable and beta npm publishing for `@knockaway/*` packages via GitHub Actions.
- Stable releases triggered by **GitHub Releases** (matches the org's existing `ts-utils` convention).
- Beta prereleases published automatically on **any push to a non-main branch** that touches the package.
- One **shared, reusable** implementation so all our publishable packages adopt it with a thin per-repo file.
- Support packages that live in a **subdirectory** (`sdk/`), not just at repo root.
- Zero changes to the existing CircleCI service build/test/deploy pipeline.

**Non-goals**
- No migration of service deploy off CircleCI.
- No changesets/semantic-release/conventional-commit automation in this iteration (version comes from the
  GitHub Release tag for stable; derived for beta).
- No monorepo/workspace restructuring of the SDKs.

## 3. Current state (context)

- Both repos: **CircleCI** owns service build → Docker image → Terraform deploy (dev on `main`, prod on
  `prod*`). **GitHub Actions** currently runs only promotion PRs (`main → prod`).
- SDK publishing has **no automation** today.
- Org already has a publish precedent: `ts-utils/.github/workflows/publish-package-on-release.yaml`
  (`release: published` → `pnpm publish` with `NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}`). It assumes the
  package is at repo root and has no beta channel.
- `secrets.NPM_TOKEN` is an **org-level secret** (referenced by many repos' workflows).
- Shared CI logic today is packaged as **one-repo-per-action** composite/JS actions
  (`knockaway/gh-action-promotion-pr-params@v3`, `knockaway/gh-action-upsert-pr@v3`), consumed as steps.

## 4. Decisions (resolved during brainstorming)

1. **Stable trigger:** GitHub Release (`release: published`) → npm dist-tag `latest`.
2. **Beta trigger:** push to any non-main branch touching the package → npm dist-tag `beta`.
3. **Beta version scheme:** `<base-version>-beta.<short-sha>` (e.g. `1.9.0-beta.a1b2c3d`).
4. **Reuse form:** a **single shared repo** `knockaway/github-actions` holding a **composite action**
   in a subfolder, consumed as a step: `uses: knockaway/github-actions/publish-npm@v1`.
   New shared actions become new subfolders. Single moving `vN` tag governs the whole repo.
5. **Node version:** action defaults to each repo's root `.nvmrc` (temporal=22, jupiter=v24); overridable
   via input.

## 5. Architecture

```
knockaway/github-actions            (NEW, single shared CI repo)
└── publish-npm/
    └── action.yml                  composite action: install → build → version → publish
                                    → uses: knockaway/github-actions/publish-npm@v1

temporal-worker/.github/workflows/publish-sdk.yml   thin caller (triggers + one step)
jupiter/.github/workflows/publish-sdk.yml            thin caller (triggers + one step)
```

Separation of concerns:
- **Triggers** (`release`, `push`) live in each consuming repo — GitHub cannot centralize `on:` in any
  shared form, so this is unavoidable and intentionally small.
- **All publish logic** (setup, build, version derivation, dist-tag selection, publish) lives once in the
  composite action.

### 5.1 Composite action — `publish-npm/action.yml`

**Inputs**

| input | required | default | purpose |
|---|---|---|---|
| `working-directory` | no | `.` | directory of the package to publish (`sdk` for the lead repos) |
| `node-version` | no | `''` | explicit Node version; when empty the action uses `node-version-file` |
| `node-version-file` | no | `.nvmrc` | repo-root file to read Node version from when `node-version` is empty |
| `npm-token` | **yes** | — | npm automation token, wired to `NODE_AUTH_TOKEN` |
| `dist-tag` | no | *(derived)* | override the computed dist-tag |
| `version` | no | *(derived)* | override the computed version |
| `access` | no | `public` | `pnpm publish --access` value |

**Steps (`runs: using: composite`)**
1. `git config user.name/email` (from `github.actor`) — keeps `pnpm version` happy.
2. `pnpm/action-setup@v4`.
3. `actions/setup-node@v4` with `registry-url: https://registry.npmjs.org` and either `node-version` or
   `node-version-file`.
4. `pnpm install` in `working-directory` (no `--frozen-lockfile`: the subpackage dirs have no own lockfile).
5. `pnpm run build` in `working-directory` — **explicit** build. (Note: `temporal-worker/sdk` uses a
   `prepublish` hook, which modern npm/pnpm no longer runs on publish; an explicit build fixes that.
   Guarded to no-op if no `build` script exists.)
6. **Derive version + dist-tag** (unless overridden by inputs), from `github` context:
   - `github.event_name == 'release'` → `version = tag_name` with any leading `v` stripped; `dist-tag = latest`.
   - otherwise (push to non-main) → `base = <package.json version>`; `version = ${base}-beta.${short_sha}`;
     `dist-tag = beta`. `short_sha = ${GITHUB_SHA::7}`.
7. `pnpm version --no-git-tag-version "$version"` in `working-directory`.
8. `pnpm publish --no-git-checks --access "$access" --tag "$dist-tag"` in `working-directory`,
   with `env: NODE_AUTH_TOKEN: ${{ inputs.npm-token }}`.

**Idempotency:** before publishing, the action checks `npm view "$pkg@$version"` and **skips** (success, with a
log line) if that exact version already exists — so re-running a job on the same sha won't hard-fail.

### 5.2 Per-repo caller — `.github/workflows/publish-sdk.yml`

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

The two repos are identical except the (implicit) package under `sdk/`.

## 6. Conventions this establishes

- **A GitHub Release on a service repo publishes its SDK.** These repos don't cut GitHub Releases today
  (deploys go through CircleCI on `prod*`), so Releases are free to adopt for this. The **release tag is the
  SDK semver** (e.g. tag `0.9.0` or `v0.9.0` → `@knockaway/workflow-client@0.9.0` on `latest`).
- **`paths: ['sdk/**']`** ensures betas only fire when the SDK actually changes — service-only commits on a
  branch do not publish to npm.
- **Every SDK-touching push to a non-main branch** publishes exactly one `beta.<sha>` version under the
  `beta` dist-tag. Consumers opt in with `pnpm add @knockaway/jupiter-sdk@beta` or a pinned `-beta.<sha>`.

## 7. Secrets & permissions

- **`NPM_TOKEN`** (org secret) must have **repository access granted** to `temporal-worker` and `jupiter`
  (they currently only use `GITHUB_TOKEN`). Verify/extend org secret visibility before first run.
- Token must be an npm **automation** token with publish rights to the `@knockaway` scope.
- Workflow `permissions`: `contents: read`; `id-token: write` reserved for a later npm-provenance upgrade.

## 8. Versioning of the shared repo

- `knockaway/github-actions` is tagged `v1.0.0`, `v1.1.0`, … with a **moving `v1`** major tag (matching the
  `gh-action-*` convention). Consumers pin `@v1`.
- All subfolder actions in the repo share that single tag line — acceptable for internal CI, and the reason
  we chose one repo over repo-per-action.

## 9. Edge cases

- **Duplicate version** (job re-run on same sha, or Release re-published): handled by the `npm view` skip.
- **Concurrent beta pushes on one branch:** `concurrency` cancels superseded in-flight runs.
- **Two branches, same base version:** both produce distinct `-beta.<sha>` versions; both land on the `beta`
  tag, last-writer-wins for the tag pointer. Acceptable; documented. (A future per-branch dist-tag option is
  possible but out of scope.)
- **Scoped first publish / access:** `--access public` is passed explicitly; both packages already exist as
  public so this is a no-op safety net.
- **`.nvmrc` with `v` prefix** (jupiter `v24`): `actions/setup-node` accepts it.

## 10. Rollout plan

1. Create `knockaway/github-actions` with `publish-npm/action.yml`, `README.md`, tag `v1.0.0` + `v1`.
2. Grant org `NPM_TOKEN` access to `temporal-worker` and `jupiter`.
3. Add `publish-sdk.yml` to `temporal-worker`; validate with a beta push (branch) and a test GitHub Release.
4. Add `publish-sdk.yml` to `jupiter`; validate the same way.
5. Document the "Release = SDK publish" convention in each repo's README / `sdk/README.md`.
6. Roll the caller workflow out to the remaining publishable packages (root-package repos pass
   `working-directory: .`).

## 11. Verification

- **Beta:** push a branch changing `sdk/**` → Action runs → `npm view <pkg>@<version> dist-tags` shows a new
  `beta` pointing at `<base>-beta.<sha>`; `pnpm add <pkg>@beta` resolves it.
- **Stable:** publish a GitHub Release tagged `X.Y.Z` → Action runs → `<pkg>@X.Y.Z` on `latest`.
- **No-op safety:** re-run the same job → skip log, no error, npm unchanged.
- **Paths filter:** push a branch changing only service `src/**` → no publish run.

## 12. Open questions

- Confirm org `NPM_TOKEN` is an **automation** token with `@knockaway` publish rights (not a personal token).
- Should the SDK version in `sdk/package.json` be committed back after a stable Release, or is the Release tag
  the single source of truth (current design: tag is source of truth; package.json is only bumped in-CI at
  publish time, not committed)?
- Any packages where the beta channel is undesirable (stable-only)? The caller can simply omit the `push:`
  trigger for those.
