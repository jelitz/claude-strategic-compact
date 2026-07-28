---
name: strategic-compact
description: Suggests manual context compaction at logical intervals to preserve context through task phases rather than arbitrary auto-compaction.
metadata:
  origin: ECC
  source: https://github.com/affaan-m/ECC/blob/main/skills/strategic-compact/SKILL.md
---

# Strategic Compact Skill

Suggests manual `/compact` at strategic points in your workflow rather than relying on arbitrary auto-compaction.

## When to Activate

- Running long sessions that approach context limits (200K+ tokens)
- Working on multi-phase tasks (research → plan → implement → test)
- Switching between unrelated tasks within the same session
- After completing a major milestone and starting new work
- When responses slow down or become less coherent (context pressure)

## Why Strategic Compaction?

Auto-compaction triggers at arbitrary points:
- Often mid-task, losing important context
- No awareness of logical task boundaries
- Can interrupt complex multi-step operations

Strategic compaction at logical boundaries:
- **After exploration, before execution** — Compact research context, keep implementation plan
- **After completing a milestone** — Fresh start for next phase
- **Before major context shifts** — Clear exploration context before different task

## How It Works

The `suggest-compact.js` script runs on PreToolUse (Edit/Write) and combines two signals:

1. **Context size (primary)** — Reads the latest `usage` record from the session transcript (`transcript_path` in the hook payload) and sums `input_tokens + cache_read_input_tokens + cache_creation_input_tokens` (the true context size of the turn). Suggests `/compact` at a window-scaled threshold — 160k tokens on a 200k window, 250k on a 1M window (detected from a `[1m]` model marker, or inferred when observed tokens already exceed 200k) — and re-reminds after every additional 60k tokens of context growth
2. **Tool-call count (secondary)** — Counts tool invocations in session; suggests at a configurable threshold (default: 50 calls), then every 25 calls after

Tool count alone is a weak proxy for window pressure: a few large file reads or MCP responses can fill the window in very few calls, while many tiny calls can cross 50 with a near-empty window. The context-size signal fires when it actually matters.

### Proactive behavior contract

Unlike a passive log line, the injected message is a **directive to the assistant**, not just a status note. When either signal fires, the assistant must, in that same response:

1. Tell the user now that this looks like a good point to compact — don't wait to be asked, don't just silently absorb the signal.
2. Propose a ready-to-run `/compact <focus instructions>` command. Per [Claude Code's own docs](https://code.claude.com/docs/en/commands), the argument to `/compact` is a **short focus instruction** for what the automatic summary should prioritize keeping (e.g. `/compact focus on the auth middleware rewrite, keep the API design decisions`) — a phrase, not a pre-written narrative summary. Claude Code's compaction step generates the actual summary from the real conversation; the assistant is not writing that summary itself and must not paste a multi-paragraph recap into the command.
3. Skip the nudge only when the current moment is clearly mid-implementation (per the Compaction Decision Guide below) — the signal still fired, but acting on it would lose costly in-flight state.

This is baked into the hook's injected text itself (`scripts/hooks/suggest-compact.js`), so it applies every time the signal fires, regardless of whether this skill was separately invoked in the session.

## Hook Setup

This was installed manually (not as a plugin). It is registered in `~/.claude/settings.json` as:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{ "type": "command", "command": "node ~/.claude/scripts/hooks/suggest-compact.js" }]
      },
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "node ~/.claude/scripts/hooks/suggest-compact.js" }]
      }
    ]
  }
}
```

## Configuration

Environment variables:
- `COMPACT_THRESHOLD` — Tool calls before first suggestion (default: 50)
- `COMPACT_CONTEXT_THRESHOLD` — Context tokens before the context-size suggestion (default: 160000 on a 200k window, 250000 on a 1M window; `0` disables the context signal)
- `COMPACT_CONTEXT_INTERVAL` — Additional context tokens before the suggestion repeats (default: 60000)
- `COMPACT_STATE_TTL_DAYS` — Days before stale per-session state files in the temp dir are swept (default: 14)
- `ECC_CONTEXT_WINDOW_TOKENS` — Explicit context-window size, in tokens, overriding auto-detection. Set this for large-window models whose reported id lacks a `[1m]` marker (e.g. 400k Opus 4.x, or a new 1M-window model family) so the threshold scales to the real window instead of defaulting to 200k and overstating context usage.
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` — Claude Code's native window-size override, in tokens; honored as a fallback when `ECC_CONTEXT_WINDOW_TOKENS` is unset.

> The context window is otherwise auto-detected from a `[1m]` model marker or inferred when observed tokens already exceed 200k. On a large-window model that carries neither signal, set one of the overrides above so the `/compact` suggestion fires at the right point.

## Compaction Decision Guide

Use this table to decide when to compact:

| Phase Transition | Compact? | Why |
|-----------------|----------|-----|
| Research → Planning | Yes | Research context is bulky; plan is the distilled output |
| Planning → Implementation | Yes | Plan is in TodoWrite or a file; free up context for code |
| Implementation → Testing | Maybe | Keep if tests reference recent code; compact if switching focus |
| Debugging → Next feature | Yes | Debug traces pollute context for unrelated work |
| Mid-implementation | No | Losing variable names, file paths, and partial state is costly |
| After a failed approach | Yes | Clear the dead-end reasoning before trying a new approach |

## What Survives Compaction

Understanding what persists helps you compact with confidence:

| Persists | Lost |
|----------|------|
| CLAUDE.md instructions | Intermediate reasoning and analysis |
| TodoWrite task list | File contents you previously read |
| Memory files (`~/.claude/memory/`) | Multi-step conversation context |
| Git state (commits, branches) | Tool call history and counts |
| Files on disk | Nuanced user preferences stated verbally |

## Best Practices

1. **Compact after planning** — Once plan is finalized in TodoWrite, compact to start fresh
2. **Compact after debugging** — Clear error-resolution context before continuing
3. **Don't compact mid-implementation** — Preserve context for related changes
4. **Act on the suggestion, don't just log it** — The hook tells the assistant *when*; the assistant decides *if* (per the table above) and, when it proceeds, surfaces the suggestion to the user in that turn rather than absorbing it silently
5. **Write before compacting** — Save important context to files or memory before compacting
6. **Always add focus instructions** — Never suggest a bare `/compact`; pass a short phrase telling it what to prioritize, e.g. `/compact Focus on implementing auth middleware next`. Do not write out a full narrative summary as the argument — that's not what compaction does with it, and Claude Code generates the real summary itself from the conversation

## Related

- [The Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) — Token optimization section
- Memory persistence hooks — For state that survives compaction
