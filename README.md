# claude-statusline

Build and customize the Claude Code status line (bottom info bar) and session name banner.

[![license](https://img.shields.io/github/license/wan-huiyan/claude-statusline)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-orange)](https://claude.com/claude-code)

## Quick Start

```
You: set up my statusline with git branch, cost, model info, and context usage
Claude: [reads SKILL.md, generates ~/.claude/statusline.sh with 4 modules, configures settings.json]

You: add cache hit ratio to my status bar
Claude: [reads existing script, surgically adds cache module to Row 2]

You: I want a minimal one-row statusline with just model and context
Claude: [generates a 5-line script with single printf output]
```

## Installation

**Claude Code (plugin install — recommended):**
```bash
# Add the marketplace, then install the plugin
claude plugin marketplace add wan-huiyan/claude-statusline
claude plugin install claude-statusline@wan-huiyan-claude-statusline
```

**Claude Code (git clone):**
```bash
git clone https://github.com/wan-huiyan/claude-statusline.git ~/.claude/skills/claude-statusline
```

**Cursor** (2.4+):
```bash
# Per-project rule (most reliable)
mkdir -p .cursor/rules
# Copy plugins/claude-statusline/SKILL.md content into .cursor/rules/claude-statusline.mdc with alwaysApply: true

# Or via npx skills CLI
npx skills add wan-huiyan/claude-statusline --global
```

## What You Get

**14 composable statusline modules** — pick what you want, arrange into rows:

| Module | Example Output |
|--------|---------------|
| Session Label | `📝 fix auth token refresh` |
| Git Branch | `⎇ feature/new-api` |
| Git Dirty | `● 3 changed +2 new` |
| Model Info | `Opus 4.6` |
| Effort Level | `[medium]` |
| Cost | `$0.4231` |
| Duration | `12m34s` |
| Lines Changed | `+142/-37 lines` |
| Context Bar | `ctx:42% ████░░░░░░ 58% left` |
| Token Counts | `in:84K out:12K` |
| Cache Ratio | `cache:73%` |
| Rate Limits | `5h:45% 7d:12%` |
| Memory Freshness | `lessons:2h proj-mem:15m` |
| Agent/Vim/Worktree | `⚙ agent:review vim:NORMAL` |

**Session name banner** with auto-naming:

- `/rename my-feature` — set a custom name
- `/rename` (no args) — auto-generate from conversation history
- `/color red` — color the banner
- Shell wrapper to auto-name every session from git branch

**Complete JSON field reference** — all 30+ fields Claude Code passes to statusline scripts.

## Typical Ad-Hoc vs With Skill

| | Ad-hoc | With claude-statusline |
|---|---|---|
| JSON field names | Guess or search docs | Complete reference with types and examples |
| Modules | Write from scratch each time | Copy-paste from 14 pre-built modules |
| Session naming | Manual `/rename` | Auto-derived from git branch / commit message |
| Pitfalls | Hit `grep -c \|\| echo 0` newline bug | Documented with fix and debug command |
| Cross-platform | Forget `stat` differs on macOS/Linux | Fallback patterns included |

## How It Works

Claude Code runs `~/.claude/statusline.sh` after every assistant response, piping session data as JSON to stdin. The script parses it with `jq`, formats with ANSI colors, and prints rows via `printf "%b\n"`.

The skill teaches Claude how to:
1. **Generate** a new statusline from user preferences (minimal to full)
2. **Modify** an existing statusline surgically (add/remove modules)
3. **Debug** common issues (newline bugs, ANSI rendering, stale data)
4. **Configure** the session name banner and auto-naming

## Key Design Decisions

- **Modular architecture** — each feature is a self-contained bash snippet with Parse + Display sections, so they compose without dependencies
- **Smart session label** — fallback chain (explicit name > commit message > branch > project dir) ensures something meaningful always shows
- **Color-coded thresholds** — green/yellow/red at consistent breakpoints across all modules for at-a-glance scanning
- **`grep -c` warning** — the skill explicitly warns against `grep -c ... || echo 0`, a pattern that injects newlines into variables and silently breaks line rendering

## Limitations

- Requires `jq` for JSON parsing (no pure-bash alternative provided)
- statusline.sh only runs after Claude responds (or on `refreshInterval` timer) — no real-time updates
- Session naming requires a shell wrapper; no native Claude Code setting for auto-naming
- Rate limit fields only available on Claude.ai Pro/Max plans, not API
- No support for multi-line module output (each module contributes to a single row)

<details>
<summary>Quality Checklist</summary>

- [x] All 30+ JSON fields documented with types, examples, and absence conditions
- [x] 14 modules with complete, tested bash code
- [x] Cross-platform stat fallback (macOS + Linux)
- [x] `grep -c || echo 0` pitfall documented with root cause and fix
- [x] Session banner section with `/rename`, `/color`, and auto-naming wrapper
- [x] Troubleshooting section for 5 common issues
- [x] Test instructions with mock JSON payload

</details>

## Version History

- **v1.1.0** (2026-04-10) — Fix effort level: read from `settings.json` `effortLevel` instead of hardcoding by model name
- **v1.0.0** (2026-04-09) — Initial release: 14 modules, session banner, `grep -c` pitfall fix, complete JSON field reference

## License

MIT
