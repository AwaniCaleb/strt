# Contributing to Session Trust

Thank you for your interest. The project is pre-alpha; please read this first.

## Developer Certificate of Origin (DCO)

Contributions are accepted under the Developer Certificate of Origin
(https://developercertificate.org). By signing off a commit you certify that you
wrote the change, or otherwise have the right to submit it under the project's
licence (Apache-2.0).

Sign off every commit with `git commit -s`, which adds a line like:

    Signed-off-by: Your Name <you@example.com>

Pull requests with commits that are not signed off cannot be merged.

## Supported development platforms

- Linux
- macOS
- Windows via WSL2, with the repository inside the Linux filesystem

Native Windows checkouts are not supported.

## Tests and lint

All tests and lint must pass before a pull request is merged. The exact commands
arrive with the development tooling and will be documented here.

## Design rules

- Module import rules: `docs/blueprint.md` §10.4.
- Security invariants: `docs/blueprint.md` §4.1. Breaking one is a release blocker.

## Dependencies

The engine's runtime uses the Python standard library only. Pull requests that add
runtime dependencies to `engine/` will be declined.

## Commit messages

Maintainer commits carry internal task prefixes (`T<n>: ...`). Contributors don't
need them; a clear summary in Conventional Commit style
(e.g. `fix(render): escape ESC in terminal mode`) plus sign-off is enough.
