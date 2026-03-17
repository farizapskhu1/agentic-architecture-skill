---
name: agentic-architecture
description: Anthropic best practices for agentic AI architecture (2025-2026). Use when designing, reviewing, or debugging agentic systems — tool definitions, context engineering, guardrails, orchestration patterns, long-running agents. Triggers on questions about agent design, tool use optimization, context management, or "how should we architect this".
---

# Agentic Architecture — Anthropic Best Practices (2025-2026)

Distilled from 8 Anthropic engineering publications (2024-2026). For full details, read the reference files.

## Core Philosophy

> "Find the simplest solution possible, and only increase complexity when needed."

Agentic systems trade **latency and cost for better task performance**. Before building an agent, ask: can a single LLM call with retrieval + in-context examples solve this? Only add agentic patterns when simpler approaches fail.

### Where 2026 updates 2024-2025 (newer wins)

| Topic | 2024-2025 said | 2026 says | What to do |
|-------|---------------|-----------|------------|
| **Tool loading** | `defer_loading` + Tool Search Tool (85% token reduction) | Code Execution with MCP — tools as filesystem APIs (98.7% reduction) | For large tool sets (100+), prefer Code Execution with MCP. Tool Search still fine for 10-50 tools. |
| **Single vs Multi-agent** | Default to single agent; multi-agent only when needed | Multi-agent coordination becoming standard; parallel reasoning across context windows | For complex workflows, **plan for multi-agent from the start**. Single-agent still fine for focused tasks. |
| **Context management** | Compaction, structured notes, sub-agents to manage context | Data stays in execution sandbox, never enters model context at all | Prefer **keeping data out of context entirely** (code execution) over managing it once it's in (compaction). |
| **Evals** | Eval-driven tool optimization; held-out test sets | Swiss Cheese model; agent-type-specific strategies; pass@k + pass^k; start from 20-50 real failures | Use the **2026 eval framework** — it's much more comprehensive. |

---

## 1. Tool Design (highest impact area)

Source: [Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents), [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)

### Tool descriptions are prompt engineering

Write tool descriptions as you would **onboard a new hire** — make implicit context explicit. Small refinements to descriptions yield dramatic improvements (Claude 3.5 Sonnet achieved SOTA on SWE-bench after tool description refinements alone).

**Do:**
- Define specialized terminology in the description (e.g. `"დაბჟენა = filling = therapy"`)
- Describe parameter enum values with synonyms and examples
- Use `input_examples` for complex parameters (72% → 90% accuracy in Anthropic tests)
- Return actionable errors: explain what went wrong + expected format + correct example
- Return high-signal data only; resolve UUIDs to human-readable names
- Namespace tools by service+resource (e.g. `asana_projects_search`)

**Don't:**
- Wrap every existing API endpoint as a tool — agents have limited context
- Create overlapping tools — if you can't tell which applies, neither can the agent
- Return raw technical IDs (uuid, mime_type) when the agent needs semantic names
- Use `list_all_*` tools — prefer `search_*` that filter server-side

### Tool consolidation

Instead of multiple low-level tools, create **task-oriented** tools:
- `list_users` + `list_events` + `create_event` → `schedule_event`
- `read_logs` → `search_logs` (returns only matching lines with context)
- `get_customer_by_id` + `list_transactions` → `get_customer_context`

### Actionable errors (critical pattern)

```
BAD:  "Error 400: Invalid parameter"
GOOD: "Invalid date format '11/06/2024'. Expected YYYY-MM-DD. Example: '2024-11-06'"
```

Errors are a **steering mechanism** — they teach the agent correct usage mid-conversation.

### input_examples (API beta feature)

JSON Schema defines structure but cannot express **usage patterns**. `input_examples` fill this gap:

```json
{
  "name": "lookup_pricing",
  "input_examples": [
    {"service": "therapy", "note": "user asked 'კბილის დაბჟენა რა ღირს?'"},
    {"service": "implant", "note": "user asked 'იმპლანტი რა ღირს?'"}
  ]
}
```

Best practices for examples:
- Use realistic data (real names, plausible values), not placeholders
- Show minimal, partial, and full specification patterns (variety)
- 1-5 examples per tool; focus on genuine ambiguities
- Examples add tokens — only use when accuracy improvement outweighs cost

### Token-efficient responses

- Implement pagination, filtering, truncation with sensible defaults
- Offer `response_format` enum: `"concise"` (72 tokens) vs `"detailed"` (206 tokens)
- When truncating, tell the agent how to narrow the search

---

## 2. Context Engineering

