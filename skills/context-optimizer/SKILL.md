---
name: context-optimizer
description: Audit and optimize Claude Code context window usage — plugins, hooks, CLAUDE.md, MCP servers, env settings. Produces a scored report with actionable fixes. Use when user says "optimize context", "reduce tokens", "why so expensive", "token audit", or "context bloat".
tools: Read, Glob, Grep, Bash, Edit, WebSearch
---

# Context Optimizer

Audit Claude Code's context overhead and produce an actionable optimization plan. Based on real-world analysis that achieved 40-50% token reduction.

## When to Use

- Token costs are unexpectedly high
- Sessions hit context limits too quickly
- User wants to audit plugin/hook overhead
- Before starting a long-running agentic workflow (ralph, autopilot, ultrawork)

## Baseline Reference

A fresh Claude Code session in an empty directory uses ~16K tokens. A real project adds ~7K more. Anything above ~25K startup overhead indicates optimization opportunity.

| Scenario | Expected Startup Tokens |
|----------|------------------------|
| Empty dir, default | ~16,000 |
| Empty dir, --tools='' | ~5,900 |
| Real project, default | ~23,000 |
| Real project, stripped | ~12,000 |
| Heavy plugins (10+) | ~35,000-50,000+ |

## Workflow

### Phase 0: Historical Cost Analysis with ccusage

