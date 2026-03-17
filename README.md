# Agentic Architecture — Claude Code Skill

A Claude Code skill with Anthropic best practices for agentic AI architecture (2025-2026). Distilled from 8 official Anthropic engineering publications.

## What's inside

**Main skill** (`.claude/skills/agentic-architecture/SKILL.md`) — covers:

1. **Tool Design** — descriptions as prompt engineering, consolidation, actionable errors, `input_examples`
2. **Context Engineering** — context rot, system prompt Goldilocks zone, just-in-time loading
3. **Advanced Tool Use** — Tool Search (`defer_loading`), Programmatic Tool Calling
4. **Composable Patterns** — prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer
5. **Long-Running Agent Harnesses** — multi-session architectures with progress tracking
6. **Eval-Driven Development** — realistic task generation, iterative improvement
7. **Agent Evals (2026)** — 3 grader types, Swiss Cheese model, `pass@k` vs `pass^k`
8. **Code Execution with MCP** — tools as filesystem APIs (98.7% token reduction)
9. **2026 Agentic Trends** — multi-agent coordination, MCP as standard

**Reference deep dives** (`references/`):
- `tool-design-deep-dive.md` — eval-driven development, response design, `input_examples` spec
- `context-engineering-deep-dive.md` — context rot, just-in-time context, state management
- `patterns-and-guardrails.md` — 5 workflow patterns, guardrail patterns, long-running harnesses
- `2026-updates.md` — agent evals, code execution with MCP, agentic coding trends

## Installation

Copy the `.claude/skills/agentic-architecture/` folder into your project:

```bash
# From your project root
mkdir -p .claude/skills
cp -r path/to/this-repo/.claude/skills/agentic-architecture .claude/skills/
```

Or for global availability (all projects):

```bash
cp -r path/to/this-repo/.claude/skills/agentic-architecture ~/.claude/skills/
```

## Usage

In Claude Code, invoke with:

```
/agentic-architecture
```

The skill triggers automatically when you ask about agent design, tool use optimization, context management, orchestration patterns, or "how should we architect this".

## Sources

All content distilled from official Anthropic engineering publications:

1. [Writing Effective Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — Sep 2025
2. [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) — Nov 2025
3. [Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Sep 2025
4. [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — Dec 2024
5. [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — Nov 2025
6. [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Jan 2026
7. [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — 2026
8. [2026 Agentic Coding Trends Report](https://resources.anthropic.com/2026-agentic-coding-trends-report) — Jan 2026
