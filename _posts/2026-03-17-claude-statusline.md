---
layout: post
title: "claude-statusline: a configurable status line for Claude Code"
description: "A customizable status line for Claude Code with built-in themes, cost tracking, context window usage, and full TOML configuration."
date: 2026-03-17
tags: [claude-code, go, open-source]
comments: true
hashnode: true
---

Claude Code lets you customize the status line at the bottom of your terminal. The [default suggestion][statusline-docs] is a bash script, which works but gets clunky fast and difficult to maintain for more complex features.

There are other tools out there that solve this (more on that below), but none written in Go. I think Go is the right fit here: fast, compiles to a single binary, and cross-platform out of the box, given my setup uses Mac/Linux/Windows. I'm a big fan of [Starship][starship], and it heavily inspired the design: presets, format strings, per-module config and so on.

So I built [claude-statusline][repo]:

![claude-statusline screenshot](/assets/images/claude-statusline.webp)

## What it shows

The default format displays five modules:

```toml
format = "$directory | $git_branch | $model | $cost | $context"
```

- `directory`: current working directory, truncated to the last 3 segments
- `git_branch`: current branch, with an indicator when you're inside a git worktree
- `model`: which Claude model is running
- `cost`: session cost in USD, color-coded by thresholds (yellow at $1, red at $5)
- `context`: context window usage as a progress bar with percentage (yellow at 50%, red at 90%)

There are two more modules disabled by default: `session_timer` and `lines_changed`. You can enable them in the config.

## Themes

claude-statusline ships with 6 built-in presets:

| Preset             | Description                       | Nerd Font |
| ------------------ | --------------------------------- | --------- |
| `default`          | Flat with pipes, standard colors  | No        |
| `minimal`          | Clean spacing, no separators      | No        |
| `pastel-powerline` | Pastel powerline arrows           | Yes       |
| `tokyo-night`      | Dark blues with rounded powerline | Yes       |
| `gruvbox-rainbow`  | Earthy rainbow powerline          | Yes       |
| `catppuccin`       | Catppuccin Mocha powerline        | Yes       |

To switch themes, set the `preset` field in your config:

```toml
preset = "tokyo-night"
```

Preview all of them with mock data:

```bash
claude-statusline themes
```

## Making it your own

The config lives at `~/.config/claude-statusline/config.toml`. You can start from a preset and override individual modules:

```toml
preset = "catppuccin"

[cost]
thresholds = [
  { above = 2.0, style = "yellow" },
  { above = 10.0, style = "red" },
]

[context]
bar_width = 10
```

Styles support named colors, hex values, 256-color codes, and attributes:

```toml
[model]
style = "fg:#11111b bg:#cba6f7 bold"
```

Rearranging the format string or adding inline styled text also works:

```toml
format = "$model | $directory | $git_branch | [$cost](dim)"
```

## Installation

With Homebrew (recommended, keeps updates simple):

```bash
brew install felipeelias/tap/claude-statusline
```

Or with Go:

```bash
go install github.com/felipeelias/claude-statusline@latest
```

Then add it to your Claude Code settings (`.claude/settings.json` or global):

```json
{
  "statusLine": {
    "type": "command",
    "command": "claude-statusline prompt"
  }
}
```

Generate a starter config with `claude-statusline init`, and use `claude-statusline test` to iterate on your config without running Claude Code.

## Alternatives

The [awesome-claude-code][awesome] list has other options worth checking out:

- [claude-powerline][claude-powerline]: vim-style powerline with usage tracking and themes
- [CCometixLine][ccometixline]: written in Rust, with interactive TUI configuration
- [claudia-statusline][claudia-statusline]: also Rust-based, with persistent stats tracking and cloud sync
- [ccstatusline][ccstatusline]: customizable formatter with token usage and metrics

[awesome]: https://github.com/hesreallyhim/awesome-claude-code
[ccometixline]: https://github.com/Haleclipse/CCometixLine
[ccstatusline]: https://github.com/sirmalloc/ccstatusline
[claudia-statusline]: https://github.com/hagan/claudia-statusline
[claude-powerline]: https://github.com/Owloops/claude-powerline
[repo]: https://github.com/felipeelias/claude-statusline
[starship]: https://starship.rs
[statusline-docs]: https://code.claude.com/docs/en/settings#status-line
