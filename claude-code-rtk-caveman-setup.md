# Setting up RTK and Caveman for Claude Code

Two productivity plugins for Claude Code: **RTK** (token savings on CLI tool calls) and **Caveman** (compressed communication mode).

---

## 1. RTK (Rust Token Killer)

RTK is a CLI proxy that intercepts shell commands from Claude Code and rewrites them into token-optimized versions — cutting 60-90% of token usage on dev operations like `git status`, `git diff`, etc.

### Install

```bash
brew install rtk
rtk --version   # verify install
```

### Hook setup

RTK integrates with Claude Code via a **PreToolUse hook** on the `Bash` matcher. Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "rtk hook claude"
      }
    ]
  }
}
```

This intercepts every Bash tool call and transparently rewrites commands to rtk proxies (e.g. `git status` → `rtk git status`).

### Documentation reference

Create `~/.claude/RTK.md` with usage reference, then reference it from your `CLAUDE.md`:

```markdown
@RTK.md
```

RTK.md should document:

- **Meta commands** (run directly, not proxied):
  - `rtk gain` — show token savings analytics
  - `rtk gain --history` — command usage history with savings
  - `rtk discover` — analyze Claude Code history for missed optimization opportunities
  - `rtk proxy <cmd>` — execute raw command without filtering (debugging)
- **Hook-based usage** — all other commands are automatically rewritten by the hook. No manual intervention needed.

### Verify it works

```bash
rtk gain   # should show savings stats, not "command not found"
```

> **Warning:** Name collision exists with `reachingforthejack/rtk` (Rust Type Kit). If `rtk gain` fails, check `which rtk` points to the correct binary.

---

## 2. Caveman

Caveman is a Claude Code plugin that compresses assistant communication — cutting ~75% of filler tokens while keeping full technical accuracy. Think: smart caveman who drops articles, filler words, and pleasantries but keeps every technical detail.

### Install

Install from the Claude Code marketplace:

```bash
claude mcp add-from-marketplace caveman@caveman
```

This registers it in `~/.claude/settings.json` automatically.

### How it works

Caveman uses two hooks (user-level, stored in `~/.claude/hooks/`):

| Hook | Script | Purpose |
|------|--------|---------|
| `SessionStart` | `caveman-activate.js` | Injects "CAVEMAN MODE ACTIVE" instructions at session start |
| `UserPromptSubmit` | `caveman-mode-tracker.js` | Re-injects reminder per prompt, tracks intensity level |

Supporting files installed alongside:
- `caveman-config.js` — configuration
- `caveman-stats.js` — token savings tracking (via `/caveman-stats`)
- Statusline scripts for mode display

### Intensity levels

Switch with `/caveman lite|full|ultra`:

| Level | Effect |
|-------|--------|
| **lite** | Drop filler, keep grammar mostly intact |
| **full** (default) | Drop articles, fragments OK, short synonyms |
| **ultra** | Maximum compression, telegraphic style |

### Skills provided

- `/caveman lite|full|ultra` — switch intensity
- `/caveman-commit` — ultra-compressed commit messages
- `/caveman-review` — ultra-compressed code review comments
- `/caveman-compress` — compress memory/doc files into caveman format
- `/caveman-help` — quick reference card

### Disable

Say `stop caveman` or `normal mode` in conversation to revert to standard English.

### Important caveat

> **Node path hardcoding:** The hook commands may hardcode the node binary path (e.g. `/opt/homebrew/Cellar/node/XX.Y.Z/bin/node`). This **breaks when Homebrew bumps the node version**. Fix: edit the hook scripts to use plain `node` instead of the absolute path, so it resolves from `$PATH`.

---

## Verify both are active

Start a new Claude Code session. You should see:

1. `SessionStart:startup hook success: CAVEMAN MODE ACTIVE` — Caveman is injecting
2. Bash commands getting rewritten transparently — RTK is proxying
3. Run `rtk gain` to confirm token savings are being tracked

---

## Summary

| Plugin | Install method | Integration | What it saves |
|--------|---------------|-------------|---------------|
| **RTK** | Homebrew | PreToolUse hook on Bash | 60-90% tokens on CLI output |
| **Caveman** | Claude Code marketplace | SessionStart + UserPromptSubmit hooks | ~75% tokens on assistant prose |
