<div align="center">

# claude-strategic-compact

**A Claude Code skill + hook for proactive, phase-aware `/compact` suggestions**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Requires Node.js](https://img.shields.io/badge/node-%3E%3D18-339933?logo=node.js&logoColor=white)](https://nodejs.org)
[![Based on ECC](https://img.shields.io/badge/based%20on-affaan--m%2FECC-blue)](https://github.com/affaan-m/ECC)

[한국어](README.md) · English

</div>

---

Claude Code compacts the conversation automatically once context fills up — at whatever point that happens to be, often mid-task. This project nudges you toward *manual* `/compact` at a moment that actually makes sense: after research wraps up, after a milestone lands, before a task switch — not wherever the token counter happens to cross a line.

Adapted from [affaan-m/ECC](https://github.com/affaan-m/ECC) (`skills/strategic-compact/`, MIT licensed), repackaged as a standalone, manually-installable slice with one behavioral change: the upstream hook only logs a passive status line, this one issues an active directive — see [What changed](#what-changed-from-upstream) below.

## Table of contents

- [How it works](#how-it-works)
- [What changed from upstream](#what-changed-from-upstream)
- [Install](#install)
- [Configuration](#configuration)
- [Scope](#scope)
- [License](#license)

## How it works

`scripts/hooks/suggest-compact.js` runs on every `Edit` / `Write` tool call via a `PreToolUse` hook, and combines two signals:

| Signal | Trigger | Notes |
|---|---|---|
| **Context size** (primary) | Real token usage from the session transcript crosses a window-scaled threshold — 160k on a 200k window, 250k on a 1M window | Re-fires every +60k tokens of further growth |
| **Tool-call count** (secondary) | 50 calls, then every 25 after | A weaker proxy — a few large reads can fill the window in very few calls, so this is a backstop, not the main signal |

When either signal fires, Claude Code receives an injected instruction. Per [Claude Code's own docs](https://code.claude.com/docs/en/commands), `/compact [instructions]` takes a **short focus instruction** — what the automatic summary should prioritize keeping — not a pre-written summary; Claude Code generates the actual summary itself from the real conversation. The full behavior contract, a compaction decision table (when *not* to compact — mid-implementation, for instance), and what survives `/compact` at all are documented in [`skills/strategic-compact/SKILL.md`](skills/strategic-compact/SKILL.md).

## What changed from upstream

| | Upstream (ECC) | This repo |
|---|---|---|
| Install target | ECC's own framework layout, its own installer | Direct copy into `~/.claude/`, no installer needed |
| Hook message | Passive status line ("context is at X%") | Active directive: speak up now, in this response |
| `/compact` argument | Not addressed | Explicitly scoped to *focus instructions*, matching documented `/compact` semantics — not a full narrative summary |

## Install

```bash
cp -r skills/strategic-compact ~/.claude/skills/strategic-compact
mkdir -p ~/.claude/scripts/hooks ~/.claude/scripts/lib
cp scripts/hooks/suggest-compact.js ~/.claude/scripts/hooks/
cp scripts/lib/*.js ~/.claude/scripts/lib/
```

Then merge the hook registration from [`settings.snippet.json`](settings.snippet.json) into `~/.claude/settings.json`, under `hooks.PreToolUse` — **merge into the existing array, don't overwrite it** if you already have other `PreToolUse` hooks configured.

Requires Node.js on `PATH` (the hook is a `node` script). Sanity-check it loads cleanly:

```bash
node --check ~/.claude/scripts/hooks/suggest-compact.js
```

## Configuration

All environment variables are optional:

| Variable | Default | Purpose |
|---|---|---|
| `COMPACT_THRESHOLD` | `50` | Tool calls before the first count-based suggestion |
| `COMPACT_CONTEXT_THRESHOLD` | `160000` (200k window) / `250000` (1M window) | Context tokens before the size-based suggestion fires; `0` disables it |
| `COMPACT_CONTEXT_INTERVAL` | `60000` | Additional tokens of growth before the suggestion re-fires |
| `COMPACT_STATE_TTL_DAYS` | `14` | Days before stale per-session state files in the temp dir are swept |
| `ECC_CONTEXT_WINDOW_TOKENS` | auto-detected | Explicit context-window size override, for large-window models whose id lacks a `[1m]` marker |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | — | Claude Code's own window-size override; honored as a fallback |

## Scope

This repo is the skill + hook only. `scripts/lib/agent-data-home.js` and `scripts/lib/path-safety.js` are pulled in only because `scripts/lib/utils.js` requires them at import time (`ECC_AGENT_DATA_HOME` resolution) — they aren't used for anything beyond satisfying that require chain here.

## License

MIT — see [LICENSE](LICENSE). Original work Copyright (c) 2026 Affaan Mustafa; modifications Copyright (c) 2026 jelitz.
