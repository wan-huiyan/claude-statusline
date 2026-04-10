# Statusline Module Library

Copy-pasteable bash snippets for each statusline feature. Each module has
a **Parse** section (add near top of script) and a **Display** section
(add where you want the output).

---

## Session Label

Auto-derives a meaningful conversation name: explicit session name > latest
commit message > branch name > project directory.

**Parse:**
```bash
SESSION_NAME=$(echo "$input" | jq -r '.session_name // ""')
SESSION_ID=$(echo "$input" | jq -r '.session_id // ""')
```

**Display:**
```bash
CONV_LABEL=""
if [ -n "$SESSION_NAME" ]; then
    CONV_LABEL="${C}📝 ${SESSION_NAME}${RST}"
else
    LATEST_COMMIT=""
    if [ -n "$GIT_DIR" ]; then
        LATEST_COMMIT=$(git -C "$CWD" log -1 --format='%s' 2>/dev/null | cut -c1-50)
    fi
    if [ -n "$LATEST_COMMIT" ]; then
        CONV_LABEL="${C}📝 ${LATEST_COMMIT}${RST}"
    elif [ -n "$BRANCH" ] && [ "$BRANCH" != "HEAD" ] && [ "$BRANCH" != "main" ] && [ "$BRANCH" != "master" ]; then
        BRANCH_SHORT=$(echo "$BRANCH" | sed 's|^feature/||;s|^fix/||;s|^chore/||;s|^bugfix/||;s|^hotfix/||')
        CONV_LABEL="${C}📝 ${BRANCH_SHORT}${RST}"
    elif [ -n "$CWD" ]; then
        CONV_LABEL="${C}📝 $(basename "$CWD")${RST}"
    fi
fi
```

**Depends on:** Git Branch module (for `$GIT_DIR`, `$BRANCH`, `$CWD`)

---

## Git Branch & Worktree

Shows current branch, worktree indicator, or detached HEAD state.

**Parse:**
```bash
CWD=$(echo "$input" | jq -r '.workspace.current_dir // ""')
WT_NAME=$(echo "$input" | jq -r '.worktree.name // ""')
WT_BRANCH=$(echo "$input" | jq -r '.worktree.branch // ""')
```

**Display:**
```bash
GIT_DIR=""
[ -n "$CWD" ] && GIT_DIR=$(git -C "$CWD" rev-parse --git-dir 2>/dev/null)

if [ -n "$GIT_DIR" ]; then
    if [ -n "$WT_BRANCH" ]; then
        BRANCH="$WT_BRANCH"
    else
        BRANCH=$(git -C "$CWD" rev-parse --abbrev-ref HEAD 2>/dev/null)
    fi
    if [ -n "$WT_NAME" ]; then
        WT_LABEL="${M}⎇ wt:${WT_NAME}${RST} ${DIM}(${BRANCH})${RST}"
    elif [ -n "$BRANCH" ] && [ "$BRANCH" != "HEAD" ]; then
        WT_LABEL="${M}⎇ ${BRANCH}${RST}"
    else
        WT_LABEL="${M}⎇ (detached)${RST}"
    fi
else
    WT_LABEL="${DIM}(no git)${RST}"
fi
```

---

## Git Dirty Status

Counts changed and untracked files, shows green check or yellow dot.

**Display** (requires Git Branch module):
```bash
if [ -n "$GIT_DIR" ]; then
    GIT_STATUS=$(git -C "$CWD" status --porcelain 2>/dev/null)
    DIRTY_COUNT=$(echo "$GIT_STATUS" | grep -v '^??' | grep -c . 2>/dev/null)
    UNTRACKED=$(echo "$GIT_STATUS" | grep -c '^??' 2>/dev/null)
    if [ "$DIRTY_COUNT" -gt 0 ]; then DIRTY_CLR="$Y"; DIRTY_SYM="●"
    else                               DIRTY_CLR="$G"; DIRTY_SYM="✓"; fi
    DIRTY_LABEL="${DIRTY_CLR}${DIRTY_SYM} ${DIRTY_COUNT} changed${RST}"
    [ "$UNTRACKED" -gt 0 ] && DIRTY_LABEL="${DIRTY_LABEL} ${DIM}+${UNTRACKED} new${RST}"
else
    DIRTY_LABEL=""
fi
```

