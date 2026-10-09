# Task Log

Format: T[n] | Title | Status | Date
Statuses: To do / In progress / Done / Blocked / Dropped. Numbers are never reused. Split tasks get new numbers with "split from T[n]".
(manual) = done by the maintainer; Claude Code only commits any file part. Blueprint roadmap item in [brackets].

## Setup
T0 | Project setup files (CLAUDE.md, task log, .gitignore) | Done | 2026-10-09

## Phase 0: Foundations
T1 | Repo files: LICENSE, NOTICE, README stub, SECURITY.md stub, CONTRIBUTING.md (DCO), CHANGELOG.md, .gitattributes; DL-1, DL-2 [0.1] | Done | 2026-10-09
T2 | (manual) GitHub settings: branch protection, passkey 2FA, private vulnerability reporting, mirror remote [0.1] | To do | —
T3 | (manual) SSH commit/tag signing + commit allowed_signers [0.2] | To do | —
T4 | Dev tooling: pyproject.toml (tool config), hashed requirements-dev.txt, Makefile, tests/conftest.py, golden helper [0.5] | To do | —
T5 | Engine skeleton: __main__, cli version, hook start/stop/end log-only stubs, io/paths.py (state roots, ID validation, containment) [0.5] | To do | —
T6 | io/fsutil.py: atomic writes, 0600/0700 enforcement, locks [0.5] | To do | —
T7 | Plugin skeleton: plugin.json, marketplace.json (next), hooks.json matchers, capture.sh, lifecycle.sh + hook-contract and atomicity tests [0.4] | To do | —
T8 | Platform spike A: hook payloads, failed calls, SessionEnd, output channels, paths/env → docs/platform-facts.md [0.3] | To do | —
T9 | Platform spike B: marketplace pinning, commands/env markers, managed installs, deny rules, local dev + claude -p, git alternates/filter overrides, macOS CLT Python [0.3] | To do | —
T10 | io/runner.py run_program + io/binaries.py (clean PATH, xcode-select guard) [0.5] | To do | —
T11 | io/gitops.py run_git hardening + rules/git_hardening.json + malicious-config canary tests [0.5] | To do | —
T12 | io/objstore.py private object store + index seeding + zero-footprint test [0.5] | To do | —
T13 | io/config.py stricter-only merge + freeze stub + tests [0.5] | To do | —
T14 | render/safe.py Untrusted/SafeText/untrusted() v0 + tests [0.5] | To do | —
T15 | Security tests: no-shell, stdlib-only, no-network, import rules + tools/check_size_budgets.py [0.6] | To do | —
T16 | CI: ci.yml (SHA-pinned, contents: read), test matrix, Windows-disable placeholder, gitleaks, dependabot [0.6] | To do | —
T17 | Capture redacted real payload fixtures → tests/fixtures/payloads/cc-<version>/ [0.7] | To do | —
T18 | Phase 0 acceptance check (real session capture, CI green, invariant tests) | To do | —

