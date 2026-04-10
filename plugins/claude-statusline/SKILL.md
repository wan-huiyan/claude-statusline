---
name: claude-statusline
description: >
  Build and customize the Claude Code status line and session banner. Use this
  skill whenever the user mentions "status line", "status bar", "bottom bar",
  "info bar", "session name", "session banner", "rename session", "/rename",
  "/color", wants to add/change/remove information displayed at the bottom of
  Claude Code, asks about what data is available in the statusline, or wants to
  show session info, git state, cost, context usage, model info, or any other
  live metrics in the terminal. Also use when the user says "statusline",
  "/statusline", asks about customizing the Claude Code UI, or wants to
  auto-name sessions.
---

# Claude Code Status Line Builder

Generate and customize the shell script that powers Claude Code's bottom-of-terminal
info bar. The statusline receives a JSON payload via stdin after every assistant
message and prints colored rows to stdout.

## How It Works

Claude Code runs your script (`~/.claude/statusline.sh`) and pipes session JSON
to stdin. Your script parses the JSON with `jq`, formats the data, and prints
rows with ANSI colors. Each `printf "%b\n"` call produces one row.

**Config** (in `~/.claude/settings.json` or project `.claude/settings.json`):
```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "refreshInterval": 10
  }
}
```

`refreshInterval` (optional, seconds) re-runs the script periodically — useful
for live clocks or polling git status even when Claude hasn't responded.

## Available Modules

The statusline is built from composable **modules** — each is a self-contained
bash snippet. Pick the ones you want, arrange them into rows, done.

| Module | What it shows | Example output |
|--------|--------------|----------------|
| **Session Label** | Auto-derived conversation name: explicit name > latest commit > branch > project dir | `📝 fix auth token refresh` |
| **Git Branch** | Branch name, worktree indicator, detached state | `⎇ feature/new-api` or `⎇ wt:review (main)` |
| **Git Dirty** | Changed + untracked file counts | `● 3 changed +2 new` or `✓ 0 changed` |
| **Model Info** | Model name and version | `Opus 4.6` |
| **Effort Level** | Effort level from `~/.claude/settings.json` `effortLevel` key (low/medium/high) | `[medium]` (green/yellow/red) |
| **Cost** | Session cost in USD | `$0.4231` |
| **Duration** | Total API wall-clock time | `12m34s` |
| **Lines Changed** | Lines added/removed this session | `+142/-37 lines` |
| **Context Bar** | Visual 10-block bar + percentage + compaction warning | `ctx:42% ████░░░░░░ 58% left` |
| **Token Counts** | Cumulative input/output tokens | `in:84K out:12K` |
| **Cache Ratio** | Cache hit percentage from last API call | `cache:73%` |
| **Rate Limits** | 5h/7d usage for Claude.ai Pro/Max plans | `5h:45% 7d:12%` |
| **Memory Freshness** | Age of lessons.md + nearest project memory | `lessons:2h proj-mem:15m` |
| **Agent/Vim/Worktree** | Active agent name, vim mode, worktree flag | `⚙ agent:review vim:NORMAL` |
| **Version** | Claude Code version | `v1.2.3` |

See `references/modules.md` for the complete copy-pasteable bash code for each module.
See `references/fields.md` for every JSON field available in the stdin payload.

## Generating a Statusline

When the user asks for a statusline, follow this process:

### 1. Understand preferences

Ask (or infer from context) which modules they want and how many rows.
Common layouts:

- **Minimal (1 row):** Model + cost + context percentage
- **Standard (2 rows):** Git + model/cost, context bar
- **Full (4 rows):** Git, model/cost/duration/lines, context bar, session/memory
- **Custom:** User picks modules and arrangement

If the user already has a statusline (`~/.claude/statusline.sh`), read it first
and modify rather than replacing — they may have custom logic worth preserving.

### 2. Assemble the script

Every statusline script follows this skeleton:

```bash
#!/bin/bash
input=$(cat)

# ANSI colors
R='\033[0;31m'; G='\033[0;32m'; Y='\033[0;33m'; B='\033[0;34m'
M='\033[0;35m'; C='\033[0;36m'; W='\033[0;37m'
BLD='\033[1m'; DIM='\033[2m'; RST='\033[0m'

# Parse JSON (only parse fields you use)
MODEL_ID=$(echo "$input" | jq -r '.model.id // ""')
# ... more fields ...

# Build modules
# ... module logic ...

# Assemble rows
ROW1="..."
ROW2="..."

# Output
printf "%b\n" "$ROW1"
printf "%b\n" "$ROW2"
```

