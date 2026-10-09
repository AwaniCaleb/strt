# Project Blueprint — Session Trust

**Plugin ID:** `session-trust` · **Blueprint date:** 2026-10-09 · **Source:** Hardening session over the Mentor blueprint (DEC-1 – DEC-29, ASM-1 – ASM-16) · **Owner:** Georgina Awani
**Status:** Design complete, not yet implemented · **Target first release:** v0.1.0 (Phase 1) · **Licence:** Apache-2.0

> **How to read this document.** Every design choice cites its decision (DEC-n) or assumption (ASM-n). The full Decision & Assumption Log is in §8.4; §13 is the build-time Change Log. Items marked *(verify)* depend on Claude Code behaviour, prices, or legal points that must be checked against current sources before relying on them. This blueprint **supersedes the Mentor blueprint**; every difference is listed with its reason in §8.3.
>
> **Headline changes from the Mentor blueprint:**
> - **Zero footprint in user repos.** Baseline objects live in a private object store; no refs; reports are external by default (DEC-1, DEC-5).
> - **Filter-driver code execution closed** in the hardened git path (DEC-13).
> - **Wider capture.** `PreToolUse(Bash|Read)` plus `PostToolUse` now covers NotebookEdit and MCP tools (DEC-1, which resolves the old OQ-1).
> - **`SessionEnd` changes.** It runs synchronously for up to 3 s, then detaches; the per-turn `Stop` tally is off by default (DEC-14).
> - **Publish guarantee restated honestly.** The `/dev/tty` confirmation is now accident prevention, with bypasses detected (DEC-15).
> - **Shape-based compatibility warnings** replace version-range warnings (DEC-20).
> - **End-to-end testing on your Claude Pro login** (DEC-22).
> - **New decisions:** concurrent-writer handling (DEC-25), early outside testers (DEC-26), platform-timing positioning (DEC-27), built-in auditability (DEC-28), developer-owned by design (DEC-29).
>
> **Platform facts confirmed while compiling the Mentor blueprint (Claude Code hooks reference, checked 2026-10-09; re-verify on your target version):**
> - `SessionEnd` exists as a hook event.
> - Exit code `2` is a *blocking* error whose stderr is fed to Claude for `PostToolUse`.
> - Stdout with exit `0` is added to Claude's context for `UserPromptSubmit` and `SessionStart`.
> - `SessionEnd` stderr is shown to the user only.
> - `CLAUDE_PROJECT_DIR` is available to hooks.
> - Plugin hooks are declared in `hooks/hooks.json`.
> - Tool events carry `tool_name`, `tool_input`, and (PostToolUse) `tool_response`.
>
> Reference: https://docs.claude.com/en/docs/claude-code/hooks

---

## Table of contents

1. App Overview & Goals
2. System Architecture
3. Data Model & Database Design
4. Security Model & Threat Mitigations
5. API Structure & Key Flows
6. Testing & Quality Strategy
7. Deployment, Infrastructure & Operations
8. Accepted Trade-offs, Overrides & Revisit Triggers
9. Build Order & Implementation Roadmap
10. Directory & File Structure
11. Additional Technical Specifications
12. Assumptions & Open Questions
Appendix: Concepts Covered
13. Change Log (always last, append-only)

---

## 1. App Overview & Goals

### 1.1 Product description

Session Trust is a **Claude Code plugin** that produces an honest, readable account of what an AI coding session actually changed and why, after the session ends. It records every file-modifying and shell action Claude takes, snapshots the working tree at git level before and after, and reconciles the two. Every change on disk is either explained by a recorded action or flagged as unexplained.

Each changed file is placed in one of three tiers: **in scope**, **likely related**, or **unexplained**, always with a stated reason (DEC-4). Risky commands (force-pushes, recursive deletes outside the project, piping downloads into a shell, writes to `~/.ssh`, destructive database commands, and so on) are flagged by severity and cross-checked against what actually happened on disk (DEC-10).

It runs entirely on the user's machine. v1 makes no network calls, and this is enforced by a test (DEC-9, DEC-28). It holds no user data centrally, writes nothing into the user's repository unless they opt in (DEC-1, DEC-5), and can never block, steer, or inject text into the Claude session it observes (DEC-14).

It is positioned as a **self-review aid owned by the developer**: not a surveillance tool and not compliance evidence. Guarantees and limits are published (DEC-24, DEC-29).

### 1.2 Target personas

| ID | Persona | Situation | Needs from Session Trust |
|---|---|---|---|
| P1 | **Autonomous-run developer** (primary) | Mid/senior developer who lets Claude Code run long multi-step tasks, often unattended | "Did it do only what I asked? What else did it touch? Did it run anything dangerous?", answered in about 30 seconds after the session |
| P2 | **Reviewer** | Teammate or lead reviewing a PR produced with Claude Code | A trustworthy scope/risk summary alongside the PR (v1.1 publish via `gh`) |
| P3 | **Cautious adopter** | Engineering manager or security lead deciding whether AI coding agents are acceptable | Honest documented guarantees, a clear data-handling story, a credible path to team use (DEC-23, DEC-24) |
| P4 | **Maintainer** (solo author) | Under $50/month budget; Claude Pro subscription; about 15 focused hours a week with no hard deadline (ASM-12, ASM-13); builds with Claude Code + Claude Web | Low support load, safe releases, measurable accuracy, no servers to run, no recurring USD costs beyond Pro (DEC-19 – DEC-22) |

### 1.3 Core value proposition

> **"Know exactly what your AI session changed, why each change happened, and what it couldn't explain, without trusting the AI's own summary."**

**Positioning (DEC-27):** the README and v0.1 lead with **ground truth, not the agent's account**: reconciliation against the git diff, shell side effects, and risky-command outcomes. Scope tiers are the secondary feature. Claude Code's own checkpoints and telemetry (*verify current features*) track the agent's own edits. Session Trust audits the agent independently.

Four properties that plain diffs, checkpoints, and AI-written summaries do not provide:

1. **Completeness inside git repos.** A git diff against a session baseline catches every on-disk change, including shell side effects such as code generators, `sed -i`, and migrations (DEC-1).
2. **Visible reasoning.** Every classification states *why*, so misfires are visible and debuggable rather than silent (DEC-4).
3. **Independence from the audited agent.** The trustworthy outputs (full report, verification, publishing) are produced and shown outside Claude's control (DEC-14, DEC-15). Attempts to tamper with the tool are surfaced at the top of the report (DEC-8).
4. **Developer-owned and auditable.** Reports stay on the developer's machine unless they publish them (DEC-29). Every trust claim is backed by a test or a command the user can run: the no-network ban, size budgets, and `doctor --audit` (DEC-28).

### 1.4 Top 3 success metrics

| # | Metric | Target | How it is measured |
|---|---|---|---|
| M1 | **Classification accuracy:** precision and recall of the "unexplained" tier | Precision ≥ 0.80, recall ≥ 0.70 (initial targets, recalibrated once the corpus exists, DEC-19) | `make eval` runs the engine over the labelled corpus (§6.4) on every CI run. A release fails if either value drops more than 0.03 below the stored baseline. The gate also prints the exact files whose tier changed, as counts and paths (DEC-19). Current values are auto-published to `docs/guarantees.md` each release (DEC-24). |
| M2 | **Zero interference, low overhead** | Capture hook p95 ≤ 10 ms per call; `SessionEnd` report ≤ 5 s for a 500-event session on the reference repo; **0** confirmed bugs where a hook blocked a session or injected text into Claude's context | CI benchmark job (`bench/run_bench.py`: 200 capture-hook runs and 3 full report runs on a fixture repo) fails above thresholds × 1.5. GitHub issues labelled `interference` are tracked per release. |
| M3 | **Adoption** (no telemetry, DEC-9) | **Leading:** 3–5 outside testers on `next` at the end of Phase 1 (DEC-26). **Lagging:** set after the v0.1 baseline; suggested starting target 100 unique cloners per 14 days by end of Phase 3 | GitHub Insights → Traffic (unique cloners and visitors, 14-day window), recorded manually each release into `docs/metrics.md`; stars; count of issues filed with a valid `doctor --bundle` |

### 1.5 Explicitly out of scope (v1.x)

| Out of scope | Reason / revisit trigger |
|---|---|
| Native Windows, Git Bash, MSYS, Cygwin | Untested security controls (DEC-18). Revisit in Phase 2/3 if many users request it. |
| Any network call from the plugin in v1.0 (update checks, telemetry, crash reports) | DEC-9, DEC-21. `gh` publishing arrives in v1.1. |
| Blocking or preventing commands | Report-only by design (DEC-10). The tool suggests Claude Code deny rules instead. |
| Writing anything into Claude's context | DEC-14. A numbers-only opt-in "inform Claude" mode may come later. |
| LLM-generated summaries or classification | Offline, deterministic v1 (DEC-2, DEC-4). Advisory `--deep` mode in Phase 3. |
| Hosted team service, dashboards, multi-tenancy | DEC-9 option C, gated on roughly 10 or more teams requesting it. |
| Publishing to claude.ai Artifacts | Likely impossible from Claude Code (ASM-7, verify). |
| Claims of tamper-proof or compliance-grade evidence | DEC-24. Evidence-grade roadmap only. |
| Agents other than Claude Code | DEC-3. The standalone engine keeps a port possible. |
| PR policy `check` command | DEC-16 option C, only on demand. |
| Inspecting changes *inside* git submodules | DEC-25: reported as pointer changes only. Revisit if testers ask for it. |
| Telling human/editor edits apart from agent edits with certainty | DEC-25: unattributed changes are labelled "no agent action recorded" rather than guessed at. |
| Central or managerial collection of reports | DEC-29: developer-owned by design. Revisit only with the team service. |

---

## 2. System Architecture

### 2.1 Architecture pattern

**Local event-capture pipeline (pipes and filters) with ground-truth reconciliation.**

- *Capture filters:* tiny, never-failing hooks write immutable event files (DEC-2, DEC-14).
- *Ground truth:* a git tree snapshot at session start, written to a private object store and diffed against the working tree at report time (DEC-1).
- *Processing:* a pure-Python engine normalises events, reconciles them against the diff, classifies scope and risk, redacts, renders, then shrinks raw data into compact records (DEC-4, DEC-6, DEC-10, DEC-11).
- *Delivery:* a markdown report with a machine-readable JSON header (DEC-7), plus a terminal CLI that the human runs outside the agent's mediation (DEC-15, DEC-16).

There is no server, database process, or long-running daemon. The only background process is the short-lived detached report worker (DEC-14). All state lives in files with explicit permissions (DEC-5).

### 2.2 Component diagram

```mermaid
flowchart TB
    subgraph CC["Claude Code process (untrusted agent runs here)"]
        EV_SS["SessionStart"]
        EV_UP["UserPromptSubmit"]
        EV_PRE["PreToolUse (matcher: Bash|Read)"]
        EV_PT["PostToolUse (matcher: Bash|Write|Edit|MultiEdit|NotebookEdit|mcp__.*)"]
        EV_ST["Stop (tally off by default)"]
        EV_SE["SessionEnd"]
        SLASH["/session-report (counts and path only)"]
    end

    subgraph PLUGIN["Plugin at CLAUDE_PLUGIN_ROOT"]
        CAP["hooks/capture.sh - POSIX sh, max 40 lines, umask 077, always exit 0, one atomic file per event"]
        LIFE["hooks/lifecycle.sh - runs python3 -m engine hook EVENT, always exit 0"]
        subgraph ENGINE["engine/ - Python standard library only, no network modules, max ~5k lines"]
            PRE["preflight, xcode-select guard, config freeze, first-run notice"]
            BASE["baseline: seeded temp index + private object store (run_git)"]
            NORM["normalise, validate shape, pair Pre/Post events"]
            CONC["concurrent-writer detection"]
            REC["reconcile diff with events"]
            SCOPE["scope tiers"]
            RISK["risk rules incl. pty and direct-publish detection"]
            TAMP["tamper detection"]
            RED["redaction"]
            REN["render via untrusted()"]
            MIN["minimise records, delete raw"]
            DET["SessionEnd: 3 s synchronous budget, then detached worker"]
            CLI["CLI: report, verify, doctor (--audit, --bundle), config, purge, feedback, publish v1.1, analyze (DEC-27 trigger)"]
        end
    end

    subgraph USERSTATE["User state folder (0700, outside repo)"]
        INBOX[("inbox/ raw event files (max_inbox_mb)")]
        SESS[("projects/PID/sessions/SID/")]
        OBJ[("sessions/SID/objects/ private git object store")]
        RPTS[("projects/PID/reports/ (default report location)")]
        KEY[("hmac.key 0600")]
        ERR[("errors.log")]
        UCFG[("user config.json - authority")]
    end

    subgraph REPO["User repository (read-only by default)"]
        GITOBJ[(".git/objects (read via alternates, never written)")]
        GITIDX[(".git/index (copied, never modified)")]
        WT["working tree"]
        PCFG[".claude/session-trust.json - stricter-only"]
        REP[".claude/reports/ (opt-in only)"]
    end

    LAUNCH["~/.local/bin/session-trust launcher"]
    TERM["User terminal - human channel, /dev/tty accident guard"]
    GH["gh CLI to GitHub gist or PR comment (v1.1)"]

    EV_UP --> CAP
    EV_PRE --> CAP
    EV_PT --> CAP
    CAP --> INBOX
    EV_SS --> LIFE
    EV_ST --> LIFE
    EV_SE --> LIFE
    LIFE --> PRE
    LIFE --> DET
    PRE --> UCFG
    PRE --> PCFG
    PRE --> SESS
    BASE --> GITIDX
    BASE --> GITOBJ
    BASE --> WT
    BASE --> OBJ
    NORM --> INBOX
    NORM --> SESS
    CONC --> SESS
    REC --> BASE
    REC --> NORM
    REC --> CONC
    SCOPE --> REC
    RISK --> REC
    TAMP --> REC
    SCOPE --> REN
    RISK --> REN
    TAMP --> REN
    RED --> REN
    REN --> RPTS
    REN -.->|"opt-in"| REP
    REN -->|"user-visible channel only"| TERM
    MIN --> SESS
    MIN --> KEY
    ERR -.-> CLI
    SLASH -->|"counts and path"| CLI
    TERM --> LAUNCH --> CLI
    CLI -->|"v1.1"| GH
```

### 2.3 Components and rationale

| Component | Technology | Responsibility | Why it exists / why this choice |
|---|---|---|---|
| `hooks/capture.sh` | POSIX `sh` (tested under `dash`); absolute paths `/bin/mv`, `/bin/cat`, `/bin/date`, `/usr/bin/od`, `/usr/bin/tr`; at most 40 lines (DEC-28) | Writes raw stdin JSON for `UserPromptSubmit`, `PreToolUse(Bash\|Read)` and `PostToolUse(Bash\|Write\|Edit\|MultiEdit\|NotebookEdit\|mcp__.*)` to one file per event in `inbox/` | Runs on every captured tool call, so it must take about 5 ms with no parsing and no interpreter start-up (DEC-2). Temp-file-then-rename is atomic, so parallel tool calls cannot interleave. It never parses the payload, so it never builds paths from untrusted data. |
| `hooks/lifecycle.sh` | POSIX `sh` → `python3 -m engine hook <event>` | Runs the engine for `SessionStart`, `Stop`, `SessionEnd` | One place to enforce "always exit 0" (DEC-14) and to invoke the frozen absolute interpreter path (DEC-13). |
| `engine/` | Python ≥ 3.9 (ASM-10). Every module starts with `from __future__ import annotations`; no 3.10+ APIs at runtime. **Standard library only, no network modules** (I-13, I-16); ≈ 5k-line budget (DEC-28) | All logic | The developer's fastest stack. Zero runtime dependencies means zero supply-chain surface (DEC-2, DEC-12). Standalone, so it can run in CI and against any diff (DEC-3, DEC-16, DEC-27). |
| `run_git()` (`engine/io/gitops.py`) | `subprocess.run([...], shell=False)`; fixed environment; `-c` overrides, including per-call filter-driver neutralisation | The **only** place git is invoked | Neutralises repo-controlled git config that executes commands: `fsmonitor`, pager, external diff, `textconv`, hooks, and **filter drivers** (DEC-13). Enforced by test and ruff rule (DEC-17). |
| Private object store (`engine/io/objstore.py`) | `GIT_OBJECT_DIRECTORY=<state>/…/sessions/<sid>/objects` + `GIT_ALTERNATE_OBJECT_DIRECTORIES=<repo>/.git/objects` | Holds baseline/final tree and blob objects for one session | Zero writes to the user's `.git`; secret-bearing untracked blobs stay in the 0700 state folder; retention and purge are a folder delete (DEC-1, DEC-6, DEC-7; ASM-15). |
| `run_program()` (`engine/io/runner.py`) | Same pattern | The only place other external programs (`gh`, `claude --version`, `xcode-select -p`) are invoked | Single hardened execution path (DEC-13). |
| Rules data files | `engine/rules/*.json` | Versioned detection rules: secrets, commands, git hardening, sensitive paths, scope weights | Rules change faster than code. A version stamp in every report shows how current they are (DEC-6, DEC-10, DEC-13). |
| State folder | Filesystem, `0700` folders, `0600` files | Raw events (session lifetime only), manifests, private object stores, compact records, default reports, HMAC key, errors | Keeps raw secrets out of the working tree, builds and repo backups (DEC-5), and holds them as briefly as possible (DEC-6). |
| Report files | Markdown + JSON header block; default `<state>/projects/<pid>/reports/`; `<repo>/.claude/reports/` only when `reports: "repo"` | The shareable artefact | One file per session with a stable machine interface (DEC-7). External by default so repos stay clean and later Claude sessions don't read old reports (DEC-5). |
| Detached report worker | `os.fork()` + `os.setsid()` from the engine | Finishes reports that exceed the 3 s synchronous `SessionEnd` budget | Survives Claude Code exit and terminal close; typical sessions still show a summary at exit (DEC-14). |
| Launcher `~/.local/bin/session-trust` | `sh`, installed by `doctor --install` | Stable CLI path across plugin updates | Plugin folder paths may change per version (DEC-15). Tamper-watched. |
| `/session-report` | Plugin command (prompt file) | In-session convenience: counts and path only | Keeps report content out of Claude's context and avoids the agent relaying its own audit (DEC-15). |
| `doctor --audit` | CLI subcommand | Shows what is captured per hook, where it is stored, bytes on disk now, oldest raw file age, retention date | Lets a sceptical user verify the data-handling story on their own machine (DEC-28). |

### 2.4 Hook map

All hooks are declared in `hooks/hooks.json` with explicit timeouts (DEC-14). Timeout values are proposals: *verify allowed maximums and exact schema* (ASM-4).

