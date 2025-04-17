# GitHub Workflows and Actions

A collection of GitHub Actions designed to reduce repetition across projects.

## Actions

### [`cabal-collect-bins`](./actions/cabal-collect-bins/action.yml)

Builds and collects the binaries for all provided cabal targets. Each target
must resolve to a single binary with `cabal list-bin` (so e.g. `all` is not
a valid target). Because we use `cabal list-bin`, this requires the use of
`cabal` 3.4.0.0 or later, although it is recommended to use `cabal-3.8.1.0`
or later to make sure that a fix for
[this `list-bin` issue](https://github.com/haskell/cabal/issues/7679)
is included.

Inputs:
- `targets`: Space or newline-delimited string of cabal targets.
- `dest`: Location to which the built binaries are copied. It is created if it
  does not already exist.

Outputs:
- `targets-json`: JSON array of the input targets, useful as an input into
  workflow job matrices for mapping target binaries to separate jobs.

Workflow Example:

```yml
name: CI
on: push
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      bins-json: ${{ steps.bins.outputs.targets-json }}
    steps:
      - uses: actions/checkout@v4

      - uses: haskell-actions/setup@v2

      - uses: GaloisInc/.github/actions/cabal-collect-bins@v1.1.2
        id: bins
        with:
          targets: |
            saw
            cryptol
          dest: dist/bin

      - uses: actions/upload-artifact@v4
        with:
          name: dist-bins
          path: dist

  run:
    runs-on: ubuntu-latest
    needs: build
    strategy:
      matrix:
        bin: ${{ fromJson(needs.build.outputs.bins-json) }}
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist-bins
          path: dist

      - run: |
          chmod +x dist/bin/*
          dist/bin/${{ matrix.bin }} --version
```

## Workflows

### [`haskell-ci`](./.github/workflows/haskell-ci.yml)

Build and test single-package Haskell projects using Cabal. Performs a sequence
of steps that are appropriate for many projects, namely:

- Uses `haskell-actions/setup` to install Cabal and GHC
- Runs `cabal sdist`, unpack the `sdist`, and perform subsequent steps there
- Uses `actions/cache` to cache the Cabal store and build directory (`dist-newstyle`)
- Runs `cabal configure`, `build`, `test`, `sdist`, `check`, and `haddock`
- Unconditionally saves the cache
- Checks that the package is compatible with GHC's bundled version of Cabal

This workflow will not be applicable to all projects, but it is applicable to
enough to be useful.

For a description of the inputs, see the workflow itself.

#### Comparison to other solutions

This workflow shares some goals with the [haskell-ci] tool. Here are a few
points of comparison:

- The haskell-ci tool generates a standalone workflow. This means that it will
  generally produce noisier diffs when updating, as it may require recreating
  large chunks of autogenerated YAML.
- The haskell-ci tool has a few extra features, such as integrations with
  `hlint` and `docspec`. We recommend running linters in a separate job, so
  that they can run in parallel with the build and test.
- This workflow can be updated by [Dependabot].
- This workflow provides extensive documentation and commentary, both user-
  facing and in the implementation.
- As a reusable workflow, this workflow is used in the `uses` part of a 
  job. So:
  
  - The calling workflow is free to customize other fields such as
    the workflow-level `on:` field.[^on]
  - Other jobs may run as part of the same calling workflow, and in particular,
    can set `inputs`.

[haskell-ci]: https://github.com/haskell-CI/haskell-ci
[Dependabot]: https://docs.github.com/en/code-security/getting-started/dependabot-quickstart-guide
[^on]: As an example of why such customization is important, it appears that the haskell-ci tool hardcodes `on:` to trigger on pushes and pull requests into the main branch. Not only may this not be appropriate for all projects (consider, for example, projects that use pull requests into a separate `dev` branch), it causes double-builds for pull requests targeting the main branch from a branch in the same repo.

#### Version support policy

This workflow aims to support:

- The latest two major/LTS versions of each macOS, Ubuntu, and Windows image
  supported by the hosted GitHub Actions runners
- As many GHC versions as is feasible without elaborate workarounds
- The latest two versions of Cabal

Support for a wide range of Cabal versions is not a priority because this
workflow contains an explicit step to check a package's compatibility against
the version of the Cabal library shipped with GHC. This ensures that the package
is compatible with older versions of Cabal without having to use them throughout
the whole workflow.

#### Example

```yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
jobs:
  ci:
    name: ${{ matrix.os }} GHC-${{ matrix.ghc }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-24.04]
        ghc: [9.8.4, 9.10.1, 9.12.1]
      fail-fast: false
    uses:
      haskell-ci:
        uses: GaloisInc/.github/workflows/haskell-ci@<SHA>
        inputs:
          ghc: ${{ matrix.ghc }}
          os: ${{ matrix.os }}
```

## Versioning policy

Each release has a version number of the form `MAJOR.MINOR.PATCH`, with
a corresponding Git tag prefixed by `v`. These versions follow [semantic
versioning].

[semantic versioning]: https://semver.org/

Example (for developers):
```bash
git tag --annotate vX.Y.Z --message vX.Y.Z
git push origin vX.Y.Z
```

We maintain tags of the form `vMAJOR` and may also maintain tags of the form
`vMAJOR.MINOR`. In accordance with the [official GitHub guidance], we move these
tags to the latest version tag of which they are a prefix.

[official GitHub guidance]: https://github.com/actions/toolkit/blob/main/docs/action-versioning.md#recommendations

Example (for developers):
```bash
git checkout vX.Y.Z
git tag --force --annotate vX --message vX
git push origin vX --force
```
