# Security policy

This repository holds generated package definitions. A vulnerability in what a cask installs, or in the cask itself, belongs to
the project that produces it, and is reported through that project's private vulnerability reporting:

| Cask | Report to |
| --- | --- |
| `monmux` | [leinardi/monmux security policy](https://github.com/leinardi/monmux/security/policy) |

Report here, through GitHub's
[private vulnerability reporting](https://github.com/leinardi/homebrew-tap/security/advisories/new), only a problem with this
repository's own workflows. Never in a public pull request.

## Supported versions

Only the latest version of each cask. Homebrew installs the version in the tap's `main` branch, and an older one is not patched.

## What a cask may do

A cask here downloads only from its project's GitHub releases, over HTTPS, and pins the archive by `sha256`. The monmux cask also
removes the macOS quarantine attribute from the one binary it installs, because the binary carries no Apple Developer ID
signature; monmux's [security document](https://github.com/leinardi/monmux/blob/main/docs/security.md) explains what that
bypasses and how to install without it.
