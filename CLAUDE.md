# CLAUDE.md — Session Trust

## What this is
Session Trust (plugin ID `session-trust`) is a Claude Code plugin that gives an honest account of what an AI coding session changed. Hooks record Claude's prompts, file edits and shell actions; git-level snapshots before and after the session are reconciled against them. Each changed file is tiered (in scope / likely related / unexplained) with a reason, and risky commands are flagged by severity.
It runs fully locally, makes no network calls, writes nothing into the user's repo by default, and never blocks or steers the session it observes.

- Design: `docs/blueprint.md`. The newest DL entry in its §13 Change Log overrides earlier text on the same topic.
- Tasks: `docs/task-log.md`.

## Stack
- Hooks: POSIX `sh`. `hooks/capture.sh` must run under `dash`; `hooks/lifecycle.sh` calls `python3 -m engine hook <event>`.
- Engine: Python, floor **3.9**, tested on 3.9 and 3.14. **Standard library only at runtime.**
- git: OS default and latest release, both tested.
- Dev-only tools (exact pins live in `requirements-dev.txt`, installed with `--require-hashes`): pytest 8.x, hypothesis 6.x, coverage 7.x, ruff 0.x, mypy 1.x, shellcheck ≥ 0.10, dash, gitleaks 8.x, Docker 27.x (optional).
- CI: GitHub Actions on ubuntu-24.04, macos-15, plus a windows-latest self-disable job.
- No Node, no yarn/npm in this project.

## Commands (Makefile and requirements-dev.txt exist from T4)
- Install: `python3 -m venv .venv && . .venv/bin/activate && pip install --require-hashes -r requirements-dev.txt`
- Run engine: `python3 -m engine version` · `python3 -m engine hook start|stop|end`
- Test: `make test` (= `pytest -q --cov=engine --cov-fail-under=90 -p no:cacheprovider`)
- Security tests: `python -m pytest tests/security -q`
- Lint: `make lint` (= `ruff check . && ruff format --check . && mypy --strict engine/ && shellcheck hooks/*.sh bin/session-trust`)
- Size budgets: `make budgets` (= `python tools/check_size_budgets.py`)
- Eval / benchmarks (Phase 1+): `make eval` · `make bench`
- Reproducible env: `make docker-test`
- Migrate / seed: none. There is no database. State migrations are engine code (Phase 2). Test fixtures are generated at test time by `tests/fixtures/repo_builder.py`; never commit binary repos.
- Load the plugin locally in Claude Code: see `docs/platform-facts.md` (verified in the Phase 0 spike).

## Folder map
- `.claude-plugin/`: `plugin.json`, `marketplace.json` (`next` / `stable` entries)
- `hooks/`: `hooks.json`, `capture.sh`, `lifecycle.sh`
- `commands/`: `/session-report` prompt file
- `bin/`: `session-trust` launcher (installed by `doctor --install`)
- `engine/`: `core/` (pure logic), `io/` (all side effects), `adapters/`, `render/`, `migrations/`, `rules/*.json`, `evaluation/`; later `publish/`, `analyze/`, `deep/`
- `schemas/`: JSON Schemas (draft 2020-12) for report, CLI, config, record
- `tests/`: `unit/`, `integration/`, `security/` (invariants), `fuzz/`, `golden/`, `fixtures/`, `windows/`
- `eval/`, `bench/`, `tools/`: accuracy corpus, benchmarks, size-budget check
- `docs/`: blueprint, task log, platform facts, guarantees, user docs
- `.github/`: workflows (`ci`, `release`, `weekly`), issue templates, dependabot

## Coding conventions
- Every Python module starts with `from __future__ import annotations`. No 3.10+ syntax or runtime APIs (no `match`, no runtime `X | Y` types, no `zip(strict=)`).
- `mypy --strict` on `engine/`. `Untrusted` and `SafeText` are separate `NewType`s; renderers accept `SafeText` only.
- Naming: `snake_case` modules and functions; `PascalCase` classes; rule IDs `<domain>.<rule>`; reason codes `snake_case[:arg]`; error codes `ST_E_<UPPER>`; schema IDs `st.<entity>/<major>`.
- Detection rules live in `engine/rules/*.json`, each with a `"version"` field. Don't hard-code rule lists.
- Tests mirror the engine layout. Golden files go through `tests/golden.py`; goldens are never updated in CI.
- Coverage: `engine/` ≥ 90% lines. 100% branch coverage on `render/safe.py`, `core/redact.py`, `io/gitops.py`, `io/objstore.py`, `io/runner.py`, `io/config.py` (merge) and `io/paths.py`.
- Keep code small: size budgets are a CI gate.

