# GitHub Workflows and Actions

A collection of github actions and other tooling designed to reduce repetition
and make adding pipelines easier.

Not every bit of logic should be contained in its own action, and when possible,
individual projects should ideally do a bit of legwork to make their workflows
more standardized. (e.g., being able to run tests without requiring a highly
customized and involved setup)

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
      - uses: actions/checkout@v2

      - uses: haskell/actions/setup@v1

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

### [`cache-cabal-store`](./actions/cache-cabal-store/action.yml)

Cache Haskell dependencies in the Cabal store.

`cabal configure` should be run before this action.

This action rebuilds dependencies on cache misses using `cabal build all --only-dependencies`.

The cache key is calculated using:

- The runner OS and arch
- The GHC and Cabal versions
- The hash of the Cabal build plan

Inputs:

- `cabal-store`: Cabal store path
- `cabal-version`: Cabal version
- `ghc-version`: GHC version

Outputs: None.

Example:

```yml
name: CI
jobs:
  ci:
    name: ${{ matrix.os }} GHC-${{ matrix.ghc }} Cabal-${{ matrix.cabal }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-22.04]
        ghc: [9.10.1, 9.12.1]
        cabal: [3.12.1.0]
    steps:
      - uses: actions/checkout@v4
      - uses: haskell-actions/setup@v2
        id: setup
        with:
          cabal-version: ${{ matrix.cabal }}
          ghc-version: ${{ matrix.ghc }}
      # `cabal configure` should come before restoring the Cabal store cache so
      # that the build plan can form a part of the cache key.
      - name: Cabal update
        shell: bash
        run: cabal update
      - name: Configure
        shell: bash
        run: cabal configure --enable-tests
      - name: Build and cache dependencies
        uses: GaloisInc/.github/actions/cache-cabal-store@PUT-SHA-HERE
        with:
          cabal-store: ${{ steps.setup.outputs.cabal-store }}
          cabal-version: ${{ steps.setup.outputs.cabal-version }}
          ghc-version: ${{ steps.setup.outputs.ghc-version }}
      # Optional: Also cache dist-newstyle
      - name: Restore build cache
        id: build-cache
        uses: actions/cache/restore@v3
        with:
          path: dist-newstyle
          key: cabal-${{ runner.os }}-${{ runner.arch }}-ghc${{ matrix.ghc }}-${{ github.ref }}
          restore-keys: cabal-${{ runner.os }}-${{ runner.arch }}-ghc${{ matrix.ghc }}-
      - run: cabal build
        shell: bash
      # Optional: Also cache dist-newstyle
      - name: Save build cache
        uses: actions/cache/save@v3
        if: always()
        with:
          path: dist-newstyle
          key: ${{ steps.build-cache.outputs.cache-primary-key }}
```

## Layout

- `actions/`: contains actions intended to be usable by workflows
- `workflow-templates/`: contains reusable workflow templates that can be used by other repositories.
  I am guessing that the following workflow variations will cover the vast majority of common use cases:
  - `$lang.yaml`: Minimal workflow that does the simplest thing
  - `$lang-matrix.yaml`: Minimal workflow that does the simplest thing across all 3 OSes.
  - `$lang-release.yaml`: `$lang-matrix`, but includes logic for creating a release automatically and manually.
  - `$lang-docker.yaml`: `$lang-matrix`, but includes logic for building a docker image automatically and manually.
  - `$lang-complete.yaml`: `$lang-release` + `$lang-docker`
