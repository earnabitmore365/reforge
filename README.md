# /reforge — 回炉再造

> Self-looping code review + beautification with project context injection.

## What it does

One command (`/reforge`) that replaces running `/simplify` + `/code-review` + `/code-simplifier` separately:

### Phase 1: Review & Fix (auto-loop)

4 Sonnet agents run **in parallel**, each checking a different dimension:

| Agent | Checks |
|-------|--------|
| Reuse | Duplicate code, shared functions, repeated constants |
| Quality | Dead code, naming, redundant state, unnecessary comments |
| Efficiency | Hot-path I/O, N+1 patterns, TOCTOU, resource leaks |
| Security | Injection, path traversal, concurrency, edge cases |

Findings are fixed, then agents re-run. **Loops until 3 consecutive CLEANs** (max 10 rounds).

### Phase 2: Beautify (auto-loop)

1 Sonnet agent simplifies structure — flatten nesting, extract helpers, improve naming. **Loops until 3 consecutive CLEANs** (max 10 rounds).

### Project Context Injection

The key improvement: every agent receives project-specific context automatically:

| Context | Source | How |
|---------|--------|-----|
| Project rules | `rules.md` INJECT section | Filter for grep-checkable rules only |
| Lessons learned | `LEARNINGS.md` | Last 20 entries |
| Anti-patterns | `wuji-verify.py` ANTI_PATTERNS | Auto-extracted pattern list |
| Documentation | `/Volumes/SSD-2TB/文档/` | Matched by file type (edit DuckDB code → inject duckdb.md) |

This means agents review code **with your project's rules and hard-won lessons**, not just generic LLM knowledge.

## Install

```bash
claude mcp add-plugin reforge --url https://github.com/earnabitmore365/reforge
```

Or clone and symlink:

```bash
git clone https://github.com/earnabitmore365/reforge.git
ln -s $(pwd)/reforge/commands/reforge.md ~/.claude/commands/reforge.md
cp reforge/scripts/reforge_context.py ~/.claude/merit/
```

## Usage

```
/reforge              # Run full review + beautify pipeline
/loop 5m /reforge     # Auto-run every 5 minutes
```

## File Structure

```
reforge/
  .claude-plugin/
    plugin.json          # Plugin metadata
  commands/
    reforge.md           # Skill definition (Phase 1 + Phase 2 + loop logic)
  scripts/
    reforge_context.py   # Context injection engine (rules/lessons/docs matching)
  README.md
```

## Context Injection Engine (`reforge_context.py`)

The engine maps changed files to relevant documentation:

```python
FILE_DOCS_MAP = {
    "adapter": ["bitmex.md"],
    "generate_seed": ["sqlite.md", "duckdb.md", "segmented_backtest.md"],
    "build_seed_report": ["duckdb.md", "seed_report_field_reference.md"],
    "merit_gate": ["claude_code.md", "minimax_api.md"],
    ...
}
```

Run standalone to preview what context would be injected:

```bash
python3 reforge_context.py merit_gate.py credit_manager.py
```

## Design Philosophy

- **Not just LLM logic** — injects YOUR rules, YOUR lessons, YOUR documentation
- **Self-looping** — no manual re-runs, stops only when truly clean
- **Two phases** — fix bugs first (Phase 1), then beautify (Phase 2)
- **Parallel agents** — 4 agents run concurrently for speed
- **Convergence guard** — max 10 rounds prevents infinite loops

Built as part of the WujiTang Merit System (天衡册).
