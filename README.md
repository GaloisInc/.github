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
`cabal` 3.4.0.0 or later.

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

      - uses: GaloisInc/.github/actions/cabal-collect-bins@v1.1
        id: bins
        with:
          targets: |
            saw
            cryptol
          dest: dist/bin

      - uses: actions/upload-artifact@v2
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
      - uses: actions/download-artifact@v2
        with:
          name: dist-bins
          path: dist

      - run: |
          chmod +x dist/bin/*
          dist/bin/${{ matrix.bin }} --version
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
