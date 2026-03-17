# Tool Design Deep Dive

Detailed notes from [Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) and [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use).

## Table of Contents
- [Tool = Contract](#tool--contract-between-deterministic-and-non-deterministic)
- [Evaluation-Driven Development](#evaluation-driven-development-detailed)
- [Tool Response Design](#tool-response-design)
- [input_examples](#input_examples--full-specification)
- [Programmatic Tool Calling](#programmatic-tool-calling--full-details)
- [Tool Search Tool](#tool-search-tool--full-details)

## Tool = Contract Between Deterministic and Non-Deterministic

Unlike traditional functions (always behave identically), agents may call a tool, answer from knowledge, or ask a clarifying question. Tools must be designed for agents, not for developers.

## Evaluation-Driven Development (detailed)

### Generating eval tasks

Prompts should be inspired by real-world uses with realistic data. Avoid overly simplistic sandbox environments. Strong tasks require multiple tool calls (potentially dozens).

**Strong tasks:**
- "Schedule a meeting with Jane next week to discuss our latest Acme Corp project. Attach the notes from our last project planning meeting and reserve a conference room."
- "Customer ID 9182 reported that they were charged three times for a single purchase attempt. Find all relevant log entries and determine if any other customers were affected."

**Weak tasks (too prescriptive):**
- "Schedule a meeting with jane@acme.corp next week."
- "Search payment logs for `purchase_complete` and `customer_id=9182`."

Each prompt paired with verifiable response. Avoid overly strict verifiers that reject correct responses due to formatting or valid alternative phrasings.

### Running evals

Run programmatically with direct LLM API calls using simple agentic loops (while-loops wrapping alternating LLM API and tool calls). Instruct agents to output reasoning and feedback blocks before tool call and response blocks.

Collect metrics:
- Total runtime
- Number of tool calls
- Total token consumption
- Tool errors (count + types)

### Analysis signals

- Agents getting stumped → read reasoning/CoT
- Redundant tool calls → consolidate/batch tools
- Frequent invalid-parameter errors → better descriptions or input_examples
- Unnecessary query additions (e.g. Claude appending "2025" to searches) → improve tool description

### Agent collaboration

Concatenate evaluation transcripts → paste into Claude Code → Claude analyzes and refactors. Held-out test sets prevent overfitting. This outperformed expert manual optimization.

## Tool Response Design

### High-signal returns

Prioritize contextual relevance over flexibility. Agents handle natural language names better than cryptic identifiers.

**Replace:** `uuid`, `256px_image_url`, `mime_type`
**With:** `name`, `image_url`, `file_type`

Resolving UUIDs to semantically meaningful language (or even 0-indexed IDs) significantly improves precision and reduces hallucinations.

### Response format enum

When agents need both natural language and technical IDs:
```
enum ResponseFormat { DETAILED = "detailed", CONCISE = "concise" }
```
- Detailed (206 tokens): all metadata
- Concise (72 tokens): essential content only — ~67% token savings

Response structure (XML, JSON, Markdown) impacts eval performance with no one-size-fits-all solution.

### Pagination and truncation

- Implement pagination, range selection, filtering with sensible defaults
- Claude Code restricts tool responses to 25,000 tokens by default
- For truncated responses: tell agent how many more results exist + how to narrow search

## input_examples — Full Specification

JSON Schema defines structural validity but cannot express usage patterns:
- Format ambiguity: "2024-11-06" vs "Nov 6, 2024" vs ISO?
- ID conventions: UUIDs vs "USR-12345" vs plain numbers?
- Nested structure: when to populate optional nested objects?
- Parameter correlations: how do escalation.level and priority relate?

### Example: support ticket tool

```json
"input_examples": [
  {
    "title": "Login page returns 500 error",
    "priority": "critical",
    "labels": ["bug", "authentication", "production"],
    "reporter": {"id": "USR-12345", "name": "Jane Smith",
                 "contact": {"email": "jane@acme.com", "phone": "+1-555-0123"}},
    "due_date": "2024-11-06",
    "escalation": {"level": 2, "notify_manager": true, "sla_hours": 4}
  },
  {
    "title": "Add dark mode support",
    "labels": ["feature-request", "ui"],
    "reporter": {"id": "USR-67890", "name": "Alex Chen"}
  },
  {"title": "Update API documentation"}
]
```

From these, Claude learns:
- Date format: YYYY-MM-DD
- User IDs: USR-XXXXX pattern
- Labels: kebab-case
- Critical bugs → full contact + escalation with tight SLAs
- Feature requests → reporter but no contact/escalation
- Internal tasks → title only

### Best practices

- Use realistic data (real city names, plausible prices)
- Show minimal, partial, and full patterns (variety)
- 1-5 examples per tool
- Focus on genuine ambiguities
- Examples add tokens — only use when accuracy gains outweigh cost

## Programmatic Tool Calling — Full Details

Traditional approach: 20 tool calls × 50-100 items each = 2,000+ items in context.

With Programmatic Tool Calling, Claude writes Python that orchestrates tools:
```python
team = await get_team_members("engineering")
levels = list(set(m["level"] for m in team))
budget_results = await asyncio.gather(*[get_budget_by_level(level) for level in levels])
budgets = {level: budget for level, budget in zip(levels, budget_results)}
expenses = await asyncio.gather(*[get_expenses(m["id"], "Q3") for m in team])
exceeded = [{"name": m["name"], "spent": sum(e["amount"] for e in exp), "limit": budgets[m["level"]]["travel_limit"]}
            for m, exp in zip(team, expenses) if sum(e["amount"] for e in exp) > budgets[m["level"]]["travel_limit"]]
print(json.dumps(exceeded))
```

Claude receives only 2-3 names (~1 KB), not 2,000+ items.

Implementation:
1. Mark tools with `"allowed_callers": ["code_execution_20250825"]`
2. Claude generates code in `code_execution` tool
3. Tool calls execute without entering main context
4. Only final `stdout` returned to conversation

## Tool Search Tool — Full Details

Mark tools with `defer_loading: true`. Claude discovers on-demand.

For MCP servers:
```json
{"type": "mcp_toolset", "mcp_server_name": "google-drive",
 "default_config": {"defer_loading": true},
 "configs": {"search_files": {"defer_loading": false}}}
```

Keep 3-5 most-used tools always loaded; defer everything else.
