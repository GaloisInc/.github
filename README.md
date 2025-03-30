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

Example:

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
