---
name: adversarial-review
description: >
  Adversarial code review of changes to the leinardi/tap Homebrew tap:
  working tree, staged diff, branch, commit range, or PR. Hunts for anything
  that could block a release pipeline's push, hand edits to generated casks,
  unpinned or off-project downloads, postflight steps that do more than
  documented, and workflow permission or pinning drift, then reports ranked
  findings. Use when the user asks to review changes, a diff, PR, branch, or
  commit; check work before committing; or assess merge readiness.
---

# Adversarial Review - homebrew-tap

Assume the change is wrong until proven right: it blocks the next release, lets a cask install
something unverified, or drifts from the projects that generate the casks. Find the concrete push,
cask, or workflow run where it fails. Do not praise or restyle the change. A review with no findings
is credible only after active attempts to break it. Read `AGENTS.md` first: it remains the source of
truth. For a cask change, also read the generator in the producing repository (for monmux:
`homebrew_casks` in its `.goreleaser.yaml`) and that project's release workflow: the fix for a
generated file is there.

## Scope

When invoked directly: review the scope the user gives (`gh pr view <n>` for intent, then `gh pr
diff <n>`); with none, the uncommitted work (`git diff HEAD` plus untracked files); on a clean tree,
`git diff main...HEAD`. Never review from memory or only from the user's description. Read the
stated intent (`git log --oneline` for the range, any linked issue or PR body): code that works but
does something other than the stated intent is a finding. Read every changed file in full, and the
workflow jobs and docs that depend on it.

## Repository invariants

- **Nothing blocks a release pipeline's push.** No branch ruleset, no required check, no workflow
  that fails a push to `main` in a way that matters: on a push, `pre-commit-all` and `cask-audit`
  report with `continue-on-error`, `ci-result` turns their failure into a warning, and every other
  job is gated to pull requests. A change that makes a push-time job gate anything, or that needs
  the bot's commit to pass a hook, is a blocker.
- **Generated files are not edited here.** A diff under `Casks/` or `Formula/` from a human pull
  request is a finding unless it is a revert of a broken release, and even then the producing
  repository must be fixed in step.
- **Downloads are pinned and on-project.** Every `url` points at the producing project's GitHub
  releases over HTTPS and has a real `sha256`; `:no_check`, another host, or an unversioned URL is a
  finding.
- **Postflight does only what is documented.** The monmux cask's `postflight_steps` removes the
  quarantine attribute from the one binary it installs and nothing else; a broader path, another
  command, or `must_succeed: true` on `xattr -d` (which exits 1 when the attribute is absent) is a
  finding.
- **The bot's commit message stays conventional.** `chore(cask): <name> vX.Y.Z` from the generator's
  `commit_msg_template`.
- **Workflows stay pinned and least-privilege.** `contents: read` at the top, extra permissions per
  job with the reason; every action pinned to a full commit SHA with a `# vX.Y.Z` comment; inputs
  reach shell through `env:`. `cask-audit` must audit this checkout, never the published tap: the
  tap step asserts the tapped HEAD is the checkout's.

## Adversarial passes

- **Next release:** replay the producing pipeline's push against the changed repository: does
  anything run, fail, or require a review on it?
- **Audit coverage:** does `cask-audit` still see every cask, and does an excluded cop or audit hide
  something beyond the documented goreleaser stanza order?
- **Install path:** for a cask change, walk `brew install --cask` on Apple Silicon and Intel (check
  which architectures the cask's `on_arm`/`on_intel` blocks cover), with and without the quarantine
  attribute, and `brew uninstall --zap`.
- **Contract drift:** compare the README table, AGENTS.md, SECURITY.md and the producing
  repositories' docs with the change.
- **Commit type matches the change:** the tap has no releases, so no commit type derives a
  version here, and a cask changes only through its producing project's release. The type still has
  to describe the change: a `feat` or `fix` here claims a shipped change the tap cannot make, and a
  fix that users need belongs in the producing repository, typed there by that repository's own
  rules (monmux derives its version from them). A cask fix typed `chore` or `ci` in the producer
  never reaches the tap: report it against the producer. Every commit lands as it is (merge
  commits), so check each one, not just the pull request title.
- **A meta-check that can no longer fail:** `scripts/check-license-headers.sh`, the `ci-result`
  `jq` filter and the tap-equals-checkout assertion in `cask-audit` guard claims nothing else
  checks. Break the guarded thing once (a tracked `.rb` outside `Casks/` without the header, a
  `NEEDS` JSON with a failed `pre-commit-all` on a pull request, a tap left at another commit) and
  see whether the check still fails. One that passes because its file list is empty, its `case`
  pattern no longer matches anything, its exemption swallows a governed file, or its tolerance
  covers more events than a push is a false pass: a finding.

## Gates

Do not re-run the repository's gates (`make check`, the `brew tap`, `brew style` and `brew audit`
commands in `AGENTS.md`). The implementer is expected to run them and CI runs them again, but do not
assume either happened: report the verification evidence you actually have (under
adversarial-review-loop, `verification.txt` when supplied, which is the implementer's claim and not
proof; when invoked directly, the CI status or gate output in the PR or session) and what is
missing, such as a `ci-result` change with no `jq` replay per event, a meta-check change with no
mutation showing it still fails, or a cask revert with no Mac audit result (rely on CI's
`cask-audit`, never imply it passed). Judge what is missing against `AGENTS.md`'s *Validation
matrix*: every changed path owes the checks its row names. A failure in that evidence caused by the
reviewed change is a confirmed finding. Never say that a check you did not see passed. Judge whether
the change's tests prove it: a behaviour change needs a test that fails on the old code. When
invoked directly, one focused command (`pre-commit run <hook-id> --files <file>`, for a hook that
checks only the files it is given) may be run to confirm a single suspected finding, never a full
suite or gate. A candidate that is neither reproduced nor traced end to end is dropped.

## Report

Rank findings by severity, worst first. Anything that can block a release pipeline's push, or a cask
that downloads unverified code, is normally a blocker. Skip pure formatting unless it changes
meaning or breaks a required gate.

For each finding:

```text
<path>:<line> - <severity: blocker | high | medium | low>: <one-line defect>
  Failure: <concrete input/state/interleaving -> wrong result or broken invariant>
  Fix: <specific corrective change>
```

Put findings first. Then list open questions or assumptions, followed by a one-line verdict
(**block**, **approve with nits** or **approve**) and one line:
`Verification evidence: seen … / missing …`. If no findings exist, say so explicitly and briefly
name the failure modes you tried to trigger. Be blunt, but never invent a finding to appear
thorough.