| Hook event | Matcher | Command | Timeout | What it does | User-visible output |
|---|---|---|---|---|---|
| `SessionStart` | — | `lifecycle.sh start` | 20 s | See the step list below this table. | Only on problems ("runtime missing", "payload shape changed", "previous session had N hook errors") or the first-run notice, via the JSON `systemMessage` field *(verify field for this event)*. **Never plain stdout**, because stdout from `SessionStart` enters Claude's context. |
| `UserPromptSubmit` | — | `capture.sh` | 5 s | Store raw prompt event | None (stdout from this hook enters Claude's context) |
| `PreToolUse` | `Bash\|Read` | `capture.sh` | 5 s | Store raw *intent*. For Read: file path only, which feeds the `explored_early` scope signal. For Bash: the command, so interrupted, denied or killed commands are still recorded as "attempted" (DEC-1, DEC-10). | None. **Must never exit 2**, which would block the tool call. |
| `PostToolUse` | `Bash\|Write\|Edit\|MultiEdit\|NotebookEdit\|mcp__.*` | `capture.sh` | 5 s | Store raw tool result; paired with its `PreToolUse` event by `tool_use_id` (ASM-14) | None |
| `Stop` | — | `lifecycle.sh stop` | 5 s | **Off by default** (`stop_tally: false`). When enabled: counts only the new critical risk findings in inbox files added since the last `Stop`. No git, no scoring. | `⚠ 1 critical command so far` via `systemMessage` *(verify)*, or nothing. **Never exit 2**, which forces Claude to continue. |
| `SessionEnd` | — | `lifecycle.sh end` | 10 s | Generate the report synchronously within a **3 s budget**. If unfinished, write the partial header, detach a worker (`fork` + `setsid`) that completes the report, shrinks records and deletes raw events, and exit 0 (DEC-14). | Summary and capture-health line, escaped (DEC-11), via `systemMessage` or stderr *(verify which is shown at exit 0)*. If detached: "report finishing in background", and the summary appears at the next `SessionStart`. Fallback: `session-trust report --latest`. |

**`SessionStart` steps, in order:**
1. Platform check; Windows variants are disabled (DEC-18).
2. On macOS, the `xcode-select -p` guard runs before any `/usr/bin` binary, so no install dialog can appear (DEC-2).
3. Resolve absolute paths for `python3`, `git`, `gh` and `claude` (DEC-13).
4. Load and **freeze** config (DEC-8).
5. Write `disabled_roots` for the capture hook.
6. State version check and migration (DEC-20).
7. Orphan sweep (DEC-6).
8. Detect concurrent live sessions on the same repo (DEC-25).
9. Write the session manifest.
10. Take the git baseline into the private object store (DEC-1).
11. Show the first-run notice if it is due (DEC-29).

**Absolute rules for every hook:**
- The process exits `0` (`trap 'exit 0' EXIT`).
- The literal exit code `2` never appears under `hooks/`; a test enforces this.
- No plain stdout from `SessionStart` or `UserPromptSubmit`.
- The detached worker closes stdin/stdout/stderr before doing any work, so it can never write into Claude's context.

### 2.5 External integration points

| Integration | Direction | Interface | Compatibility check | Notes |
|---|---|---|---|---|
| Claude Code hooks | Claude Code → plugin | JSON on stdin; exit codes; JSON output fields | **Payload-shape validation** by the adapter at collection time; Claude Code version recorded for diagnostics only (DEC-20) | Payload fields used: `session_id`, `cwd`, `hook_event_name`, `tool_name`, `tool_input`, `tool_response`, `tool_use_id` (ASM-14), `prompt`, `source` *(verify names)*. Env: `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`. |
| Claude Code plugin system | Distribution | `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `hooks/hooks.json`, `commands/*.md` | *verify current schema* | Marketplace entries `stable` and `next` pinned to tags; fallback to protected `release/*` branches if tag pinning is unsupported (DEC-12, DEC-20) |
| Claude Code managed settings | Organisation → plugin install | Managed settings may install plugins on a developer's behalf *(verify)* | Detected at `SessionStart` if possible *(verify detectability)* | Triggers the first-run notice again (DEC-29) |
| git | Plugin → git | `run_git()` only | ≥ 2.35.2 (`safe.directory` behaviour, *verify*) | Fixed env, hardened `-c` flags including filter-driver neutralisation; private object directory via env; never overrides `safe.directory` (DEC-13) |
| macOS developer tools | Plugin → OS | `xcode-select -p` exit code via `run_program()` | macOS only | If it fails, `/usr/bin/python3` and `/usr/bin/git` are treated as absent (DEC-2) |
| GitHub CLI `gh` | Plugin → GitHub (v1.1) | `gh gist create`, `gh pr comment`, `gh repo view --json visibility` | ≥ 2.40 *(verify)* | Argument lists only; body via temp file; terminal-only; `/dev/tty` confirmation is an accident guard (DEC-9, DEC-15) |
| GitHub | Hosting | Repo, Releases, Actions, Pages, Discussions, Security Advisories | — | Public repo, so Actions minutes are free (ASM-9) |
| Claude Code headless (`claude -p`) | Testing only | Maintainer's **Claude Pro** login | — | Manual end-to-end runs only; never in CI (DEC-17, DEC-22). *Verify Pro terms cover personal manual testing.* |

### 2.6 End-to-end data flow

**1. Session start.** The user opens Claude Code in a repo. `SessionStart` runs `lifecycle.sh start`, which execs the engine with the cached absolute interpreter path. On macOS, a candidate under `/usr/bin` is used only if `xcode-select -p` succeeds. The engine then:
- detects the platform and exits with a notice on Windows variants;
- loads user config and applies only the parts of project config that make settings stricter;
- writes `config.frozen.json` and refreshes `<state>/disabled_roots`;
- runs state migrations if needed;
- sweeps orphaned sessions whose inbox files are older than 24 h (partial report if possible, then shrink and delete);
- lists live manifests for the same `repo_root` with no `ended_at` and records them as `concurrent_sessions` (DEC-25);
- writes `manifest.json`;
- takes the **baseline** (§11.6):
  - copy the real index (path from `git rev-parse --git-path index`) to `<state>/tmp/index-<sid>`, preserving the stat cache;
  - with `GIT_INDEX_FILE` pointing at that copy and `GIT_OBJECT_DIRECTORY=<session>/objects` plus `GIT_ALTERNATE_OBJECT_DIRECTORIES=<repo>/.git/objects`, run `git add -A`. Only changed and untracked files are rehashed; untracked files over 5 MB are skipped; budget 10 s; filter drivers are neutralised;
  - run `git write-tree`;
  - store the resulting tree hash in the manifest. No ref is written, and nothing is written into the user's `.git`;
  - a resumed session gets a fresh baseline in a new segment;
- shows the first-run notice once per machine, or again when a managed-settings install is detected (DEC-29).

**2. During the session.** `capture.sh` exits immediately on a Windows variant or if `$CLAUDE_PROJECT_DIR` matches a disabled root. Otherwise it writes each of these to `inbox/<epoch>-<pid>-<rand>.json` (temp file then rename, `0600`):
- user prompts;
- `Read` and `Bash` intents (`PreToolUse`);
- tool results (`PostToolUse`).

Nothing is parsed and nothing is printed. `Stop` does nothing unless `stop_tally` is enabled.

**3. Session end.** `SessionEnd` runs the engine with a 3 s synchronous budget:
1. Take the session lock. Collect inbox files whose parsed `session_id` matches (after strict regex validation) by **renaming** them into `sessions/<sid>/events/`. Malformed files are counted, not fatal. If the inbox exceeded `max_inbox_mb`, oversized content was already reduced to metadata.
2. Normalise into the internal event schema (§3.3.2) via the adapter. Shape mismatches become capture-health warnings (DEC-20). Pair `PreToolUse` and `PostToolUse` events by `tool_use_id` (fallback: order + tool name + input hash). A Bash intent without a result is marked `attempted_no_completion`.
3. Take the **final** tree the same way as the baseline, into the same private object store.
4. Diff baseline vs final with `diff-tree -r -M --no-ext-diff --no-textconv --name-status` plus mode changes. Submodule gitlink changes are reported as pointer changes only (DEC-25).
5. **Reconcile** (§11.2):
   - each changed file is matched to an edit event (explained), to a Bash event ("plausibly caused by"), or to nothing;
   - "nothing" is labelled **"no agent action recorded: may be you, your editor, or another session"**;
   - gitignored files written via Write/Edit/NotebookEdit are added from the event log;
   - re-check concurrent sessions; if any overlapped, set `shared_worktree: true` (DEC-25).
6. **Scope:** tier each file using only signals derived from user prompts and early exploration (DEC-4, §11.1).
7. **Risk:** `shlex`-parse each Bash command, apply the rules (including pty-wrapper and direct-publish detection), cross-check outcomes against the diff, and suggest deny rules (DEC-10, §11.3).
8. **Tamper:** flag events touching the plugin root, state folder (including object stores), configs or launcher, and check the manifest backstop (DEC-8, DEC-15).
9. **Redact** every output-bound string (DEC-6, §11.4), then **render** through `untrusted()` (DEC-11, §11.5). The header and skeleton are written first, so a timeout leaves a labelled partial report (DEC-13).
10. **If 3 s have elapsed:** fork and detach a worker (`setsid`, stdio closed) to finish steps 5–13. The hook process emits "report finishing in background" and exits 0 (DEC-14).
11. Write the report to `<state>/projects/<pid>/reports/session-<UTC>-<sid8>.md` (default), or to `<repo>/.claude/reports/` if `reports: "repo"` (DEC-5).
12. Emit the escaped summary and capture-health line through the user-visible channel. If the worker finished, the summary is shown at the next `SessionStart`.
13. **Shrink** events into `record.json` (paths, redacted commands, HMAC-SHA256 content hashes, sizes, tiers, findings), then **delete** `events/` (DEC-6).

**4. After the session.** The user runs `session-trust report --latest`, `session-trust verify <file>` or `session-trust doctor --audit` in a terminal (DEC-15, DEC-28). From v1.1, `session-trust publish`:
- re-redacts and strips prompt excerpts;
- checks repo visibility;
- shows a preview;
- reads a typed confirmation from `/dev/tty` (an accident guard, DEC-15);
- calls `gh` (DEC-9, DEC-23).

**5. Retention.** The sweep at each `SessionStart` deletes session folders (records, events remnants and **private object stores**), external reports, and migration backups older than 30 days (configurable) (DEC-6, DEC-7, DEC-20). Reports the user opted to write into the repo are the user's responsibility.

---

## 3. Data Model & Database Design

### 3.1 Storage engine and rationale

**No database engine.** Storage is the local filesystem with JSON documents (Python standard `json`), plus a **private, per-session git object store** for tree snapshots. That store lives in the state folder and reads the user's repo objects through git alternates (DEC-1; ASM-15).

| Alternative | Why rejected |
|---|---|
| SQLite (`sqlite3`, also standard library) | Concurrent writes from many hook processes would need locking in the fast path, and a shell hook cannot write SQLite without a binary. A single database file concentrates sensitive data and makes selective deletion harder. |
| Server database | No server exists in v1 (DEC-9). |

One file per event gives lock-free parallel capture (DEC-2). Folders per project and per session make `purge` and retention a matter of removing a folder. Every document is small and read whole.

**Atomic write helper** (used for every engine-written file):

```python
# engine/io/fsutil.py
import json, os, secrets
from pathlib import Path

def atomic_write_json(path: Path, obj: object) -> None:
    tmp = path.with_name(f".tmp-{os.getpid()}-{secrets.token_hex(4)}")
    fd = os.open(tmp, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
    with os.fdopen(fd, "w", encoding="utf-8") as f:
        json.dump(obj, f, ensure_ascii=False, sort_keys=True, separators=(",", ":"))
        f.flush()
        os.fsync(f.fileno())
    os.replace(tmp, path)
```

### 3.2 State folder layout

State root (DEC-5):
- **Linux/WSL:** `${XDG_STATE_HOME:-$HOME/.local/state}/session-trust`
- **macOS:** `$HOME/Library/Application Support/session-trust` (detected in shell by `[ -d "$HOME/Library/Application Support" ]`, which avoids forking `uname` on every call)

User config: `${XDG_CONFIG_HOME:-$HOME/.config}/session-trust/config.json` on both platforms.

```text
<state-root>/                              0700
├── state.json                             state_version, created_at, last_migrated_at
├── hmac.key                               0600 · 32 random bytes (os.urandom) · per install
├── disabled_roots                         0600 · one glob per line, written at SessionStart from user config
├── errors.log                             0600 · redacted + escaped JSON lines · rotated at 1 MiB → errors.log.1
├── tmp/                                   temporary git index files, seeded from a copy of the real index (deleted after use)
├── cache/
│   └── binaries.json                      resolved absolute paths + claude --version, keyed by path+mtime
├── inbox/                                 raw events from capture.sh (all projects, all live sessions)
│   ├── .tmp-<pid>-<rand>                  in-flight temp file (renamed when complete)
│   └── <epoch>-<pid>-<rand>.json          one raw hook payload per file
├── backups/
│   └── <timestamp>-v<from>/               pre-migration copy · deleted after verified migration or by retention
└── projects/
    └── <project-id>/
        ├── project.json                   id method, repo root (realpath), first-seen
        ├── index.json                     session list for --latest / --list (§3.4)
        ├── reports/                       DEFAULT report location (DEC-5); session-<UTC>-<sid8>.md
        ├── published.json                 publish ledger (v1.1, §3.3.10)
        └── sessions/
            └── <session-id>/
                ├── .lock                  O_EXCL lock while SessionEnd / report runs (stale after 10 min)
                ├── manifest.json          written at SessionStart (backstop, binaries, versions)
                ├── config.frozen.json     effective config for this session
                ├── events/                collected raw events (deleted after minimisation)
                ├── objects/               PRIVATE git object store: baseline/final trees + new blobs (DEC-1)
                ├── record.json            compact redacted record (kept for retention period)
                └── report.path            where the .md report was written
```

### 3.3 Entities and schemas

All timestamps are UTC ISO 8601 with `Z`. All IDs are validated against strict regular expressions **before** use in any filesystem path.

#### 3.3.1 Raw event (inbox file)

| Aspect | Specification |
|---|---|
| Filename | `^(\d{10})-(\d{1,10})-([0-9a-f]{16})\.json$`: epoch seconds, shell PID, 16 hex characters from `/dev/urandom` |
| Content | Exact stdin bytes from Claude Code (one JSON document); never parsed by shell |
| Max size accepted by engine | 8 MiB. Larger files are counted as `oversized` and only filename and size are kept. |
| Permissions | `0600` (`umask 077`) |
| Lifetime | Capture → collection at `SessionEnd`, or orphan sweep (24 h) |

**`capture.sh` reference implementation:**

```sh
#!/bin/sh
# Session Trust capture hook. POSIX sh. MUST always exit 0 (DEC-14). Never parses input (DEC-2).
umask 077
trap 'exit 0' EXIT INT TERM HUP
# Windows variants are unsupported (DEC-18): shell builtins only, no forks.
[ -n "$MSYSTEM" ] && exit 0
[ -n "$WINDIR" ] && exit 0
if [ -d "$HOME/Library/Application Support" ]; then
  ROOT="$HOME/Library/Application Support/session-trust"
else
  ROOT="${XDG_STATE_HOME:-$HOME/.local/state}/session-trust"
fi
# User-level opt-out (DEC-3 amended / DEC-8): glob patterns written by SessionStart.
if [ -n "$CLAUDE_PROJECT_DIR" ] && [ -f "$ROOT/disabled_roots" ]; then
  while IFS= read -r pat; do
    [ -n "$pat" ] || continue
    # shellcheck disable=SC2254
    case "$CLAUDE_PROJECT_DIR" in $pat) exit 0 ;; esac
  done < "$ROOT/disabled_roots"
fi
INBOX="$ROOT/inbox"
[ -d "$INBOX" ] || /bin/mkdir -p "$INBOX" 2>/dev/null || exit 0
RAND=$(/usr/bin/od -An -N8 -tx1 /dev/urandom 2>/dev/null | /usr/bin/tr -d ' \n')
[ -n "$RAND" ] || RAND="0000000000000000"
NOW=$(/bin/date +%s)
TMP="$INBOX/.tmp-$$-$RAND"
/bin/cat > "$TMP" 2>/dev/null || { /bin/rm -f "$TMP"; exit 0; }
/bin/mv -f "$TMP" "$INBOX/$NOW-$$-$RAND.json" 2>/dev/null || /bin/rm -f "$TMP"
exit 0
```

`doctor` confirms that every absolute path in this script exists on the host OS. Never use `PATH` lookups (DEC-13).

#### 3.3.2 Normalised event (internal; also the evaluation-corpus format, DEC-19)

```json
{
  "schema": "st.event/1",
  "seq": 17,
  "captured_at": "2026-10-09T10:22:31Z",
  "session_id": "8f1c2a9b-0d4e-4c11-9a77-3b2f6d1e8c40",
  "kind": "tool",
  "hook": "PostToolUse",
  "tool": "Bash",
  "tool_use_id": "toolu_01AbC…",
  "paired_seq": 16,
  "cwd": "/home/dev/app",
  "prompt_seq": null,
  "input": { "command": "npm run codegen", "file_path": null, "content_size": null },
  "result": { "exit_code": 0, "interrupted": false },
  "completion": "completed",
  "source_cc_version": "x.y.z",
  "flags": []
}
```

| Field | Type | Constraints |
|---|---|---|
| `schema` | string | Constant `st.event/1`; bump on breaking change |
| `seq` | int | Capture order (epoch, then filename), 0-based |
| `captured_at` | string | From filename epoch (seconds precision) |
| `session_id` | string | `^[A-Za-z0-9_-]{1,128}$`; non-matching events are rejected |
| `kind` | enum | `prompt` \| `read` \| `intent` (PreToolUse Bash) \| `tool` |
| `hook` | enum | `UserPromptSubmit` \| `PreToolUse` \| `PostToolUse` |
| `tool` | string or null | `Read` \| `Bash` \| `Write` \| `Edit` \| `MultiEdit` \| `NotebookEdit` \| `mcp__<server>__<tool>` \| null (prompts) |
| `tool_use_id` | string or null | From the payload when present (ASM-14); `^[A-Za-z0-9_-]{1,128}$` |
| `paired_seq` | int or null | `seq` of the matching Pre/Post event (by `tool_use_id`; fallback order + tool + input hash) |
| `completion` | enum or null | Bash only: `completed` \| `attempted_no_completion` (intent with no result: interrupted, denied, killed or failed without a PostToolUse event) |
| `cwd` | string | Absolute path; kept raw (`Untrusted`) for matching |
| `prompt_seq` | int or null | Prompts only: 1-based index, used as "prompt #n" in reports (DEC-23) |
| `prompt_text` | string | Prompts only. **In memory only**; persisted only as a redacted ≤ 120-character excerpt in `record.json` |
| `input.command` | string or null | Bash only; truncated at 16 KiB for analysis (DEC-10) |
| `input.file_path` | string or null | Read/Write/Edit/MultiEdit target; raw, for matching |
| `input.content_size` | int or null | Byte length of written content. **Content itself is discarded at normalisation.** |
| `result` | object | Fields copied from `tool_response` when present |
| `flags` | string[] | e.g. `missing_field:cwd`, `shape_mismatch`, `oversized`, `inbox_budget`, `malformed_json`, `unparseable_command`, `subagent`, `unpaired` |

For MCP tools, only the tool name, the target path if one is recognisable, and the result size are kept. `tool_input`/`tool_response` content is discarded at normalisation (DEC-6).

Raw-to-normalised field mapping lives in `engine/adapters/cc_v<n>.py`, one adapter per Claude Code payload shape (DEC-19, DEC-20).

#### 3.3.3 Session manifest (`manifest.json`)

```json
{
  "schema": "st.manifest/1",
  "session_id": "8f1c2a9b-0d4e-4c11-9a77-3b2f6d1e8c40",
  "segment": 1,
  "started_at": "2026-10-09T09:58:02Z",
  "source": "startup",
  "project_id": "p_3fa9c2d1e07b4a55",
  "repo_root": "/home/dev/app",
  "git_available": true,
  "baseline_tree": "4b825dc642cb6eb9a060e54bf8d69288fbee4904",
  "object_dir": "projects/p_3fa9c2d1e07b4a55/sessions/8f1c2a9b-…/objects",
  "index_seeded_from": "real_index",
  "concurrent_sessions": [],
  "baseline_partial": false,
  "baseline_skipped": [{"path_hmac": "hmac-sha256:…", "size": 73400320, "reason": "size>5MB"}],
  "binaries": {"python3": "/usr/bin/python3", "git": "/usr/bin/git", "gh": null, "claude": "/home/dev/.local/bin/claude"},
  "versions": {"plugin": "0.1.0", "engine": "0.1.0", "rules_secrets": "1.0.0", "rules_commands": "1.0.0",
               "rules_git": "1.0.0", "rules_scope": "1.0.0", "claude_code": "x.y.z", "git": "2.43.0", "python": "3.12.3"},
  "payload_shape": {"adapter": "cc_v1", "ok": true, "missing": []},
  "first_run_notice_shown": false,
  "expected_layout_hmac": "hmac-sha256:…"
}
```

- `segment` increments on resume (DEC-1 amendment).
- `object_dir` is relative to the state root. `index_seeded_from` is `real_index`, or `empty` when the repo has no index yet.
- `concurrent_sessions` lists other live session IDs on the same `repo_root` at start (DEC-25).
- `payload_shape` replaces the version-range check (DEC-20). The Claude Code version stays in `versions` for diagnostics only.
- `expected_layout_hmac` is the HMAC of the sorted list of files expected in this session's folder. It is re-checked at `SessionEnd` as the tamper backstop (DEC-8).

#### 3.3.4 Compact session record (`record.json`, kept for the retention period)

```json
{
  "schema": "st.record/1",
  "session_id": "8f1c2a9b-0d4e-4c11-9a77-3b2f6d1e8c40",
  "project_id": "p_3fa9c2d1e07b4a55",
  "segments": [{"segment": 1, "baseline_tree": "…", "final_tree": "…", "started_at": "…", "ended_at": "…"}],
  "versions": {"…": "as manifest"},
  "prompts": [{"prompt_seq": 1, "excerpt_redacted": "Fix the login redirect bug in …", "length": 412}],
  "files": [{
    "path": "app/Http/Controllers/AuthController.php",
    "change": "modified",
    "tier": "in_scope",
    "score": 1.0,
    "reasons": ["mentioned_in_prompt:1", "explored_early"],
    "attribution": {"type": "edit", "event_seqs": [12, 15]},
    "content_hmac_before": "hmac-sha256:…",
    "content_hmac_after": "hmac-sha256:…",
    "size_before": 4120, "size_after": 4388,
    "flags": []
  }],
  "commands": [{
    "event_seq": 21,
    "command_redacted": "git push --force origin «redacted:github-token»",
    "program": "git",
    "completion": "completed",
    "findings": [{"rule": "git.force_push", "severity": "critical", "outcome": "attempted"}],
    "opaque": false,
    "suggested_deny": "Bash(git push --force:*)"
  }],
  "tamper": [{"event_seq": 30, "target": "state_dir", "severity": "critical"}],
  "ignored_project_settings": ["allow_hosts"],
  "capture_health": {"events": 64, "malformed": 0, "oversized": 0, "missing_fields": 0, "hook_errors": 0,
                     "unpaired": 0, "payload_shape_ok": true, "inbox_budget_hit": false},
  "shared_worktree": false,
  "report_mode": "sync",
  "budgets_hit": [],
  "counts": {"in_scope": 5, "likely_related": 2, "unexplained": 1, "critical": 1, "warn": 2, "info": 3, "tamper": 1},
  "report_path": "<state>/projects/p_3fa9c2d1e07b4a55/reports/session-20261009T102944Z-8f1c2a9b.md",
  "created_at": "2026-10-09T10:29:44Z",
  "expires_at": "2026-11-08T10:29:44Z"
}
```

Constraints:
- `tier` ∈ {`in_scope`, `likely_related`, `unexplained`}.
- `attribution.type` ∈ {`edit`, `bash_plausible`, `none`}. `none` is rendered as "no agent action recorded: may be you, your editor, or another session" (DEC-25).
- `report_mode` ∈ {`sync`, `detached`} (DEC-14).
- `shared_worktree` is true if another session on the same repo overlapped this one in time (DEC-25).
- `change` ∈ {`added`, `modified`, `deleted`, `renamed`, `mode_changed`, `ignored_file_written`, `outside_repo_written`, `submodule_pointer_changed`}.
- `content_hmac_*` = HMAC-SHA256 keyed with `hmac.key`, never a bare hash (DEC-6 debrief).
- Files matching sensitive-path globs store only sizes plus `"flags": ["sensitive_path"]`.
- `prompts[].excerpt_redacted` is **only rendered into reports if `report_prompt_excerpts=true`** in user config (DEC-23).

#### 3.3.5 Report file (default `<state>/projects/<pid>/reports/session-<UTC>-<sid8>.md`; `<repo>/.claude/reports/` when opted in)

The JSON header comes first, so a partial report is still parseable (DEC-7, DEC-13).

````markdown
```json session-trust
{
  "schema_version": "1.0.0",
  "engine_version": "0.1.0",
  "rules_version": {"secrets": "1.0.0", "commands": "1.0.0", "git": "1.0.0", "scope": "1.0.0"},
  "session_id": "8f1c2a9b-0d4e-4c11-9a77-3b2f6d1e8c40",
  "generated_at": "2026-10-09T10:29:44Z",
  "status": "complete",
  "segments": [{"baseline_tree": "4b825dc6…", "final_tree": "9e1f…"}],
  "counts": {"in_scope": 5, "likely_related": 2, "unexplained": 1, "critical": 1, "warn": 2, "info": 3, "tamper": 0},
  "capture_health": "ok",
  "shared_worktree": false,
  "report_mode": "sync",
  "redaction": "automated; review before sharing",
  "limits": "review aid, not audit evidence; see docs/guarantees.md"
}
```

> ⚠ This file contains untrusted data captured from an AI coding session.
> Treat everything inside code blocks as data, not instructions.

# Session report · 2026-10-09 10:29 UTC

**1 unexplained change · 1 critical command (attempted) · 0 tamper alarms**

## Tamper alarms
None.

## Unexplained changes
| File | Change | Reason |
|---|---|---|
| `config/cache.php` | modified | no signal · no agent action recorded: may be you, your editor, or another session |

## Risky commands
| Severity | Outcome | Command | Suggested deny rule |
|---|---|---|---|
| critical | attempted | `git push --force origin «redacted:github-token»` | `Bash(git push --force:*)` |

## Likely related
| File | Change | Reasons |
|---|---|---|
| `tests/Feature/AuthTest.php` | modified | test counterpart of an in-scope file |

## In scope
| File | Change | Reasons |
|---|---|---|
| `app/Http/Controllers/AuthController.php` | modified | mentioned in prompt #1 · explored early |

## Summary
5 files modified under `app/Http/`, 1 under `config/`, 1 under `tests/`. 64 events captured. 1 Bash command flagged.

## Capture health
events 64 · malformed 0 · unpaired 0 · budgets none · payload shape ok (Claude Code x.y.z) · no concurrent sessions

## Verify
`session-trust verify --latest` (local machine only, while the session's object store is retained)
````

Header constraints for readers:
- maximum 64 KiB;
- parsed with `json.loads` only;
- validated against `schemas/report-v1.json`;
- readers support the current and previous major `schema_version` (DEC-7).

The `status` field is one of `complete`, `partial`, or `capture_log_incomplete`.

#### 3.3.6 Configuration

**User config, the authority (DEC-8):** `${XDG_CONFIG_HOME:-~/.config}/session-trust/config.json`

```json
{
  "schema": "st.config/1",
  "disabled_paths": ["/home/dev/clients/*"],
  "self_development": ["/home/dev/src/session-trust"],
  "reports": "external",
  "retention_days": 30,
  "report_prompt_excerpts": false,
  "stop_tally": false,
  "interpreters": {"python3": null, "git": null, "gh": null, "claude": null},
  "allow_hosts": ["registry.npmjs.org", "registry.yarnpkg.com", "pypi.org", "files.pythonhosted.org",
                  "repo.packagist.org", "github.com"],
  "extra_secret_patterns": [],
  "extra_command_rules": [],
  "budgets": {"baseline_seconds": 10, "untracked_file_max_mb": 5, "max_events": 20000, "max_diff_mb": 50,
              "max_inbox_mb": 256, "session_end_sync_seconds": 3}
}
```

**Project config, stricter-only (DEC-8):** `<repo>/.claude/session-trust.json`

| Field | Effect | Allowed in project config? |
|---|---|---|
| `extra_secret_patterns` | Adds redaction patterns (validated for linear-time safety, §11.4) | ✅ stricter |
| `extra_command_rules` | Adds risk rules | ✅ stricter |
| `reports` | `external` (default) \| `repo` | ✅ neutral (the report is always printed); `repo` is an explicit opt-in (DEC-5) |
| `retention_days` | Accepted only if **lower** than the user value | ✅ if lower |
| `disabled_paths`, `self_development`, `allow_hosts`, `interpreters`, `report_prompt_excerpts`, higher `budgets`, `stop_tally` | Weakening, privacy-reducing, or execution-related | ❌ Ignored; recorded as `ignored_project_settings` |

Config is merged at `SessionStart` and frozen into `config.frozen.json`. Mid-session edits apply from the next session at the earliest.

#### 3.3.7 Project identity

```text
project_id = "p_" + hex(sha256(root_commit_sha + "\0" + realpath(repo_root)))[:16]
fallback   = "p_" + hex(sha256("path\0" + realpath(cwd)))[:16]   # no commits, shallow clone without root, or no git
```

`project.json` records `"id_method": "root_commit+path" | "path"` (DEC-5 debrief). The root commit is found with `git rev-list --max-parents=0 HEAD`; if that fails or returns more than one commit, the lexicographically smallest is used.

#### 3.3.8 Private git object store (replaces in-repo refs, DEC-1)

| Item | Location | Created | Deleted |
|---|---|---|---|
| Baseline tree + new blobs | `<state>/projects/<pid>/sessions/<sid>/objects/` | `SessionStart` (and each resume segment) | Retention sweep, `purge` (folder delete) |
| Final tree + new blobs | Same folder | `SessionEnd` | Same |
| Temporary index | `<state>/tmp/index-<sid>` (copy of the real index) | Each snapshot | Immediately after `write-tree` |

Every snapshot runs with:
- `GIT_OBJECT_DIRECTORY=<session objects>`;
- `GIT_ALTERNATE_OBJECT_DIRECTORIES=<repo git-common-dir>/objects`, so unchanged blobs are read from the repo and never copied;
- `GIT_INDEX_FILE=<temp index>`.

**Nothing is written into the user's `.git`:** no objects, no refs, no index changes, no stash. Tree hashes are kept in the manifest and record; no ref is needed, because the folder itself keeps the objects alive until retention removes it.

**Limit:** if the user later garbage-collects blobs that a stored tree references through the alternate (for example a blob only reachable from a deleted branch), `verify` reports `ST_E_TREE_MISSING` honestly.

*Verify on the git floor version that `add`, `write-tree` and `diff-tree` honour both variables (ASM-15). Phase 0 spike.*

#### 3.3.9 Error log (`errors.log`)

One JSON object per line: `{"ts", "component", "code", "message_redacted", "session_id"}`. Every message passes through redaction (§11.4) and terminal-mode escaping (§11.5). Rotated at 1 MiB, keeping one previous file. Never printed unescaped (DEC-14).

#### 3.3.10 Publish ledger (v1.1): `<state>/projects/<pid>/published.json`

`[{"report_sha256", "destination": "gist" | "pr", "url", "published_at", "included_prompts": false}]`. This is used for publish idempotency (§5.6) and for `purge` reminders ("this report was published to `<url>`; purge cannot remove it").

### 3.4 Indexing strategy (tied to query patterns)

| Query pattern | Where | Strategy |
|---|---|---|
| Collect this session's events at `SessionEnd` | `inbox/` | One `os.scandir` pass; parse each file; keep those whose `session_id` matches. The inbox stays small because every `SessionEnd` and sweep drains it. |
| `report --latest` for current repo | `projects/<pid>/index.json` | List `[{session_id, started_at, ended_at, report_path, counts}]` sorted by `ended_at`, rewritten atomically after each report; O(1) read of the last entry |
| `report --session ID` | Path | Direct `projects/<pid>/sessions/<sid>/record.json` after regex validation |
| `report --list --since DATE --limit N` | `index.json` | Filter and slice in memory (at most a few hundred entries per project per retention window) |
| Retention sweep | `index.json` + `expires_at` | Iterate and delete expired session folders (records + object stores) and external reports |
| Orphan sweep | `inbox/` filenames | Epoch in the filename gives age without parsing; files older than 24 h are grouped by `session_id` |
| `verify FILE` / `--latest` | Report header + manifest | Locate the session's object store; `git cat-file -e` both trees with the object-dir env; recompute diff; compare file list and counts |
| Concurrent-session check | `projects/<pid>/sessions/*/manifest.json` | Scan live manifests (no `ended_at`, lock not stale) for the same `repo_root`; at most a handful per project |

### 3.5 Migration approach (DEC-20)

- `state.json.state_version` (int, starts at 1). Each schema (`st.event`, `st.manifest`, `st.record`, `st.config`) carries its own version string.
- Migrations live in `engine/migrations/v<from>_to_<to>.py`, each an idempotent `migrate(root: Path) -> None`, with fixture tests.
- When `state_version < CURRENT`:
  1. copy the state folder to `backups/<ts>-v<from>/` (same permissions);
  2. run migrations in order;
  3. validate;
  4. bump `state_version`;
  5. delete the backup (otherwise retention deletes it).
- When `state_version > CURRENT` (plugin downgraded): **refuse** with a visible notice and point to `session-trust doctor --reset-state` (typed `/dev/tty` confirmation; warns that history is lost).
- Config fields are deprecated for at least one minor release (with a `doctor` warning) before removal.
- Report schema: additive header field = minor bump; rename or removal = major bump.

### 3.6 Caching design

| Cached | Where | Key | Invalidation rule |
|---|---|---|---|
| Resolved absolute binary paths | `cache/binaries.json` | Candidate path + file mtime | Re-resolve at `SessionStart` if a path vanished or its mtime changed. User `interpreters` overrides always win. |
| `claude --version` output | `cache/binaries.json` | `claude` path + mtime | As above (DEC-20) |
| Effective config | `sessions/<sid>/config.frozen.json` | Session | Immutable for the session; rebuilt next session (DEC-8) |
| Compiled rule sets | In memory | Rules file version | Per process only |
| Baseline tree | Private object store | Session segment | Replaced on resume (new segment); deleted with the session folder by retention |
| `xcode-select -p` result (macOS) | `cache/binaries.json` | Developer-tools path + mtime | Re-checked at `SessionStart` if missing or changed |

### 3.7 Audit trail, deletion and retention

| Data | Location | Retention | Deletion |
|---|---|---|---|
| Raw inbox/event files (may contain secrets) | State folder | Until `SessionEnd`, or ≤ 24 h + next `SessionStart` for orphans; capped at `max_inbox_mb` | Hard `unlink` after shrinking (DEC-6) |
| Compact records | State folder | 30 days (configurable; project may only lower) | Hard delete by sweep or `purge` |
| HMAC key | State folder | Lifetime of install | `purge --all` |
| Migration backups | State folder | Until validated; at most the retention period | Hard delete |
| Private object stores (may contain untracked file contents) | State folder | 30 days | Hard delete of the folder (DEC-1, DEC-6) |
| Reports (default, external) | State folder | 30 days | Hard delete by sweep or `purge` |
| Reports written to the repo (opt-in) | Repo | User-controlled (git history) | User's responsibility; `purge` docs explain `git filter-repo` and its limits (DEC-23) |
| Publish ledger | State folder | Same as records | `purge` (does **not** delete the remote gist or comment; prints its URL) |
| `errors.log` | State folder | Rotating 2 × 1 MiB | Rotation |

There are **no soft deletes**: removal is real removal, because the point is holding less data. The docs state honestly that deletion on SSDs and copy-on-write filesystems is not secure wiping, and recommend full-disk encryption (DEC-23).

The tool's own audit trail per report consists of: `capture_health`, `versions`, `budgets_hit`, `ignored_project_settings`, and `status`. Together these record *how* the report was produced so readers can judge it (DEC-24).

### 3.8 Tenancy

v1 is **single-user and local** (ASM-1), so there is no tenancy. The future hosted team service (DEC-9 option C) is sketched in §11.8: PostgreSQL 16, `org_id` on every row, row-level security policies, per-org API tokens.

---

## 4. Security Model & Threat Mitigations

### 4.1 Security principles (non-negotiable invariants)

Each invariant has an enforcing test (§6.3). Breaking one is a release blocker.

| ID | Invariant | Source | Enforced by |
|---|---|---|---|
| I-1 | No hook can block, steer, or inject into a Claude session: always exit 0; never `2`; no plain stdout from `SessionStart`/`UserPromptSubmit` | DEC-14 | `tests/security/test_hook_contract.py` (static search + runtime runs with fault injection) |
| I-2 | Nothing is written into the user's repository (working tree or `.git`) unless `reports: "repo"` is set, and then only under `.claude/reports/` | DEC-1, DEC-5 | Integration test snapshots the repo directory tree and `.git` (objects, refs, index mtime and hash) before and after a full lifecycle; asserts zero differences |
| I-3 | Raw events are deleted after the report is generated; orphans are processed at the next start | DEC-6 | Integration test |
| I-4 | Git is invoked only via `run_git()` with hardened flags (including per-call filter-driver neutralisation) and a fixed environment | DEC-13 | ruff `S603/S607` + custom AST test banning `subprocess` imports outside `io/gitops.py` and `io/runner.py` |
| I-5 | No `shell=True`, no `os.system`, no `eval`/`exec` anywhere in `engine/` | DEC-9, DEC-13 | ruff `S602`, `S605`, `S307` + AST test |
| I-6 | Every untrusted string reaching output passes `untrusted()` (markdown or terminal mode); renderers accept only `SafeText` | DEC-11 | `mypy --strict` with `NewType` + golden tests + fuzzing |
| I-7 | Output-bound strings are redacted with the current rules before rendering, including at publish time | DEC-6, DEC-9 | Redaction fixture suite (100% of seeded secrets) |
| I-8 | Project config can never weaken settings | DEC-8 | Property test over generated configs |
| I-9 | Events touching the tool's own files raise a critical tamper finding | DEC-8, DEC-15 | Integration tests per target path |
| I-10 | IDs are regex-validated before any path use; paths are resolved and checked to stay inside expected roots | DEC-2 debrief | Unit + fuzz tests |
| I-11 | **Accident guard, not a security boundary:** publish confirmation is read only from `/dev/tty`; publish refuses without a TTY or when session env markers are present. Bypasses are **detected**: pty wrappers (`script`, `unbuffer`, `expect`, Python `pty`) and direct `gh gist create` / `gh pr comment` calls raise critical findings | DEC-15, DEC-10 | Integration tests: no TTY → refuse; env marker → refuse; each bypass pattern → critical finding |
| I-12 | All files created `0600`, folders `0700` | DEC-5 | Integration test (`os.stat`) |
| I-13 | Zero runtime third-party dependencies | DEC-2, DEC-12 | CI check: `engine/` imports resolve to the standard library only (`sys.stdlib_module_names`) |
| I-14 | Content hashes are HMAC-keyed, never bare | DEC-6 | Unit test |
| I-15 | Prompt text never appears in reports or published output unless explicitly opted in | DEC-23 | Golden tests |
| I-16 | No network-capable module is imported anywhere in `engine/`: `socket`, `ssl`, `urllib`, `http`, `ftplib`, `smtplib`, `poplib`, `imaplib`, `xmlrpc`, `asyncio` streams/servers, `telnetlib` (where present) | DEC-9, DEC-28 | `tests/security/test_no_network.py` (AST walk) + ruff `banned-api` |
| I-17 | Size budgets: `hooks/capture.sh` ≤ 40 lines; `engine/` (excluding `rules/` and `schemas/`) ≤ ~5,000 non-blank, non-comment lines | DEC-28 | `tools/check_size_budgets.py` in CI; README states current numbers |
| I-18 | No repo-controlled command runs during snapshots: filter drivers, `fsmonitor`, hooks, external diff, `textconv`, pager | DEC-13 | Canary repo whose `.git/config` defines `filter.evil.clean`/`process` (matched by `.gitattributes`) plus the existing canaries; the canary file must never be created |
| I-19 | No hook path can open a GUI or install prompt: on macOS, `/usr/bin` binaries are invoked only after `xcode-select -p` succeeds | DEC-2 | Unit test with a mocked `xcode-select` failure → `/usr/bin` candidates are skipped |

### 4.2 Authentication mechanism

| Phase | Mechanism | Implementation notes |
|---|---|---|
| v1.0 (local) | **None by design.** The tool acts as the local OS user and inherits OS account security. | No credentials stored or handled. Folder permissions (`0700`/`0600`) separate users on shared machines. |
| v1.1 (publish) | **Delegated to GitHub CLI.** `gh` holds the user's GitHub credentials. | Pre-check with `gh auth status` (exit code only). The tool never reads, stores, logs, or prints the token. `run_program()` passes an allowlisted environment: `HOME`, `PATH` set to the clean system path, `GH_CONFIG_DIR`, `GH_HOST`, and `GH_TOKEN`/`GITHUB_TOKEN` (passed only to `publish`, never logged). |
| v1.1 (publish) | **Accident guard.** Confirmation is read from `/dev/tty`. | The user must type the destination's short identifier (gist: `publish`; PR: the PR number). This stops *accidental* publishing, including Claude running the command without a terminal. It is **not** a security boundary: an agent can allocate a pseudo-terminal, and it already has the user's `gh` credentials. Those paths are **detected** and flagged as critical instead (DEC-15, I-11). |
| Future team service | GitHub OAuth login (GitHub App) for the web UI; API tokens for CLI and CI | Token format `st_<env>_<base62 32 bytes>`. Stored as SHA-256 of the full token (acceptable because the token is high-entropy random), with an 8-character prefix for lookup. Per-token scopes (`reports:write`, `reports:read`), expiry (90 days default), revocation, last-used timestamp. |

### 4.3 Session / token strategy

- **v1.0:** no sessions or tokens. The only long-lived secret the tool creates is `hmac.key` (32 random bytes, `0600`), used only to make content hashes non-reversible.
- **v1.1:** `gh` manages its own token lifecycle. Session Trust holds no token.
- **Future service:** short-lived web sessions (HttpOnly, Secure, SameSite=Lax cookies; 12 h idle timeout; rotate on login); API tokens as above; CI uses GitHub OIDC token exchange rather than stored tokens where possible.

### 4.4 Authorization model (trust boundaries)

v1 has no users or roles. Authorization means **who can influence the tool's behaviour**.

| Principal / input | Trust level | Can | Cannot |
|---|---|---|---|
| Human user at a terminal | **Trusted** | Everything: configure, purge, publish (with `/dev/tty` confirmation) | — |
| User config file | **Authority** | Set every option, including weakening options (`disabled_paths`, `allow_hosts`, `interpreters`, `self_development`, prompt excerpts) | — |
| Project config file | **Semi-trusted** (anyone who commits to the repo) | Make settings stricter (add rules or patterns, lower retention) and choose report location | Disable the tool, weaken detection, add hosts, change interpreters, enable prompt excerpts (DEC-8) |
| Claude (the audited agent) | **Untrusted actor** | Anything the OS user can do, through its tools, including publishing via `gh` directly | Do it **silently** (detection, not prevention). Tool self-tampering raises a critical alarm (DEC-8); `publish` from a session is refused and flagged; pty wrappers and direct `gh` publishing are flagged (DEC-15); frozen config ignores mid-session edits |
| Repo content (files, git config, filenames, `.envrc`) | **Untrusted input** | Appear in reports as escaped data | Execute code through the tool (DEC-13), inject into rendered output (DEC-11), or widen scope (DEC-4: only user prompts are scope sources) |
| Committed reports from others | **Untrusted input** | Be read by `verify`/`report` | Exceed header caps, smuggle HTML or instructions (DEC-7) |
| Subagent prompts (Task tool) | **Untrusted** (written by Claude) | Be logged as events | Count as scope sources (DEC-4 debrief) |

**Tenant isolation:** not applicable in v1. In the future service, isolation is enforced by PostgreSQL RLS keyed on `org_id`, set per request via `SET LOCAL app.org_id` (§11.8).

### 4.5 Threat model (STRIDE)

**Flow 1: Capture** (tool call → `capture.sh` → inbox file)

| | Threat | Mitigation | Status |
|---|---|---|---|
| S | Forged event files planted to fake attribution | State folder `0700`; manifest backstop; events must match the session ID; forged events can only *explain* changes, never hide them from the diff | ✅ DEC-5, DEC-8 |
| T | Agent deletes or edits events mid-session | Critical tamper alarm (command/path match) + manifest check; git diff remains complete; report status `capture_log_incomplete` | ✅ DEC-1, DEC-8 |
| R | Agent denies running a command | Event captured by hook outside the agent's control; deletion is detected (above) | ✅ |
| I | Raw secrets on disk | Session-length only; `0600`; outside repo; deleted after shrinking | ✅ DEC-5, DEC-6 |
| D | Parallel writes corrupt the log; disk fills | One atomic file per event; 8 MiB per-event cap at read; `max_events` budget | ✅ DEC-2, DEC-13 |
| E | `PATH` hijack (repo `bin/` via `.envrc`) makes the hook run attacker programs | Absolute paths only in `capture.sh` and `lifecycle.sh` | ✅ DEC-13 |

**Flow 2: Report generation** (`SessionEnd` → engine → git → report)

| | Threat | Mitigation | Status |
|---|---|---|---|
| S | Spoofed session ID used for path traversal | Strict regex; resolved-path containment check | ✅ I-10 |
| T | Hostile filenames or content alter the report's meaning or rendering | `untrusted()` code-fencing, invisible-character escapes, no links or images, lookalike detection | ✅ DEC-11 |
| R | Report falsified after the fact | Tree hashes + `verify` (honest drift only); signing is on the roadmap | 🟡 Accepted limit (DEC-7, DEC-24) |
| I | Secrets in report | Redaction rules + entropy + sensitive paths; minimal free text; prompt excerpts off by default | ✅ DEC-6, DEC-11, DEC-23 |
| D | Huge sessions, diffs or repos stall the engine past hook timeout | Budgets; partial-first writing; `status: partial` | ✅ DEC-13 |
| D | ReDoS via crafted command strings | Linear-time patterns, 16 KiB cap, token-based matching | ✅ DEC-10 |
| E | Repo `.git/config` executes code via `fsmonitor`, external diff, `textconv`, pager, hooks | Hardened `run_git()`; minimal env (`GIT_CONFIG_NOSYSTEM=1`, no `GIT_*` passthrough) | ✅ DEC-13 |
| E | **Filter drivers** (`filter.<name>.clean` / `.process`, selected by repo `.gitattributes`) run during `git add -A` at baseline/final snapshot | Per-call enumeration of `filter.*` keys and empty overrides + `required=false` (ASM-16); fallback: snapshot via `hash-object --no-filters` (§11.6) | ✅ DEC-13, I-18 |
| T | Snapshot writes alter the user's repo (objects, index lock, refs) | Copied temp index; private object directory with alternates; `GIT_OPTIONAL_LOCKS=0`; no refs | ✅ DEC-1, I-2 |
| E | `safe.directory` bypass | Never overridden; report "git refused" honestly | ✅ DEC-13 |

**Flow 3: Publish** (v1.1: report → re-redact → `gh` → GitHub)

| | Threat | Mitigation | Status |
|---|---|---|---|
| S | Agent impersonates the human to publish | `/dev/tty` confirmation and in-session refusal stop accidents. A pty wrapper or direct `gh` call can bypass them, so those are **detected** as critical findings in the session report | 🟡 Detection only (DEC-15, I-11) |
| T | Report modified between preview and upload | Preview and upload use the same in-memory bytes; SHA-256 shown in preview and stored in ledger | ✅ |
| R | User disputes what was published | Publish ledger (hash, URL, time) | ✅ |
| I | Report becomes public unexpectedly; secrets leak | Visibility check with explicit public-repo confirmation; re-redaction with current rules; prompt excerpts always stripped unless `--include-prompts` + warning | ✅ DEC-9, DEC-23 |
| D | Oversized comment rejected | Collapsible `<details>`; truncation with pointer to committed file | ✅ DEC-9 debrief |
| E | Command injection into `gh`; fake `gh` on `PATH` | Argument list; body via temp file; absolute `gh` path | ✅ DEC-9, DEC-13 |

**Accepted residual risks** (documented in `docs/guarantees.md`):
1. A determined agent with shell access as the same OS user can falsify events or the state folder *without* matching any detection pattern (for example through an obfuscated interpreter one-liner). The git diff still shows all on-disk changes, and manifest checks raise the bar, but this is **detection, not prevention**.
2. Reports can be edited by whoever controls the repo.
3. Redaction is best-effort.
4. Scope tiers are heuristic, with published error rates.
5. Publishing safeguards prevent accidents, not a determined agent: the agent holds the user's `gh` credentials. Bypasses are detected and reported, not prevented (DEC-15).
6. Edits by the developer, their editor, or a concurrent session cannot be told apart from unrecorded agent effects. They are labelled honestly rather than attributed (DEC-25).

### 4.6 Mapping to OWASP Top 10

*The 2025 edition list is used below; verify final category names. Equivalent 2021 categories are noted in brackets.*

| OWASP category | Relevance to Session Trust | Mitigations |
|---|---|---|
| A01 Broken Access Control | Project config or agent weakening the tool; path traversal via IDs | Stricter-only config (DEC-8); frozen config; regex + containment checks (I-10) |
| A02 Security Misconfiguration [2021 A05] | Permissive file modes; risky git config honoured | `umask 077`, explicit modes (I-12); hardened git (DEC-13); secure defaults (prompt excerpts off, publish off) |
| A03 Software Supply Chain Failures [2021 A06/A08] | Compromised account, CI action, or release reaching every user | Signed tagged releases; marketplace pinned to tags (or protected release branches); SHA-pinned actions; zero CI secrets; zero runtime dependencies; size budgets keep the code auditable (DEC-12, DEC-28) |
| A04 Cryptographic Failures [2021 A02] | Reversible hashes of secret files | HMAC-SHA256 with per-install 32-byte key (I-14) |
| A05 Injection [2021 A03] | Shell injection via filenames or commands; markdown/HTML injection; terminal escape injection; prompt injection into later AI readers | `shell=False` everywhere (I-5); `untrusted()` markdown and terminal modes (I-6); "data, not instructions" notice |
| A06 Insecure Design [2021 A04] | Observer that can steer the observed; audit relayed by the audited agent; auditor writing into the audited repo | Never-steer hook contract (I-1); human terminal path (DEC-15); report-only (DEC-10); zero-footprint snapshots (I-2) |
| A07 Authentication Failures [2021 A07] | Agent passing as the human for publish | `/dev/tty` accident guard plus bypass detection (I-11); delegated GitHub auth; documented as detection, not prevention |
| A08 Software or Data Integrity Failures [2021 A08] | Tampered reports or state; unsigned updates | Tree-hash verification; manifest backstop; signed tags (DEC-7, DEC-8, DEC-12) |
| A09 Security Logging & Alerting Failures [2021 A09] | Silent breakage hides gaps | Capture-health line; `errors.log`; tamper alarms; version and payload checks (DEC-14, DEC-20) |
| A10 Mishandling of Exceptional Conditions [new in 2025] | Hook crash or timeout produces no report or blocks session | Fail-open hooks with partial-first reports; malformed events counted, not fatal (DEC-13, DEC-14) |

**OWASP Top 10 for LLM Applications** (relevant because the tool observes an LLM agent):

| Category | Mitigation |
|---|---|
| LLM01 Prompt Injection | Only user prompts define scope; reports fence untrusted data; nothing is written to Claude's context |
| LLM02 Sensitive Information Disclosure | Redaction; short-lived raw data; excerpts opt-in |
| LLM06 Excessive Agency | The tool *surfaces* agency (risky commands, unexplained changes) and suggests deny rules |

### 4.7 Secrets management

| Secret | Where it lives | Who can read it | Rotation / handling |
|---|---|---|---|
| `hmac.key` | `<state>/hmac.key` (`0600`) | Local OS user | Created once; destroyed by `purge --all`; rotating it invalidates hash comparisons with older records (acceptable) |
| GitHub token (v1.1) | `gh`'s own storage (OS keychain or `gh` config) | `gh` | Managed by `gh`; never touched by Session Trust |
| Release signing key (SSH) | Maintainer's hardware key or password manager; public key in `allowed_signers` | Maintainer | Never deleted from `allowed_signers`; add date ranges on rotation (DEC-12) |
| Claude Pro login (manual e2e) | Claude Code's own credential storage on the maintainer's machine | Maintainer | Managed by Claude Code; never in repo or CI. **No Anthropic API key exists for this project** (DEC-22) |
| Private corpus encryption key (`age`) | Maintainer's password manager + offline backup | Maintainer | Rotate if exposed; re-encrypt corpus (DEC-19) |
| CI secrets | **None** | — | Zero-secret CI is an explicit invariant (DEC-12) |

### 4.8 Rate limiting, resource budgets and abuse prevention

There is no network API in v1, so there is no request rate limiting. Abuse prevention takes the form of **resource budgets** (DEC-13):

| Budget | Default | On hit |
|---|---|---|
| Baseline time | 10 s | Stop adding files; `baseline_partial: true`; report status `partial` |
| Untracked file size in baseline | 5 MB per file | Skip the file, recorded as `baseline_skipped` (path HMAC + size) |
| Events per session | 20,000 | Further events counted, not analysed |
| Single raw event size | 8 MiB | Metadata only (`oversized`) |
| Total inbox size | 256 MiB (`max_inbox_mb`) | Collection keeps metadata only for events beyond the budget; flag `inbox_budget`; summary notes it (DEC-6) |
| `SessionEnd` synchronous work | 3 s (`session_end_sync_seconds`) | Detach the worker; header `report_mode: detached` (DEC-14) |
| Diff size analysed | 50 MiB | Remaining files listed without content hashes |
| Command analysis length | 16 KiB | Analysed partially; finding `oversized_command` (info) |
| Report header size (readers) | 64 KiB | Rejected with `ST_E_SCHEMA_INVALID` |
| Files detailed in report | 500 per tier | Remaining files summarised by directory, with counts |
| `errors.log` | 1 MiB × 2 | Rotation |
| Hook timeouts | §2.4 | Partial report already written; orphan sweep completes it next session |

The future team service will add per-token rate limits of 60 requests/minute and 10 MB/day of uploads per organisation (initial values).

### 4.9 Audit logging

| What | Where | Content | Never contains |
|---|---|---|---|
| Per-session capture health | Report header + `record.json` | Event counts, malformed, oversized, missing fields, hook errors, Claude Code version in range | Raw payloads |
| Tool self-integrity | Report `Tamper alarms` section | Target category (state, object store, config, plugin, launcher), event sequence number, severity | Command arguments beyond redacted text |
| Shared working tree | Report header + summary | `shared_worktree`, count of concurrent sessions | Other sessions' content |
| Ignored project settings | Report + record | Field names only | Values |
| Internal errors | `errors.log` | Component, error code, redacted message | Tracebacks with local values; file contents; prompts |
| Publish actions (v1.1) | Publish ledger | Report hash, destination type, URL, time | Report body |

### 4.10 Data privacy and compliance obligations

*Legal points require verification by a qualified professional.*

**Data inventory:**

| Data | Personal data? | Stored where | Retention | Leaves the machine? |
|---|---|---|---|---|
| User prompts (full) | Likely yes | Memory + raw inbox (session only) | Minutes | No |
| Prompt excerpts (≤ 120 characters, redacted) | Likely yes | `record.json` | 30 days | Only if the user opts in **and** publishes with `--include-prompts` |
| File paths, branch names, commit context | Possibly (names inside paths) | Records, reports | 30 days / git history | Only via user commit or publish |
| Commands (redacted) | Possibly | Records, reports | 30 days / git history | Only via user commit or publish |
| File contents | Possibly | Raw inbox only (session) | Minutes; then HMAC + size only | No |
| Diagnostics bundle | Minimal (versions, counts) | User-created file | User-controlled | Only if the user posts it |

**Applicable frameworks** (general guidance, verify):
- GDPR / UK GDPR, if users or their colleagues are in the EU or UK.
- Nigeria Data Protection Act 2023.
- Similar laws elsewhere (for example POPIA, Kenya DPA 2019, CCPA/CPRA for California employers).

**Architectural impact:**
1. **The author is likely neither controller nor processor** for v1, because no data reaches the author (ASM-11). The organisation or individual running the tool is the controller of their own data.
2. **Data minimisation by default:** no prompt text in reports (DEC-23); short raw lifetime (DEC-6); no telemetry (DEC-9, DEC-21).
3. **Erasure:** `session-trust purge` covers local records, external reports, private object stores and the key. Opted-in repo reports in git history and published copies are outside its reach, and the docs say so (DEC-23).
4. **Workplace monitoring (DEC-29):** team deployments may count as employee monitoring. Session Trust is **developer-owned by design**:
   - reports stay on the developer's machine unless *they* publish;
   - a project config cannot force publishing or prompt excerpts;
   - a first-run notice tells the developer the tool is active. It reappears when the plugin was installed through managed settings (*verify detectability*).

   `docs/teams.md` ships a ready-to-send developer notice template for leads, DPIA prompts, and a note on consulting works councils where required. It is clearly marked *not legal advice*. A professional review is scheduled before v1.0 (OQ-12).
5. **Future hosted service:** the author becomes a processor (or controller) and needs a privacy notice, DPA templates, sub-processor list, data-residency choice (EU region recommended for the first customers), breach-notification procedure (72 h under GDPR; NDPA has its own timelines, verify), and records of processing.

---

## 5. API Structure & Key Flows

There is no HTTP API in v1. The "API" is three contracts: the **hook contract** (Claude Code ↔ plugin), the **CLI contract** (`session-trust` ↔ users, scripts, CI), and the **report/schema contract** (files ↔ readers).

### 5.1 Interface style and versioning

| Interface | Style | Version carrier | Compatibility rule |
|---|---|---|---|
| Hook contract | JSON on stdin; exit codes; JSON output fields | Adapter name per payload shape (`cc_v1`, …); Claude Code version recorded for diagnostics only | Field validation at collection: a missing or changed required field → visible capture-health warning. Version numbers alone never warn (DEC-20) |
| CLI | Subcommands; `--format json` | `cli_schema_version` in every JSON output (`schemas/cli-v1.json`) | Additive = minor; breaking = major with one-minor deprecation (DEC-16) |
| Report file | Markdown + JSON header | `schema_version` (`schemas/report-v1.json`) | Readers support current and previous major (DEC-7) |
| Config | JSON | `schema` field `st.config/1` | Deprecate one minor before removal |
| Internal state | JSON | `state_version` + per-entity `schema` | Forward-only migrations (DEC-20) |

**The version numbers** (documented in `docs/versioning.md`):

| Number | Example | Bumped when |
|---|---|---|
| Plugin/engine release | `0.4.2` → `1.0.0` | Any release (semantic versioning; 0.x until end of Phase 2) |
| Report `schema_version` | `1.0.0` | Report header changes |
| CLI `cli_schema_version` | `1.0.0` | JSON output shape changes |
| Rules versions | `secrets 1.3.0` | Rule data files change |
| `state_version` | `3` | Internal storage layout changes |

### 5.2 Complete interface listing

#### 5.2.1 Hook endpoints

See §2.4 for the full table. Contract per hook: input is a JSON payload on stdin; output is exit `0` always, with optional JSON `{"systemMessage": "<escaped text>", "suppressOutput": true}` *(verify field names and per-event support)*; no plain stdout for `SessionStart` and `UserPromptSubmit`.

#### 5.2.2 CLI commands

The launcher `~/.local/bin/session-trust` execs `<resolved python3> -m engine "$@"`.

| Command | Flags | Trust requirement | Description | Exit codes |
|---|---|---|---|---|
| `report` | `--latest` (default) \| `--session ID` \| `--list`; `--since DATE`; `--limit N` (default 20); `--format md\|text\|json`; `--out PATH`; `--fail-on SPEC`; `--project PATH` | Local user; read-only, safe inside a session | Print or regenerate a report from records; `--list` lists sessions | 0, 1, 3, 4, 5 |
| `verify` | `FILE` \| `--latest`; `--format text\|json` | Local user, on the machine that produced the report | Check a report against the trees in its session's private object store: recompute the diff, compare files and counts | 0 (match), 1 (mismatch), 3, 4 |
| `doctor` | `--format text\|json`; `--install` (create launcher); `--bundle PATH`; `--audit`; `--reset-state` | Local user; `--reset-state` needs `/dev/tty` | Health check of paths, versions, permissions, absolute binaries, `xcode-select` status (macOS), platform, last payload-shape result, WSL `/mnt/c` warning. `--bundle` writes the diagnostics file (DEC-21). `--audit` shows, per hook, what is captured and where; bytes on disk now (inbox, object stores, records, reports); oldest raw file age; next retention date; and the current size-budget numbers (DEC-28) | 0, 1 (problems found), 3, 4 |
| `config show` | `--format text\|json`; `--project PATH` | Local user | Effective merged config, including ignored project settings | 0, 3, 4 |
| `purge` | `--session ID` \| `--project` \| `--all`; `--yes-i-understand` is **not** accepted (always `/dev/tty`) | Human at terminal | Delete records, private object stores, external reports, key (`--all`); print reminders for published URLs and opted-in repo reports | 0, 3, 4, 6 |
| `feedback` | `REPORT FILE_PATH --tier in_scope\|likely_related\|unexplained`; `--export PATH` | Local user | Record a tier correction locally; `--export` writes a re-redacted file for a GitHub issue (DEC-19) | 0, 3, 4 |
| `publish` (v1.1) | `[FILE \| --latest]`; `--to gist\|pr`; `--pr N`; `--include-prompts` | **Human at terminal only**; refuses inside a session or without a TTY | Re-redact, strip prompt excerpts, visibility check, preview, `/dev/tty` confirmation, upload via `gh`, record ledger | 0, 3, 4, 6 |
| `analyze` (Phase 3) | `--base REF --head REF`; `--prompt-file F`; `--format json\|md`; `--fail-on SPEC`; `--allow-partial` | CI or local; read-only | Diff-only mode: scope and risk on any diff with no session log (DEC-16) | 0, 1, 3, 4, 5 |
| `version` | `--format text\|json` | Any | All version numbers | 0 |
| `hook start\|stop\|end` | — | **Internal**: called only by `lifecycle.sh` | Lifecycle entry points; always exit 0 regardless of internal result | 0 |
| `eval` | `--corpus DIR`; `--baseline FILE`; `--update-baseline` | Developer only (not in launcher help) | Accuracy evaluation (DEC-19) | 0, 1, 3, 4 |

**`--fail-on SPEC` grammar:** comma-separated terms from `critical` (any critical finding), `warn` (any warn or above), `tamper` (any tamper alarm), `unexplained:N` (more than N unexplained files), `partial` (status not complete; implied in CI unless `--allow-partial`).

**Exit codes** (DEC-16; **`2` is never used**):

| Code | Meaning |
|---|---|
| 0 | OK |
| 1 | Findings exceeded `--fail-on` threshold / verify mismatch / doctor found problems |
| 3 | Usage error |
| 4 | Internal error |
| 5 | Partial result (budget hit) |
| 6 | Refused for safety (inside a session, no TTY, user declined confirmation, newer state version) |

#### 5.2.3 In-session slash command

`commands/session-report.md`: instructs Claude to run `session-trust report --latest --format json` *(verify inline-execution syntax and allowed-tools frontmatter)* and print **only**: the counts, the report path, and the fixed sentence *"Summary relayed by Claude. The file is the source of truth: run `session-trust report --latest` in your terminal."* No file lists or command text are included (DEC-15).

### 5.3 Key flows (sequence diagrams)

**Journey 1: Normal session, capture to report**

```mermaid
sequenceDiagram
    autonumber
    actor U as Developer
    participant CC as Claude Code
    participant LS as lifecycle.sh
    participant CS as capture.sh
    participant EN as engine
    participant ST as State folder
    participant G as git (run_git)
    participant R as Reports (external by default)

    U->>CC: start session in repo
    CC->>LS: SessionStart payload
    LS->>EN: hook start
    EN->>ST: freeze config, write manifest, sweep orphans
    EN->>ST: detect concurrent sessions, first-run notice if due
    EN->>G: copy real index, add -A (filters neutralised), write-tree into private object dir
    G-->>EN: baseline tree hash (stored in manifest, no ref)
    LS-->>CC: exit 0 (no stdout)
    U->>CC: prompt "Fix login redirect"
    CC->>CS: UserPromptSubmit payload
    CS->>ST: inbox file (atomic rename)
    CS-->>CC: exit 0
    loop each tool call
        CC->>CS: PreToolUse(Bash|Read) intent, then PostToolUse result
        CS->>ST: inbox file
        CS-->>CC: exit 0
    end
    U->>CC: end session
    CC->>LS: SessionEnd payload
    LS->>EN: hook end
    EN->>ST: collect inbox files for SID, normalise
    EN->>EN: pair Pre/Post by tool_use_id, validate payload shape
    EN->>G: final tree into private object dir, diff-tree baseline final
    G-->>EN: changed files
    EN->>R: write header and skeleton first
    alt finished within 3 s budget
        EN->>EN: reconcile, scope, risk, tamper, redact, render
        EN->>R: complete session report
        EN->>ST: write record.json, delete raw events
        LS-->>CC: exit 0 with systemMessage summary
        CC-->>U: one-line summary and capture health
    else budget exceeded
        EN->>EN: fork + setsid detached worker (stdio closed)
        LS-->>CC: exit 0, "report finishing in background"
        EN->>R: worker completes report
        EN->>ST: worker writes record.json, deletes raw events
        Note over U,EN: summary shown at next SessionStart
    end
```

**Journey 2: Verifying a report**

```mermaid
sequenceDiagram
    autonumber
    actor U as Developer or reviewer
    participant L as session-trust launcher
    participant EN as engine
    participant G as git (run_git)

    U->>L: session-trust verify report.md
    L->>EN: verify report.md
    EN->>EN: read header, cap 64 KiB, json.loads, schema check
    alt header invalid or unsupported major version
        EN-->>U: ST_E_SCHEMA error, exit 3
    else header valid
        EN->>G: cat-file -e baseline and final trees (private object dir + alternates)
        alt trees missing (expired, other machine)
            EN-->>U: ST_E_TREE_MISSING with explanation, exit 4
        else trees present
            EN->>G: diff-tree baseline final (hardened)
            G-->>EN: file list
            EN->>EN: compare file list and counts with report body
            EN-->>U: match (exit 0) or itemised mismatches (exit 1)
        end
    end
```

**Journey 3: Publishing a report (v1.1)**

```mermaid
sequenceDiagram
    autonumber
    actor U as Developer at terminal
    participant L as session-trust launcher
    participant EN as engine
    participant T as /dev/tty
    participant GH as gh CLI
    participant GHub as GitHub

    U->>L: session-trust publish --latest --to pr --pr 42
    L->>EN: publish
    EN->>EN: refuse if session env markers present or no TTY (exit 6), an accident guard
    Note over EN: pty wrappers or direct gh calls inside a session are flagged critical in that session's report (detection, not prevention)
    EN->>EN: re-redact with current rules, strip prompt excerpts
    EN->>GH: repo view --json visibility (argv list)
    GH-->>EN: PUBLIC or PRIVATE
    EN->>EN: check ledger for same report hash
    EN->>T: show preview, SHA-256, visibility warning
    T-->>EN: user types 42 (or declines, exit 6)
    EN->>GH: pr comment 42 --body-file tmp (argv list)
    GH->>GHub: create comment
    GHub-->>GH: comment URL
    GH-->>EN: URL
    EN->>EN: append ledger entry
    EN-->>U: published URL, exit 0
```

### 5.4 Error format and conventions

**JSON error envelope** (any command with `--format json`):

```json
{
  "cli_schema_version": "1.0.0",
  "ok": false,
  "error": {
    "code": "ST_E_TREE_MISSING",
    "message": "Baseline tree for this report is no longer available.",
    "hint": "Trees are kept for 30 days in the private object store on the machine that produced the report.",
    "details": {"tree": "4b825dc6…"}
  }
}
```

Successful JSON outputs carry `"ok": true` plus a `data` object. All strings in `message`, `hint` and `details` pass through redaction and terminal escaping.

**Error code catalogue:**

| Code | Exit | Meaning |
|---|---|---|
| `ST_E_USAGE` | 3 | Bad flags or arguments |
| `ST_E_NOT_GIT` | 4 | Not a git repo (commands that require git) |
| `ST_E_GIT_REFUSED_OWNERSHIP` | 4 | `safe.directory` refusal; never bypassed |
| `ST_E_GIT_FAILED` | 4 | Other git failure (exit code + redacted stderr) |
| `ST_E_STATE_NEWER` | 6 | State written by a newer version; see `doctor --reset-state` |
| `ST_E_SCHEMA_INVALID` | 3 | Report header too large, malformed, or fails schema |
| `ST_E_SCHEMA_UNSUPPORTED` | 3 | Major version not supported |
| `ST_E_TREE_MISSING` | 4 | Session object store expired or purged, report from another machine, or referenced repo blobs garbage-collected |
| `ST_E_DEVTOOLS_MISSING` | 4 | macOS: `xcode-select -p` failed and no non-`/usr/bin` interpreter or git was found |
| `ST_E_NOT_FOUND` | 4 | Session or report not found |
| `ST_E_BUDGET` | 5 | Budget hit; partial output produced |
| `ST_E_NO_TTY` | 6 | Interactive confirmation required |
| `ST_E_IN_SESSION` | 6 | Command refused inside a Claude Code session |
| `ST_E_DECLINED` | 6 | User declined confirmation |
| `ST_E_GH_MISSING` / `ST_E_GH_UNAUTH` | 4 | `gh` not found / not authenticated |
| `ST_E_PLATFORM_UNSUPPORTED` | 4 | Windows variant |
| `ST_E_INTERNAL` | 4 | Unexpected exception (details in `errors.log`; traceback to stderr only with `--debug`, never from hooks) |

**Human output conventions:**
- One line per finding, severity first (`CRITICAL`, `WARN`, `INFO`).
- No ANSI colour unless stdout is a TTY and `NO_COLOR` is unset.
- All untrusted text is terminal-escaped.

### 5.5 Pagination, filtering and sorting

| Context | Standard |
|---|---|
| `report --list` | `--limit N` (default 20, max 500), `--since YYYY-MM-DD`; sorted by `ended_at` descending; JSON includes `next_since` when more results exist |
| Findings within a report and in JSON output | Sorted deterministically: severity (critical → info), then `event_seq`, then path (byte order) |
| Files within tiers | Path ascending (byte order); > 500 per tier summarised by top-level directory |
| JSON object keys | `sort_keys=True` for diffable output (DEC-16) |
| Future service API | Cursor pagination (`?cursor=<opaque>&limit=50`), filtering by `repo`, `since`, `severity`; sort by `generated_at` desc |

### 5.6 Idempotency and concurrency for retry-prone operations

| Operation | Risk | Rule |
|---|---|---|
| `SessionEnd` runs twice, or concurrently with `/session-report` | Double collection, duplicate reports | `sessions/<sid>/.lock` via `O_EXCL` (stale after 10 min). Inbox collection **moves** files, so a file is collected once. The report path is deterministic per session and segment, so a re-run overwrites the same file. `report` (CLI) never deletes raw events. |
| Orphan sweep interrupted | Half-processed session | Each step is a marker in `record.json.status`; the sweep resumes from the last completed step |
| Minimise-then-delete | Deletion before the record is safely written | Write `record.json` atomically and `fsync` **before** deleting `events/` |
| State migration interrupted | Corrupt state | Backup first; migrations idempotent; `state_version` bumped last |
| `publish` retried | Duplicate gist or comment | Ledger keyed by `report_sha256` + destination; a repeat prompts "already published at URL, publish again? (y/N)" via `/dev/tty` |
| Snapshot object writes | Two processes writing the same session object store | The session lock covers snapshots; git object writes are content-addressed and idempotent |
| Detached worker vs a later `report` call | Partial report read while the worker is still running | Worker holds the session lock; `report` shows `status: partial, report_mode: detached, in progress` until the lock is released |
| Future service: `POST /v1/reports` | Retries create duplicates | `Idempotency-Key` header = report SHA-256; unique `(org_id, report_sha256)` constraint; replay returns the original 201 body |

---

## 6. Testing & Quality Strategy

### 6.1 Test types and share of effort (DEC-17)

| Share | Type | Scope | Runs |
|---|---|---|---|
| ~60% | **Unit** | Pure functions: prompt signal extraction, stem normalisation, scope scoring, `shlex` parsing and rules, redaction patterns and entropy, `untrusted()`, config merge (stricter-only), path containment, ID validation, header building, migrations | Every push |
| ~30% | **Integration** | Real git in temporary repos; `capture.sh` under `dash` fed crafted stdin; full `hook start` → events → `hook end` runs; golden reports (mostly JSON records); permissions; retention; resume segments; publish with a fake `gh` binary at an absolute test path | Every push |
| (cross-cutting) | **Property/fuzz** (`hypothesis`) | `untrusted()`, command parser, redaction, config merge, report header reader | Every push (fixed seed in CI); longer runs weekly |
| (cross-cutting) | **Security invariants** | I-1 … I-19 (§4.1) | Every push; release blocker |
| (cross-cutting) | **Accuracy eval** | Labelled corpus precision/recall (DEC-19) | Every push (fast subset) + full run before release |
| (cross-cutting) | **Benchmarks** | Capture p95, report time (M2) | Every push on `ubuntu-24.04` only |
| ~10% | **End-to-end** | Real Claude Code via `claude -p` with the plugin installed from the local checkout, under the maintainer's **Claude Pro** login (shares Pro usage limits; 2–4 runs a month is negligible) | Manual: before each release and when a payload-shape warning or the weekly drift job flags a Claude Code change (DEC-17, DEC-20, DEC-22) |

### 6.2 Tools (development-only; runtime stays standard library)

*Pin exact versions in `requirements-dev.txt` with `--require-hashes`; versions below are majors to verify as current.*

| Tool | Version | Purpose |
|---|---|---|
| Python | 3.9 floor (ASM-10) and 3.14 | Test matrix; the 3.9 leg enforces `from __future__ import annotations` and no 3.10+ runtime APIs (DEC-2) |
| pytest | 8.x | Test runner |
| hypothesis | 6.x | Property-based fuzzing |
| coverage.py | 7.x | Coverage gates |
| ruff | latest 0.x | Lint + format + security rules (`S` = flake8-bandit set), import bans |
| mypy | 1.x, `--strict` on `engine/` | Types, including `Untrusted`/`SafeText` `NewType` separation |
| shellcheck | ≥ 0.10 | `hooks/*.sh`, `bin/session-trust` |
| dash | distro package | Runs `capture.sh` to catch bash-isms |
| gitleaks | 8.x | Secret scanning of the repo, including fixtures |
| git | OS default + latest | Matrix (DEC-18) |
| Docker | 27.x | `Dockerfile.dev` reproducible env; disposable PoC sandbox (`--network none`) |

The golden-file helper is custom (`tests/golden.py`, about 60 lines), so no snapshot-library dependency: compare normalised JSON/markdown, with an `--update-golden` pytest flag gated behind an environment variable that is never set in CI.

### 6.3 Paths that must have tests before any release

| Area | Required tests |
|---|---|
| Hook contract (I-1) | Static: no `exit 2` / `sys.exit(2)` in `hooks/` or hook code paths. Runtime: run every hook with malformed stdin, missing state folder, read-only disk, killed engine. Assert exit 0 and empty stdout for `SessionStart`/`UserPromptSubmit`. |
| Capture atomicity | 50 parallel `capture.sh` invocations with 2 MiB payloads produce 50 valid files and 0 temp leftovers |
| Pre/Post pairing | Paired by `tool_use_id`; fallback pairing when the ID is absent; a Bash intent with no result → `attempted_no_completion` + risk outcome `attempted`; NotebookEdit and `mcp__*` events normalised with content discarded |
| Baseline/diff completeness | Fixture repo covering: modified, added untracked, deleted, renamed, mode change, gitignored write via Write event, symlink, Unicode/lookalike names, file > 5 MB untracked, Bash `sed -i` side effect, codegen side effect |
| Hardened git (I-4, I-18) | Repo with malicious `.git/config` (`core.fsmonitor` = script writing a canary file; `diff.external`; `textconv` driver; `core.pager`; `core.hooksPath`; **`filter.evil.clean` and `filter.evil.process` selected by `.gitattributes` `* filter=evil`**) → canary never created during baseline, final, diff or verify. The filter config is also added **mid-session** (simulating the agent) → still never created |
| Zero footprint (I-2) | Full lifecycle in a fixture repo: repo tree, `.git/objects`, `.git/refs`, `packed-refs` and `.git/index` (content hash) are byte-identical before and after; the object store exists only under the state folder |
| Index seeding | Baseline on a 20k-file fixture repo with a warm index completes ≤ 2 s on `ubuntu-24.04`; the result equals a from-scratch baseline (tree hash match) |
| Concurrent writers (DEC-25) | Two overlapping sessions on one repo → both reports `shared_worktree: true`; a file changed outside any event → attribution `none` with the human/editor label; submodule pointer change reported as `submodule_pointer_changed` |
| Clean environment | `PATH` with fake `git`/`python3`/`mv` first → fakes never executed |
| Redaction (I-7) | 100% of seeded secrets (§11.4 catalogue × 3 variants each) redacted in reports, records, `errors.log`, bundle, publish preview; allowlist cases (git SHAs, UUIDs, integrity hashes) not redacted |
| Rendering (I-6) | Filenames with backticks (fence-length escape), `<script>`, `![x](http://…)`, `\u202e`, zero-width, ESC sequences, very long names → safe output in markdown and terminal modes |
| Config (I-8) | Property test: for any generated project config, the merged result is never weaker than user config on any weakening field |
| Tamper (I-9) | Events targeting state folder (incl. object stores and reports), config files, plugin root, launcher (direct and via `cd` + relative path) → critical; `self_development` path → info |
| Lifecycle | Resume creates a new segment and baseline; orphan sweep completes a killed session; double `SessionEnd` produces one report; a slow fixture (> 3 s) detaches, the hook exits 0 within budget, and the worker produces a complete report and releases the lock; the worker never writes to inherited stdout |
| Retention/purge | Records, object stores, external reports, backups older than retention deleted; `purge --all` leaves no state except the empty root |
| Platform | Windows-variant detection (env `MSYSTEM`, `WINDIR`, `uname` MINGW/MSYS/CYGWIN) disables cleanly (Windows CI job); macOS `xcode-select -p` failure → `/usr/bin` candidates skipped (I-19) |
| Auditability (I-16, I-17) | AST network-module ban; size-budget script; `doctor --audit` JSON validates against `schemas/cli-v1.json` and its byte counts match a fixture state folder |
| First-run notice (DEC-29) | Shown once per machine on the user-visible channel only; never on stdout of `SessionStart`; re-shown when the managed-install marker is set *(marker per Phase 0 findings)* |
| CLI contract | JSON outputs validate against `schemas/cli-v1.json`; exit codes per §5.2.2; `2` never returned |
| Publish (v1.1) | No TTY → refuse; inside session (env set) → refuse; public repo → extra confirmation; ledger idempotency; detection of `script -q /dev/null session-trust publish`, `unbuffer`, `expect`, Python `pty`, and direct `gh gist create` / `gh pr comment` inside a session → critical findings |

**Coverage gates:** overall `engine/` ≥ 90% lines. **100% branch coverage** on `render/safe.py`, `core/redact.py`, `io/gitops.py`, `io/objstore.py`, `io/runner.py`, `io/config.py` (merge), and `io/paths.py`.

### 6.4 Test data and seeding strategy

| Data | Location | How it's produced |
|---|---|---|
| Fixture repos | Generated at test time by `tests/fixtures/repo_builder.py` | Declarative spec → temp dir with `git init`, commits, `.gitignore`, untracked files, modes, symlinks. Never committed as binary repos. |
| Hook payload fixtures | `tests/fixtures/payloads/cc-<version>/*.json` | Captured from real sessions in Phase 0 and on each Claude Code release; **run through the tool's own redaction, reviewed by hand, gitleaks-scanned** before commit (DEC-17 debrief) |
| Synthetic sessions | `tests/fixtures/sessions/*.yaml`, converted at test time by `tests/fixtures/session_builder.py` (standard library, no YAML dependency: use `.json`) | Scenario → normalised events + repo spec |
| Public evaluation corpus | `eval/corpus/<case-id>/{events.jsonl, repo_spec.json, labels.json}` | Synthetic scenarios: vague prompts, refactors, codegen (`yarn`/`npm`/`composer`/`php artisan`), scope expansion mid-session, **scope drift across 10+ prompts** (DEC-4), flat Laravel `app/Http/Controllers` and `app/Models`, TypeScript `src/` feature folders, planted scope-creep cases, **concurrent human/editor edits** (DEC-25), interrupted Bash commands |
| Private evaluation corpus | Separate private repo `session-trust-corpus-private`, `age`-encrypted at rest | The maintainer's real sessions, redacted (DEC-19) |
| Labelling guide | `eval/LABELLING.md` | Rules for tiers (tests, lockfiles, formatting-only changes, generated files) |
| Eval baseline | `eval/baseline.json` | Stored precision/recall; updated deliberately with `engine eval --update-baseline` in a reviewed commit |

### 6.5 CI quality gates

| Gate | Tool | Blocks merge? |
|---|---|---|
| Lint + format | `ruff check`, `ruff format --check` | Yes |
| Shell lint | `shellcheck hooks/*.sh bin/session-trust` | Yes |
| Types | `mypy --strict engine/` | Yes |
| Unit + integration | pytest on matrix (ubuntu-24.04, macos-15) × (Python floor, 3.14) × (git default, latest) | Yes |
| Windows self-disable | `windows-latest` job: run hooks under Git Bash and assert clean disable | Yes |
| Security invariants | `tests/security/` | Yes |
| Standard-library-only imports | `tests/security/test_stdlib_only.py` (runs on Python 3.14 only, because `sys.stdlib_module_names` is 3.10+) | Yes |
| No network modules (I-16) | `tests/security/test_no_network.py` | Yes |
| Size budgets (I-17) | `python tools/check_size_budgets.py` | Yes |
| Secret scan | gitleaks (full history on default branch, diff on PRs) | Yes |
| Coverage | coverage thresholds (§6.3) | Yes |
| Accuracy | `engine eval` on public corpus vs baseline (−0.03 tolerance); output lists every file whose tier changed (DEC-19) | Yes |
| Benchmarks | `bench/run_bench.py` vs M2 thresholds × 1.5 | Yes |
| Schema validation | Golden outputs validate against `schemas/*.json` | Yes |
| Dependency review | Dependabot for GitHub Actions + `requirements-dev.txt` | Advisory |
| DCO | DCO check on PRs | Yes (once outside contributions start) |

**CI workflow sketch (`.github/workflows/ci.yml`):**

```yaml
name: ci
on:
  pull_request:          # never pull_request_target (DEC-16)
  push:
    branches: [main]
permissions:
  contents: read         # minimal (DEC-12)
jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<full-sha>        # pin by SHA, comment the tag
      - uses: actions/setup-python@<full-sha>
        with: { python-version: "3.14" }
      - run: pip install --require-hashes -r requirements-dev.txt
      - run: ruff check . && ruff format --check .
      - run: mypy --strict engine/
      - run: sudo apt-get install -y shellcheck dash && shellcheck hooks/*.sh bin/session-trust
  test:
    strategy:
      matrix:
        os: [ubuntu-24.04, macos-15]
        python: ["3.9", "3.14"]                  # floor per ASM-10
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@<full-sha>
      - uses: actions/setup-python@<full-sha>
        with: { python-version: "${{ matrix.python }}" }
      - run: pip install --require-hashes -r requirements-dev.txt
      - run: pytest -q --cov=engine --cov-fail-under=90 -p no:cacheprovider
        env: { HYPOTHESIS_PROFILE: ci }
  windows-disable:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@<full-sha>
      - shell: bash
        run: tests/windows/test_disable.sh
  security:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<full-sha>
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@<full-sha>
      - run: python -m pytest tests/security -q
      - run: python tools/check_size_budgets.py
  eval-bench:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<full-sha>
      - uses: actions/setup-python@<full-sha>
        with: { python-version: "3.14" }
      - run: python -m engine eval --corpus eval/corpus --baseline eval/baseline.json
      - run: python bench/run_bench.py --fail-over 1.5
```

A scheduled `weekly.yml` runs the full matrix plus the latest git release and longer `hypothesis` runs, to catch ecosystem drift with no code change (§7.5).

---

## 7. Deployment, Infrastructure & Operations

### 7.1 Services used

| Service | Use | Plan / cost | Phase |
|---|---|---|---|
| GitHub (public repo `session-trust`) | Source, issues, Discussions, Releases, Pages (docs), Security Advisories, private vulnerability reporting | Free | 0 |
| GitHub Actions | CI (Linux, macOS, Windows-disable) | Free on public repos for standard runners *(verify current terms)* (ASM-9) | 0 |
| GitHub private repo `session-trust-corpus-private` | Encrypted real-session corpus | Free | 2 |
| Claude Code marketplace (your repo's `.claude-plugin/marketplace.json`) | Distribution (`stable`, `next`) | Free | 1 |
| Claude Pro (existing subscription) | Manual end-to-end tests via `claude -p` | $0 marginal (already paid; *verify Pro terms cover personal manual testing*) | 1 |
| Password manager (existing) | Signing key backup, `age` key | Existing | 0 |
| Mirror remote (e.g. Codeberg or GitLab free tier) | Off-GitHub backup of the source repo | Free | 0 |
| Domain (optional) | Docs credibility | ~$10–15/year | 3 (optional) |
| Hetzner Cloud (or similar VPS), only on the DEC-9 trigger | Future team service | ~€4–6/month for a small instance *(verify)* | 3+ |

### 7.2 Environment layout

There are no servers, so "environments" are **release channels** and machines.

| Environment | What it is | Who uses it | Promotion rule |
|---|---|---|---|
| **Local dev** | Your checkout, loaded into Claude Code from a local path (local marketplace entry or plugin-dir flag, *verify mechanism*), with `self_development` set for the checkout path | You | Any commit |
| **CI** | GitHub Actions matrix (§6.5) | Automated | All gates green to merge to `main` |
| **`next` channel** (≈ staging) | Marketplace entry pinned to signed `vX.Y.Z-rc.N` tags | You (dogfooding), opt-in testers (documented as unsupported) | ≥ 3 days of your own daily use, no P1 bugs, manual end-to-end checklist passed |
| **`stable` channel** (≈ production) | Marketplace entry pinned to signed `vX.Y.Z` tags | All users | Release checklist complete (`docs/release-checklist.md`) |

### 7.3 CI/CD pipeline stages

```mermaid
flowchart LR
    A["PR opened"] --> B["Lint: ruff, mypy, shellcheck"]
    B --> C["Tests: unit, integration, fuzz on matrix"]
    C --> D["Security: invariants, gitleaks, stdlib-only"]
    D --> E["Eval and benchmarks"]
    E --> F["Merge to main (signed commit, no force-push)"]
    F --> G["Maintainer creates signed tag vX.Y.Z-rc.N locally"]
    G --> H["release.yml: verify tag signature against allowed_signers, run full gates"]
    H --> I["GitHub pre-release + marketplace next ref bump (signed commit)"]
    I --> J["Dogfood 3+ days + manual e2e checklist with claude -p"]
    J --> K["Signed tag vX.Y.Z"]
    K --> L["release.yml: verify, full gates, GitHub Release, CHANGELOG"]
    L --> M["marketplace stable ref bump (signed commit)"]
    M --> N{"Security fix?"}
    N -->|"yes"| O["Publish GitHub Security Advisory, request CVE if warranted"]
    N -->|"no"| P["Update known-issues page if needed"]
```

**Release checklist (`docs/release-checklist.md`):**
1. CI green on `main`.
2. Review the diff of `hooks/`, `engine/io/`, `engine/render/`, `engine/rules/` since the last tag.
3. CHANGELOG entry.
4. Version numbers bumped as appropriate (§5.1).
5. Eval numbers regenerated into `docs/guarantees.md`.
6. Tag signed and verifiable with `git tag -v`.
7. Dogfooding period complete on `next`.
8. Manual end-to-end checklist complete (`docs/e2e-checklist.md`) on the latest Claude Code.
9. Marketplace ref bump committed with a signature.

### 7.4 Containerisation strategy

| Context | Containers? | Details |
|---|---|---|
| The plugin itself | **No** | Runs as scripts inside the user's environment; containers would add install friction and break hook semantics |
| Reproducible dev/test env | Optional `Dockerfile.dev` | `ubuntu:24.04` + python3 + git + dash + shellcheck + gitleaks; `make docker-test` runs the suite |
| Vulnerability PoC reproduction | **Mandatory** | `docker run --rm --network none -v "$PWD/poc:/poc:ro" session-trust-dev`; never reproduce reports on your daily machine (DEC-21 debrief) |
| Future team service | Yes | Docker Compose: `app` (Flask 3.x + gunicorn), `db` (PostgreSQL 16), `proxy` (Caddy 2 with automatic TLS); images pinned by digest |

### 7.5 Observability (no telemetry)

| Signal | Source | How you see it |
|---|---|---|
| Per-session health | Capture-health line in every report and summary | Users see it; included in `doctor --bundle` |
| Local errors | `errors.log` | `doctor` summarises counts; bundle includes the last 50 redacted lines |
| Install health | `session-trust doctor` | Users run it; issue form requires the bundle |
| Build health | CI on every push + weekly scheduled run | GitHub notifications on failure |
| Platform drift | Payload-shape warnings in your own daily sessions (dogfooding) + weekly CI drift job + Claude Code changelog subscription | A shape warning or a hooks-related changelog entry triggers the end-to-end checklist and a known-issues update. Plain version bumps do not (DEC-20) |
| Platform timing (DEC-27) | Claude Code changelog and Anthropic announcements | An announced native session-change summary triggers moving `analyze` ahead of `publish` |
| Dependency/CI-action drift | Dependabot | PR notifications |
| Security reports | GitHub private vulnerability reporting | Email notification; triage within the stated window |
| Accuracy | Eval numbers per release | `docs/guarantees.md` history |
| Adoption | GitHub Traffic insights (M3) | Recorded manually in `docs/metrics.md` per release |
| Uptime alerts | Not applicable (no service) | Future service: Uptime Kuma (self-hosted) or a free external monitor + Sentry free tier *(verify)* |

### 7.6 Backup schedule and disaster recovery

| Asset | Backup | RPO | RTO | Recovery procedure |
|---|---|---|---|---|
| Source repo | GitHub + local clone + mirror remote (push on every release, weekly otherwise) | ≤ 1 week (0 for released code) | 1 h | Re-push from local clone or mirror |
| Signed tags / releases | In git (all three copies) | 0 | 1 h | As above; tags verify with `allowed_signers` |
| Release signing key | Hardware key + password-manager backup of a secondary key listed in `allowed_signers` | 0 | 1 day | Switch to the backup key; add a dated entry |
| Private corpus | Encrypted private repo + monthly encrypted offline copy | ≤ 1 month | 1 day | Restore and decrypt with the `age` key from the password manager |
| `age` key | Password manager | 0 | 1 h | Restore from the password manager; rotate and re-encrypt the corpus if exposed |
| GitHub account | Passkey + recovery codes stored offline | — | 1 day | GitHub recovery process |

**Incident runbook: malicious or broken release reaches `stable`:**
1. Immediately commit (signed) a marketplace `stable` ref change back to the last known-good tag.
2. Publish a GitHub Security Advisory (or a pinned issue for non-security breakage).
3. If account compromise is suspected: revoke all sessions and tokens, rotate the signing key, audit tags and commits since the last good release, and add the compromised key to the notes in `allowed_signers`.
4. Ship a fixed patch release through the full pipeline.
5. Write a post-incident note in `docs/incidents/`.

### 7.7 Cost estimate

*Prices are approximate as of 2026-10-09 and must be verified. Billing currency is USD; your card is confirmed to pay recurring USD charges (ASM-13).*

| Item | At launch (monthly) | At 10× users (monthly) | Assumptions |
|---|---|---|---|
| GitHub (repo, Actions, Pages, Releases) | $0 | $0 | Public repo; standard runners free (ASM-9) |
| Private corpus repo | $0 | $0 | Free private repos |
| Claude Pro (end-to-end tests) | $0 marginal | $0 marginal | Existing subscription (~$20/month, already paid for development, *verify*); 2–4 manual runs a month fit within Pro limits. At 10× users, more re-testing may hit limits; fallback is a capped API key (DEC-22 option C, ~$10–15/month) |
| Domain (optional) | $0 | ~$1 | Phase 3 only |
| Mirror remote | $0 | $0 | Free tier |
| **Total new spend (plugin only)** | **≈ $0** | **≈ $0–1** (or ≈ $10–16 if the API fallback is needed) | Users run the tool locally, so infrastructure cost does **not** grow with users. What grows is your support time. |
| *If the team service is triggered* | — | +$10–20 | Small VPS (~€4–6) + backups (~€1–2) + domain; Sentry and uptime monitoring on free tiers |

Even with the team service and the API fallback, total new spend stays around $20–36/month, inside the $50/month budget (the existing Pro subscription is counted separately). The real scaling cost is maintainer time (support, Claude Code compatibility), which the bundle, issue forms and known-issues page are designed to contain.

---

## 8. Accepted Trade-offs, Overrides & Revisit Triggers

### 8.1 Accepted trade-offs

| Decision | Gained | Sacrificed | Revisit trigger |
|---|---|---|---|
| DEC-1 Hybrid capture, private object store, seeded index, Pre/Post capture | Completeness inside git; zero writes to the user's `.git`; fast baselines on large repos; interrupted and denied commands recorded | Complexity; Bash attribution is "plausible", not proven; `verify` is local-only; blobs referenced through alternates can disappear after the user's `gc`; more inbox files per session | Most users outside git (ASM-3); exact attribution demanded (per-command snapshots); git drops alternates semantics (ASM-15) |
| DEC-1 (amend.) `SessionEnd` + `UserPromptSubmit` | No undocumented transcript dependency; one report per session | Reports depend on `SessionEnd` or the orphan sweep | `SessionEnd` unreliable in practice (OQ-3) |
| DEC-2 Shell + Python stdlib, 3.9 floor | ~5 ms capture; zero dependencies; reaches stock macOS | Writing to an end-of-life interpreter's API surface; macOS users without developer tools get no report until they install them | 3.9 leaves current Command Line Tools → raise floor to 3.10; > 10% of bundles show no usable Python → Go engine |
| DEC-3 Native plugin, standalone engine | One-command install; CI and diff-only reuse | Claude Code only; plugin-schema churn | Demand from other agents (see DEC-27) |
| DEC-4 Three tiers with reasons | Debuggable misfires; offline | Hand-tuned weights; vague prompts weak; scope accumulates over long sessions | Unexplained > 1/3 of files in eval, or precision < 0.7 after tuning → `--deep` becomes primary |
| DEC-5 Split storage, external reports by default | Raw secrets never in the working tree; clean repos; later sessions don't read old reports | Two locations; reviewers need `report --out` or `publish` | Team adopters (P2) dominate → recommend `reports: "repo"` in team docs |
| DEC-6 Shrink + delete, inbox budget | Minutes of raw exposure; bounded disk use | Old reports can't be regenerated with newer rules; events beyond 256 MiB keep metadata only | Team mode with encrypted central storage |
| DEC-7 Single `.md` + JSON header | One file; stable machine interface; local `verify` | Header editable; `verify` local-only and expires with retention | Team service (canonical JSON server-side) or signing demand |
| DEC-8 Stricter-only config | Hostile repos can't neuter the tool | Teams can't relax rules centrally | Org policy needs → signed org policy / team service |
| DEC-9 No network → `gh` → service | No data custody; GitHub handles identity | No live sharing in v1.0; gists are unlisted, not private | ~10+ teams request central reports |
| DEC-10 Parse + diff cross-check, report-only | Honest severities; no workflow breakage; bypass patterns surfaced | No prevention; rule catalogue upkeep | Users ask for blocking → generate deny rules automatically |
| DEC-11 Strict rendering + minimal text | Injection closed by construction | Less descriptive reports | `--deep` mode needs its own injection design |
| DEC-12 Tagged signed releases | Compromise must pass the release gate | Slower fix delivery; no update notifications in v1 | Second maintainer (add review); team buyers (Sigstore, DEC-28) |
| DEC-13 Hardened execution incl. filter drivers | No repo-controlled code execution, including via `git add` | LFS-tracked files are hashed raw during snapshots (consistent within a session); hardening list upkeep | New git config keys that execute commands; ASM-16 fails → `hash-object --no-filters` path |
| DEC-14 Never steer; 3 s sync `SessionEnd`; tally off | Zero interference; no per-turn latency; reports survive Claude Code exit | One fork path to test; large sessions show their summary next time instead of at exit | Phase 0 shows `SessionEnd` output never visible → always detach |
| DEC-15 Human terminal path, honest guarantee | Audit not mediated by the agent; accidents prevented; claims match reality | Publishing is protected by detection, not prevention | Claude Code adds a user-only confirmation channel |
| DEC-16 Stable CLI contract | Scriptable, CI-ready | Compatibility promise to maintain | Team traction → bring `analyze` forward (DEC-27) |
| DEC-17 Pyramid + manual e2e on Pro | Cheap deterministic coverage of security properties; $0 e2e | Drift caught manually; e2e shares Pro limits | Official offline hook simulator → automate e2e |
| DEC-18 Linux/macOS only | No untested security controls | Windows users excluded | Significant Windows demand (OQ-13) |
| DEC-19 Labelled corpus | Measurable tuning; no telemetry | Labelling effort; single-labeller bias | Corpus stale after a payload change; outside labellers available |
| DEC-20 Shape-based compatibility + migrations | Warnings mean real breakage; safe upgrades | A semantic change with the same shape goes unflagged until e2e | Anthropic publishes a versioned hook API |
| DEC-21 Bundle + forms, no telemetry | Fast diagnosis, no data held | Depends on users reporting | Team service |
| DEC-22 GitHub + Pro for e2e | ≈ $0 new monthly spend; no API key to protect | Pro usage limits shared with development | Pro limits block testing → capped API key |
| DEC-23 No personal text in reports | Erasure-resistant store free of personal free text | Reviewers can't read prompts | Compliance teams need prompts → team service with access control |
| DEC-24 Honest positioning, Apache-2.0 | Credibility; liability protection | Softer compliance pitch | Paying team commits → evidence-grade roadmap |
| DEC-25 Concurrent writers labelled, not attributed | Honest reports in shared working trees; no false blame on the agent | More "unexplained" entries when the developer edits during a session; submodule internals not inspected | Testers report the label is too noisy → add an mtime-window heuristic marked as such; submodule demand |
| DEC-26 Outside testers after Phase 1 | Real feedback months earlier; corpus data | Support load starts during Phase 2 | Testers unavailable → dogfood only and recruit at v1.0 |
| DEC-27 Plugin-first positioning | Fastest path to the hard-to-copy value | Platform risk until `analyze` ships | Native Anthropic session summary announced, or ≥ 2 testers want PR/CI use → move `analyze` ahead of `publish` |
| DEC-28 Auditability as a feature | Every trust claim testable by a sceptic | Size budgets constrain features; ~1.5 days of work | Team or compliance buyer → external assurance (Sigstore, paid review) |
| DEC-29 Developer-owned by design | A monitoring risk turned into a P3 selling point | Managers get no central visibility in v1 | Team service → full redesign in its own session |
| SK-3 Bus factor (accepted risk) | No coordination overhead | Single maintainer and account | First regular outside contributor → candidate second maintainer with release review rights |

### 8.2 Overrides of recommendations

**No architectural overrides.** In the hardening session, every pivot was accepted as recommended: DEC-1, 2, 5, 13, 14, 15, 20 and 22, plus the notes on DEC-4, 6, 7, 10, 12, 17 and 19, and new DEC-25 to DEC-29. The original 24 Mentor decisions also matched their recommendations.

**One process override:** the template asked for the blueprint in 6 parts, one per response. You asked for a single Markdown file instead.
- **Implication:** none on content. The blueprint was produced section by section against the original at full depth, so no section was shortened to fit.

### 8.3 Differences from the Mentor blueprint, and why

| # | What changed | From → to | Why | Refs |
|---|---|---|---|---|
| 1 | Baseline object storage | Objects in user `.git/objects` + `refs/session-trust/*` → private per-session object store in the state folder via `GIT_OBJECT_DIRECTORY` + alternates; no refs | Zero footprint in user repos; secret-bearing untracked blobs stay in the 0700 folder; deletion is a folder removal | DEC-1, DEC-7, ASM-15, I-2 |
| 2 | Baseline index | Empty temp index (full rehash) → copy of the real index (stat cache kept) | Large repos stayed inside the 10 s budget only partially; completeness was silently lost | DEC-1, DEC-13 |
| 3 | Capture matchers | `PostToolUse(Bash\|Write\|Edit\|MultiEdit)` + proposed `PreToolUse(Read)` → `PreToolUse(Bash\|Read)` + `PostToolUse(Bash\|Write\|Edit\|MultiEdit\|NotebookEdit\|mcp__.*)` paired by `tool_use_id` | Interrupted, denied or killed commands and other file writers were invisible; resolves OQ-1 | DEC-1, DEC-10, ASM-14 |
| 4 | Python floor | "3.9 if shipped" with 3.10+ code samples → 3.9 enforced with `__future__` annotations; `xcode-select -p` guard | Code contradicted the floor; macOS shims could open an install dialog from a hook | DEC-2, ASM-5, ASM-10, I-19 |
| 5 | Report location default | `repo` → `external` (`repo` opt-in) | No clutter or accidental commits; later Claude sessions don't read old reports | DEC-5 |
| 6 | Git hardening | Missing filter drivers; untracked cache disabled → per-call filter neutralisation; untracked cache allowed | `git add` ran repo-selected filter commands; disabling the cache only slowed large repos | DEC-13, ASM-16, I-18 |
| 7 | `SessionEnd` and `Stop` | Fully synchronous report; per-turn tally on → 3 s synchronous budget then detached worker; tally off by default and critical-count only | Reports killed on exit; per-turn latency | DEC-14 |
| 8 | Publish guarantee | `/dev/tty` as a security control → accident guard + bypass detection | A pseudo-terminal or direct `gh` call defeats it; the claim must be honest | DEC-15, DEC-10, DEC-24, I-11 |
| 9 | Compatibility warnings | Declared version range → payload-shape validation | Frequent Claude Code releases would cause alert fatigue on the health line | DEC-20 |
| 10 | E2E testing cost | Dedicated capped API key (~$10–15/month) → Claude Pro login | $0 new spend; one fewer secret | DEC-22, DEC-17, ASM-13 |
| 11 | Concurrent writers and submodules | Unhandled → `shared_worktree` flag, "no agent action recorded" label, worktree guidance, submodule pointer changes | Protects M1 precision; honest labelling | DEC-25 |
| 12 | Outside testing | After v1.0 → 3–5 testers on `next` after Phase 1 | ~15 h/week puts v1.0 months away; feedback is needed sooner | DEC-26, ASM-12 |
| 13 | Positioning | Generic "review aid" → lead with ground truth, shell side effects and risky outcomes; `analyze` trigger | Platform timing risk | DEC-27 |
| 14 | Auditability | Implicit → I-16 network ban, I-17 size budgets, `doctor --audit`, `docs/what-we-capture.md` | "No network" was unenforced; trust claims must be testable | DEC-28 |
| 15 | Monitoring stance | Docs note → first-run notice, rollout template, "self-review, not surveillance" | Transparency for developers under managed installs | DEC-29 |
| 16 | Inbox budget | None → `max_inbox_mb` 256 | Long sessions could hold hundreds of MB of raw content | DEC-6 |
| 17 | Minor | S2 file reads moved to `io/`; eval gate prints tier changes; drift and concurrency corpus cases; marketplace fallback to protected branches | Module boundary fix; readable gate; coverage; resilience | DEC-4, DEC-19, DEC-12 |
| 18 | Document structure | §13 Decision Log → §8.4; §13 is now the build-time Change Log; §3 and §5 renamed to template titles | Template conformance | — |

### 8.4 Decision & Assumption Log (source of truth)

```text
DECISION LOG — Session Trust — Source: Mentor session (DEC-1–24, ASM-1–11) + Hardening session (amendments, DEC-25–29, ASM-12–16) — Date: 2026-10-09
DEC-1 | Architecture | Change-capture strategy | Chosen: Hybrid — git-tree baseline at SessionStart (temp index, write-tree) + diff at report time + narrow PostToolUse logger (Bash|Write|Edit|MultiEdit); AMENDED: prompts via UserPromptSubmit (transcript not read); report at SessionEnd, Stop gives at most a counts-only tally; resumed sessions get a fresh baseline segment | Recommended: D staged (C + first prompt, then logger) | Override: No | Reason: each source's blind spot is covered by another; the gap between them is the trust signal | AMENDED (hardening): private per-session object store via GIT_OBJECT_DIRECTORY + alternates, no refs in user repos; temp index seeded from a copy of the real index; PreToolUse(Bash|Read) + PostToolUse(Bash|Write|Edit|MultiEdit|NotebookEdit|mcp__.*) paired by tool_use_id (resolves OQ-1)
DEC-2 | Architecture | Runtime and language | Chosen: POSIX sh capture hook (umask 077, one atomic file per event, no parsing) + Python stdlib-only engine + SessionStart runtime check that degrades visibly | Recommended: D with Python | Override: No | Reason: near-zero per-call cost, zero dependencies, fastest for the developer | AMENDED (hardening): floor 3.9 enforced (from __future__ import annotations, no 3.10+ runtime APIs); xcode-select -p guard before any /usr/bin binary on macOS
DEC-3 | Architecture | Packaging and distribution | Chosen: Native Claude Code plugin from own GitHub marketplace; engine standalone (python3 -m engine); AMENDED: opt-out via user-level disabled_paths | Recommended: A structured as D | Override: No | Reason: lowest-friction, most trustworthy install, CI reuse kept open
DEC-4 | Architecture | Scope classification | Chosen: Three tiers (in scope / likely related / unexplained) from deterministic signals with a reason per file; only user prompts are scope sources; sibling rule capped; stem normalisation; --deep LLM pass later, advisory | Recommended: D | Override: No | Reason: misfires visible and debuggable; offline and dependency-free | NOTE (hardening): S2 file reads moved to io/; corpus case for scope drift across many prompts
DEC-5 | Data | Storage locations | Chosen: Raw events in per-user state folder outside repo (0700/0600, XDG respected); redacted reports in .claude/reports or external per config; never modify user .gitignore | Recommended: D | Override: No | Reason: raw secrets never enter the working tree; zero footprint option | AMENDED (hardening): reports default to external (<state>/projects/<pid>/reports/); repo is opt-in
DEC-6 | Data | Redaction and retention | Chosen: Shrink raw events to redacted records (HMAC-SHA256 content hashes) at report time and delete raw; orphan sweep at next start; 30-day configurable retention; pattern + entropy + sensitive-path redaction with typed placeholders; versioned rules | Recommended: B + redaction layer | Override: No | Reason: minimum time holding sensitive data without slowing capture | NOTE (hardening): max_inbox_mb budget (256)
DEC-7 | Data | Report format and versioning | Chosen: One .md per session with JSON header (schema/engine/rules versions, session id, baseline/final tree hashes, counts, redaction notice); full data in compact record; trees pinned under refs/session-trust for retention period only | Recommended: C | Override: No | Reason: git-friendly single file, stable machine interface, verifiable against git | NOTE (hardening): trees kept in the private object store, not refs; verify is local-only and documented as such
DEC-8 | Auth | Config authority and self-tamper | Chosen: User config is authority; project config stricter-only (weakening ignored and noted); config frozen at SessionStart; critical alarm on events touching plugin/state/config/refs/launcher; manifest backstop; self_development exemption | Recommended: C | Override: No | Reason: the audited agent cannot silently disable its auditor
DEC-9 | Auth | Sharing and team access | Chosen: v1.0 no network; v1.1 publish via gh (secret gist or PR comment) with visibility check, re-redaction, preview, typed confirmation; hosted multi-tenant service only after ~10+ team requests | Recommended: A then B, C on demand | Override: No | Reason: safest data is data never held; GitHub handles identity
DEC-10 | Security | Risky-command detection | Chosen: shlex parsing with wrapper recursion, per-program rules with severities, opaque labelling, diff cross-check (attempted vs happened), report-only, suggested Claude Code deny rules | Recommended: D | Override: No | Reason: intent plus outcome gives honest severity without becoming a permission system | NOTE (hardening): rules for pty wrappers (script/unbuffer/expect/pty) and direct gh gist/pr publishing
DEC-11 | Security | Untrusted content in reports | Chosen: Single untrusted() renderer (markdown + terminal modes, code-span fencing, visible escapes for controls/bidi/zero-width, length caps); no images or links; fixed data-not-instructions notice; minimal free text with template summary; lookalike detection; Untrusted/SafeText type separation | Recommended: B + D | Override: No | Reason: closes rendering and injection channels by construction
DEC-12 | Security | Supply chain and releases | Chosen: Marketplace pinned to signed tags; release checklist; SHA-pinned actions; minimal CI permissions; zero CI secrets; no force-push; required checks; zero runtime deps; Sigstore later for teams; official directory when available | Recommended: B | Override: No | Reason: compromise must pass the release gate instead of reaching all users instantly | NOTE (hardening): fallback to protected release/* branches if marketplace tag pinning is unsupported
DEC-13 | Security | STRIDE residual mitigations | Chosen: Absolute binary paths from clean system path, frozen per session; minimal env; single hardened run_git() (fsmonitor, pager, external diff, textconv, hooks disabled); resource budgets with partial-first reports; interpreter overrides in user config only | Recommended: B | Override: No | Reason: closes repo-controlled code execution and silent timeouts at the source | AMENDED (hardening): filter drivers neutralised per run_git() call (fallback hash-object --no-filters); untrackedCache no longer disabled
DEC-14 | Interface | Hook I/O and failure contract | Chosen: Always exit 0 via trap; exit 2 banned by test; nothing written to Claude's context; user notices only via documented user-visible channel, escaped; explicit timeouts; capture-health line; redacted rotated errors.log; numbers-only inform-Claude mode possible later as opt-in | Recommended: B | Override: No | Reason: an auditor must never steer what it audits | AMENDED (hardening): SessionEnd synchronous up to 3 s then detached worker; Stop tally off by default, critical-count only when on
DEC-15 | Interface | On-demand report surface | Chosen: /session-report returns counts and path only, labelled relayed by Claude; terminal CLI (report, verify, doctor, config show, publish in v1.1) is the trusted path; publish terminal-only with /dev/tty confirmation and flagged if invoked from a session; stable tamper-watched launcher in ~/.local/bin | Recommended: D | Override: No | Reason: anything that must be trustworthy runs where the agent cannot mediate it | AMENDED (hardening): /dev/tty confirmation reframed as accident guard; pty and direct-gh bypasses detected as critical; I-11 rewritten
DEC-16 | Interface | Engine CLI contract | Chosen: --format json with published JSON Schemas; exit codes 0/1/3/4/5/6 (never 2); --fail-on; no prompts without TTY; deterministic output; partial = failure in CI unless --allow-partial; analyze diff-only mode in Phase 3; PR policy check only on demand | Recommended: B now, D Phase 3, C on demand | Override: No | Reason: cheap stable contract now, CI bridge to teams later
DEC-17 | Testing | Test strategy and effort split | Chosen: ~60% unit, ~30% integration (real git, capture.sh under dash, golden mostly JSON), ~10% manual e2e via claude -p (spend-capped, per release and per Claude Code release); hypothesis fuzzing; ruff security rules; exit-2 ban test; shellcheck; mypy --strict; gitleaks | Recommended: B | Override: No | Reason: cheap deterministic checks on the security properties; e2e only for what only it can catch | NOTE (hardening): manual e2e under the maintainer's Claude Pro login
DEC-18 | Testing | Platform support and CI matrix | Chosen: Linux and macOS (incl. WSL) supported; Windows/Git Bash/MSYS/Cygwin detected and disabled with notice, tested in CI; Python floor = macOS developer-tools version; test oldest + 3.14 against OS-default and latest git; pinned runner images; public repo | Recommended: D | Override: No | Reason: never ship untested security controls; fail honestly elsewhere
DEC-19 | Testing | Heuristic accuracy measurement | Chosen: 50–100 session hand-labelled corpus (public synthetic + planted-creep; private real sessions encrypted), normalised format, labelling guide, precision/recall CI gate (initial >=0.8/>=0.7), opt-in local feedback with re-redacted export | Recommended: B + C | Override: No | Reason: measurable tuning without telemetry | NOTE (hardening): CI gate also reports tier changes as counts and paths
DEC-20 | Deployment | Release, versioning and compatibility | Chosen: Semver 0.x until end of Phase 2; declared Claude Code range checked at SessionStart (known install locations, cached); payload field validation; versioned state with backed-up forward-only migrations, refuse on downgrade, doctor --reset-state; one-minor deprecation; stable/next channels; security patches with advisories | Recommended: B | Override: No | Reason: version drift becomes visible instead of silent; state protected across upgrades | AMENDED (hardening): compatibility warnings driven by payload-shape validation; Claude Code version recorded for diagnostics only
DEC-21 | Deployment | Learning about failures | Chosen: doctor --bundle (versions, redacted frozen config, health counts, re-redacted errors; never events/records/prompts); issue forms with review checkbox and CC-version labels; SECURITY.md + GitHub private vulnerability reporting with realistic response window; known-issues page; Discussions; PoC reproduction only in disposable environments | Recommended: B + D | Override: No | Reason: fast diagnosis with no telemetry and no data held
DEC-22 | Deployment | Infrastructure and costs | Chosen: GitHub only (public repo, Actions, Releases, Pages, Discussions, private encrypted corpus repo) + dedicated workspace API key capped ~$10–15/month for manual e2e; domain optional later; team hosting documented not provisioned | Recommended: B without domain | Override: No | Reason: predictable isolated cost, no idle attack surface | AMENDED (hardening): Claude Pro subscription for manual e2e; dedicated API key dropped (OQ-14 closed); capped key only as a fallback
DEC-23 | Compliance | Personal data in reports | Chosen: Prompts referenced by number only; no names/emails/usernames; excerpts opt-in via user config only; publish always strips excerpts unless --include-prompts with warning; data-processing page; team monitoring guidance (not legal advice); purge command with honest deletion caveats | Recommended: B + C | Override: No | Reason: keep personal free text out of permanent git history by default
DEC-24 | Compliance | Product claims and positioning | Chosen: Positioned as session review aid; guarantees and limitations page with CI-generated accuracy; limits line in report header; Apache-2.0 + DCO; evidence-grade roadmap (signed reports, append-only central storage, change-management control mapping) gated on team trigger | Recommended: B + C | Override: No | Reason: honest claims are the product and protect the solo author
DEC-25 | Architecture | Concurrent writers and submodules | Chosen: detect other live sessions on the same repo (shared_worktree flag); unattributed changes labelled "no agent action recorded: may be you, your editor, or another session"; git worktree guidance; submodules reported as pointer changes only | Recommended: same | Override: No | Reason: protects M1 precision and labels honestly instead of guessing
DEC-26 | Delivery | Early outside testing | Chosen: 3-5 outside testers on the next channel at the end of Phase 1; their bundles and accuracy issues feed DEC-19 and Phase 2 priorities | Recommended: same | Override: No | Reason: ~15 h/week puts v1.0 months away; real feedback first
DEC-27 | Product | Platform-timing response (SK-1) | Chosen: plugin-first, README and v0.1 lead with ground truth, shell side effects and risky outcomes; move analyze ahead of publish if Anthropic announces a native session-change summary or >=2 testers ask for PR/CI use | Recommended: A now, B on trigger | Override: No | Reason: fastest path to the hard-to-copy value; engine is already standalone
DEC-28 | Security | Auditability as a feature (SK-2) | Chosen: I-16 network-module ban, I-17 size budgets (capture.sh <=40 lines, engine <=~5k lines), doctor --audit, docs/what-we-capture.md; external assurance on buyer demand | Recommended: B | Override: No | Reason: every trust claim backed by a test or a command
DEC-29 | Compliance | Developer-owned by design (SK-5) | Chosen: first-run notice (again on managed installs), team rollout notice template + DPIA prompts in docs/teams.md, "self-review, not surveillance" positioning; legal review before v1.0 | Recommended: B | Override: No | Reason: turns monitoring risk into the P3 selling point
ASM-1 | Scope | v1 is single-user and local; team mode is a later phase | Impact if wrong: server, auth and tenant isolation move into v1 scope
ASM-2 | Compliance | Not built for a specific regulated industry | Impact if wrong: certified retention, tamper evidence and data residency become requirements
ASM-3 | Architecture | Most target users work inside git repos | Impact if wrong: DEC-1 completeness guarantee rarely applies; log-only mode dominates
ASM-4 | Platform | Hook payload fields, SessionEnd availability/timeout and per-hook output semantics are as described (partly confirmed 2026-10-09) | Impact if wrong: capture lifecycle and output contract need rework | NOTE (hardening): spike also covers tool_use_id, PostToolUse on failure, managed-install detection
ASM-5 | Runtime | python3 is present on most target machines | Impact if wrong: many silent no-reports; engine moves to Go | NOTE (hardening): macOS /usr/bin shims require the xcode-select guard
ASM-6 | Runtime | Language mismatch (Flask vs Laravel/TypeScript) resolved in favour of Python | Impact if wrong: TypeScript engine adds a build step
ASM-7 | Sharing | claude.ai Artifacts cannot be published from Claude Code | Impact if wrong: Artifact route becomes an available publish target
ASM-8 | Interface | Claude Code sets a detectable session env var; slash-command inline output enters model context | Impact if wrong: publish detection relies on /dev/tty alone; DEC-15 split may simplify | NOTE (hardening): env-var detection is bypassable; acceptable because DEC-15 is now an accident guard
ASM-9 | Deployment | Public plugin repo, so GitHub Actions minutes are free | Impact if wrong: macOS CI minutes consume the budget
ASM-10 | Testing | Python floor follows what current macOS developer tools ship | Impact if wrong: floor is end-of-life or excludes Mac users | NOTE (hardening): 3.9 is end-of-life since Oct 2025; verify the version current Command Line Tools ship
ASM-11 | Compliance | Tool author is neither data controller nor processor for v1 (needs professional verification) | Impact if wrong: privacy notice, legal basis and possibly DPAs required even for v1
ASM-12 | Delivery | No hard deadline; ~15 focused hours/week assumed (hours not stated) | Impact if wrong: calendar estimates in §9 scale; phase order unchanged
ASM-13 | Billing | Maintainer has Claude Pro and a card that reliably pays recurring USD | Impact if wrong: DEC-22 reverts to a capped API key or no automated e2e
ASM-14 | Platform | PreToolUse and PostToolUse payloads carry a shared tool_use_id | Impact if wrong: fallback pairing by order + tool name + input hash, less exact for parallel calls
ASM-15 | Platform | git honours GIT_OBJECT_DIRECTORY + GIT_ALTERNATE_OBJECT_DIRECTORIES for add, write-tree, diff-tree, cat-file on the floor version | Impact if wrong: fall back to in-repo objects with refs (Mentor design) behind a footprint warning
ASM-16 | Security | Empty filter.<name>.clean/.process overrides with required=false disable the driver without errors | Impact if wrong: snapshot via hash-object --no-filters + update-index --index-info
```

---

## 9. Build Order & Implementation Roadmap

Effort is in **person-days for a solo developer working with Claude Code and Claude Web**. It includes testing and documentation, assumes about 5 focused hours per person-day, and should be treated as ±30% estimates.

**Calendar (ASM-12):** at about 15 focused hours a week, one person-day is about one-third of a week. There is no hard deadline, so the calendar is a planning aid, not a commitment.

| Phase | Effort | Calendar at ~15 h/week | Release |
|---|---|---|---|
| 0 Foundations | ≈ 7.5 person-days | ≈ 2.5 weeks | — |
| 1 MVP core | ≈ 25 person-days | ≈ 8.5 weeks | v0.1.0 … v0.3.0 on `next` + 3–5 outside testers (DEC-26) |
| 2 Production hardening | ≈ 18.5 person-days | ≈ 6 weeks | v1.0.0 on `stable` |
| **Total to v1.0** | **≈ 51 person-days** | **≈ 17 weeks (~4 months)** | |
| 3 Growth | Demand-driven | — | v1.1+ |

```mermaid
flowchart LR
    P0["Phase 0 Foundations ~7.5 days"] --> P1["Phase 1 MVP core ~25 days, v0.1 to v0.3"]
    P1 --> T["Outside testers on next (DEC-26)"]
    T --> P2["Phase 2 Production hardening ~18.5 days, v1.0.0"]
    P2 --> P3["Phase 3 Growth, demand-driven, v1.1+"]
    P2 -.->|"DEC-27 trigger"| AN["analyze moves ahead of publish"]
```

### Phase 0: Foundations (≈ 7.5 person-days)

**Goal:** a skeleton that installs, captures raw payloads, passes CI, and a written record of verified platform facts.

| # | Deliverable | Effort |
|---|---|---|
| 0.1 | Public repo with Apache-2.0 `LICENSE`, `NOTICE`, `README` stub, `SECURITY.md` stub, `CONTRIBUTING.md` (DCO), `CHANGELOG.md`; branch protection (no force-push, required checks); passkey 2FA; private vulnerability reporting on | 0.5 |
| 0.2 | SSH signing set up; `allowed_signers` committed; signed commits and tags verified locally | 0.5 |
| 0.3 | **Platform verification spike** → `docs/platform-facts.md`, recording the Claude Code version and date. See the checklist below this table. | 2.5 |
| 0.4 | Plugin skeleton: `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` (`next` entry), `hooks/hooks.json` with the new matchers, final `capture.sh` (≤ 40 lines), stub `lifecycle.sh` → `python3 -m engine hook <event>` (logs only) | 0.5 |
| 0.5 | Engine skeleton: `engine/__main__.py`, `cli.py` with `version`; `io/paths.py`, `io/fsutil.py`, `io/gitops.py` (`run_git` with filter neutralisation), `io/objstore.py` (private object dir + index seeding), `io/runner.py` (`run_program`), `io/binaries.py` (with `xcode-select` guard), `render/safe.py` (`Untrusted`, `SafeText`, `untrusted()` v0); `pyproject.toml` (tool config only), `requirements-dev.txt` (hashed) | 1.5 |
| 0.6 | CI: `ci.yml` (lint, types, shellcheck, test matrix incl. Python 3.9, Windows-disable placeholder, gitleaks, security tests incl. I-16 and size budgets), Actions pinned by SHA, `permissions: contents: read`; Dependabot | 1.5 |
| 0.7 | Capture real payload fixtures (redacted, reviewed) → `tests/fixtures/payloads/cc-<version>/` | 0.5 |

**Phase 0 spike checklist (item 0.3).** Each question is resolved or recorded as "not possible", with its design consequence:
- **Hook payloads:** field names per event, including `tool_use_id` for Pre/Post pairing (ASM-4, ASM-14).
- **Failed calls:** whether `PostToolUse` fires on failed or interrupted calls.
- **`SessionEnd`:** availability to plugins, its timeout, and whether a detached child survives Claude Code exit (OQ-3).
- **Output channels:** which one is user-visible but not model-visible, per event (OQ-2).
- **Paths and env:** `CLAUDE_PROJECT_DIR`/`CLAUDE_PLUGIN_ROOT` behaviour and plugin folder layout across versions (OQ-5).
- **Marketplace:** ref pinning and update behaviour (OQ-4).
- **Commands and env markers:** slash-command inline execution; session environment variables (ASM-8).
- **Managed installs:** whether a managed-settings install is detectable (OQ-15).
- **Rules and local dev:** deny-rule syntax (OQ-6); local plugin loading for development; `claude -p` hook behaviour under a Pro login.
- **git:** `GIT_OBJECT_DIRECTORY` + alternates on the floor version (ASM-15); empty filter-driver overrides (ASM-16).
- **macOS:** the Python version current Command Line Tools ship (ASM-10).

**"Auth scaffold" equivalent for this product:** the trust-boundary code is created here. That means:
- `io/config.py` with the stricter-only merge and freeze stub;
- `io/paths.py` containment checks;
- `run_git()`/`run_program()` hardening, including filter-driver neutralisation;
- the private object store, which guarantees zero footprint.

Each comes with its first security tests.

**Dependencies:** none.

**Acceptance criteria:**
- CI green on Linux and macOS for Python 3.9 and 3.14.
- In a real Claude Code session with the dev plugin loaded, every prompt, Read/Bash intent and captured tool result produces exactly one inbox file, with no visible output and no noticeable slowdown.
- `docs/platform-facts.md` answers every spike question.
- Security tests for I-1 (static), I-2 (zero footprint on a baseline-only run), I-4, I-5, I-13, I-16, I-17 and I-18 exist and pass.

### Phase 1: MVP Core (≈ 25 person-days) → v0.1.0 … v0.3.0 on `next`

**Goal:** a useful, safe, honest report at the end of every session on Linux and macOS, in front of outside testers.

| # | Deliverable | Effort | Depends on |
|---|---|---|---|
| 1.1 | `SessionStart`: platform detection and disable; binary resolution and cache (with `xcode-select` guard); config load, merge and freeze; `disabled_roots`; manifest; **baseline via seeded temp index into the private object store**; resume segments | 3.5 | 0.x |
| 1.2 | Event collection and normalisation: inbox scan, ID validation, adapter `cc_v1` with shape validation, **Pre/Post pairing**, `max_inbox_mb`, size caps, flags, lock file | 2.5 | 0.7 |
| 1.3 | Diff and reconciliation (§11.2), including ignored-file writes, outside-repo writes, NotebookEdit/MCP writes, submodule pointer changes | 2 | 1.1, 1.2 |
| 1.4 | Scope engine v1 (§11.1) with `rules/scope.json`; S2 definition extraction in `io/` | 3 | 1.3 |
| 1.5 | Risk engine v1 (§11.3) with `shlex` parsing, wrapper recursion, outcome cross-check (including `attempted_no_completion`), deny suggestions | 3 | 1.3 |
| 1.6 | Redaction v1 (§11.4), sensitive paths, HMAC hashing | 2 | 0.5 |
| 1.7 | Renderer: `untrusted()` markdown and terminal modes, report writer (header first), summary line, template-based plain-English summary, "no agent action recorded" label | 2 | 1.4–1.6 |
| 1.8 | `SessionEnd` orchestration: **3 s synchronous budget, detached worker**, report (external by default), record, delete raw, `index.json`, retention sweep, orphan sweep | 2 | 1.7 |
| 1.9 | CLI: `report` (`--latest`/`--session`/`--list`/`--format`), `verify` (`--latest`), `config show`, `doctor` (basic + `--install` launcher), `version`; exit codes; JSON schemas `report-v1`, `cli-v1` | 1.5 | 1.8 |
| 1.10 | `/session-report` command (counts and path only); optional `Stop` tally (off by default, critical-count only) | 0.5 | 1.9 |
| 1.11 | README leading with ground truth, shell side effects and risky outcomes (DEC-27); install; what it does and doesn't do; size-budget numbers; first draft of `docs/data-processing.md` | 0.5 | — |
| 1.12 | Windows-disable CI job finalised; benchmark job (M2) incl. 20k-file index-seeding benchmark | 0.5 | 1.8 |
| 1.13 | Concurrent-writer detection and `shared_worktree` flag (DEC-25) | 1 | 1.1, 1.3 |
| 1.14 | `doctor --audit`, `docs/what-we-capture.md`, first-run notice (DEC-28, DEC-29) | 1 | 1.9 |
| 1.15 | Tester onboarding: `next` install guide, short feedback form (issue template), bundle instructions (DEC-26) | 0.5 | 1.9 |

**Acceptance criteria (v0.1.0):**
- Reference scenarios produce correct reports. Each scenario has an integration test:
  - (a) a scoped bug fix gives 0 unexplained;
  - (b) an unrelated config edit gives 1 unexplained with "no signal";
  - (c) `yarn codegen` / `npm run codegen` changes show as "plausibly caused by";
  - (d) a Write to gitignored `.env` appears as `ignored_file_written` with the sensitive-path flag and no content hash;
  - (e) `git push --force` shows critical/attempted with a deny suggestion;
  - (f) `rm -rf ../other` shows critical;
  - (g) an interrupted `rm -rf` shows `attempted_no_completion`;
  - (h) a file edited by "the human" mid-session shows the "no agent action recorded" label;
  - (i) two overlapping sessions both show `shared_worktree`.
- **Zero footprint:** the user repo and `.git` are byte-identical before and after a session with default config (I-2).
- 100% of seeded fixture secrets are redacted in all outputs.
- Malicious `.git/config` canary tests pass, including filter drivers (I-4, I-18).
- `SessionStart`/`UserPromptSubmit` produce empty stdout across 1,000 fuzzed payloads (I-1).
- Capture p95 ≤ 10 ms; report ≤ 5 s for 500 events (M2); baseline on a 20k-file warm repo ≤ 2 s.
- You have used it daily for at least 5 working days on `next`.
- **3–5 outside testers** have installed from `next` and produced at least one report each (DEC-26).

### Phase 2: Production Hardening (≈ 18.5 person-days) → v1.0.0 on `stable`

**Goal:** tamper-aware, measurable, supportable, honest, ready for strangers. Priorities inside this phase are reordered by tester feedback.

| # | Deliverable | Effort | Depends on |
|---|---|---|---|
| 2.1 | Resource budgets with partial-first rendering and `status: partial` (DEC-13) | 1.5 | Phase 1 |
| 2.2 | Tamper detection: target matching with path canonicalisation, manifest backstop, `self_development`, launcher integrity in `doctor`, `publish`-from-session rule, pty-wrapper and direct-`gh` rules (§11.3) | 2 | Phase 1 |
| 2.3 | Lookalike-character detection; ignored-project-settings notes | 0.5 | 1.7 |
| 2.4 | Migration framework, `state_version`, backups, `doctor --reset-state`; **payload-shape warning UX** (DEC-20) | 2 | Phase 1 |
| 2.5 | Evaluation: labelling guide, public synthetic corpus (≥ 40 cases incl. drift and concurrency cases), private corpus (≥ 20 of your sessions + consenting tester sessions, encrypted), `engine eval` with tier-change output, baseline, CI gate; tune `scope.json` | 4 | 1.4 |
| 2.6 | `hypothesis` fuzzing for renderer, parser, redaction, config merge, header reader | 1.5 | Phase 1 |
| 2.7 | Support kit: `doctor --bundle`, GitHub issue forms (bug, accuracy), `SECURITY.md` final with response window, Discussions, `docs/known-issues.md` | 1.5 | 1.9 |
| 2.8 | Compliance and positioning docs: `docs/guarantees.md` (CI-generated accuracy block; honest DEC-15 and DEC-25 limits), `docs/data-processing.md` final, `docs/teams.md` with rollout notice template and DPIA prompts (DEC-29), limits line in report header; `purge` and `feedback` commands | 2.5 | 2.5 |
| 2.9 | Release machinery: `release.yml` (tag signature verification), `stable`/`next` marketplace entries (or protected-branch fallback), `docs/release-checklist.md`, `docs/e2e-checklist.md`, first manual end-to-end run under Pro; `weekly.yml` | 1.5 | 0.6 |
| 2.10 | Tester-feedback triage buffer (DEC-26) | 1 | Phase 1 |
| 2.11 | Hardening review: walk the §4.5 STRIDE tables against the code, close gaps, update `docs/guarantees.md` | 0.5 | all |

**Acceptance criteria (v1.0.0):**
- All invariant tests I-1 … I-19 pass on the full matrix.
- Eval on the combined corpus is at precision ≥ 0.80 and recall ≥ 0.70, *or* recalibrated targets are documented with reasons in `docs/guarantees.md`.
- Fuzzing: ≥ 10,000 examples per target in the weekly run without failure.
- 10 consecutive days on `next` with no P1 bug.
- Manual end-to-end checklist passes on the latest Claude Code.
- A stranger who isn't one of the Phase 1 testers installs from the marketplace and gets a report by following only the README.
- Professional review of `docs/data-processing.md` and `docs/teams.md` done (OQ-12).
- v1.0.0 tag signed; marketplace `stable` pinned.

### Phase 3: Growth Features (demand-driven, ≈ 20+ person-days)

Build in this order, unless a trigger says otherwise. **DEC-27:** if Anthropic announces a native session-change summary, or two or more testers ask for PR/CI use, item 3.3 (`analyze`) moves ahead of 3.1.

| # | Deliverable | Effort | Trigger / dependency | Acceptance criteria |
|---|---|---|---|---|
| 3.1 | **v1.1 `publish`** via `gh` (gist / PR comment), visibility check, re-redaction, prompt stripping, `/dev/tty` accident guard, ledger, size handling | 3 | v1.0.0; OQ-9 | Journey 3 integration tests pass with a fake `gh`; manual publish to a private test repo; no-TTY and in-session refusals proven; bypass patterns detected |
| 3.2 | Resolve opaque `yarn`/`npm`/`pnpm run`, `composer run`, `make` targets by parsing scripts (records which version of the file was used) | 2 | v1.0.0 | Codegen scenarios move from "opaque" to rule-matched where applicable |
| 3.3 | **`analyze` diff-only mode** + composite GitHub Action (`action.yml`) + docs for `pull_request` (never `pull_request_target`) | 4 | DEC-27 trigger or team interest | Runs on fork PRs with read-only token; JSON validates; partial fails CI unless `--allow-partial` |
| 3.4 | Experimental opt-in "inform Claude" mode (numbers only) | 1.5 | User requests; data from 3.3 | Only integers reach context; off by default; documented risks |
| 3.5 | `--deep` advisory LLM pass (§11.9) | 4 | Eval shows vague-prompt weakness | Never changes deterministic tiers; per-session cost cap enforced; injection test suite passes |
| 3.6 | Native Windows support | 6–8 | Windows demand (OQ-13) | All invariants pass on `windows-latest`, including ACL permissions and console-based confirmation |
| 3.7 | Sigstore/`gitsign` provenance + build attestations | 1.5 | Team or compliance buyer (DEC-28) | Verification documented and tested |
| 3.8 | Official plugin directory listing | 0.5 | Directory accepts submissions | Listed |
| 3.9 | **Team service** (separate project plan; §11.8) | 30–40 | ≈ 10+ teams request central reports (DEC-9) | New mentor session for its tenancy, auth, compliance and monitoring design (DEC-29) before building |

---

## 10. Directory & File Structure

### 10.1 Repository strategy

**Monorepo for the product, plus one separate private repo for sensitive evaluation data.**

- `session-trust` (public): plugin, engine, tests, public corpus, docs, CI. One release unit means one version number set and one signed tag. Hooks and engine change together (the hook contract), so separating them would only create version skew.
- `session-trust-corpus-private` (private, `age`-encrypted): your real sessions for evaluation (DEC-19). Kept separate because its access control and risk profile differ completely.
- The future team service would be a **separate repo** (possibly under a different licence), importing the engine as a pinned dependency.

### 10.2 Directory tree

```text
session-trust/
├── .claude-plugin/
│   ├── plugin.json                 # plugin manifest: name, version, description, known-good Claude Code version for docs only; compatibility is judged by payload shape (DEC-20)
│   └── marketplace.json            # marketplace entries "session-trust" (stable, pinned tag) and "session-trust-next" (rc tag)
├── hooks/
│   ├── hooks.json                  # hook declarations + matchers + explicit timeouts (§2.4)
│   ├── capture.sh                  # HOT PATH: raw payload → inbox file. POSIX sh, ≤ 40 lines (I-17), absolute paths, exit 0 always
│   └── lifecycle.sh                # SessionStart/Stop/SessionEnd → frozen python3 -m engine hook <event>, exit 0 always
├── commands/
│   └── session-report.md           # /session-report: counts + path only, labelled "relayed by Claude"
├── bin/
│   └── session-trust               # launcher template installed to ~/.local/bin by `doctor --install`
├── engine/
│   ├── __init__.py
│   ├── __main__.py                 # `python3 -m engine` → cli.main()
│   ├── version.py                  # ENGINE_VERSION, SCHEMA versions, CLI_SCHEMA_VERSION, STATE_VERSION
│   ├── cli.py                      # argparse; maps exit codes; never returns 2
│   ├── hooks_entry.py              # start/stop/end orchestration; catches everything; always returns 0
│   ├── core/                       # PURE logic: no filesystem, subprocess or network access
│   │   ├── model.py                # dataclasses + NewTypes (Untrusted, CanonicalPath, SessionId)
│   │   ├── reconcile.py            # diff ↔ events attribution (§11.2)
│   │   ├── scope.py                # signals, scoring, tiers (§11.1)
│   │   ├── prompts.py              # path/identifier extraction from prompts, stem normalisation
│   │   ├── commands.py             # shlex parsing, wrapper recursion, rule evaluation (§11.3)
│   │   ├── tamper.py               # self-target matching
│   │   ├── redact.py               # pattern + entropy + allowlist redaction (§11.4)
│   │   ├── lookalike.py            # mixed-script path detection
│   │   ├── budgets.py              # budget accounting objects
│   │   └── summary.py              # template-based plain-English summary (no free text from data)
│   ├── io/                         # ALL side effects live here
│   │   ├── paths.py                # state/config roots, containment checks, ID validation
│   │   ├── fsutil.py               # atomic writes, 0600/0700 enforcement, locks
│   │   ├── gitops.py               # run_git(): the ONLY git invocation point; filter-driver neutralisation (§11.6)
│   │   ├── objstore.py             # private per-session object store + index seeding (GIT_OBJECT_DIRECTORY + alternates)
│   │   ├── worker.py               # SessionEnd budget + fork/setsid detached worker (stdio closed)
│   │   ├── concurrency.py          # live-session detection on the same repo (DEC-25)
│   │   ├── runner.py               # run_program(): the ONLY other subprocess point (gh, claude --version)
│   │   ├── binaries.py             # clean-path resolution + cache; macOS xcode-select -p guard (I-19)
│   │   ├── config.py               # load, validate, stricter-only merge, freeze
│   │   ├── state.py                # state.json, migrations driver, index.json
│   │   ├── inbox.py                # collection (rename-based), orphan sweep
│   │   ├── retention.py            # minimise + delete, retention sweep, purge
│   │   ├── errors_log.py           # redacted rotating error log
│   │   ├── platform.py             # Windows-variant / WSL / macOS detection
│   │   ├── audit.py                # doctor --audit data (what is captured, where, bytes, ages) (DEC-28)
│   │   ├── notice.py               # first-run / managed-install notice state (DEC-29)
│   │   └── tty.py                  # /dev/tty confirmation; session detection
│   ├── adapters/
│   │   ├── __init__.py             # select adapter by Claude Code version / payload shape
│   │   └── cc_v1.py                # raw payload → st.event/1
│   ├── render/
│   │   ├── safe.py                 # untrusted() → SafeText (markdown + terminal modes) (§11.5)
│   │   ├── header.py               # JSON header builder/reader (caps, schema check)
│   │   ├── markdown.py             # report body; accepts SafeText only
│   │   └── terminal.py             # summary lines; accepts SafeText only
│   ├── migrations/
│   │   └── __init__.py             # ordered registry; v1_to_v2.py etc. added as needed
│   ├── publish/                    # v1.1: gh integration, ledger, preview
│   ├── analyze/                    # Phase 3: diff-only mode
│   ├── deep/                       # Phase 3: advisory LLM pass
│   ├── evaluation/
│   │   └── run.py                  # `engine eval` (precision/recall vs baseline)
│   └── rules/                      # versioned data files (each has "version" field)
│       ├── secrets.json
│       ├── sensitive_paths.json
│       ├── commands.json
│       ├── git_hardening.json      # -c overrides list for run_git()
│       └── scope.json              # signal weights + thresholds
├── schemas/                        # JSON Schemas (draft 2020-12) for public contracts
│   ├── report-v1.json
│   ├── cli-v1.json
│   ├── config-v1.json
│   └── record-v1.json
├── tests/
│   ├── conftest.py                 # temp state roots, env isolation, hypothesis profiles
│   ├── golden.py                   # golden-file helper (no dependency)
│   ├── unit/                       # mirrors engine/core and engine/render
│   ├── integration/                # real git, real capture.sh under dash, full lifecycle
│   ├── security/                   # invariants I-1..I-19 (release blockers)
│   ├── fuzz/                       # hypothesis suites
│   ├── windows/
│   │   └── test_disable.sh         # Windows CI job
│   ├── golden/                     # expected JSON records (+ a few rendered .md)
│   └── fixtures/
│       ├── repo_builder.py         # declarative temp-repo builder
│       ├── session_builder.py      # scenario → normalised events
│       ├── sessions/*.json         # scenario specs
│       ├── malicious/              # hostile git configs, filenames, payloads
│       └── payloads/cc-<version>/  # redacted real payloads per Claude Code version
├── eval/
│   ├── LABELLING.md                # tier labelling rules
│   ├── corpus/<case-id>/           # events.jsonl, repo_spec.json, labels.json (public, synthetic)
│   └── baseline.json               # stored precision/recall
├── bench/
│   └── run_bench.py                # M2 benchmarks
├── docs/
│   ├── index.md                    # GitHub Pages home
│   ├── guarantees.md               # guarantees + limits; accuracy block generated in CI; DEC-15/DEC-25 honest limits
│   ├── what-we-capture.md          # one redacted raw event beside the record it shrinks into (DEC-28)
│   ├── data-processing.md          # what data, where, how long; nothing sent
│   ├── teams.md                    # team rollout: developer notice template, DPIA prompts (not legal advice) (DEC-29)
│   ├── testers.md                  # how to join the `next` channel and send feedback (DEC-26)
│   ├── versioning.md               # the version numbers explained
│   ├── platform-facts.md           # verified Claude Code facts per version (Phase 0)
│   ├── known-issues.md             # by Claude Code version
│   ├── release-checklist.md
│   ├── e2e-checklist.md
│   ├── metrics.md                  # M1–M3 per release
│   └── incidents/                  # post-incident notes
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── release.yml             # tag-triggered; verifies signature; creates Release
│   │   └── weekly.yml              # scheduled drift + long fuzz
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug.yml                 # requires doctor --bundle + review checkbox
│   │   ├── accuracy.yml            # requires feedback --export
│   │   └── config.yml              # links to Discussions + SECURITY.md
│   └── dependabot.yml
├── allowed_signers                 # every release-signing key ever used, with date ranges
├── Dockerfile.dev                  # reproducible test env + PoC sandbox base
├── tools/
│   └── check_size_budgets.py       # I-17: capture.sh and engine line budgets (CI gate)
├── Makefile                        # make test | lint | eval | bench | budgets | docker-test
├── pyproject.toml                  # tool config only (ruff, mypy, pytest, coverage). NO runtime dependencies
├── requirements-dev.txt            # pinned, hashed dev dependencies
├── CHANGELOG.md
├── CONTRIBUTING.md                 # DCO sign-off, test expectations, module rules
├── SECURITY.md                     # private reporting, response window, scope = guarantees.md
├── LICENSE                         # Apache-2.0
├── NOTICE
└── README.md
```

### 10.3 Naming conventions

| Item | Convention | Example |
|---|---|---|
| Python modules and functions | `snake_case` | `run_git`, `reconcile.py` |
| Classes / dataclasses | `PascalCase` | `NormalisedEvent`, `FileFinding` |
| Rule IDs | `<domain>.<rule>` | `git.force_push`, `net.download_exec` |
| Scope reason codes | `snake_case[:arg]` | `mentioned_in_prompt:2`, `explored_early` |
| Error codes | `ST_E_<UPPER>` | `ST_E_TREE_MISSING` |
| Schema IDs | `st.<entity>/<major>` | `st.record/1` |
| Report files | `session-<YYYYMMDDTHHMMSSZ>-<first 8 of sid>.md` | `session-20261009T102944Z-8f1c2a9b.md` |
| Private object stores | `<state>/projects/<pid>/sessions/<sid>/objects/` (no refs anywhere) | — |
| Default report location | `<state>/projects/<pid>/reports/session-<UTC>-<sid8>.md` | — |
| Tags | `vX.Y.Z`, `vX.Y.Z-rc.N` | `v1.0.0-rc.2` |
| Branches | `feat/…`, `fix/…`, `sec/…`, `docs/…` | `sec/git-hardening-textconv` |
| Commits | Conventional Commits + `Signed-off-by:` | `fix(render): escape ESC in terminal mode` |

### 10.4 Module boundary rules (what may import what)

| Module | May import | Must not import |
|---|---|---|
| `engine/core/*` | `core/*`, standard library pure modules (`re`, `shlex`, `json`, `dataclasses`, `typing`, `hashlib`, `hmac`, `unicodedata`) | `io/*`, `render/*`, `os`, `subprocess`, any file reads (S2 definitions are extracted in `io/` and passed in), network modules (I-16) |
| `engine/render/*` | `core/model`, `render/*` | `io/*` (renderers receive data, they don't fetch it); `subprocess` |
| `engine/io/*` | `core/*`, `render/safe` (for errors log), standard library except network modules (I-16) | `subprocess` **except** `io/gitops.py` and `io/runner.py`; `os.fork`/`os.setsid` **except** `io/worker.py` |
| `engine/adapters/*` | `core/model` | Everything else |
| `engine/hooks_entry.py`, `engine/cli.py` | Anything in `engine/` | — (orchestration layer) |
| `engine/publish/*`, `engine/analyze/*`, `engine/deep/*` | `core`, `io`, `render` | Each other |
| Anything in `engine/` | Standard library only | Third-party packages (I-13) |
| `tests/*` | Anything | — |

Enforcement: a custom AST test (`tests/security/test_import_rules.py`) walks every module and fails on violations. ruff `banned-api` covers `subprocess` and `os.system`.

---

## 11. Additional Technical Specifications

**Applicable areas:**
- the scope engine and reconciliation (including concurrent writers);
- risk rules;
- redaction and rendering;
- hardened git execution and private snapshots;
- the detached `SessionEnd` worker;
- platform detection;
- the future team-service sketch;
- AI cost controls for `--deep`.

Mobile and real-time sections are **not applicable**. For emerging markets: the tool is offline-first by nature (it works with no connectivity and costs users nothing), handles no money, and needs no payment rails. The maintainer's foreign-currency billing is resolved (ASM-13, DEC-22).

### 11.1 Scope classification algorithm (DEC-4)

**Inputs:** user prompts (only `UserPromptSubmit`; subagent prompts excluded), `Read` intents (`PreToolUse(Read)`, DEC-1), edit events, reconciled change list.

**Prompt extraction (`core/prompts.py`):**
- Path-like tokens matching `(?:[\w.-]+/)*[\w.-]+\.[A-Za-z0-9]{1,8}`, plus anything inside backticks.
- Identifiers: PascalCase `\b[A-Z][a-z0-9]+(?:[A-Z][a-z0-9]+)+\b`, snake_case `\b[a-z][a-z0-9]*(?:_[a-z0-9]+)+\b`, kebab-case `\b[a-z][a-z0-9]*(?:-[a-z0-9]+)+\b`.
- Free words ≥ 3 characters, lowercased, minus a stopword list (`rules/scope.json`).
- **Normalisation:** split stems on case changes, `_`, `-`, `.`; lowercase; drop tokens shorter than 3 characters. "the user controller", `UserController` and `user_controller.py` all normalise to `[user, controller]`.

**Signals per changed file:**

| Signal | Reason code | Weight (initial) | Definition |
|---|---|---|---|
| S1 | `mentioned_in_prompt:n` | 1.00 | Prompt n contains the file's path suffix, its basename, or its full normalised stem sequence (contiguous) |
| S2 | `identifier_match:n` | 0.85 | A prompt identifier normalises to the file stem tokens, or appears as a class/function name in the file's first 64 KiB (definition line only) |
| S3 | `explored_early` | 0.60 | File had a `Read` intent before the session's first Write/Edit/MultiEdit/NotebookEdit event (`PreToolUse(Read)` intents, DEC-1; disabled automatically if Phase 0 finds Read intents unavailable) |
| S4 | `test_counterpart` | 0.60 | File is a test of an in-scope file: `tests/**/<Stem>Test.php`, `**/<stem>.test.ts`, `**/<stem>.spec.ts`, `**/test_<stem>.py`, `**/<stem>_test.go` |
| S5 | `manifest_lock` | 0.70 | Lockfile changed alongside its manifest (`package.json` ↔ `yarn.lock`/`package-lock.json`/`pnpm-lock.yaml`; `composer.json` ↔ `composer.lock`; `pyproject.toml` ↔ `poetry.lock`/`uv.lock`) |
| S6 | `generated_by_command:seq` | 0.50 | Reconciliation attributes the change to a Bash command whose program is in the generator list (§11.2) |
| S7 | `sibling` | 0.40 | Same directory as an in-scope file **and** (directory has ≤ 15 entries **or** the stems share a ≥ 4-character prefix token) |
| S8 | `same_feature_dir` | 0.30 | Shares the deepest common directory (depth ≥ 2) with ≥ 2 in-scope files |

**Score:** `score = max(weights) + 0.05 × (number of additional distinct signals)`, capped at 1.0. Max-based scoring keeps reasons explainable; the small bonus rewards corroboration.

**Tiers:** `in_scope` if score ≥ 0.80; `likely_related` if ≥ 0.40; otherwise `unexplained` with reason `no_signal`. Weights and thresholds live in `rules/scope.json` (versioned) and are tuned only via the eval loop (DEC-19).

**Overrides, applied after scoring:**
- `sensitive_path` (`.env*`, `*.pem`, `*.key`, `id_*`, `credentials*`, `*.p12`, `*.keystore`, `secrets.*`) not mentioned by S1 → force `unexplained` + warn finding `fs.sensitive_file_changed`.
- `outside_repo_written` (Write/Edit target resolves outside the repo root) → force `unexplained` + warn finding.
- `ignored_file_written` without S1/S2 → `unexplained` with reason `ignored_file`.
- Paths in `self_development` → tamper findings downgraded to info (DEC-8).

**Two-pass classification:** pass 1 computes S1, S2, S3, S5 and S6. The files that reach in-scope in pass 1 then seed S4, S7 and S8 in pass 2. There is no further iteration, which avoids runaway expansion.

### 11.2 Reconciliation and attribution (DEC-1)

For each changed path `P` from `diff-tree baseline final` (plus ignored and outside-repo writes from the event log):

1. **Edit attribution:** if any Write/Edit/MultiEdit event's canonical `file_path` equals `P` → `{"type": "edit", "event_seqs": [...]}`.
2. **Bash attribution** (candidates are Bash events, latest first):
   - **Token match:** `P`, its basename, or its parent directory appears among the parsed argument tokens (resolved against the tracked `cwd` across `cd` within the same command), **or**
   - **Generator match:** the program/subcommand is in the generator list and `P` lies under the command's effective `cwd`. Generator list: `npm|yarn|pnpm run|exec|dlx`, `npx`, `composer`, `php artisan`, `make`, `sed -i`, `perl -pi`, `git checkout|restore|apply|am|merge|rebase|stash|mv|rm`, `prettier --write`, `eslint --fix`, `black`, `ruff --fix|format`, `gofmt -w`, `cp`, `mv`, `rm`, `touch`, `tee`, `patch`, `tar -x`, `unzip`, codegen tools listed in `rules/commands.json`.
   - The first match → `{"type": "bash_plausible", "event_seqs": [seq]}`, rendered as "plausibly caused by `<command>`".
3. **Unattributed** → `{"type": "none"}`, rendered as **"no agent action recorded: may be you, your editor, or another session"**. It feeds the "unexplained" headline together with the scope tier (DEC-25). Bash intents with `attempted_no_completion` still count as attribution candidates (a killed command may have partially run).
4. **Edited-then-reverted:** edit events whose target shows no change in the diff → info line "edited, net unchanged".
5. **Deletions and renames:** reported from `diff-tree -M50%` status codes; renames carry `old_path`.
6. **No git (ASM-3 failure case):** log-only mode. Files come solely from edit events; header `status` notes `no_git_baseline`; the completeness guarantee is not claimed.
7. **Other writers (DEC-25):** NotebookEdit targets count as edit attribution. `mcp__*` events whose input names a path count as `bash_plausible`-strength attribution with type `mcp_plausible`. Submodule gitlink changes are reported as `submodule_pointer_changed` without inspecting submodule contents.
8. **Shared working tree (DEC-25):** if another session on the same `repo_root` overlapped in time, the header sets `shared_worktree: true`. Unattributed files keep the human/editor/another-session label; they are **never** attributed to the other session, because that session's events are not read.

### 11.3 Risk rule catalogue (DEC-10, v1)

The parser uses `shlex.shlex(posix=True, punctuation_chars=True)` to split on `|`, `||`, `&&`, `;`, `&`. Each segment's program is resolved after skipping environment assignments (`FOO=bar cmd`), `sudo`/`env`/`nohup`/`time` prefixes (recorded), and wrappers (`bash -c`, `sh -c`, `zsh -c`, which recurse up to depth 3). Unparseable input → `exec.unparseable` (info) + opaque.

| Rule ID | Matches | Severity | Outcome check |
|---|---|---|---|
| `git.force_push` | `git push` with `-f`, `--force`, `+<refspec>`; `--force-with-lease` → warn | critical | Remote not verifiable → `attempted` |
| `git.reset_hard` | `git reset --hard` | warn | Diff shows reverted files → `happened` |
| `git.clean` | `git clean` with `-f` (`-x` → critical) | warn | Deleted untracked files visible in baseline vs final |
| `git.history_rewrite` | `git rebase`, `git commit --amend`, `git filter-branch`, `git filter-repo` | warn | — |
| `git.config_write` | `git config` without `--get`/`--list`; critical if key ∈ {`core.fsmonitor`, `core.hooksPath`, `core.sshCommand`, `diff.*.textconv`, `diff.external`, `core.pager`, `credential.helper`, `include.path`} | warn/critical | — |
| `fs.rm_recursive_outside` | `rm -r`/`-R`/`-rf` (any order) where a target resolves outside the repo root or equals `~`, `/`, `..` | critical | Inside repo → `fs.rm_recursive` info |
| `fs.find_delete` | `find … -delete` or `-exec rm` | warn | Diff deletions |
| `fs.chmod_risky` | `chmod 777`, `o+w`, `a+w`, `-R` | warn | Mode changes in diff → `happened` |
| `fs.write_sensitive_home` | Redirection (`>`, `>>`) or `cp`/`mv`/`tee`/`ln` targets under `~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.config/gh`, `~/.netrc`, `~/.bashrc`, `~/.zshrc`, `~/.profile`, `~/.gitconfig` | critical | Not in repo diff → `attempted (outside repo)` |
| `fs.sensitive_file_changed` | (from scope overrides) | warn | — |
| `net.download_exec` | `curl`/`wget` piped into `sh`/`bash`/`zsh`/`python`/`node`/`php`; `bash <(curl …)`; `sh -c "$(curl …)"` | critical | — |
| `net.non_allowlisted_host` | `curl`, `wget`, `http`, `nc`, `ncat`, `scp`, `rsync`, `ssh`, `ftp` with a host not in `allow_hosts` (localhost/127.0.0.1/::1 always allowed) | warn | — |
| `net.upload` | `curl -T`/`--upload-file`/`-F`/`--data @file`/`-d @file`; `scp`/`rsync` with a remote destination | critical | — |
| `priv.escalation` | `sudo`, `doas`, `su`, `pkexec` | critical | — |
| `exec.eval` | `eval`; `base64 -d`/`--decode` piped to a shell; `xxd -r` piped to a shell | critical | — |
| `exec.opaque` | `python -c`, `node -e`, `php -r`, `ruby -e`, `perl -e`, `./<script>`, `bash <file>`, package-manager scripts, `make` | info | Linked to unexplained changes when present |
| `pkg.install` | `npm/yarn/pnpm add|install`, `pip install`, `composer require`, `go get`, `cargo add`; warn if source is a URL/git/tarball | info/warn | Lockfile change via diff |
| `env.secret_read` | `printenv`, `env` (bare), `cat`/`less`/`head` of sensitive paths, `echo $<NAME containing KEY|TOKEN|SECRET|PASSWORD>` | warn | — |
| `db.destructive` | SQL `DROP TABLE|DATABASE|SCHEMA`, `TRUNCATE`, `DELETE FROM` without `WHERE` (in `psql -c`, `mysql -e`, `sqlite3`); `php artisan migrate:fresh|migrate:reset|db:wipe`; `rails db:drop|db:reset`; `prisma migrate reset`; `dropdb` | critical | — |
| `proc.kill` | `kill -9`, `pkill`, `killall` | info | — |
| `tool.self_publish` | `session-trust publish` or `python3 -m engine publish` | critical | DEC-15 |
| `tool.pty_wrapper` | `script` (any flags) wrapping a command, `unbuffer`, `expect`, `python`/`python3 -c` code importing `pty`, `socat … pty` | critical if the wrapped command is `session-trust`/`engine`; warn otherwise | DEC-15 bypass detection |
| `tool.direct_publish` | `gh gist create`, `gh pr comment`, `gh issue comment`, `gh api` with `POST`/`--method POST` | critical (an agent sharing content externally via the user's credentials) | DEC-15 |
| `tool.tamper` | Any token or redirection resolving to the plugin root, state root (including private object stores and reports), user config, project config, or the launcher | critical | Manifest backstop (DEC-8) |
| `git.filter_config` | `git config` writing `filter.*` keys or edits to `.gitattributes` adding `filter=` | warn (critical when combined with a `filter.*.clean/process` command) | Neutralised by DEC-13 regardless; flagged for visibility |
| `exec.unparseable` | `shlex` failure, here-docs, `$( )`/backticks it can't resolve | info | Opaque |
| `exec.oversized_command` | Command > 16 KiB | info | Partial analysis |

**Deny-rule suggestions:** each critical or warn rule carries a template, e.g. `Bash(git push --force:*)`, `Bash(curl:*|*sh)`. *Verify the current Claude Code permission rule syntax; omit suggestions whose syntax cannot express the rule.*

### 11.4 Redaction rules (DEC-6, v1)

Applied to every output-bound string: report, record, `errors.log`, terminal summary, bundle, feedback export, publish preview. Every pattern must be linear-time (no nested quantifiers). User and project `extra_secret_patterns` are rejected if they fail a static check for nested quantifiers or backreferences.

| Type label | Pattern (Python `re`) |
|---|---|
| `aws-access-key` | `\b(?:AKIA|ASIA)[0-9A-Z]{16}\b` |
| `aws-secret` | `(?i)aws.{0,20}?(?:secret|key).{0,5}?[=:]\s*['"]?[A-Za-z0-9/+=]{40}` |
| `github-token` | `\bgh[pousr]_[A-Za-z0-9]{36,255}\b` |
| `github-pat` | `\bgithub_pat_[A-Za-z0-9_]{22,255}\b` |
| `anthropic-key` | `\bsk-ant-[A-Za-z0-9_-]{20,}\b` |
| `openai-key` | `\bsk-(?:proj-)?[A-Za-z0-9_-]{20,}\b` |
| `stripe-key` | `\b(?:sk|rk)_(?:live|test)_[A-Za-z0-9]{16,}\b` |
| `paystack-key` | `\b(?:sk|pk)_(?:live|test)_[0-9a-f]{40}\b` |
| `slack-token` | `\bxox[abprs]-[A-Za-z0-9-]{10,}\b` |
| `google-api-key` | `\bAIza[0-9A-Za-z_-]{35}\b` |
| `jwt` | `\beyJ[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\b` |
| `private-key` | `-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----` (applied with a 64 KiB window) |
| `auth-header` | `(?i)\bauthorization\s*:\s*\S+(?:\s+\S+)?` |
| `bearer` | `(?i)\bbearer\s+[A-Za-z0-9._~+/-]{8,}=*` |
| `url-credentials` | `(?<=://)[^/\s:@]{1,64}:[^/\s@]{1,128}(?=@)` |
| `assignment` | `(?i)\b(?:password|passwd|pwd|secret|token|api[_-]?key|access[_-]?key|client[_-]?secret)\s*[=:]\s*['"]?[^\s'"]{4,}` (value only redacted) |
| `connection-string` | `(?i)\b(?:postgres(?:ql)?|mysql|mongodb(?:\+srv)?|redis|amqp)://[^\s]+` (credentials portion only) |

**Entropy pass:**
- Tokens of length ≥ 24 from `[A-Za-z0-9+/=_-]` with Shannon entropy ≥ 4.0 bits/char are redacted as `«redacted:high-entropy»`.
- Allowlist (not redacted): 40/64-hex strings in git contexts (`commit`, `tree`, `sha`, after `git` commands); UUIDs (`8-4-4-4-12` hex); `sha256-`/`sha384-`/`sha512-` integrity strings; strings inside paths that match a tracked file path.

**Sensitive-path suppression:** content-derived values for files matching `rules/sensitive_paths.json` globs are never emitted, only `modified` and sizes.

**Replacement format:** `«redacted:<type>»`. The type stays visible (DEC-6).

### 11.5 `untrusted()` specification (DEC-11)

```python
SafeText = NewType("SafeText", str)
Untrusted = NewType("Untrusted", str)

def untrusted(value: Untrusted, *, mode: Literal["md_inline", "md_block", "terminal"],
              max_len: int = 512) -> SafeText: ...
```

Steps, in order:
1. Redact (§11.4).
2. Apply Unicode NFC normalisation for display only (raw values are kept separately for matching).
3. Replace C0/C1 controls (except `\n` in `md_block`), `\x7f`, bidi controls (U+202A–202E, U+2066–2069, U+200E/200F), and zero-width characters (U+200B–200D, U+2060, U+FEFF) with visible `\uXXXX`.
4. In `terminal` mode, additionally strip ESC sequences entirely before step 3 is applied to any remainder.
5. Truncate to `max_len` code points, adding `…(+N chars)`.
6. In `md_inline` mode, wrap in a backtick run one longer than the longest internal run, padding with spaces if the content starts or ends with a backtick. In `md_block` mode, wrap in a fence of backticks one longer than any internal run, with no info string.
7. Return `SafeText`.

Renderers accept only `SafeText`, enforced by `mypy --strict`. Report headings, table headers and the JSON header's free-text fields use constants only.

### 11.6 Hardened git execution and snapshots (DEC-1, DEC-13)

```python
# engine/io/gitops.py
from __future__ import annotations   # required on the 3.9 floor (DEC-2)

GIT_ENV = {
    "PATH": "/usr/bin:/bin:/usr/local/bin:/opt/homebrew/bin",
    "HOME": os.environ.get("HOME", ""),
    "LC_ALL": "C",
    "GIT_CONFIG_NOSYSTEM": "1",
    "GIT_TERMINAL_PROMPT": "0",
    "GIT_OPTIONAL_LOCKS": "0",
}
GIT_HARDENING = [  # loaded from rules/git_hardening.json (versioned)
    "-c", "core.fsmonitor=false",
    "-c", "core.pager=cat",
    "-c", "diff.external=",
    "-c", "core.hooksPath=/dev/null",
    "-c", "protocol.allow=never",
    "-c", "credential.helper=",
    "-c", "core.sshCommand=false",
    # core.untrackedCache is NOT disabled: it is index data, not execution, and speeds up add -A
]

def _filter_overrides(repo: CanonicalPath) -> list[str]:
    # Reading config is safe (no execution). Re-run on every snapshot, because the agent may add drivers mid-session.
    names = _config_filter_driver_names(repo)          # git config --name-only --get-regexp '^filter\.'
    out: list[str] = []
    for n in sorted(names):                            # driver names validated against ^[A-Za-z0-9._-]{1,64}$
        out += ["-c", f"filter.{n}.clean=", "-c", f"filter.{n}.smudge=",
                "-c", f"filter.{n}.process=", "-c", f"filter.{n}.required=false"]
    return out

def run_git(repo: CanonicalPath, *args: str, index_file: Path | None = None,
            object_dir: Path | None = None, timeout: float = 10.0) -> GitResult:
    env = dict(GIT_ENV)
    if index_file is not None:
        env["GIT_INDEX_FILE"] = str(index_file)
    if object_dir is not None:
        env["GIT_OBJECT_DIRECTORY"] = str(object_dir)
        env["GIT_ALTERNATE_OBJECT_DIRECTORIES"] = str(_common_dir(repo) / "objects")
    cmd = [FROZEN_BINARIES.git, "-C", str(repo), *GIT_HARDENING, *_filter_overrides(repo), *args]
    # diff/diff-tree calls add: --no-ext-diff --no-textconv
    return _run(cmd, env=env, timeout=timeout)  # shell=False, stdin=DEVNULL, capture output, size-capped
```

**Snapshot procedure (`engine/io/objstore.py`), used for both baseline and final trees:**
1. Resolve the real index with `git rev-parse --git-path index` and the common dir with `git rev-parse --git-common-dir`. This works for worktrees.
2. Copy the real index to `<state>/tmp/index-<sid>-<n>` (`shutil.copyfile`, then `chmod 0600`). If it doesn't exist (fresh repo), start empty and record `index_seeded_from: "empty"`. The real index is never modified.
3. Run `run_git(repo, "add", "-A", index_file=tmp, object_dir=session_objects)`:
   - the stat cache means only changed or untracked files are rehashed;
   - untracked files over 5 MB are pre-filtered via `git ls-files -o --exclude-standard -z` + `os.stat`, and excluded with pathspec magic `:(exclude,literal)<path>`;
   - the 10 s budget applies.
4. Run `run_git(repo, "write-tree", index_file=tmp, object_dir=session_objects)`. Its output is the tree hash.
5. Delete the temp index. Store the tree hash in the manifest or record.
6. Diff with `run_git(repo, "diff-tree", "-r", "-M50%", "--no-ext-diff", "--no-textconv", "--name-status", base, final, object_dir=session_objects)`.

**Notes:**
- Never pass `-c safe.directory=*`.
- `GIT_OPTIONAL_LOCKS=0` avoids taking the user's index lock during status-like operations.
- Because filters are neutralised, Git LFS files are hashed raw in both snapshots. The diff is consistent within a session; only paths and sizes are reported for them.
- **Fallback if ASM-16 fails** (an empty override doesn't disable a driver on some git version):
  1. hash changed and untracked files with `git hash-object --no-filters -w --stdin-paths` (object dir set);
  2. build the temp index with `git update-index --index-info`, starting from the copied index;
  3. then run `write-tree`.

  No filter or attribute is consulted on this path.
- **Fallback if ASM-15 fails** (alternates not honoured): revert to the Mentor design (objects in the repo + refs), with a visible footprint warning and `reports: external` retained.
- *Verify each `-c` key and both env variables against the minimum supported git version.*

### 11.7 Platform detection (DEC-18)

| Check | Where | Result |
|---|---|---|
| `MSYSTEM` or `WINDIR` env set | `capture.sh` (builtins) | Exit 0 immediately |
| `platform.system()` = `Windows`, or `uname -s` starts with `MINGW`/`MSYS`/`CYGWIN` | `SessionStart` (Python) | Disable, with notice "native Windows not yet supported; use WSL" |
| `/proc/sys/kernel/osrelease` contains `microsoft` | `SessionStart` | WSL: supported; if repo is under `/mnt/<drive>/`, `doctor` and the summary warn about permission bits and performance |
| `Darwin` | `SessionStart` | macOS: Application Support state root; case-insensitive path canonicalisation for tamper and scope matching |
| `Darwin` + `xcode-select -p` fails | `SessionStart` (`io/binaries.py`) | `/usr/bin/python3` and `/usr/bin/git` treated as absent (no install dialog can open, I-19); Homebrew / python.org / `uv` / pyenv locations and `interpreters` overrides still tried; if none found → visible "install developer tools or Python ≥ 3.9" notice and disable |

### 11.8 Future team service (sketch only, gated on DEC-9 trigger)

- **Stack:** Flask 3.x + gunicorn, PostgreSQL 16, Caddy 2 (TLS), Docker Compose on a small VPS (EU region recommended for first customers).
- **Auth:** GitHub App OAuth for humans; API tokens (§4.2) and GitHub OIDC for CI.

```sql
CREATE TABLE orgs        (id uuid PRIMARY KEY, name text NOT NULL, created_at timestamptz NOT NULL DEFAULT now());
CREATE TABLE users       (id uuid PRIMARY KEY, github_id bigint UNIQUE NOT NULL, login text NOT NULL);
CREATE TABLE memberships (org_id uuid REFERENCES orgs, user_id uuid REFERENCES users,
                          role text NOT NULL CHECK (role IN ('owner','admin','member','viewer')),
                          PRIMARY KEY (org_id, user_id));
CREATE TABLE api_tokens  (id uuid PRIMARY KEY, org_id uuid NOT NULL REFERENCES orgs, prefix text NOT NULL,
                          token_sha256 bytea NOT NULL UNIQUE, scopes text[] NOT NULL, expires_at timestamptz NOT NULL,
                          revoked_at timestamptz, last_used_at timestamptz);
CREATE TABLE reports     (id uuid PRIMARY KEY, org_id uuid NOT NULL REFERENCES orgs, repo text NOT NULL,
                          report_sha256 bytea NOT NULL, header jsonb NOT NULL, body_redacted text NOT NULL,
                          schema_version text NOT NULL, generated_at timestamptz NOT NULL,
                          received_at timestamptz NOT NULL DEFAULT now(), UNIQUE (org_id, report_sha256));
CREATE INDEX reports_org_generated ON reports (org_id, generated_at DESC);
CREATE INDEX reports_org_repo ON reports (org_id, repo, generated_at DESC);
ALTER TABLE reports ENABLE ROW LEVEL SECURITY;
CREATE POLICY reports_isolation ON reports USING (org_id = current_setting('app.org_id')::uuid);
-- same pattern for api_tokens and memberships; app sets `SET LOCAL app.org_id = ...` per request
```

- **Ingestion:** `POST /v1/reports`, `Idempotency-Key` = report SHA-256; server re-validates the header and re-redacts. Append-only (no UPDATE grants), which is the basis for the evidence-grade roadmap (DEC-24).
- **This sketch requires its own mentor session** (tenancy, compliance, DPA, residency) before building.

### 11.9 AI/ML: `--deep` advisory pass (Phase 3)

| Concern | Specification |
|---|---|
| Model | Smallest current Claude model via the user's own API key (`ANTHROPIC_API_KEY` from env or keychain). *Check current model IDs and pricing at docs.claude.com before implementing.* |
| Input | Only: prompt excerpts (redacted, ≤ 120 characters each), file paths, deterministic tiers and reasons, diff stats. **No file contents by default.** `--deep-with-diffs` opt-in sends redacted hunks capped at 20 KiB. |
| Output | Strict JSON `{files: [{path, suggested_tier, rationale}]}` validated against a schema; invalid output is discarded |
| Effect | **Never changes deterministic tiers.** Shown in a separate "Advisory (AI)" section, labelled, via `untrusted()` |
| Cost controls | Per-session cap of 8k input / 1k output tokens; refuse if estimated cost exceeds a configurable cap (default $0.02, *verify pricing*); disabled in CI unless `--deep` is given |
| Injection defence | All data in one JSON blob inside delimiters; system prompt states it is data; no tools; output schema-validated; rationale strings escaped |
| Versioning and evaluation | Model ID and prompt version recorded in the header; eval corpus measures whether advisory suggestions would have improved precision/recall before promoting |

---

### 11.10 Detached `SessionEnd` worker (DEC-14)

| Concern | Specification |
|---|---|
| Budget | `budgets.session_end_sync_seconds` (default 3). Timed from engine start, not hook start. |
| Before detaching | Header and skeleton already written with `status: partial`, `report_mode: detached`; the session lock is held |
| Detach | `os.fork()`. Parent: emits the escaped "report finishing in background" line on the user-visible channel and exits 0. Child: `os.setsid()`, then `os.closerange(0, 1024)` and reopens `/dev/null` for stdio (so nothing can reach Claude's context), sets `umask 077`, `nice +10`, continues the pipeline. Only `io/worker.py` may call `fork`/`setsid` (§10.4). |
| Completion | Rewrites the report atomically with `status: complete`, writes `record.json`, deletes raw events, updates `index.json`, writes `pending_summary` into the session folder, then releases the lock |
| Hard cap | Worker exits after 5 minutes with `status: partial` and `budgets_hit: ["worker_timeout"]` |
| Next `SessionStart` | Shows pending summaries (escaped, at most 3 lines) on the user-visible channel and deletes the `pending_summary` markers |
| Crash safety | If the worker dies, the lock goes stale after 10 minutes and the orphan sweep finishes from the last completed step (§5.6) |
| Verify | Phase 0 confirms a `setsid` child survives Claude Code exit (OQ-3). If not → orphan sweep only, with summaries shown at next start |

### 11.11 First-run and managed-install notice (DEC-29)

| Concern | Specification |
|---|---|
| When | First `SessionStart` on a machine (`state.json.first_run_notice_at` unset), and again when a managed-settings install is detected (OQ-15) |
| Channel | User-visible only (`systemMessage` or stderr per Phase 0 findings); **never** stdout; never in Claude's context |
| Text (constant) | "Session Trust is active. Reports stay on this machine unless you publish them. Run `session-trust doctor --audit` to see what is captured." |
| Opt-out | None for the first display; it is a transparency control. It is shown at most once per trigger. |

## 12. Assumptions & Open Questions

### 12.1 Assumptions

| ID | Area | Assumption | Impact if wrong | Resolve by |
|---|---|---|---|---|
| ASM-1 | Scope | v1 is single-user and local; team mode later | Server, auth and tenancy move into v1; budget and security posture change completely | Phase 3 trigger review |
| ASM-2 | Compliance | Not built for a specific regulated sector | Certified retention, tamper evidence, residency needed | Before marketing to regulated buyers |
| ASM-3 | Architecture | Most users work inside git repos | The completeness guarantee rarely applies; log-only mode becomes primary | Phase 2 (bundle data) |
| ASM-4 | Platform | Hook payload fields, `SessionEnd` availability and time limit, per-hook output semantics as described (partly confirmed 2026-10-09) | Lifecycle and output contract rework | **Phase 0 spike** |
| ASM-5 | Runtime | `python3` widely present on target machines. On macOS, the `/usr/bin` shims count only when developer tools are installed (`xcode-select` guard, I-19) | Many visible "runtime missing" notices; engine moves to Go | Phase 2 (bundle data) |
| ASM-6 | Runtime | Language mismatch (Flask vs Laravel/TypeScript preferences) resolved in favour of Python | A TypeScript engine adds a build step | Decided |
| ASM-7 | Sharing | claude.ai Artifacts can't be published from Claude Code | The Artifact route becomes an alternative publish target | Phase 0 spike |
| ASM-8 | Interface | A session environment variable exists; slash-command inline output enters Claude's context. The variable is bypassable (`env -u`), which is acceptable now that DEC-15 is an accident guard backed by detection | In-session refusal relies on `/dev/tty` alone; detection rules unchanged | Phase 0 spike |
| ASM-9 | Deployment | Public repo, so Actions minutes are free | macOS CI minutes cost money | Phase 0 |
| ASM-10 | Testing | Python floor = 3.9, the version current macOS developer tools ship (*verify*). 3.9 has been end-of-life since Oct 2025; the code stays 3.9-compatible via `__future__` annotations | If Apple moves on, raise the floor to 3.10 (DEC-2 option B) | Phase 0 |
| ASM-11 | Compliance | The author is neither controller nor processor for v1 | Privacy notice, legal basis, possibly DPAs needed for v1 | Before v1.0.0 (professional review) |
| ASM-12 | Delivery | No hard deadline; ~15 focused hours/week (hours not stated) | Calendar estimates in §9 scale; phase order unchanged | Revisit at end of Phase 0 with actual velocity |
| ASM-13 | Billing | You have Claude Pro and a card that reliably pays recurring USD | DEC-22 falls back to a capped API key, or e2e stays manual-only without `claude -p` | Confirmed in hardening session |
| ASM-14 | Platform | `PreToolUse` and `PostToolUse` payloads share a `tool_use_id` | Fallback pairing by order + tool name + input hash; less exact for parallel tool calls | **Phase 0 spike** |
| ASM-15 | Platform | git honours `GIT_OBJECT_DIRECTORY` + `GIT_ALTERNATE_OBJECT_DIRECTORIES` for `add`, `write-tree`, `diff-tree`, `cat-file` on the floor version | Revert to in-repo objects + refs with a visible footprint warning (§11.6) | **Phase 0 spike** |
| ASM-16 | Security | Empty `filter.<name>.clean/.smudge/.process` overrides with `required=false` disable the driver cleanly | Snapshot via `hash-object --no-filters` + `update-index --index-info` (§11.6) | **Phase 0 spike** |

### 12.2 Open questions

| ID | Question | Why it matters | Resolve by |
|---|---|---|---|
| OQ-1 | ~~Capture `Read` intents via `PreToolUse`?~~ **Resolved: yes**, as part of DEC-1 (`PreToolUse(Bash\|Read)`). Phase 0 still confirms the payload shape. | — | Closed (hardening session) |
| OQ-2 | Which output channel is user-visible but not model-visible for `SessionStart`, `Stop`, `SessionEnd` (`systemMessage`? stderr?) | DEC-14 notices and summaries depend on it | Phase 0 |
| OQ-3 | Does `SessionEnd` have a time limit; does it fire for `claude -p`, `/clear`, terminal close and crashes; and does a detached (`setsid`) child survive Claude Code exit? | Report reliability; detached-worker design (DEC-14); orphan-sweep frequency | Phase 0 |
| OQ-4 | Does the marketplace support pinning to a tag/ref, and does it auto-update? | DEC-12 release gate and update notification | Phase 0 |
| OQ-5 | Does the plugin install path include a version (affects the launcher)? | DEC-15 launcher design | Phase 0 |
| OQ-6 | Exact Claude Code permission deny-rule syntax for Bash prefixes and pipes | Deny suggestions (DEC-10) | Phase 1 |
| OQ-7 | Is the `source` field present on `SessionStart` (startup/resume/clear)? | Resume segments (DEC-1) | Phase 0 |
| OQ-8 | Do hooks fire inside subagents, and how are those events marked? | Subagent prompt exclusion (DEC-4) and attribution | Phase 0 |
| OQ-9 | GitHub comment size limit and gist policies in target organisations | Publish design (DEC-9) | Phase 3 (v1.1) |
| OQ-10 | Realistic M3 adoption target | Success metric calibration | After v0.1.0 + 1 month |
| OQ-11 | Calibrated precision/recall targets | M1 and CI gate | Phase 2 |
| OQ-12 | Professional review of `docs/data-processing.md`, `docs/teams.md` and ASM-11 | Legal exposure | Before v1.0.0 |
| OQ-13 | Share of Windows users among early adopters | DEC-18 revisit | 3 months after v1.0.0 |
| OQ-14 | ~~Card support for recurring USD billing?~~ **Resolved:** yes, and e2e uses Claude Pro instead of an API workspace (ASM-13, DEC-22) | — | Closed (hardening session) |
| OQ-15 | Can a plugin detect that it was installed via managed settings rather than by the user? | First-run notice re-display for managed installs (DEC-29) | Phase 0 |
| OQ-16 | Does `PostToolUse` fire for failed or interrupted tool calls (and is there a separate failure event)? | Completeness of `attempted_no_completion` vs failure outcomes (DEC-1, DEC-10) | Phase 0 |
| OQ-17 | Do Claude Pro terms cover using `claude -p` for personal manual testing of a plugin? | DEC-22 | Phase 1 |
| OQ-18 | How noisy is the "no agent action recorded" label in real use (developer editing during sessions)? | DEC-25 revisit (possible mtime-window heuristic) | Phase 2 (tester feedback) |

---

## Appendix: Concepts Covered

| Concept | Why it mattered for Session Trust | Study further (mid/senior level) |
|---|---|---|
| **Ground truth vs self-report** | The agent's own account (or its tool log) can be incomplete; the git diff is the independent source of truth | Reconciliation patterns in auditing systems; event sourcing vs state snapshots |
| **Git plumbing** (`write-tree`, temporary index, `diff-tree`, `gc` and reachability) | Baselines without touching the user's index; keeping snapshots verifiable; `verify` | *Pro Git* ch. 10 "Git Internals"; `git help gitrepository-layout` |
| **Git alternates and object directories** (`GIT_OBJECT_DIRECTORY`, `GIT_ALTERNATE_OBJECT_DIRECTORIES`, index stat cache) | Zero-footprint snapshots and fast baselines (DEC-1) | `git help git` (environment variables); `gitrepository-layout` (`objects/info/alternates`); `git help update-index` (racy-git, stat info) |
| **Git filter drivers and attributes** (`clean`/`smudge`/`process`, `.gitattributes`) | `git add` executes repo-selected commands, the hole found during hardening (DEC-13) | `git help gitattributes` (filter section); Git LFS architecture |
| **Pseudo-terminals** (`pty`, `script`, `expect`) | Why `/dev/tty` confirmation is an accident guard, not a security boundary (DEC-15) | `man 7 pty`; `man 1 script`; Python `pty` module docs |
| **Process detachment** (`fork`, `setsid`, closing inherited stdio) | Finishing reports after Claude Code exits without writing into its context (DEC-14) | APUE (Stevens) ch. 9 and 13 (daemon processes); Python `os.fork` caveats |
| **Atomic file operations** (`O_EXCL`, write-then-rename, `fsync`) | Lock-free parallel capture; crash-safe records | POSIX `rename(2)` guarantees; durability on APFS/ext4 |
| **Fail-open vs fail-closed** | Hooks fail open (never block work); CI fails closed (partial = fail) | Designing safety-critical failure modes; circuit breakers |
| **Trust boundaries and confused deputies** | The tool runs code in untrusted repos for a possibly untrusted agent | Confused-deputy problem; capability-based security |
| **Git config as code execution** (`fsmonitor`, `textconv`, external diff, `safe.directory`) | The most serious STRIDE finding (DEC-13) | CVE-2022-24765 and related git advisories; Git's security docs |
| **STRIDE threat modelling** | Structured walk of capture, report and publish flows | Adam Shostack, *Threat Modeling: Designing for Security*; OWASP Threat Modeling Cheat Sheet |
| **Output encoding and injection** (markdown/HTML, terminal escapes, Unicode bidi/"Trojan Source", lookalikes) | Reports are built from attacker-influenced strings (DEC-11) | Trojan Source paper (Boucher & Anderson); terminal escape injection write-ups |
| **Prompt injection (indirect)** | Reports and repo content may be read by future AI sessions | OWASP Top 10 for LLM Applications (LLM01); Simon Willison's prompt-injection writing |
| **Secret detection** (patterns + entropy + allowlists) | Redaction quality defines whether reports are safe to share | gitleaks and trufflehog rule sets; false-positive tuning |
| **Keyed hashing (HMAC) vs plain hashing** | Plain hashes of small secrets are brute-forceable | Dictionary attacks on unsalted hashes; HMAC construction |
| **ReDoS** | Regex over attacker input inside a time-limited hook | Catastrophic backtracking; linear-time regex engines (RE2) |
| **Supply-chain security** (signed tags, SHA-pinned actions, zero-secret CI, `pull_request_target` risks) | The plugin runs on every tool call of every user | SLSA framework; Sigstore/`gitsign`; GitHub Actions hardening guide |
| **Semantic versioning of multiple contracts** | Plugin, report schema, CLI schema, rules, state version | Semver spec; schema evolution with JSON Schema |
| **Precision/recall and evaluation sets** | Turns heuristic tuning into measurement (DEC-19) | Building labelled datasets; inter-annotator agreement (Cohen's kappa) |
| **Property-based testing** | Fuzzing the renderer, parser and redaction | `hypothesis` docs; "Choosing properties for property-based testing" |
| **Data minimisation and privacy by default** | Git history as a permanent store; employee-monitoring implications | GDPR Art. 5 and 25; NDPA 2023 overview; DPIA basics (*verify with a professional*) |
| **Honest positioning / guarantees documents** | Overclaiming is the main liability risk for a solo security-tool author | Read security-tool threat-model and limitations pages (e.g. Signal, age, Sigstore) |

---

## 13. Change Log

*No entries yet. This section is filled in during the build, when a decision has to change.*

**Rules:**
- Entries are only ever added at the bottom; nothing above this section is edited or deleted.
- The newest entry overrides the original blueprint text and any earlier entry on the same topic.

**Entry format:**

```text
DL-[n] | [date] | Replaces: [section, DEC-x or DL-y] | Change: [what changed to what] | Why: [reason]
```

DL-1 | 2026-10-09 | Replaces: §10.3 Naming conventions (Branches and Commits rows) | Change: All work happens on the dev branch and reaches main only via pull request (no feat/, fix/, sec/, docs/ branches). Commit messages use `T<n>: <type>(<scope>): <summary>` (Conventional Commit types) with DCO sign-off via `git commit -s` | Why: task-number traceability required by the build workflow, while keeping Conventional Commit types and DCO

DL-2 | 2026-10-09 | Replaces: §7.2 Environment layout (Local dev row: adds a development-environment constraint) | Change: Local development, tests and Claude Code sessions run on WSL2 (Ubuntu 24.04) with the repo inside the Linux filesystem; native Windows checkouts are not used; .gitattributes enforces LF line endings | Why: the maintainer's machine is Windows, while the product supports only Linux and macOS (DEC-18); POSIX file modes, dash, setsid and the hooks need a real Linux environment
