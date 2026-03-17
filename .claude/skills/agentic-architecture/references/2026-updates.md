# 2026 Updates Deep Dive

Detailed notes from three 2026 Anthropic publications.

## Table of Contents
- [Demystifying Evals for AI Agents](#demystifying-evals-for-ai-agents-jan-2026)
- [Code Execution with MCP](#code-execution-with-mcp-2026)
- [2026 Agentic Coding Trends Report](#2026-agentic-coding-trends-report)

## Demystifying Evals for AI Agents (Jan 2026)

Source: [anthropic.com/engineering/demystifying-evals-for-ai-agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### Definition

An eval = structured test: give an AI an input → apply grading logic → measure success. Focus: automated evals runnable during development without real users.

### Eval Complexity Spectrum

1. **Single-turn**: one prompt, one response, grade
2. **Multi-turn**: back-and-forth interaction
3. **Agent evals**: tools across many turns, modifying environment state, adapting

### Graders (detailed)

**Code-based**: string matching, binary tests, static analysis, outcome verification.
- Pros: fast, cheap, objective, reproducible
- Cons: brittle to valid variations

**Model-based**: rubric scoring, NL assertions, pairwise comparison.
- Pros: flexible, nuanced
- Cons: needs calibration with human graders; non-deterministic

**Human**: expert review.
- Pros: gold standard quality
- Cons: expensive, slow

### Agent-Type-Specific Strategies

**Coding agents**: does code run + tests pass? Combine: deterministic tests + LLM rubrics + static analysis + state checks + tool call verification. Benchmarks: SWE-bench Verified, Terminal-Bench.

**Conversational agents**: multidimensional — ticket resolved (state check) + under N turns (transcript constraint) + appropriate tone (LLM rubric). Need verifiable end-state outcomes.

**Research agents** (hardest): groundedness checks (claims ← sources), coverage checks (key facts included), source quality checks (authoritative).

**Computer use agents**: real or sandboxed environments. DOM-based = fast but token-heavy; screenshot-based = slower but token-efficient.

### Eight-Step Roadmap

**Steps 0-3: Dataset**
- Start with 20-50 tasks from real failures
- User-reported failures → test cases (ensures real-world coverage)
- Task clarity: two domain experts would reach same pass/fail verdict
- Include reference solutions proving solvability
- Balanced sets: test both positive and negative cases

**Steps 4-5: Harness + Graders**
- Each trial: clean, isolated environment (shared state = noise)
- Deterministic graders where possible; LLM where necessary
- Avoid rigid grading — agents find valid approaches designers didn't anticipate
- Partial credit for multi-component tasks

**Steps 6-8: Maintenance**
- Read transcripts regularly — only way to know if graders work
- Monitor capability saturation (all solvable tasks pass → need harder tasks)
- Domain experts + product teams own eval tasks + run evals

### Critical Pitfall: Eval Bugs > Agent Bugs

Opus 4.5 on CORE-Bench: scored **42%** due to:
- Rigid grading penalizing `96.12` when expecting `96.124991...`
- Ambiguous task specs
- Stochastic tasks

After fixing eval bugs → **95%**. Always check: is the eval broken, or is the agent broken?

### Swiss Cheese Model

No single layer catches everything. Combine:
- Automated evals (development-time)
- Production monitoring (real user behavior at scale)
- A/B testing (actual user outcomes)
- User feedback (sparse but surfaces unexpected issues)
- Manual transcript review (builds intuition)
- Systematic human studies (gold standard, expensive)

### Non-Determinism Metrics

- **pass@k**: probability ≥1 of k attempts succeeds → approaches 100% at high k
- **pass^k**: probability ALL k trials succeed → approaches 0% at high k
- At k=1, identical. At k=10, opposite stories. Report both.

### Claude Code's Journey

1. Fast iteration on employee + external user feedback
2. Added narrow evals: concision, file edits
3. Expanded to complex behaviors: over-engineering
4. Evals focused research-product collaborations on real gaps

---

## Code Execution with MCP (2026)

Source: [anthropic.com/engineering/code-execution-with-mcp](https://www.anthropic.com/engineering/code-execution-with-mcp)

### The token problem at scale

1. **Definition overhead**: 1000+ MCP tools = hundreds of thousands of tokens for schemas alone
2. **Intermediate result duplication**: data flows through model context between tool calls. 2-hour meeting transcript = 50K+ tokens passing through twice.

### Architecture: tools as filesystem code APIs

MCP servers exposed as directory structure:
```
servers/
  google-drive/
    getDocument.ts
    index.ts
  salesforce/
    updateRecord.ts
    index.ts
```

Each tool = TypeScript function that agents import and compose. Discovery by reading filesystem, not pre-loaded schemas.

### Results: 150K → 2K tokens (98.7% reduction)

Why:
- Agents load only needed tool definitions (progressive disclosure)
- Data filtering happens in execution sandbox, not model context
- Results don't pass through model between operations

### Five benefits (detailed)

1. **Progressive disclosure**: `search_tools()` with configurable detail levels (name only / name+description / full schema)

2. **Data filtering**: 10K spreadsheet rows → filter locally → return only relevant rows to model

3. **Control flow**: loops, conditionals, error handling in standard code. Reduces latency vs chaining tool calls through model.

4. **Privacy preservation**: intermediate data stays in sandbox. PII auto-tokenized — real contact details flow Google Sheets → Salesforce while model sees only `[EMAIL_1]`, `[PHONE_1]`.

5. **State persistence + skills**: write intermediate results to files for cross-session resumption. Persist reusable code patterns as skills.

### Trade-offs

- Requires secure sandboxing, resource limits, monitoring
- Infrastructure overhead vs direct tool calls
- Not worth it for simple single-tool calls

### Validation

Cloudflare published similar findings, calling the pattern "Code Mode." Core insight: LLMs excel at writing code — leverage that for tool composition rather than forcing direct function calls.

---

## 2026 Agentic Coding Trends Report

Source: [resources.anthropic.com/2026-agentic-coding-trends-report](https://resources.anthropic.com/2026-agentic-coding-trends-report)

### Eight trends

1. **Role shift**: implementation → agent supervision, system design, output review
2. **Multi-agent coordination**: parallel reasoning across separate context windows replaces single-agent workflows
3. **Extended horizons**: minutes → days/weeks, agents build full systems with human checkpoints
4. **MCP as standard**: de facto standard for agent-tool integration
5. **AI-automated review**: agents automate code review to scale human oversight
6. **Beyond engineering**: domain experts use agentic tools (not just developers)
7. **Security-first**: architecture embedded from day one, not bolted on
8. **Digital assembly lines**: human-guided multi-step workflows create business value

### Multi-agent in practice

If 2025 = single AI assistants, 2026 = coordinated teams. Orchestrator delegates to specialized agents working simultaneously in separate context windows, then synthesizes.

Real-world: Fountain (workforce platform) achieved:
- 50% faster screening
- 40% quicker onboarding
- 2x candidate conversions
- Using Claude for hierarchical multi-agent orchestration

### Four strategic priorities

1. Master multi-agent coordination (parallel context windows)
2. Scale human-agent oversight (AI-automated review)
3. Extend agentic tools beyond engineering
4. Embed security as core architecture
