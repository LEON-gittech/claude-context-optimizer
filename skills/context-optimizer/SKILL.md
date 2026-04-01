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

## Quick Wins Checklist

- [ ] Disable plugins for wrong tech stack
- [ ] Remove duplicate Stop hooks
- [ ] Narrow `*` matchers to specific tools where possible
- [ ] Keep CLAUDE.md < 5K tokens per file
- [ ] Remove duplicated rules across CLAUDE.md levels
- [ ] Create .claudeignore
- [ ] Set CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50
- [ ] Disable unused MCP servers
- [ ] Review PostToolUse hooks with >10s timeout
