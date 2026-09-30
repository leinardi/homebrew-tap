# AGENTS.md

## What this is

The Homebrew tap `leinardi/tap`. It holds generated package definitions and the tooling around them, nothing else:
`Casks/monmux.rb` is written by [leinardi/monmux](https://github.com/leinardi/monmux)'s release workflow (goreleaser's
`homebrew_casks`) and pushed straight to `main` by `github-actions[bot]` on every release. Issues are disabled; problems with a
cask belong to the project it installs.

## Common commands

```bash
make check        # pre-commit on all files
make check-stage  # pre-commit on the staging area only
```

The Makefile pulls shared snippets from `leinardi/make-common@v1` into `.mk/` on first run. To refresh: `make mk-common-update`.
Project targets live in local `.mk/*.mk` files listed in `MK_LOCAL_FILES`, never as recipes in the Makefile.

`brew style` and `brew audit` run in CI (`cask-audit`, on macOS) against this checkout tapped as `leinardi/tap`. To run them
locally on a Mac: `brew tap leinardi/tap "$PWD"`, then `brew style --except-cops Cask/StanzaOrder leinardi/tap` and
`brew audit --tap leinardi/tap --strict --online`.

## Invariants

- **Never hand-edit a generated file.** `Casks/` (and a future `Formula/`) is written by the producing repository's release
  pipeline, and the next release overwrites any edit. Change the generator (e.g. `homebrew_casks` in monmux's
  `.goreleaser.yaml`) instead. The pre-commit fixers exclude these directories for the same reason.
- **Nothing may block a release pipeline's push.** The default branch has no ruleset on purpose (gh-leinardi-iac:
  `default_branch_ruleset_enabled = false`, "would fail every release"), local hooks never run on the bot's push, and CI on a
  push to `main` only reports: `cask-audit` is `continue-on-error` there. Do not add a required check, a branch ruleset or a
  push-time gate.
- **The release pipelines' commit message stays conventional and scoped.** goreleaser commits as `chore(cask): monmux vX.Y.Z`
  (`commit_msg_template` in monmux's `.goreleaser.yaml`). A generator change that drops the type or the scope makes the tap's
  history fail the rules below.
- **A cask downloads only from the producing project's GitHub releases, over HTTPS, pinned by `sha256`.** A cask with
  `sha256 :no_check`, a URL outside that project's releases, or a `postflight` that does more than documented is a finding.
  The monmux cask's `postflight_steps` removes the quarantine attribute from the one binary it installs; why, and how to
  install without it, is in monmux's `docs/security.md`.

## Project skills

Skills live in `.agents/skills/` (symlinked as `.claude/skills`). Load `adversarial-review` for any review request ("review my
diff", "is this ready to merge").

## Commit messages

All commits MUST be Conventional Commits 1.0.0 **with a scope**: `<type>(<scope>)[!]: <description>`. Enforced by the
`conventional-pre-commit` `commit-msg` hook (`--force-scope`, installed by `make pre-commit-install`) and by the
`conventional-commits` CI job on pull requests. Examples: `ci(audit): run brew style on every cask`,
`docs(readme): list the monmux cask`. The tap has no releases, so no version is derived from these types.
