# Statusline JSON Fields Reference

Complete reference of every field Claude Code passes to the statusline script
via stdin. All fields use `jq` dot notation.

## Model

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `model.id` | string | `"claude-opus-4-6"` | Full model identifier |
| `model.display_name` | string | `"Opus"` | Human-readable name |

## Workspace

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `cwd` | string | `"/Users/me/project"` | Alias for `workspace.current_dir` |
| `workspace.current_dir` | string | `"/Users/me/project"` | Preferred over `cwd` |
| `workspace.project_dir` | string | `"/Users/me/project"` | Directory where Claude Code was launched |
| `workspace.added_dirs` | array | `["/other/dir"]` | Dirs added via `/add-dir` or `--add-dir` |
| `workspace.git_worktree` | string | `"review-wt"` | Only present inside a linked git worktree |

## Cost & Duration

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `cost.total_cost_usd` | number | `0.4231` | Cumulative session cost |
| `cost.total_duration_ms` | number | `45000` | Total wall-clock time (ms) |
| `cost.total_api_duration_ms` | number | `38000` | Time waiting for API (ms) |
| `cost.total_lines_added` | number | `142` | Lines of code added |
| `cost.total_lines_removed` | number | `37` | Lines of code removed |

## Context Window

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `context_window.total_input_tokens` | number | `84000` | Cumulative input tokens |
| `context_window.total_output_tokens` | number | `12000` | Cumulative output tokens |
| `context_window.context_window_size` | number | `200000` | Max context (200K or 1M for extended) |
| `context_window.used_percentage` | number | `42.5` | Percentage of context used |
| `context_window.remaining_percentage` | number | `57.5` | Percentage remaining |

### Current Usage (last API call)

| Field | Type | Notes |
|-------|------|-------|
| `context_window.current_usage.input_tokens` | number | Input tokens in current context |
| `context_window.current_usage.output_tokens` | number | Output tokens generated |
| `context_window.current_usage.cache_creation_input_tokens` | number | Tokens written to cache |
| `context_window.current_usage.cache_read_input_tokens` | number | Tokens read from cache |

`current_usage` is `null` before the first API call.

## Rate Limits (Claude.ai Pro/Max only)

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `rate_limits.five_hour.used_percentage` | number | `45.2` | 5-hour window usage (0-100) |
| `rate_limits.five_hour.resets_at` | number | `1712345678` | Unix epoch when window resets |
| `rate_limits.seven_day.used_percentage` | number | `12.1` | 7-day window usage (0-100) |
| `rate_limits.seven_day.resets_at` | number | `1712900000` | Unix epoch when window resets |

Only present for Claude.ai Pro/Max users, after the first API response.

## Session

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `session_id` | string | `"abc123def456"` | Unique, stable for session lifetime |
| `session_name` | string | `"my-analysis"` | Set via `--name` or `/rename`; absent if not set |
| `transcript_path` | string | `"/path/to/transcript.json"` | Path to conversation transcript |

## Version & Style

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `version` | string | `"1.2.3"` | Claude Code version |
| `output_style.name` | string | `"default"` | Current output style |

## Vim Mode

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `vim.mode` | string | `"NORMAL"` or `"INSERT"` | Only present when vim mode enabled |

## Agent

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `agent.name` | string | `"review-agent"` | Only present when `--agent` flag used |

## Worktree

Only present during `--worktree` sessions.

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `worktree.name` | string | `"review"` | Worktree name |
| `worktree.path` | string | `"/path/to/worktree"` | Absolute path |
| `worktree.branch` | string | `"feature/x"` | Branch name (may be absent for hook-based) |
| `worktree.original_cwd` | string | `"/path/to/original"` | Dir before entering worktree |
| `worktree.original_branch` | string | `"main"` | Branch before entering worktree |

## Other

| Field | Type | Notes |
|-------|------|-------|
| `exceeds_200k_tokens` | boolean | True when total tokens exceed 200K threshold |

## Fields That May Be Absent

These fields are only present in specific conditions:
- `session_name` — only when explicitly set
- `workspace.git_worktree` — only inside a linked git worktree
- `vim` — only when vim mode is enabled
- `agent` — only when `--agent` flag or agent settings configured
- `worktree.*` — only during `--worktree` sessions
- `rate_limits` — only for Claude.ai Pro/Max, after first API response

Always use `// ""` or `// 0` defaults in jq to handle absent fields gracefully.
