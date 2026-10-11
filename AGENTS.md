# AGENTS.md

## What this is

The Homebrew tap `leinardi/tap`. It holds generated package definitions and the tooling around them,
nothing else: `Casks/monmux.rb` is written by
[leinardi/monmux](https://github.com/leinardi/monmux)'s release workflow (goreleaser's
`homebrew_casks`) and pushed straight to `main` by `github-actions[bot]` on every release. Issues
are disabled; problems with a cask belong to the project it installs.

## Common commands

```bash
make check                       # pre-commit on all files: the local gate, same as CI's pre-commit-all
make check-stage                 # pre-commit on the staging area only
make pre-commit-install          # install the pre-commit and commit-msg hooks
make mk-common-update            # refresh the shared make-common snippets
scripts/check-license-headers.sh # the license-header check alone (also a pre-commit hook)
```

The Makefile pulls shared snippets from `leinardi/make-common@v1` into `.mk/` on first run. To
refresh: `make mk-common-update`. Project targets live in local `.mk/*.mk` files listed in
`MK_LOCAL_FILES`, never as recipes in the Makefile.

`brew style` and `brew audit` run in CI (`cask-audit`, on macOS) against this checkout tapped as
`leinardi/tap`. To run them locally on a Mac (the excluded cops, and why, are in `ci.yaml`):

```bash
brew tap leinardi/tap "$PWD"
brew style --cask \
  --except-cops Cask/StanzaOrder,Cask/StanzaGrouping,Layout/FirstArrayElementIndentation \
  leinardi/tap/monmux
brew audit --tap leinardi/tap --strict --online
```

CI (`ci.yaml`): `pre-commit-all` runs every hook over every file and `cask-audit` audits every cask;
the reviewdog jobs only annotate pull requests, and `conventional-commits` checks a pull request's
commits. `ci-result` stands for `pre-commit-all`, `conventional-commits` and `cask-audit` in one
check name. Nothing requires it: the default branch has no ruleset (see Invariants).

## Generated or vendored: never hand-edit

- `Casks/` (and a future `Formula/`): written by the producing projects' release pipelines (see
  Invariants).
- `.mk/help.mk`, `.mk/pre-commit.mk` (the Makefile's `MK_COMMON_FILES`), `.mk/.mk-common-version`
  and `scripts/bootstrap-mk-common.sh`: fetched from `leinardi/make-common`; refresh them with
  `make mk-common-update`, and port a change upstream.

## Invariants

- **Never hand-edit a generated file.** `Casks/` (and a future `Formula/`) is written by the
  producing repository's release pipeline, and the next release overwrites any edit. Change the
  generator (e.g. `homebrew_casks` in monmux's `.goreleaser.yaml`) instead. The pre-commit fixers
  exclude these directories for the same reason.
- **Nothing may block a release pipeline's push.** The default branch has no ruleset on purpose
  (gh-leinardi-iac: `default_branch_ruleset_enabled = false`, "would fail every release"), local
  hooks never run on the bot's push, and CI on a push to `main` only reports: `pre-commit-all` and
  `cask-audit` are `continue-on-error` there, `ci-result` turns their failure into a warning, and
  every other job runs on pull requests only. Do not add a required check, a branch ruleset or a
  push-time gate.
- **The release pipelines' commit message stays conventional and scoped.** goreleaser commits as
  `chore(cask): monmux vX.Y.Z` (`commit_msg_template` in monmux's `.goreleaser.yaml`). A generator
  change that drops the type or the scope makes the tap's history fail the rules below.
- **A cask downloads only from the producing project's GitHub releases, over HTTPS, pinned by
  `sha256`.** A cask with `sha256 :no_check`, a URL outside that project's releases, or a
  `postflight` that does more than documented is a finding. The monmux cask's `postflight_steps`
  removes the quarantine attribute from the one binary it installs; why, and how to install without
  it, is in monmux's `docs/security.md`.

## Validation matrix

Run before considering an edit done. `make check` applies to every row.

| You edited | Run |
| --- | --- |
| `.github/workflows/**` | `pre-commit run actionlint --all-files`; a `ci-result` change: replay its `jq` with a `NEEDS` JSON for a pull request, a push and a dispatch |
| `.pre-commit-config.yaml` | `make check` (it is the change's own test) |
| `Makefile` | `make help`, `make check` (checkmake) |
| `scripts/check-license-headers.sh` | `scripts/check-license-headers.sh`; then break it once (a tracked `.rb` outside `Casks/` without the header) and see it fail |
| `.mk/*.mk`, `scripts/bootstrap-mk-common.sh` | never by hand: `make mk-common-update`, then `make help` |
| `Casks/**`, `Formula/**` | never by hand: change the producing repository's generator; CI's `cask-audit` (macOS) audits the result |
| `*.md`, `.agents/skills/**` | `make check`: markdownlint holds prose to 100 columns; tables, code blocks and headings are exempt |

## License headers

Every tracked source file the tap owns (shell, Ruby and Python files, repository-local `.mk` files
and the `Makefile`) carries the MIT header, dated with the year of the file's first commit;
`scripts/check-license-headers.sh`, an always-run pre-commit hook, fails on any that lacks it. Copy
the header from the script itself. Exempt: `Casks/` and `Formula/`, which the release pipelines
rewrite (a header would be dropped on every release), and the files fetched from make-common, which
carry make-common's own header. YAML, JSON and Markdown are never governed.

## Project skills

Skills live in `.agents/skills/` (symlinked as `.claude/skills`). Load `adversarial-review` for any
review request ("review my diff", "is this ready to merge").

## Commit messages

All commits MUST be Conventional Commits 1.0.0 **with a scope**:
`<type>(<scope>)[!]: <description>`. Enforced by the `conventional-pre-commit` `commit-msg` hook
(`--force-scope`, installed by `make pre-commit-install`) and by the `conventional-commits` CI job
on pull requests. Examples: `ci(audit): run brew style on every cask`,
`docs(readme): list the monmux cask`. The tap has no releases, so no version is derived from these
types: a cask's version comes from the producing project's release, and a cask changes only through
it. Pick the type that describes the change (`ci`, `build`, `docs` or `chore` for nearly everything
here); a `feat` or `fix` would claim a shipped change the tap cannot make. PRs land as merge
commits, so every commit lands on `main` as it is.
