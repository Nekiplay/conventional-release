# conventional-release ![main branch workflow](https://github.com/mgoltzsche/conventional-release/actions/workflows/workflow.yaml/badge.svg?branch=main)

A GitHub Action to automate releases based on [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) using [git-sv](https://github.com/thegeeklab/git-sv).

## Features

* Supports fully automated releases driven by Conventional Commits.
* Allows to disable automated versioning in favour of manually pushing tags (`auto-release: false`) or of passing the version explicitly (`manual-version`).
* Supports version schemes with more than three components (e.g. `2.0.4.2.1`), which plain semver cannot express.
* Reports non-conventional commit messages without blocking the release (`validate-commit-messages: false`).
* Recovers from a failed release build automatically (when the next commit is pushed).
* Allows to use the same workflow/job definition for both pull request and release builds.
* Fails builds that leave uncommitted changes.

## Fork changes

This fork differs from `mgoltzsche/conventional-release` in the following ways.

* **`manual-version`** — releases an explicit version instead of deriving one
  from the commit log. This makes `workflow_dispatch` driven releases possible,
  since upstream only recognised a release when the workflow was triggered by a
  tag push.
* **Version formats** — `manual-version` accepts any number of dot-separated
  numeric segments with an optional prerelease. Upstream only accepted 1 to 3
  components.
* **`generate-changelog`** — a new script renders the release notes. Upstream
  used `git-sv release-notes`, which resolves the version through
  [Masterminds semver](https://github.com/Masterminds/semver) and therefore
  fails on versions such as `2.0.4.2.1`. The new script only needs the commit
  range, resolves it with plain `git`, and groups commits into **Breaking
  Changes / Features / Bug Fixes / Performance / Other Changes**. Commits that
  do not follow Conventional Commits are listed under Other Changes rather than
  dropped.
* **`validate-commit-messages`** — defaults to `false`. Upstream always failed
  the build on the first malformed commit message, which makes adoption
  impossible for a repository with pre-existing history. Set it to `true` to
  restore the strict behaviour.
* **`github-release-title`** — a release title independent of the tag name,
  supporting a `%s` placeholder for the version.
* Release notes are passed to `gh release create` via `--notes-file` rather than
  `--notes`, so multi-line markdown is not subject to shell quoting.

## Usage

To enable automated releases within your workflow, add a step to runs this Action after `actions/checkout` and before your actual build step(s):

```
    - id: release
      name: Prepare release
      uses: mgoltzsche/conventional-release@v2
```

For all supported Action inputs and outputs, see [`./action.yml`](./action.yml).

Please note that the releasing job needs to have write permissions for `contents` in order to push a git tag and create a GitHub release.
In case of pull request builds, the token still provides read-only access this way.
Correspondingly the `actions/checkout` Action should be configured with `persist-credentials: false`.

Please also note that the `actions/checkout` Action must be configured with `fetch-depth: 0` to work with the release Action.

To run subsequent steps conditionally depending on whether it is a release build, use the Action output `publish` as condition.
(In case of a release build, a git tag is pushed and a GitHub release created by the Action's post-entrypoint only after all steps within the job succeeded.)

Corresponding to its outputs, the Action exports the following environment variables to subsequent steps:

* `RELEASE_VERSION`: The semantic version (or manually pushed tag) of the release without leading `v`. During non-release builds this holds the next version with a `-dev-<SHA>` suffix.
* `RELEASE_PUBLISH`: Is `true` when release build, otherwise empty.

### Example workflow

A workflow that creates releases based on commits on the main branch automatically and that validates pull requests can look as follows:

```yaml
name: Build and release

on:
  push:
    branches:
    - main
  pull_request:
    branches:
    - main

concurrency: # Run release builds sequentially, cancel outdated PR builds
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

permissions: # Grant write access to github.token within non-pull_request builds
  contents: write

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest

    steps:
    - name: Check out code
      uses: actions/checkout@v7
      with:
        fetch-depth: 0
        persist-credentials: false

    - id: release
      name: Prepare release
      uses: mgoltzsche/conventional-release@v2

    # ... Build artifact ...

    - name: Publish artifact
      if: steps.release.outputs.publish # To run only when release build
      run: |
        set -u
        echo Publishing $RELEASE_VERSION
        ...
```

### Manually triggered release

Use `manual-version` to release a version that you choose, e.g. from a
`workflow_dispatch` input:

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version number (e.g. 2.0.4.2.1)"
        required: true

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
        persist-credentials: false

    - # ... build the artifacts to attach ...

    - name: Create release
      uses: Nekiplay/conventional-release@main
      with:
        manual-version: ${{ inputs.version }}
        github-release-draft: true
        github-release-title: "My Project ${{ inputs.version }}"
        github-release-files: |
          dist/my-app
          dist/my-app-setup.exe
```

The Action creates and pushes the tag and creates the GitHub Release in its
post step, so it must be the **last** step of the job. A tag that already exists
makes the Action fail rather than silently retag a published release.

### Changelog format

`generate-changelog` produces:

```markdown
Full changelog: [v2.0.4.1...v2.0.4.2.1](https://github.com/owner/repo/compare/v2.0.4.1...v2.0.4.2.1)

## Breaking Changes

- **cli**: drop the legacy --quick flag ([1234567](https://github.com/owner/repo/commit/...))

## Features

- add arm64 installer for Windows ([2345678](https://github.com/owner/repo/commit/...))

## Bug Fixes

- guard against an empty program list ([3456789](https://github.com/owner/repo/commit/...))

## Performance

- parallelize directory scanning ([4567890](https://github.com/owner/repo/commit/...))

## Other Changes

- bump deps ([5678901](https://github.com/owner/repo/commit/...))
```

`perf` gets its own section; every other type except `feat` and `fix` goes to
Other Changes. Commits carrying `[skip ci]` are omitted.

The [workflow used to publish this Action](./.github/workflows/workflow.yaml) is another example that shows how to release a container image, add a release commit and force-push a major version tag.

## Design considerations

See [design considerations](./DESIGN.md).
