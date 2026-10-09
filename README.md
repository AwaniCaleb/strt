# Session Trust

Session Trust is a Claude Code plugin that produces an honest, readable account of
what an AI coding session actually changed, after the session ends. It records the
file-modifying and shell actions Claude takes, snapshots the working tree at git level
before and after, and reconciles the two, so every change on disk is either explained
by a recorded action or flagged as unexplained. Each changed file is placed in one of
three tiers (in scope, likely related, unexplained) with a stated reason, and risky
commands are flagged by severity. It runs entirely on your machine, makes no network
calls, writes nothing into your repository unless you opt in, and never blocks or
steers the session it observes. It is a self-review aid owned by the developer, not a
surveillance tool and not compliance evidence.

## Status

Pre-alpha: under construction, not usable yet.

## Platforms

Linux and macOS (Windows via WSL2); native Windows not supported.

## Licence

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
