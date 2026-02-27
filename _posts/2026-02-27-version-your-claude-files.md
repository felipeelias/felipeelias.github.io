---
layout: post
title: "You Should Be Versioning Your ~/.claude Config"
description: "Your Claude Code configuration is worth keeping in version control. Here's what to track and what to ignore."
date: 2026-02-27
tags: [claude-code, git]
comments: true
hashnode: true
---

If you've been using [Claude Code][claude-code] for a while, you've probably accumulated a decent amount of configuration: settings, skills, custom agents, CLAUDE.md instructions. Losing all of that would be annoying.

The fix: `git init` inside `~/.claude`, with some caveats.

## What lives in ~/.claude

Based on the [official docs][docs], here's what's worth versioning:

- `CLAUDE.md`
- `settings.json`
- `skills/**/SKILL.md`
- `agents/<name>.md`
- `commands/<name>.md`
- `statusline.sh`

## What to ignore

Claude Code generates transient data that you should ignore. Add this to your `.gitignore`.

```text
# Credentials
.credentials.json
credentials.json

# Internal state
.claude.json
.claude.json.backup.*
security_warnings_*.json
stats-cache.json
mcp-needs-auth-cache.json

# Session data
history.jsonl
backups
cache
debug
file-history
paste-cache
session-env
shell-snapshots

# Agent and team state
plans
plugins
tasks
teams
todos

# Telemetry
statsig
telemetry
usage-data

# IDE integration
ide/
```

## Note about projects

The `projects/` directory contains per-project auto-memory and conversation logs. Claude Code [recently enabled memory per project][memory], which is worth adding to git. The easiest approach is to ignore the log files instead:

```text
projects/**/*.jsonl
projects/**/*.txt
```

## Quick setup

Remember to push it to GitHub or GitLab, below is an example with GitHub:

```bash
cd ~/.claude
git init
git add .gitignore CLAUDE.md settings.json
git add skills/ agents/ commands/ statusline.sh 2>/dev/null
git commit -m "feat: initial claude config"
gh repo create claude-config --private --source=. --push
```

That's it.

[claude-code]: https://claude.ai/code
[docs]: https://code.claude.com/docs/en/settings
[memory]: https://code.claude.com/docs/en/memory
