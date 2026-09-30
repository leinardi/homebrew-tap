# Contributing

The casks here are generated. A change to a cask, or a problem with one, belongs to the project it installs: its release
pipeline writes the cask and pushes it here, and the next release overwrites a hand edit. Issues are disabled in this repository
for that reason.

| Cask | Report to and change in |
| --- | --- |
| `monmux` | [leinardi/monmux](https://github.com/leinardi/monmux) (`homebrew_casks` in `.goreleaser.yaml`) |

Pull requests here are for the tooling: the workflows, the pre-commit configuration, the Makefile and the docs.

## Setup

Install [`pre-commit`](https://pre-commit.com/), then install the hooks once with `make pre-commit-install`. It installs both the
`pre-commit` and the `commit-msg` hooks. `make check` runs the full suite on every file, `make check-stage` on the staged files
only.

## Commit messages

All commits must follow [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) with a scope:
`<type>(<scope>)[!]: <description>`, e.g. `ci(audit): run brew style on every cask`. The `conventional-pre-commit` hook enforces
this on `commit-msg`, and the `conventional-commits` CI job checks every commit of a pull request. Pull requests are merged with
merge commits, so every commit lands on `main` as it is.