---

## Model Info

Shows model name and version number.

**Parse:**
```bash
MODEL_ID=$(echo "$input" | jq -r '.model.id // ""')
MODEL_NAME=$(echo "$input" | jq -r '.model.display_name // "Claude"')
```

**Display:**
```bash
MODEL_SHORT="$MODEL_NAME"
if echo "$MODEL_ID" | grep -qi "opus";   then MODEL_SHORT="Opus";   fi
if echo "$MODEL_ID" | grep -qi "sonnet"; then MODEL_SHORT="Sonnet"; fi
if echo "$MODEL_ID" | grep -qi "haiku";  then MODEL_SHORT="Haiku";  fi
VER=$(echo "$MODEL_ID" | grep -oE '[0-9]+[-._][0-9]+' | head -1 | tr '-' '.')
[ -n "$VER" ] && MODEL_LABEL="${MODEL_SHORT} ${VER}" || MODEL_LABEL="$MODEL_SHORT"

MODEL_OUT="${BLD}${MODEL_LABEL}${RST}"
```

---

## Effort Level

Reads the effort level from `~/.claude/settings.json` (`effortLevel` key: low/medium/high).
This is the authoritative source — do NOT hardcode effort based on model name.

**Display:**
```bash
EFFORT_LEVEL=$(jq -r '.effortLevel // ""' "${HOME}/.claude/settings.json" 2>/dev/null | tr -d '[:space:]')

RE_CLR="$W"
case "$EFFORT_LEVEL" in
    low)    RE_CLR="$G" ;;
    medium) RE_CLR="$Y" ;;
    high)   RE_CLR="$R" ;;
esac

EFFORT_LABEL=""
[ -n "$EFFORT_LEVEL" ] && EFFORT_LABEL="${RE_CLR}[${EFFORT_LEVEL}]${RST}"
```

---

## Cost

Session cost in USD, formatted to 4 decimal places.

**Parse:**
```bash
COST_RAW=$(echo "$input" | jq -r '.cost.total_cost_usd // 0')
```

**Display:**
```bash
COST_FMT=$(printf '$%.4f' "$COST_RAW")
COST_LABEL="${Y}${COST_FMT}${RST}"
```

---

## Duration

Total API wall-clock time in minutes and seconds.

**Parse:**
```bash
DURATION_MS=$(echo "$input" | jq -r '.cost.total_duration_ms // 0')
```

**Display:**
```bash
DURATION_SEC=$(( ${DURATION_MS:-0} / 1000 ))
MINS=$(( DURATION_SEC / 60 ))
SECS=$(( DURATION_SEC % 60 ))
DUR_LABEL="${DIM}${MINS}m${SECS}s${RST}"
```

---

## Lines Changed

Lines added/removed this session (proxy for session productivity).

**Parse:**
```bash
LINES_ADDED=$(echo "$input" | jq -r '.cost.total_lines_added // 0')
LINES_REMOVED=$(echo "$input" | jq -r '.cost.total_lines_removed // 0')
```

**Display:**
```bash
PLAN_LABEL=""
if [ "${LINES_ADDED:-0}" -gt 0 ] 2>/dev/null || [ "${LINES_REMOVED:-0}" -gt 0 ] 2>/dev/null; then
    PLAN_LABEL="${G}+${LINES_ADDED}${RST}/${R}-${LINES_REMOVED}${RST} lines"
fi
```

---

## Context Bar

Visual 10-block progress bar showing context window usage with compaction warning.

**Parse:**
```bash
CTX_USED=$(echo "$input" | jq -r '.context_window.used_percentage // 0' | cut -d. -f1)
CTX_REM=$(echo "$input" | jq -r '.context_window.remaining_percentage // 100' | cut -d. -f1)
```

