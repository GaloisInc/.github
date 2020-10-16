# github-actions

A collection of github actions and other tooling designed to reduce repetition and make adding pipelines easier.

Not every bit of logic should be contained in its own action, and when possible, individual projects should ideally do a bit of legwork to make their workflows more standardized.
(e.g., being able to run tests without requiring a highly customized and involved setup)

## Layout

- `actions/`: contains actions intended to be usable by workflows
- `workflow-templates/`: contains reusable workflow templates that can be used by other repositories.
  I am guessing that the following workflow variations will cover the vast majority of common use cases:
  - `$lang.yaml`: Minimal workflow that does the simplest thing
  - `$lang-matrix.yaml`: Minimal workflow that does the simplest thing across all 3 OSes.
  - `$lang-release.yaml`: `$lang-matrix`, but includes logic for creating a release automatically and manually.
  - `$lang-docker.yaml`: `$lang-matrix`, but includes logic for building a docker image automatically and manually.
  - `$lang-complete.yaml`: `$lang-release` + `$lang-docker`