**Key principles:**
- Only parse JSON fields the chosen modules actually use (each `jq` call costs ~5ms)
- Use `// ""` or `// 0` defaults so missing fields don't break the script
- Color-code thresholds (green/yellow/red) to make the bar scannable at a glance
- **Never use `|| echo 0` with `grep -c`** — `grep -c` always outputs a count
  (even 0), but returns exit code 1 when the count is 0. The `|| echo 0` fallback
  fires on that exit code and appends a second `0`, producing `"0\n0"` — a literal
  newline embedded in your variable. This silently breaks line rendering. Use
  `grep -c . 2>/dev/null` without any `||` fallback.
- Test with: `echo '{"model":{"id":"claude-opus-4-6","display_name":"Opus"}}' | ~/.claude/statusline.sh`

### 3. Write and configure

1. Write the script to `~/.claude/statusline.sh`
2. Make it executable: `chmod +x ~/.claude/statusline.sh`
3. Ensure `settings.json` has the statusLine config (check both user and project scope)
4. Tell the user changes appear after the next Claude response

### Cross-platform notes

- `stat -f %m` (macOS) vs `stat -c %Y` (Linux) — use fallback pattern
- `date +%s` works on both
- `jq` is required — if not installed, suggest `brew install jq` or `apt install jq`
- The script runs in the user's default shell environment

## Modifying an Existing Statusline

When the user wants to add/change/remove a specific element:

1. Read `~/.claude/statusline.sh` first
2. Identify the relevant module/section
3. Edit surgically — don't rewrite the whole file
4. If adding a new module, insert the JSON parsing near the top with the other
   `jq` calls, and add the display logic in the appropriate row

## Session Name Banner

The session name banner is a centered divider that appears across the terminal
(separate from the statusline). It shows the session name and can be colored.

### Commands

- `/rename my-feature` — set a custom session name, banner appears immediately
- `/rename` (no args) — auto-generates a name from conversation history
- `/color red` — color the session banner (red, blue, green, yellow, purple, orange, pink, cyan, default)

### Auto-Naming Every Session

There's no built-in setting to auto-name sessions. The banner only appears when
you explicitly use `--name` at launch or `/rename` during the session.

To get the banner automatically on every session, add a shell wrapper to `~/.zshrc`
(or `~/.bashrc`) that overrides the `claude` command:

```bash
# Auto-named Claude Code sessions — always shows the session banner
claude() {
  # Skip auto-naming for non-session commands and when --name is already set
  if [[ "$*" == *"--resume"* ]] || [[ "$*" == *"--name"* ]] || \
     [[ "$1" == "plugin" ]] || [[ "$1" == "config" ]] || [[ "$1" == "mcp" ]]; then
    command claude "$@"
    return
  fi
  local name=$(git rev-parse --abbrev-ref HEAD 2>/dev/null \
    | sed 's|^feature/||;s|^fix/||;s|^chore/||')
  [ -z "$name" ] || [ "$name" = "main" ] || [ "$name" = "master" ] \
    && name=$(basename "$PWD")
  command claude --name "$name" "$@"
}
```

This derives the session name from the git branch (stripped of prefixes) or
project directory. `command claude` calls the real binary to avoid recursion.
It skips auto-naming for `--resume`, `--name`, `plugin`, `config`, and `mcp`
subcommands.

After adding, run `source ~/.zshrc` to activate. Every `claude` invocation
will show the banner immediately.

### Session Label in the Statusline

The Session Label module (see `references/modules.md`) shows the session name
in the statusline's bottom bar — complementary to the banner. It uses the same
fallback chain: explicit name > latest commit message > branch > project dir.
The banner and the statusline label can coexist — one at the top, one at the bottom.

## Troubleshooting

Common issues:
- **Nothing appears:** Check `settings.json` has `statusLine.command` set, script is executable
- **Garbled output:** ANSI codes not being interpreted — ensure `printf "%b\n"` not `echo`
- **Line wraps unexpectedly:** Check for embedded newlines in variables. The most
  common cause: `grep -c ... || echo 0` produces `"0\n0"` because `grep -c`
  returns exit code 1 when count is 0, triggering the fallback. Fix: remove the
  `|| echo 0`. Debug with: `printf '%s' "$VAR" | xxd` to spot `0a` bytes.
- **Stale data:** Script only runs after Claude responds (unless `refreshInterval` is set)
- **Slow script:** Too many `jq` calls or expensive git commands — batch jq queries, cache git output
