# Composable Patterns & Guardrails

Detailed notes from [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) and [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents).

## Table of Contents
- [Agents vs Workflows](#agents-vs-workflows)
- [Five Composable Patterns](#five-composable-patterns)
- [Guardrail Patterns](#guardrail-patterns)
- [Long-Running Agent Harnesses](#long-running-agent-harnesses)
- [Agent-Computer Interface (ACI)](#agent-computer-interface-aci)

## Agents vs Workflows

| | Workflows | Agents |
|---|---|---|
| Control | Predefined code paths | LLM dynamically directs |
| Behavior | Predictable, consistent | Flexible, adaptive |
| Cost | Lower (fewer LLM calls) | Higher (multi-turn reasoning) |
| Use when | Task is well-defined | Task requires flexibility |

**Default to workflows. Upgrade to agents only when flexibility is needed.**

## Five Composable Patterns

### 1. Prompt Chaining
Sequential steps: output of step N → input of step N+1.
- Add programmatic "gates" between steps (validation, filtering)
- Good when task cleanly decomposes into fixed subtasks
- Trades latency for accuracy

**Example:** Generate marketing copy → Check for brand guideline compliance → Translate

### 2. Routing
Classify input → direct to specialized handler.
- Enables separation of concerns
- Route easy queries to Haiku, hard ones to Sonnet
- Works when distinct categories exist and classification is accurate

**Example:** Customer support → classify as billing/technical/general → specialized prompt per category

### 3. Parallelization

**Sectioning:** break task into independent subtasks run simultaneously.
**Voting:** run same task multiple times for diverse outputs / confidence.

**Example (sectioning):** Analyze document → extract key terms + summarize + identify action items (all in parallel)
**Example (voting):** Code review from 3 independent perspectives → merge findings

### 4. Orchestrator-Workers
Central LLM dynamically breaks down task → delegates to workers → synthesizes.
Key difference from parallelization: subtask list is **dynamic**, not pre-defined.

**Example:** "Refactor the auth module" → orchestrator identifies affected files → spawns worker per file → synthesizes changes

### 5. Evaluator-Optimizer
One LLM generates → another evaluates → loop until quality criteria met.
Good when clear evaluation criteria exist and iteration adds measurable value.

**Example:** Generate response → evaluate for accuracy/tone/completeness → refine → re-evaluate

## Guardrail Patterns

### Parallel guardrails (recommended)

Run guardrails **as a separate parallel instance**, not inline:
```
User message
    ├─→ [Main agent] → generates response
    └─→ [Guard agent] → screens for issues
         └─→ blocks/modifies if needed
```

This is a sectioning variant of the parallelization pattern. Performs better than single-LLM inline checks.

### Agent guardrail principles

1. **Sandbox execution environments** — limit blast radius
2. **Maximum iterations** — prevent runaway loops (e.g., max 5 tool calls)
3. **Human checkpoints** — for high-stakes actions (send message, make payment)
4. **Transparency** — show planning steps, tool calls, reasoning
5. **Compounding error awareness** — agents can go off-track; each step may amplify errors

### Input validation as guardrail

Tools can validate their own inputs:
```python
def execute_tool(params):
    if params["date"] and not re.match(r"\d{4}-\d{2}-\d{2}", params["date"]):
        return {"error": f"Invalid date '{params['date']}'. Use YYYY-MM-DD."}
    # ... proceed
```

This is idiomatic in agentic systems — the tool protects itself and returns actionable errors.

## Long-Running Agent Harnesses

### The core challenge
Each new context window starts with **no memory** of previous sessions.

### Two-part solution

1. **Initializer agent** (first session only):
   - Sets up environment (repos, dependencies)
   - Creates progress tracking file (`claude-progress.txt`)
   - Writes feature lists, acceptance criteria

2. **Coding agents** (each subsequent session):
   - Reads progress file + git history
   - Makes incremental progress
   - Updates progress file with what was done + what's next
   - Commits work

### Progress file pattern
```markdown
# claude-progress.txt

## Completed
- [x] Set up database schema
- [x] Implement user authentication

## Current
- [ ] Add payment processing (in progress: Stripe webhook handler)

## Blocked
- Waiting for API credentials for email service

## Architecture Decisions
- Using PostgreSQL (not MongoDB) for ACID transactions
- JWT tokens with 24h expiry
```

### Key insight
The progress file + git history together enable fast state recovery. The file captures **intent and decisions** that git history alone doesn't convey.

## Agent-Computer Interface (ACI)

From Appendix 2 of Building Effective Agents:

> "Invest in the ACI as much as you would in HCI."

Principles:
- Give the model enough tokens to think before writing
- Keep formats close to natural text (markdown > JSON for code)
- Avoid formatting overhead (accurate line counts, string escaping)
- Write tool definitions like great docstrings for a junior developer
- Include examples, edge cases, input format requirements, boundaries with other tools
- **Poka-yoke** (error-proof) your tools: change arguments to make mistakes harder
  - Example: require absolute filepaths (not relative) to prevent path confusion
- Test extensively, iterate on tool definitions
