# claude-strategic-compact

A Claude Code skill + hook that nudges you toward manual `/compact` at logical task boundaries — instead of context filling up and Claude Code's automatic compaction firing wherever it happens to land.

Adapted from [affaan-m/ECC](https://github.com/affaan-m/ECC) (`skills/strategic-compact/`, MIT licensed), with two changes:

1. **Manual-install paths.** ECC ships this as part of a much larger framework with its own installer. This repo is the standalone slice, laid out for a direct copy into `~/.claude/` (see Install below) rather than ECC's own directory conventions.
2. **A proactive behavior contract.** The upstream hook only emits a passive status line ("context is at X%, consider compacting"). This version's injected message is a directive: when it fires, the assistant is instructed to actively tell you *now*, in that response, rather than silently absorbing the signal — see [SKILL.md](skills/strategic-compact/SKILL.md#proactive-behavior-contract) for the exact contract and why it stops short of writing the `/compact` summary for you.

## How it works

`scripts/hooks/suggest-compact.js` runs on every `Edit`/`Write` tool call (via a `PreToolUse` hook) and combines two signals:

- **Context size (primary)** — reads the real token usage from the session transcript, compares it to a window-scaled threshold (160k on a 200k window, 250k on 1M), and re-fires every +60k tokens of growth.
- **Tool-call count (secondary)** — a weak proxy for window pressure, fires at 50 calls then every 25 after.

When either fires, the injected message tells the assistant to speak up immediately and propose a `/compact <focus instructions>` — a short phrase telling the compactor what to prioritize (per [Claude Code's own docs](https://code.claude.com/docs/en/commands): `/compact [instructions]` takes *focus instructions*, not a pre-written summary — Claude Code generates the actual summary itself from the real conversation). Full details, the compaction decision table, and what does/doesn't survive `/compact` are in [SKILL.md](skills/strategic-compact/SKILL.md).

## Install

Copy the skill and scripts into your global Claude Code config:

```bash
cp -r skills/strategic-compact ~/.claude/skills/strategic-compact
mkdir -p ~/.claude/scripts/hooks ~/.claude/scripts/lib
cp scripts/hooks/suggest-compact.js ~/.claude/scripts/hooks/
cp scripts/lib/*.js ~/.claude/scripts/lib/
```

Then merge the hook registration from [`settings.snippet.json`](settings.snippet.json) into `~/.claude/settings.json` under `hooks.PreToolUse` — **merge into the existing array, don't overwrite it** if you already have other `PreToolUse` hooks.

Requires Node.js on PATH (the hook is a `node` script). Verify it loads cleanly:

```bash
node --check ~/.claude/scripts/hooks/suggest-compact.js
```

## Configuration

Environment variables (all optional):

| Variable | Default | Purpose |
|---|---|---|
| `COMPACT_THRESHOLD` | `50` | Tool calls before the first count-based suggestion |
| `COMPACT_CONTEXT_THRESHOLD` | `160000` (200k window) / `250000` (1M window) | Context tokens before the size-based suggestion; `0` disables it |
| `COMPACT_CONTEXT_INTERVAL` | `60000` | Additional tokens of growth before the suggestion re-fires |
| `COMPACT_STATE_TTL_DAYS` | `14` | Days before stale per-session state files in the temp dir are swept |
| `ECC_CONTEXT_WINDOW_TOKENS` | auto-detected | Explicit context-window size override, for large-window models whose id lacks a `[1m]` marker |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | — | Claude Code's own window-size override; honored as a fallback |

## Scope

This repo is the skill + hook only. `scripts/lib/agent-data-home.js` and `scripts/lib/path-safety.js` are pulled in only because `scripts/lib/utils.js` requires them at import time (`ECC_AGENT_DATA_HOME` resolution) — they aren't used for anything beyond satisfying that require chain here.

## License

MIT — see [LICENSE](LICENSE). Original work Copyright (c) 2026 Affaan Mustafa; modifications Copyright (c) 2026 jelitz.