## Phase 1: MVP Core
T19 | SessionStart: platform detection + disable notice (io/platform.py) [1.1] | To do | —
T20 | SessionStart: binary cache, config load/merge/freeze, disabled_roots [1.1] | To do | —
T21 | SessionStart: manifest, baseline into private object store, resume segments [1.1] | To do | —
T22 | Event collection: inbox scan, ID validation, lock, max_inbox_mb, orphan sweep [1.2] | To do | —
T23 | Adapter cc_v1: shape validation, Pre/Post pairing, size caps, flags [1.2] | To do | —
T24 | Diff and reconciliation incl. ignored, outside-repo, NotebookEdit/MCP, submodule pointers [1.3] | To do | —
T25 | Scope engine: signal extraction + rules/scope.json + S2 extraction in io/ [1.4] | To do | —
T26 | Scope engine: scoring, tiers, reason codes [1.4] | To do | —
T27 | Risk engine: shlex parsing, wrapper recursion, rules/commands.json [1.5] | To do | —
T28 | Risk engine: outcome cross-check (attempted_no_completion), deny suggestions [1.5] | To do | —
T29 | Redaction v1, sensitive paths, HMAC hashing [1.6] | To do | —
T30 | Renderer: markdown/terminal modes, header builder, report writer [1.7] | To do | —
T31 | Renderer: summary line, template summary, "no agent action recorded" label [1.7] | To do | —
T32 | SessionEnd: 3 s sync budget + detached worker [1.8] | To do | —
T33 | SessionEnd: record, raw deletion, index.json, retention and orphan sweeps [1.8] | To do | —
T34 | CLI: report, verify, config show, version, exit codes, schemas report-v1/cli-v1 [1.9] | To do | —
T35 | CLI: doctor basic + --install launcher [1.9] | To do | —
T36 | /session-report command + optional Stop tally [1.10] | To do | —
T37 | README (DEC-27 positioning) + first draft docs/data-processing.md [1.11] | To do | —
T38 | Windows-disable CI final, benchmark job, 20k-file index-seeding benchmark [1.12] | To do | —
T39 | Concurrent-writer detection + shared_worktree [1.13] | To do | —
T40 | doctor --audit, docs/what-we-capture.md, first-run notice [1.14] | To do | —
T41 | Tester onboarding: next install guide, feedback issue template, bundle instructions [1.15] | To do | —
T42 | v0.1.0 acceptance: reference scenarios a–i, zero footprint, redaction, performance | To do | —
T43 | (manual) Signed v0.1.0-rc tag, next channel, 5 days dogfooding | To do | —
T44 | (manual) Recruit 3–5 outside testers on next | To do | —

## Phase 2: Production Hardening
T45 | Resource budgets, partial-first rendering, status: partial [2.1] | To do | —
T46 | Tamper detection: target matching, path canonicalisation [2.2] | To do | —
T47 | Tamper: manifest backstop, self_development, launcher integrity, publish/pty/gh rules [2.2] | To do | —
T48 | Lookalike-character detection + ignored-project-settings notes [2.3] | To do | —
T49 | Migration framework, state_version, backups, doctor --reset-state [2.4] | To do | —
T50 | Payload-shape warning UX [2.4] | To do | —
T51 | Labelling guide + public synthetic corpus (≥ 40 cases) [2.5] | To do | —
T52 | engine eval, baseline, CI accuracy gate with tier-change output [2.5] | To do | —
T53 | Private encrypted corpus + tune scope.json [2.5] | To do | —
T54 | hypothesis fuzzing: renderer, parser, redaction, config merge, header reader [2.6] | To do | —
T55 | doctor --bundle + GitHub issue forms [2.7] | To do | —
T56 | SECURITY.md final, Discussions, docs/known-issues.md [2.7] | To do | —
T57 | docs/guarantees.md (CI accuracy block) + report limits line [2.8] | To do | —
T58 | docs/data-processing.md final + docs/teams.md [2.8] | To do | —
T59 | purge + feedback commands [2.8] | To do | —
T60 | release.yml, stable/next entries, release + e2e checklists [2.9] | To do | —
T61 | weekly.yml + first manual e2e run under Pro [2.9] | To do | —
T62 | Tester-feedback triage [2.10] | To do | —
T63 | Hardening review against STRIDE tables [2.11] | To do | —
T64 | v1.0.0 acceptance + (manual) signed release on stable | To do | —

## Phase 3: Growth (demand-driven; order per blueprint §9 and DEC-27)
T65 | publish via gh (v1.1) [3.1] | To do | —
T66 | Resolve opaque yarn/npm/pnpm/composer/make scripts [3.2] | To do | —
T67 | analyze diff-only mode + GitHub Action [3.3] | To do | —
T68 | Experimental "inform Claude" mode [3.4] | To do | —
T69 | --deep advisory LLM pass [3.5] | To do | —
T70 | Native Windows support [3.6] | To do | —
T71 | Sigstore/gitsign provenance [3.7] | To do | —
T72 | Official plugin directory listing [3.8] | To do | —
T73 | Team service (separate plan and mentor session first) [3.9] | To do | —