Source: [Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

> "Context engineering is finding the smallest possible set of high-signal tokens that maximize the likelihood of the desired outcome."

### Context rot

As tokens increase, recall accuracy **decreases** (n-squared pairwise relationships strain attention). Context is a finite resource with diminishing returns. Every token competes for attention.

### System prompts — the Goldilocks zone

Two failure modes:
1. **Too rigid:** hardcoded complex brittle logic
2. **Too vague:** high-level guidance without specifics

Target the middle: **direct language, organized sections (XML/Markdown), minimal information set**.
Start with minimal prompts on the best model → add instructions for discovered failure modes.

### Just-in-time context (key pattern)

Don't pre-load everything. Maintain lightweight identifiers (file paths, queries, links) and dynamically load data at runtime via tools. This mirrors human cognition.

Hybrid approach: retrieve **some** data upfront for speed (config, facts) + enable just-in-time retrieval (search, lookup) for the rest.

### Long-running state management

1. **Compaction**: summarize conversations nearing limits; preserve decisions, discard tool outputs
2. **Structured notes**: agents write persistent notes (NOTES.md, to-do lists) outside context
3. **Sub-agents**: focused sub-agents with clean windows return condensed summaries (10K explored → 1-2K returned)

---

## 3. Advanced Tool Use Features (API beta)

Source: [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)

Enable via `betas=["advanced-tool-use-2025-11-20"]`.

### Tool Search Tool — fix context bloat from definitions

Problem: 5 MCP servers ≈ 55K tokens before any work begins.
Solution: `defer_loading: true` — Claude discovers tools on-demand.

| Metric | Before | After |
|--------|--------|-------|
| Token usage | ~77K | ~8.7K (85% reduction) |
| Opus 4 accuracy | 49% | 74% |
| Opus 4.5 accuracy | 79.5% | 88.1% |

**When to use:** >10K tokens in tool defs, 10+ tools, multiple MCP servers.
Keep 3-5 most-used tools always loaded; defer the rest.

### Programmatic Tool Calling — fix intermediate result bloat

Claude writes Python code orchestrating multiple tools. Only final output enters context.

| Metric | Before | After |
|--------|--------|-------|
| Token usage | 43,588 | 27,297 (37% reduction) |
| Knowledge retrieval | 25.6% | 28.5% |

**When to use:** large datasets needing aggregation, 3+ dependent tool calls, data filtering before model sees it.

### Prioritize by bottleneck

- **Context bloat from definitions** → Tool Search
- **Large intermediate results** → Programmatic Calling
- **Parameter errors** → input_examples

---

## 4. Composable Patterns

Source: [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)

Five workflow patterns (composable, not prescriptive):

| Pattern | When to use |
|---------|------------|
| **Prompt Chaining** | Task cleanly decomposes into fixed sequential subtasks |
| **Routing** | Distinct categories exist; classify then specialize |
| **Parallelization** | Independent subtasks (sectioning) or need diverse outputs (voting) |
| **Orchestrator-Workers** | Complex tasks with unpredictable subtask decomposition |
| **Evaluator-Optimizer** | Clear eval criteria + iterative refinement adds measurable value |

### Agents vs Workflows

- **Workflows**: predefined code paths, predictable, consistent
- **Agents**: LLM dynamically directs its own process, flexible, higher cost

### Guardrails

- Run guardrails **in parallel** with the main task (sectioning pattern) — a separate model instance screens while the main one works
- Sandbox execution environments
- Set maximum iterations / stopping conditions
- Add human feedback checkpoints for high-stakes actions

---

## 5. Long-Running Agent Harnesses

Source: [Effective Harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

For tasks spanning hours/days across multiple context windows:

1. **Initializer agent**: sets up environment, writes progress tracking files
2. **Coding agents**: make incremental progress per session, read/update progress artifacts
3. **Progress file** (e.g. `claude-progress.txt`): persists state across context resets alongside git history

Key insight: each new session starts with **no memory** — the harness must enable fast state recovery.

---

## 6. Evaluation-Driven Development

Source: [Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)

### Generate realistic tasks

**Strong:** "Customer ID 9182 reported triple-charge. Find all relevant logs and check if other customers were affected."
**Weak:** "Search payment logs for `customer_id=9182`."

Tasks should require multiple tool calls, use realistic data, and pair with verifiable outcomes.

### Run and analyze

- Run programmatically (while-loop: LLM API ↔ tool calls)
- Enable chain-of-thought / interleaved thinking
- Collect: runtime, tool call count, token consumption, tool errors
- Redundant calls → consolidate tools; frequent parameter errors → better descriptions/examples

### Iterate with Claude

Concatenate eval transcripts → paste into Claude Code → Claude analyzes failures and refactors tools. Use held-out test sets to avoid overfitting. This process outperformed expert manual optimization in Anthropic's internal tests.

---

## 7. Agent Evals (2026)

Source: [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Jan 2026

> "20-50 simple tasks drawn from real failures is a great start."

### Three grader types

| Grader | Use when |
|--------|----------|
| **Code-based** (string match, tests, static analysis) | Objective, reproducible checks — "does the code pass tests?" |
| **Model-based** (rubric scoring, pairwise comparison) | Nuanced quality — tone, completeness, appropriateness |
| **Human** (expert review) | Gold standard, but expensive — use to calibrate model graders |

### Eval by agent type

- **Coding agents**: does the code run + tests pass? (SWE-bench, Terminal-Bench)
- **Conversational agents**: task completed + under N turns + appropriate tone (multidimensional)
- **Research agents**: groundedness + coverage + source quality (hardest to eval)

### Key principles

- **Start from real failures**: convert user-reported bugs into test cases
- **Avoid overly rigid grading**: Opus 4.5 scored 42% on CORE-Bench due to eval bugs (expected `96.124991`, got `96.12`); after fixing graders → 95%
- **Swiss Cheese Model**: no single eval layer catches everything — combine automated evals + production monitoring + A/B tests + manual review
- **pass@k vs pass^k**: at k=1 identical, at k=10 opposite stories. Use both.
- **Partial credit**: for multi-component tasks, grade each component separately
- **0% pass rate across many trials** = most likely broken task, not incapable agent

### Claude Code's eval journey

Started with fast iteration on user feedback → added narrow evals (concision, file edits) → expanded to complex behaviors (over-engineering). Evals focused research-product collaborations on real gaps.

---

## 8. Code Execution with MCP (2026)

Source: [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — 2026

> 150,000 tokens → 2,000 tokens = **98.7% reduction** in token usage

### The problem with direct tool calls at scale

1. **Definition overhead**: 1000+ MCP tools = hundreds of thousands of tokens just for schemas
2. **Intermediate result duplication**: data flows through model context between tool calls (e.g. 2-hour meeting transcript = 50K+ tokens passed through twice)

### The solution: tools as code APIs

Instead of pre-loading all tool schemas, MCP servers are presented as **filesystem-based code APIs**:

```
servers/
  google-drive/
    getDocument.ts
  salesforce/
    updateRecord.ts
```

Agents write code to import, compose, and execute tools — loading only what they need.

### Five benefits

1. **Progressive disclosure**: `search_tools()` function lets agents find tools on-demand
2. **Data filtering**: process 10K rows locally, return only relevant ones to model
3. **Control flow**: loops/conditionals in code vs chained tool calls (lower latency)
4. **Privacy**: intermediate data stays in sandbox; PII auto-tokenized (`[EMAIL_1]`)
5. **State persistence**: write results to files for cross-session resumption

### When to use

Best for: many MCP servers, large data processing, multi-step workflows, privacy-sensitive data.
Trade-off: requires secure sandboxing + resource limits + monitoring overhead.

---

## 9. 2026 Agentic Trends

Source: [2026 Agentic Coding Trends Report](https://resources.anthropic.com/2026-agentic-coding-trends-report)

### Eight trends

1. Engineering roles shift from implementation → agent supervision + system design + output review
2. **Multi-agent systems replace single-agent workflows** — parallel reasoning across separate context windows
3. Task horizons expand from minutes → days/weeks with strategic human checkpoints
4. MCP becomes de facto standard for agent-tool integration
5. Agent-automated code review scales human oversight
6. Agentic tools extend beyond engineering to domain experts
7. Security architecture embedded from day one (not bolted on)
8. Business value from "digital assembly lines" — human-guided multi-step workflows

### Multi-agent coordination (key 2026 shift)

Orchestrator delegates subtasks to specialized agents working **simultaneously** in separate context windows, then stitches results together. Real-world: Fountain achieved 50% faster screening, 40% quicker onboarding, 2x conversions using hierarchical multi-agent orchestration.

### Strategic priorities for 2026

1. Master multi-agent coordination (parallel reasoning across context windows)
2. Scale human-agent oversight through AI-automated review
3. Extend agentic tools beyond engineering teams
4. Embed security as core architecture, not afterthought

---

## Quick Reference: Design Checklist

- [ ] Can a single LLM call solve this? (don't over-architect)
- [ ] Are tool descriptions written for a new hire, not a developer?
- [ ] Do enum parameters have descriptive values with synonyms?
- [ ] Do errors explain what went wrong + how to fix?
- [ ] Is the tool set minimal? No overlapping tools?
- [ ] Are tool responses token-efficient? Pagination/filtering?
- [ ] Is system prompt in the Goldilocks zone? (not too rigid, not too vague)
- [ ] Is context loaded just-in-time where possible?
- [ ] Are guardrails running in parallel, not inline?
- [ ] Do you have realistic evals with verifiable outcomes?
- [ ] Are evals starting from real failures (not synthetic tasks)?
- [ ] For large tool sets: is code execution with MCP an option to cut tokens?

---

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