Before auditing, measure actual token spending using [ccusage](https://github.com/LEON-gittech/ccusage) (forked with 14x cache speedup):

```bash
# Install (one-time)
npm install -g ccusage   # or use our fork: gh repo clone LEON-gittech/ccusage

# Daily cost breakdown by model
ccusage --period day

# Weekly trend
ccusage --period week

# Identify which days/models consume most tokens
# Look for: cache_write >> cache_read (indicates poor cache reuse from too many subagents)
# Look for: high total tokens on days with agentic workflows (ralph, autopilot)
```

**Key metrics to capture:**
- Daily total tokens and cost
- Cache write vs cache read ratio (target: read > 5x write = good cache reuse)
- Model distribution (Opus vs Sonnet vs Haiku — is Opus overused?)

### Phase 1: Measure Current State

Run these commands to establish baseline:

```bash
# 1. Count enabled plugins
grep -c '"true"' ~/.claude/settings.json

# 2. Count total hooks across all sources
find ~/.claude/plugins/cache -name "hooks.json" -exec cat {} \; | grep '"type"' | wc -l

# 3. Measure CLAUDE.md cascade size
find . -name "CLAUDE.md" -exec wc -c {} \; 2>/dev/null
wc -c ~/.claude/CLAUDE.md 2>/dev/null

# 4. Count MCP servers
grep -c '"command"' ~/.claude/settings.json

# 5. Check for .claudeignore
test -f .claudeignore && echo "EXISTS" || echo "MISSING"
```

### Phase 2: Audit Plugins

Read `~/.claude/settings.json` and list all `enabledPlugins`. For each enabled plugin:

1. **Find its hooks**: `~/.claude/plugins/cache/<marketplace>/<name>/<version>/hooks/hooks.json`
2. **Count hook events**: How many hook types does it register?
3. **Check matcher scope**: `"*"` or `""` matchers fire on EVERY tool call (most expensive)
4. **Assess relevance**: Does this plugin serve the current project's tech stack?

**Scoring per plugin:**

| Criteria | Score |
|----------|-------|
| Has hooks with `*` matcher on PostToolUse | -3 (fires every tool call) |
| Has hooks with `*` matcher on PreToolUse | -2 (fires every tool call) |
| Has Stop hook | -1 (fires every stop attempt) |
| Has SessionStart hook | -1 (fires every session/subagent) |
| No hooks (pure skill) | 0 (zero overhead when not invoked) |
| Relevant to current project tech stack | +2 |
| Not relevant to current project | -2 |

**Automatic red flags:**
- PostToolUse hook with timeout > 30s (blocking)
- Plugin for wrong tech stack (e.g., rust-analyzer on Python project)
- Multiple plugins with overlapping Stop hooks (persistence conflicts)
- SessionStart hooks that inject large context blobs

### Phase 3: Audit Hooks

Compile a complete hook inventory from ALL sources:

1. **User settings**: `~/.claude/settings.json` → `hooks` section
2. **Each enabled plugin**: `hooks.json` in plugin cache
3. **Project settings**: `.claude/settings.json` in project root

For each hook event, count total hooks and identify duplicates:

```
Event               | Total Hooks | Critical Duplicates?
--------------------|-------------|--------------------
SessionStart        |     ?       | Multiple context injectors?
UserPromptSubmit    |     ?       | Multiple keyword detectors?
PreToolUse          |     ?       | Multiple enforcers on same matcher?
PostToolUse         |     ?       | Multiple observers on * matcher?
Stop                |     ?       | Multiple persistent-mode hooks?
SubagentStart/Stop  |     ?       |
PreCompact          |     ?       |
SessionEnd          |     ?       |
```

**Per-tool-call overhead formula:**
```
hooks_per_call = PreToolUse_hooks + PostToolUse_hooks
tokens_per_call = hooks_per_call * avg_hook_output_tokens (~150-300)
iteration_overhead = tool_calls_per_iteration * tokens_per_call
```

**Duplicate detection rules:**
- Two persistent-mode Stop hooks → remove one
- PreToolUse enforcer + separate prompt-guard on Write|Edit → redundant
- PostToolUse context-monitor + Stop context-guard → overlapping
- Multiple SessionStart context injectors → likely redundant
- PostToolUse hook spawning `claude -p` sub-processes → each trigger creates new processes with independent context windows. Unless explicitly needed, remove.

### Phase 4: Audit CLAUDE.md

Find and analyze all CLAUDE.md files in the hierarchy:

```bash
# Discovery
find / -maxdepth 6 -name "CLAUDE.md" 2>/dev/null | head -20
ls ~/.claude/CLAUDE.md 2>/dev/null
```

For each file, check:

| Criterion | Target | Check |
|-----------|--------|-------|
| Size | < 5K tokens (~20KB) | `wc -c` |
| Duplication | 0 repeated rules across levels | Diff adjacent levels |
| Relevance | 100% relevant to project | Scan for wrong tech stack refs |
| Hierarchy | Each level adds unique value only | No copy-paste from parent |

**Common CLAUDE.md bloat patterns:**
- Behavioral rules repeated at every level (should only be in top-level)
- Build commands for wrong tech stack (npm in Python project)
- Full MCP server table duplicated across files
- Security rules copied verbatim from parent
- Large architecture docs that should be in separate files
- "Important Instruction Reminders" sections that duplicate earlier rules

**Target CLAUDE.md cascade:**
```
L1 Global (~/.claude/CLAUDE.md)     — framework config, 1-2K tokens max
L2 Parent (project root/CLAUDE.md)  — behavioral rules, security, generic setup
L3 Worktree/Package (CLAUDE.md)     — ONLY unique additions, reference parent
L4 Subdir (CLAUDE.md)               — ONLY subdir-specific context
```

### Phase 5: Audit MCP Servers & Environment

Check for unused MCP servers and env overhead:

```bash
# List MCP servers from settings
grep -A3 '"mcpServers"' ~/.claude/settings.json

# Check env vars that affect context
env | grep -i "CLAUDE\|ANTHROPIC" | sort
```

**Key env optimizations:**
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` — compact earlier (default 80%)
- `MAX_THINKING_TOKENS=10000` — cap extended thinking
- `CLAUDE_CODE_SUBAGENT_MODEL=haiku` — cheaper subagents for simple tasks
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` — if using external memory system

**MCP overhead**: Each enabled MCP server adds tool definitions to system prompt. Disable unused servers with `disabledMcpServers` in project `.claude/settings.json`.

**MCP connection churn**: MCP servers can drop and reconnect in loops, causing I/O bloat and CPU spikes. Check:
```bash
# Large debug files indicate connection errors
find ~/.claude/debug/ -size +500k -ls
grep -l "Connection error\|token expired" ~/.claude/debug/*.txt 2>/dev/null
```
If found, disable the unstable MCP server in settings.json.

### Phase 6: Check for .claudeignore

```bash
cat .claudeignore 2>/dev/null || echo "MISSING — should create"
```

**Essential .claudeignore entries:**
```
node_modules/
dist/
build/
.git/
*.lock
*.min.js
__pycache__/
*.pyc
.venv/
results/          # large generated outputs
*.sqlite
*.db
```

### Phase 7: Generate Report

Output a structured report:

```markdown
## Context Optimization Report

### Startup Overhead Estimate
- Plugins: X enabled (Y with hooks)
- Hooks: X total across all events
- CLAUDE.md cascade: X tokens
- MCP servers: X active
- Estimated startup: ~XK tokens

### Score: X/100

### Critical Issues (fix immediately)
1. [Issue] — [Impact] — [Fix]

### High Priority (significant savings)
1. [Issue] — [Impact] — [Fix]

### Medium Priority (incremental improvement)
1. [Issue] — [Impact] — [Fix]

### Estimated Savings
- Before: ~XK tokens/session startup
- After: ~XK tokens/session startup
- Per-tool-call reduction: X fewer hooks
- For agentic workflows (100 tool calls): ~XK tokens saved/iteration
```

### Phase 8: Apply Fixes (with user approval)

Present each fix and wait for user confirmation before applying:

1. **Plugin disables**: Edit `enabledPlugins` in settings.json
2. **Hook removals**: Edit hooks section in settings.json
3. **CLAUDE.md dedup**: Edit CLAUDE.md files to remove duplicates
4. **MCP disables**: Add `disabledMcpServers` to project settings
5. **Env additions**: Suggest env var changes
6. **Create .claudeignore**: Write if missing

## Anti-Patterns to Flag

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| PostToolUse hook with `*` matcher and >30s timeout | Blocks every tool call | Reduce timeout to 5-10s or narrow matcher |
| Plugin for wrong tech stack | Wastes startup tokens loading irrelevant tools | Disable plugin |
| CLAUDE.md > 5K tokens | Loaded every session, every subagent | Split into skills (load on demand) |
| Multiple persistent-mode Stop hooks | Race conditions, duplicate state reads | Keep one authoritative version |
| SessionStart hook injecting full file contents | Massive context per session | Use progressive disclosure (index → fetch) |
| No .claudeignore | Claude reads irrelevant files | Create with standard exclusions |
| All subagents use Opus | 5x cost vs Sonnet for simple tasks | Set CLAUDE_CODE_SUBAGENT_MODEL=haiku for simple work |
| Hooks from disabled plugins still running | Ghost hooks waste resources | Verify plugin disable removes hooks |
| `alwaysThinkingEnabled: true` | Adds 30-50% latency to every response | Set to `false`, use `/think` manually |
| `effortLevel: "high"` for all tasks | Wastes tokens on simple queries | Use `"medium"` for daily work |
| Zombie MCP processes (stale sessions) | 300+ processes, 30+ GB RAM, CPU contention | Kill old claude sessions, `pkill -f` stale MCPs |
| Zombie claude processes (>2 days old) | Each holds memory, MCP children, context state | `pgrep claude` + check age, kill old ones |
| Subagent session accumulation | 100+ session records clutter resume list | Limit parallel Agent calls, consolidate analysis |
| Stale persistent-mode state files | `active: true` in `.omc/state/` blocks session exit | Delete stale state files after workflow ends |
| PostToolUse hooks spawning `claude -p` sub-processes | Invisible process and context cost | Remove or gate behind explicit command |
| MCP server connection churn | Reconnection loops → I/O bloat, CPU spikes | Disable unstable servers, check debug logs |
| Unused `mcpServers` in settings.json | Spawns extra processes on every session | Remove entries not actively used |
| CWD deleted but process alive | Worktree removed, claude process lingers | Check `readlink /proc/$pid/cwd` for `(deleted)` |
| Ghost sessions from CronCreate spam | Thousands of junk sessions polluting devtools and wasting storage | Detect and purge ghost session JSONL files, kill source zombie process |

## Zombie Process Detection

A common hidden cost: old Claude sessions leave behind orphaned processes that accumulate over days/weeks.

### Zombie Claude Processes

Long-running sessions (from agentic workflows, tmux, persistent-mode hooks) can become zombies when their CWD is deleted, the workflow stalls, or the user forgets about them.

```bash
# List all claude processes with age, PID, and working directory
for pid in $(pgrep claude); do
  ppid=$(ps -o ppid= -p $pid 2>/dev/null | tr -d ' ')
  age=$(ps -o etime= -p $pid 2>/dev/null | tr -d ' ')
  cwd=$(readlink /proc/$pid/cwd 2>/dev/null)
  echo "PID=$pid AGE=$age CWD=$cwd"
done

# Quick check: any claude processes older than 1 day?
for pid in $(pgrep claude); do
  age=$(ps -o etimes= -p $pid 2>/dev/null | tr -d ' ')
  if [ "$age" -gt 86400 ] 2>/dev/null; then
    days=$((age / 86400))
    cwd=$(readlink /proc/$pid/cwd 2>/dev/null)
    echo "ZOMBIE: PID=$pid running ${days}d CWD=$cwd"
  fi
done
```

**Expected**: 1-3 claude processes (current session + maybe tmux). **Problem**: 5+ processes, or any running >2 days, or CWD showing `(deleted)`.

**Red flags:**
- `CWD` shows `(deleted)` — the worktree/branch was removed but the process lingers
- Age >2 days — likely a forgotten agentic workflow or stale persistent-mode session
- Multiple processes in the same directory — subagent that never terminated

**Fix**:
```bash
# Kill specific zombie (verify PID first with the detection script above)
kill <zombie-pid>

# Kill all claude processes older than 2 days (DANGEROUS — verify first!)
# for pid in $(pgrep claude); do
#   age=$(ps -o etimes= -p $pid 2>/dev/null | tr -d ' ')
#   [ "$age" -gt 172800 ] 2>/dev/null && echo "Would kill PID=$pid ($(readlink /proc/$pid/cwd 2>/dev/null))"
# done
```

### Zombie MCP Processes

Old Claude sessions also leave behind orphaned MCP server processes.

```bash
# Check for zombie MCP processes
ps aux | grep -c '[p]laywright-mcp'
ps aux | grep -c '[c]ontext7-mcp'
ps aux | grep -c '[s]equential-thinking'
ps aux | grep -c '[t]avily-mcp'

# Total memory used by MCP servers
ps aux | grep -E '(playwright-mcp|context7-mcp|sequential-thinking|tavily-mcp)' | grep -v grep | awk '{sum += $6} END {printf "%.0f MB\n", sum/1024}'
```

**Expected**: 0-2 of each MCP type. **Problem**: 10+ of any type indicates process leak.

**Fix**:
```bash
# Kill the parent claude process (preferred — MCPs die with it)
kill <old-claude-pid>

# Or clean up orphaned MCPs directly
pkill -f 'playwright-mcp'
pkill -f 'context7-mcp'
pkill -f 'mcp-server-sequential-thinking'
pkill -f 'tavily-mcp'
```

## Ghost Session Detection (CronCreate Spam)

Long-running Claude sessions (e.g., idle in tmux) may have `CronCreate` scheduled tasks that continue firing indefinitely, generating thousands of junk sessions every ~10 minutes. These sessions:
- Pollute `~/.claude/projects/` with 6-20 line JSONL files
- Show up in claude-devtools as fake sessions with canned analysis prompts
- Waste storage and API credits if the prompts actually execute

### Detection

```bash
# Count ghost sessions (first line is a canned analysis prompt)
find ~/.claude/projects/ -name "*.jsonl" | while read f; do
  head -1 "$f" | grep -q "Analyze this codebase for\|Analyze test coverage\|Analyze this codebase for performance" && echo "$f"
done | wc -l

# Check if new ones are still being created (run twice, 15min apart)
find ~/.claude/projects/ -name "*.jsonl" -mmin -15 | while read f; do
  head -1 "$f" | grep -q "Analyze this codebase for\|Analyze test coverage\|Analyze this codebase for performance" && echo "ACTIVE: $f"
done
```

### Cleanup

```bash
# Build list and delete
find ~/.claude/projects/ -name "*.jsonl" | while read f; do
  head -1 "$f" | grep -q "Analyze this codebase for\|Analyze test coverage\|Analyze this codebase for performance" && echo "$f"
done > /tmp/ghost_sessions.txt
xargs rm -f < /tmp/ghost_sessions.txt
echo "Deleted $(wc -l < /tmp/ghost_sessions.txt) ghost sessions"
```

### Known Ghost Session Patterns

| First Line Pattern | Source | Frequency |
|-------------------|--------|-----------|
| `"Analyze this codebase for security vulnerabilities..."` | CronCreate in stale session | ~10min |
| `"Analyze test coverage and identify gaps..."` | CronCreate in stale session | ~10min |
| `"Analyze this codebase for performance optimizations..."` | CronCreate in stale session | ~10min |

### Prevention

- Always `/exit` Claude instances in tmux before detaching
- Set `autoCompactThreshold` to limit runaway sessions
- Periodically check for ghost sessions as part of `ccusage` workflow
- Kill zombie Claude processes (see above) — ghost sessions stop when the source process dies

## Subagent Session Accumulation

When Claude Code uses the `Agent` tool to spawn subagents, each creates a session record in `~/.claude/projects/<project>/`. These pile up in the session resume list, making it hard to find real sessions.

### Detection

```bash
# Count total session directories for current project
PROJECT_DIR=$(echo "$PWD" | sed 's|/|-|g; s|^-||')
SESSION_DIR="$HOME/.claude/projects/-${PROJECT_DIR}"
echo "Total sessions: $(ls -d "$SESSION_DIR"/*/ 2>/dev/null | wc -l)"

# Count sessions with subagent children
SUBAGENT_PARENTS=0
TOTAL_SUBAGENTS=0
for dir in "$SESSION_DIR"/*/subagents/ 2>/dev/null; do
  count=$(ls "$dir"/*.meta.json 2>/dev/null | wc -l)
  if [ "$count" -gt 0 ]; then
    SUBAGENT_PARENTS=$((SUBAGENT_PARENTS + 1))
    TOTAL_SUBAGENTS=$((TOTAL_SUBAGENTS + count))
  fi
done
echo "Sessions with subagents: $SUBAGENT_PARENTS (total subagents: $TOTAL_SUBAGENTS)"
```

**Expected**: <50 session dirs. **Problem**: 100+ sessions means heavy subagent usage is accumulating records.

### Common Causes

| Cause | Description | Solution |
|-------|-------------|----------|
| Agentic workflows (autopilot, ultrawork, ralph) | Each iteration may spawn multiple Agent calls | Use `model=haiku` for simple subagents, limit parallel agents |
| Parallel analysis patterns | Spawning 3+ agents for security/performance/test analysis | Consolidate into a single comprehensive agent call |
| persistent-mode Stop hook | Prevents sessions from terminating, extending their lifetime | Ensure stale-state threshold (default 2h) is working |
| Background agents (`run_in_background: true`) | Agents run detached and may not be noticed | Check `pgrep claude` after workflows complete |

### Persistent-Mode Hook Audit

The `persistent-mode.mjs` Stop hook is the most common reason sessions won't terminate. Audit its state files:

```bash
# Check for active OMC mode states that may be keeping sessions alive
STATE_DIR=".omc/state"
for f in ralph-state.json autopilot-state.json ultrawork-state.json \
         ultrapilot-state.json pipeline-state.json team-state.json \
         ultraqa-state.json swarm-summary.json skill-active-state.json; do
  if [ -f "$STATE_DIR/$f" ]; then
    active=$(python3 -c "import json; d=json.load(open('$STATE_DIR/$f')); print(d.get('active', False))" 2>/dev/null)
    echo "$f: active=$active"
  fi
done

# Check session-scoped state files
for dir in "$STATE_DIR"/sessions/*/; do
  session=$(basename "$dir")
  for f in "$dir"*.json; do
    [ -f "$f" ] && echo "Session $session: $(basename $f)"
  done
done 2>/dev/null
```

**If stale state files have `active: true`**: Delete them to unblock session termination:
```bash
# Remove stale state files (only after confirming no active workflow)
rm -f .omc/state/*-state.json
rm -rf .omc/state/sessions/
```

## Key Env Optimizations (New)

| Env / Setting | Default | Optimized | Impact |
|--------------|---------|-----------|--------|
| `alwaysThinkingEnabled` | true (some configs) | false | -30-50% latency |
| `effortLevel` | high | medium | -40% tokens on simple tasks |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 80 | 50 | Earlier compaction |
| `MAX_THINKING_TOKENS` | unlimited | 10000 | Cap thinking cost |
| `CLAUDE_CODE_SUBAGENT_MODEL` | sonnet | haiku (simple tasks) | 5x cheaper subagents |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 0 | 1 (if using external) | No duplicate memory |

## Companion Tool: ccusage

Use [ccusage](https://github.com/LEON-gittech/ccusage) to measure actual token spending before and after optimization. Our fork includes a 14x cache speedup for large datasets.

```bash
# Before optimization — capture baseline
ccusage --period day > /tmp/before-optimization.txt

# After optimization — compare
ccusage --period day > /tmp/after-optimization.txt
diff /tmp/before-optimization.txt /tmp/after-optimization.txt
```

## Quick Wins Checklist

- [ ] Run `ccusage --period day` to establish cost baseline
- [ ] Check for zombie claude processes (`pgrep claude` + check age >2 days)
- [ ] Kill zombie claude processes with deleted CWD or age >2 days
- [ ] Check for zombie MCP processes (ps aux | grep mcp)
- [ ] Kill old claude sessions spawning orphaned processes
- [ ] Audit subagent session count (>100 in one project = problem)
- [ ] Check `.omc/state/` for stale `active: true` state files
- [ ] Clean stale persistent-mode state files after workflows end
- [ ] Disable plugins for wrong tech stack
- [ ] Remove duplicate Stop hooks
- [ ] Narrow `*` matchers to specific tools where possible
- [ ] Keep CLAUDE.md < 5K tokens per file
- [ ] Remove duplicated rules across CLAUDE.md levels
- [ ] Create .claudeignore
- [ ] Set CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50
- [ ] Disable unused MCP servers
- [ ] Review PostToolUse hooks with >10s timeout
- [ ] Remove PostToolUse hooks that spawn `claude -p` sub-processes
- [ ] Set `alwaysThinkingEnabled: false` if currently true
- [ ] Set `effortLevel: "medium"` for daily work
- [ ] Remove unused `mcpServers` entries from settings.json
- [ ] Check `~/.claude/debug/` for large files (>500KB) indicating MCP connection churn