## Module boundaries (enforced by tests/security/test_import_rules.py)
- `core/*`: pure; only `re`, `shlex`, `json`, `dataclasses`, `typing`, `hashlib`, `hmac`, `unicodedata` and other `core/*`. No `io/`, `render/`, `os`, `subprocess` or file reads.
- `render/*`: only `core/model` and `render/*`. No `io/`, no `subprocess`.
- `io/*`: `core/*`, `render/safe`, standard library. `subprocess` only in `io/gitops.py` and `io/runner.py`; `os.fork`/`os.setsid` only in `io/worker.py`.
- `adapters/*`: `core/model` only.
- `cli.py`, `hooks_entry.py`: orchestration; may import anything in `engine/`.
- `publish/`, `analyze/`, `deep/`: `core`, `io`, `render`; never each other.

## Hard rules (blueprint §4.1; breaking one is a release blocker)
- I-1 Hooks never block, steer or inject: always exit 0, never exit 2 anywhere; no plain stdout from `SessionStart` or `UserPromptSubmit`.
- I-2 Zero footprint: nothing written into the user's working tree or `.git` (objects, refs, index) unless `reports: "repo"`, and then only `.claude/reports/`. Snapshot objects go in the private per-session object store (`GIT_OBJECT_DIRECTORY` + alternates). No refs.
- I-3 Raw events are deleted after the report; orphans are processed at the next start.
- I-4 git is invoked only through `run_git()` in `io/gitops.py`: hardened `-c` flags, per-call filter-driver neutralisation, fixed env, absolute binary path.
- I-5 No `shell=True`, `os.system`, `eval` or `exec` in `engine/`.
- I-6 Every untrusted string reaching output passes `untrusted()` (markdown or terminal mode).
- I-7 Output-bound strings are redacted with the current rules before rendering.
- I-8 Project config can only make settings stricter. Config is frozen at `SessionStart`.
- I-9 Events touching the tool's own files raise a critical tamper finding.
- I-10 IDs are regex-validated before any path use; resolved paths must stay inside expected roots.
- I-11 Publish confirmation is read only from `/dev/tty`. This is an accident guard, not a security boundary.
- I-12 Files are created `0600`, folders `0700` (`umask 077`).
- I-13 Zero runtime third-party dependencies in `engine/`.
- I-14 Content hashes are HMAC-SHA256 with the per-install key, never bare.
- I-15 Prompt text never appears in reports or published output unless opted in; prompts are referenced by number.
- I-16 No network modules in `engine/`: `socket`, `ssl`, `urllib`, `http`, `ftplib`, `smtplib`, `poplib`, `imaplib`, `xmlrpc`, `asyncio` streams/servers, `telnetlib`.
- I-17 Size budgets: `capture.sh` ≤ 40 lines; `engine/` (excluding `rules/`, `schemas/`) ≤ ~5,000 non-blank, non-comment lines.
- I-18 No repo-controlled command runs during snapshots: filter drivers, fsmonitor, hooks, external diff, textconv, pager.
- I-19 No hook path can open a GUI or install prompt: on macOS, `/usr/bin` binaries are used only after `xcode-select -p` succeeds.
- `capture.sh` never parses payloads. It writes a temp file, then renames it into `inbox/`.
- CLI exit codes are 0, 1, 3, 4, 5, 6 only; never 2.
- Repo content, filenames, git config, hook payloads and committed reports are untrusted input.
- Reproduce security PoCs only in Docker with `--network none`, never on the host.
- Blueprint items marked *(verify)*: check `docs/platform-facts.md` or current docs first. If it can't be verified, stop and report.

## Commit format
- `T<n>: <type>(<scope>): <summary>`, e.g. `T11: feat(io): add hardened run_git`. Types: feat, fix, test, docs, chore, ci, sec.
- Sign off every commit with `git commit -s` (DCO). Commits are SSH-signed once T3 is done.

## CLAUDE CODE RULES
- Work only on the dev branch. Never commit to, push to, merge into or rebase main. Never force-push.
- Every piece of work belongs to a numbered task (T1, T2, T3…). Start each commit message with the task number, e.g. "T7: add expense form".
- No AI attribution: no Co-Authored-By lines, no "Generated with" lines, no mention of Claude or AI in commits, pull request titles or descriptions.
- Never commit secrets (.env files, keys, passwords, tokens).
- A task is Done only when its tests and lint pass. Then update its line in docs/task-log.md: T[n] | [title] | [status] | [date].
- If the blueprint's plan won't work, stop and report it. Don't change approach on your own. After approval, add a DL entry at the bottom of the Change Log in docs/blueprint.md. Never edit or delete existing blueprint text. The newest DL entry overrides earlier text on the same topic.
- Ask before: deleting files or data, destructive database commands, adding dependencies, changing anything outside the current task, or touching any live environment.
- You may open a pull request from dev to main when asked, but never merge it.
- Output longer than about 5 lines (reports, plans, explanations, error details, questions) goes into docs/output.md, never the terminal. Overwrite the file completely each time. In the terminal, write only one line, e.g. "Done: T7. Report in docs/output.md".
- docs/output.md is in .gitignore. Never commit it.
- End every task with a report in docs/output.md: what changed, files touched, test results, and any problems or questions.
