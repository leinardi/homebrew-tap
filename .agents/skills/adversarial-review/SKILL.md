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

Assume the change is wrong until proven right: it blocks the next release, lets
a cask install something unverified, or drifts from the projects that generate
the casks. Find the concrete push, cask, or workflow run where it fails. Do not
praise or restyle the change. A review with no findings is credible only after
active attempts to break it.

This skill defines the review procedure and reporting. `AGENTS.md` remains the
source of truth.

Copy this checklist and tick items as you go:

```text
Review progress:
- [ ] 1. Diff and intent established (default scope if none given)
- [ ] 2. AGENTS.md and, for a cask change, the producing generator read
- [ ] 3. Repository invariants checked
- [ ] 4. Adversarial passes run
- [ ] 5. Findings confirmed or dropped; gates run
- [ ] 6. Report written
```

## 1. Establish the diff

Never review from memory or only from the user's description. Read the actual
diff and determine its intent. With no scope given, review the uncommitted
work; if the tree is clean, review the branch against `main`.

| User intent | Command |
| --- | --- |
| "my work", "before I commit", uncommitted changes | `git status --short`, then `git diff HEAD`; inspect untracked files too |
| staged changes only | `git diff --staged` |
| branch, "this PR", "ready to merge" | `git diff main...HEAD` |
| specific commit range | `git diff <base>..<head>` |
| GitHub PR number | `gh pr view <n>` for intent and metadata, then `gh pr diff <n>` |

Read `git log --oneline` for the reviewed range and any linked issue or PR
body. Code that works but does something other than the stated intent is a
finding.

Read every changed file in full, and the workflow jobs and docs that depend on
it.

## 2. Load project authority

Always read `AGENTS.md`. For a cask change, also read the generator in the
producing repository (for monmux: `homebrew_casks` in its `.goreleaser.yaml`)
and that project's release workflow: the fix for a generated file is there.

## 3. Repository invariants

- **Nothing blocks a release pipeline's push.** No branch ruleset, no required
  check, no workflow that fails a push to `main` in a way that matters: on a
  push, `cask-audit` reports with `continue-on-error`, and every other job is
  gated to pull requests. A change that makes a push-time job gate anything, or
  that needs the bot's commit to pass a hook, is a blocker.
- **Generated files are not edited here.** A diff under `Casks/` or `Formula/`
  from a human pull request is a finding unless it is a revert of a broken
  release, and even then the producing repository must be fixed in step.
- **Downloads are pinned and on-project.** Every `url` points at the producing
  project's GitHub releases over HTTPS and has a real `sha256`; `:no_check`,
  another host, or an unversioned URL is a finding.
- **Postflight does only what is documented.** The monmux cask's
  `postflight_steps` removes the quarantine attribute from the one binary it
  installs and nothing else; a broader path, another command, or
  `must_succeed: true` on `xattr -d` (which exits 1 when the attribute is
  absent) is a finding.
- **The bot's commit message stays conventional.** `chore(cask): <name> vX.Y.Z`
  from the generator's `commit_msg_template`.
- **Workflows stay pinned and least-privilege.** `contents: read` at the top,
  extra permissions per job with the reason; every action pinned to a full
  commit SHA with a `# vX.Y.Z` comment; inputs reach shell through `env:`.
  `cask-audit` must audit this checkout, never the published tap: the tap step
  asserts the tapped HEAD is the checkout's.

## 4. Adversarial passes

- **Next release:** replay the producing pipeline's push against the changed
  repository: does anything run, fail, or require a review on it?
- **Audit coverage:** does `cask-audit` still see every cask, and does an
  excluded cop or audit hide something beyond the documented goreleaser
  stanza order?
- **Install path:** for a cask change, walk `brew install --cask` on Apple
  Silicon and Intel (check which architectures the cask's `on_arm`/`on_intel`
  blocks cover), with and without the quarantine attribute, and
  `brew uninstall --zap`.
- **Contract drift:** compare the README table, AGENTS.md, SECURITY.md and the
  producing repositories' docs with the change.

For each candidate finding, reproduce it or trace the failing push, cask or
workflow run end to end. If that confirms it, report it. If not, dig once
more; if it is still unconfirmed, drop it.

## 5. Verify findings and gates

| Diff touched | Run |
| --- | --- |
| workflows, Makefile, pre-commit config | `make check` |
| a cask (revert only) | on a Mac: `brew tap leinardi/tap "$PWD"`, `brew style --except-cops Cask/StanzaOrder leinardi/tap`, `brew audit --tap leinardi/tap --strict --online` |
| docs or skill only | `make check` |

A failing gate is a confirmed finding when caused by the reviewed change. If a
gate cannot run (no Mac for the cask audit), say so and rely on CI's
`cask-audit`; never imply it passed.

## 6. Report

Rank findings by severity, worst first. Anything that can block a release
pipeline's push, or a cask that downloads unverified code, is normally a
blocker. Skip pure formatting unless it changes meaning or breaks a required
gate.

For each finding:

```text
<path>:<line> - <severity: blocker | high | medium | low>: <one-line defect>
  Failure: <concrete input/state/interleaving -> wrong result or broken invariant>
  Fix: <specific corrective change>
```

Put findings first. Then list open questions or assumptions, followed by a
one-line verdict: **block**, **approve with nits**, or **approve**. Include gates
actually run and gates not run. If no findings exist, say so explicitly and
briefly name the failure modes you tried to trigger. Be blunt, but never invent
a finding to appear thorough.