**Display:**
```bash
CTX_BAR=""
for i in 1 2 3 4 5 6 7 8 9 10; do
    filled=$(( i * 10 ))
    if [ "${CTX_USED:-0}" -ge "$filled" ] 2>/dev/null; then
        CTX_BAR="${CTX_BAR}█"
    else
        CTX_BAR="${CTX_BAR}░"
    fi
done
if   [ "${CTX_REM:-100}" -le 5 ] 2>/dev/null;  then CTX_CLR="$R"
elif [ "${CTX_REM:-100}" -le 10 ] 2>/dev/null;  then CTX_CLR="$Y"
else CTX_CLR="$G"; fi

COMPACT_WARN=""
if [ "${CTX_REM:-100}" -le 10 ] 2>/dev/null; then
    COMPACT_WARN=" ${R}⚡compact ~${CTX_REM}%${RST}"
fi
CTX_LABEL="${CTX_CLR}ctx:${CTX_USED}%${RST} ${CTX_BAR} ${DIM}${CTX_REM}% left${RST}${COMPACT_WARN}"
```

---

## Token Counts

Cumulative input and output tokens, formatted as K (thousands).

**Parse:**
```bash
IN_TOKENS=$(echo "$input" | jq -r '.context_window.total_input_tokens // 0')
OUT_TOKENS=$(echo "$input" | jq -r '.context_window.total_output_tokens // 0')
```

**Display:**
```bash
IN_K=$(( ${IN_TOKENS:-0} / 1000 ))
OUT_K=$(( ${OUT_TOKENS:-0} / 1000 ))
TOKEN_LABEL="${DIM}in:${RST}${W}${IN_K}K${RST} ${DIM}out:${RST}${W}${OUT_K}K${RST}"
```

---

## Cache Ratio

Shows cache hit percentage from the last API call — indicates how effectively
prompt caching is being used.

**Parse:**
```bash
CACHE_CREATE=$(echo "$input" | jq -r '.context_window.current_usage.cache_creation_input_tokens // 0')
CACHE_READ=$(echo "$input" | jq -r '.context_window.current_usage.cache_read_input_tokens // 0')
```

**Display:**
```bash
CACHE_LABEL=""
CACHE_TOTAL=$(( ${CACHE_CREATE:-0} + ${CACHE_READ:-0} ))
if [ "$CACHE_TOTAL" -gt 0 ] 2>/dev/null; then
    CACHE_PCT=$(( ${CACHE_READ:-0} * 100 / CACHE_TOTAL ))
    if   [ "$CACHE_PCT" -ge 70 ]; then CACHE_CLR="$G"
    elif [ "$CACHE_PCT" -ge 30 ]; then CACHE_CLR="$Y"
    else                                CACHE_CLR="$R"; fi
    CACHE_LABEL="${CACHE_CLR}cache:${CACHE_PCT}%${RST}"
fi
```

---

## Rate Limits (Claude.ai Pro/Max only)

Shows 5-hour and 7-day rate limit usage with time until reset.

**Parse:**
```bash
RL_5H=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // ""')
RL_5H_RESET=$(echo "$input" | jq -r '.rate_limits.five_hour.resets_at // ""')
RL_7D=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // ""')
```

**Display:**
```bash
RL_LABEL=""
if [ -n "$RL_5H" ]; then
    RL_5H_INT=$(echo "$RL_5H" | cut -d. -f1)
    if   [ "${RL_5H_INT:-0}" -ge 80 ] 2>/dev/null; then RL_CLR="$R"
    elif [ "${RL_5H_INT:-0}" -ge 50 ] 2>/dev/null; then RL_CLR="$Y"
    else                                                  RL_CLR="$G"; fi
    RL_LABEL="${RL_CLR}5h:${RL_5H_INT}%${RST}"
    # Add reset countdown
    if [ -n "$RL_5H_RESET" ] && [ "${RL_5H_INT:-0}" -ge 50 ] 2>/dev/null; then
        NOW=$(date +%s)
        RESET_IN=$(( ${RL_5H_RESET:-0} - NOW ))
        if [ "$RESET_IN" -gt 0 ] 2>/dev/null; then
            RESET_MINS=$(( RESET_IN / 60 ))
            RL_LABEL="${RL_LABEL}${DIM}(${RESET_MINS}m)${RST}"
        fi
    fi
    if [ -n "$RL_7D" ]; then
        RL_7D_INT=$(echo "$RL_7D" | cut -d. -f1)
        RL_LABEL="${RL_LABEL} ${DIM}7d:${RL_7D_INT}%${RST}"
    fi
fi
```

