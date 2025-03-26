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

### [`cabal-ghc-compat`](./actions/cabal-ghc-compat/action.yml)

Check that the Cabal configuration file is compatible with the version of the
Cabal library bundled with GHC.

Many Haskell projects support compilation with older GHC versions to help
users build the project even if they lack access to the latest GHC and Cabal
packages, such as when using versions provided by their OS package manager
on older systems. However, compatibility with older Cabal versions is often
overlooked. By default, commands like cabal build use the Cabal library version
bundled with the cabal-install tool to process configuration files (.cabal,
cabal.project{,.freeze}). If a project's CI configuration only tests recent
cabal-install versions, it may build with older GHC versions but only with
newer Cabal versions. This undermines compatibility, as users with older Cabal
versions will still be unable to build the project.

This action verifies that the project is buildable with the Cabal library
version [bundled with GHC][bundled]. These GHC/Cabal versions are typically
available in the same OS package sets. Using this action to check GHC/Cabal
compatibility helps prevent accidental failures to support older combinations
that users may rely on.

[bundled]: https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary/libraries/version-history

This action performs this check by using Nix to install the specified version
of GHC, creating a minimal `Setup.hs` file (which forces the project to be built
using GHC's version of Cabal), and running `cabal clean` (which is sufficient
to cause validation of the Cabal configuration files). Thus, it should be run
either entirely before any `cabal {build,install,test}` steps or entirely after
them. This strategy is more foolproof than explicitly including older Cabal
versions in, e.g., a `matrix`, as it specifically checks the version of Cabal
bundled with the specified version GHC.

Inputs:

- `dirs`: Directories in which to run
- `ghc`: GHC version, e.g., "9.10.1"
- `pkgs`: Extra packages to make available in the Nix shell
- `token`: Set to `secrets.GITHUB_TOKEN`

This action has no explicit outputs, but sets the `DO_IN_NIX_SHELL` environment
variable to `nix shell ${GHC_NIXPKGS}#cabal-install ${GHC_NIXPKGS}#${GHC}
${pkgs}` where `${GHC_NIXPKGS}` is a version of nixpkgs that contains the
specified version of GHC from the `ghc` input, and ${pkgs}` is the list of
packages from the `pkgs` input.

Example:

```yaml
name: CI
on: [push]
jobs:
  ci:
    name: ${{ matrix.os }} GHC-${{ matrix.ghc }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-24.04]
        ghc: [9.12.1, 9.10.1]
    steps:
      - uses: actions/checkout@v4
      - uses: haskell-actions/setup@v2
        id: setup-haskell
        with:
          ghc-version: ${{ matrix.ghc }}
      # (Any cache/build/test/install steps go here)
      - name: Check Cabal/GHC compatibility
        uses: GaloisInc/.github/actions/cabal-ghc-compat@SHA
        with:
          ghc: ${{ matrix.ghc }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