---

## Memory Freshness

Age of global lessons.md and nearest project memory — helps spot stale context.

**Display:**
```bash
now_epoch=$(date +%s)
fmt_age() {
    local path="$1"
    [ ! -f "$path" ] && echo "n/a" && return
    local mod_epoch
    mod_epoch=$(stat -f %m "$path" 2>/dev/null) || mod_epoch=$(stat -c %Y "$path" 2>/dev/null)
    [ -z "$mod_epoch" ] && echo "?" && return
    local diff=$(( now_epoch - mod_epoch ))
    if   [ "$diff" -lt 60 ];    then echo "${diff}s"
    elif [ "$diff" -lt 3600 ];  then echo "$(( diff / 60 ))m"
    elif [ "$diff" -lt 86400 ]; then echo "$(( diff / 3600 ))h"
    else                             echo "$(( diff / 86400 ))d"
    fi
}

LESSONS_AGE=$(fmt_age "${HOME}/.claude/lessons.md")

PROJ_MEMORY_AGE="n/a"
if [ -n "$CWD" ]; then
    CHECK_PATH="$CWD"
    while [ "$CHECK_PATH" != "/" ] && [ -n "$CHECK_PATH" ]; do
        PROJ_KEY=$(echo "$CHECK_PATH" | sed 's|/|-|g')
        PROJ_MEM_DIR="${HOME}/.claude/projects/${PROJ_KEY}/memory"
        if [ -d "$PROJ_MEM_DIR" ]; then
            NEWEST_MEM=$(find "$PROJ_MEM_DIR" -name "*.md" -type f -exec stat -f '%m %N' {} + 2>/dev/null | sort -rn | head -1 | cut -d' ' -f2-)
            [ -n "$NEWEST_MEM" ] && PROJ_MEMORY_AGE=$(fmt_age "$NEWEST_MEM")
            break
        fi
        CHECK_PATH=$(dirname "$CHECK_PATH")
    done
fi
MEM_LABEL="${DIM}lessons:${RST}${W}${LESSONS_AGE}${RST}  ${DIM}proj-mem:${RST}${W}${PROJ_MEMORY_AGE}${RST}"
```

---

## Agent / Vim / Worktree Flags

Shows active agent name, vim mode, and worktree indicator.

**Parse:**
```bash
AGENT_NAME=$(echo "$input" | jq -r '.agent.name // ""')
VIM_MODE=$(echo "$input" | jq -r '.vim.mode // ""')
```

**Display:**
```bash
FLAGS=""
[ -n "$AGENT_NAME" ] && FLAGS="${C}⚙ agent:${AGENT_NAME}${RST}"
[ -n "$VIM_MODE" ]   && FLAGS="${FLAGS:+${FLAGS} }${B}vim:${VIM_MODE}${RST}"
[ -n "$WT_NAME" ]    && FLAGS="${FLAGS:+${FLAGS} }${M}worktree${RST}"
```

---

## Version

Claude Code version number.

**Parse:**
```bash
CC_VERSION=$(echo "$input" | jq -r '.version // ""')
```

**Display:**
```bash
VER_LABEL=""
[ -n "$CC_VERSION" ] && VER_LABEL="${DIM}v${CC_VERSION}${RST}"
```

---

## ANSI Color Palette

Standard palette used by all modules. Add at top of script.

```bash
R='\033[0;31m'   # Red    — errors, high thresholds
G='\033[0;32m'   # Green  — success, low thresholds
Y='\033[0;33m'   # Yellow — warnings, medium thresholds
B='\033[0;34m'   # Blue   — vim mode, info
M='\033[0;35m'   # Magenta — git branch, worktree
C='\033[0;36m'   # Cyan   — session label, agent
W='\033[0;37m'   # White  — neutral values
BLD='\033[1m'    # Bold   — model name
DIM='\033[2m'    # Dim    — labels, secondary info
RST='\033[0m'    # Reset
```
