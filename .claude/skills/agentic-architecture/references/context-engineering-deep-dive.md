# Context Engineering Deep Dive

Detailed notes from [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).

## Core Concept

> "Building with language models is becoming less about finding the right words for prompts, and more about answering: what configuration of context is most likely to generate the desired behavior?"

Context engineering = curating and maintaining the optimal set of tokens during LLM inference.

## Context Rot

Transformers create n-squared pairwise relationships for n tokens. As context lengthens:
- Capturing relationships becomes strained
- Models trained on shorter sequences have less experience with context-wide dependencies at longer lengths
- Performance creates gradients, not hard cliffs
- Every token added competes for attention with existing tokens

**Implication:** context is a finite resource with diminishing returns. Don't pre-load everything.

## System Prompt Design

### The Goldilocks Zone

Two failure modes:
1. **Too rigid (hardcoded):** complex brittle logic that breaks on edge cases
2. **Too vague (high-level):** "be helpful" without specifics

### Principles

- Simple, direct language at the right altitude
- Organize with XML tags or Markdown headers into distinct sections
- Pursue the **minimal information set** fully outlining expected behavior
- Start with minimal prompts on the best model → add instructions based on discovered failure modes
- Few-shot examples are "pictures worth a thousand words" — more effective than articulating every rule

## Tool Description Quality

Tools must be:
- **Self-contained**: all context needed to use correctly
- **Robust to error**: clear error messages that steer agent
- **Extremely clear**: no ambiguity about intended use

### Common failures
- **Bloated tool sets** covering excessive functionality
- **Ambiguous decision points** between similar tools
- If engineers cannot state which tool applies → agents cannot either

### Design rules
- Input parameters: descriptive, unambiguous, leverage model strengths
- Token-efficient returns: encourage efficient agent behaviors
- Curate minimal viable tool sets → improves reliability and context pruning

## Just-in-Time Context (Key Pattern)

Rather than pre-loading all data, agents maintain:
- Lightweight identifiers (file paths, queries, links)
- Dynamically load data at runtime using tools

This mirrors human cognition — we don't memorize everything, we know where to look.

### Hybrid approach (recommended)
- **Upfront:** config, facts, frequently-needed context (like CLAUDE.md)
- **Just-in-time:** search, lookup, retrieval tools for everything else

General advice: "do the simplest thing that works."

## Long-Horizon State Management

### 1. Compaction
Summarize conversations nearing context limits, reinitialize new windows with summaries.
- **Preserve:** architectural decisions, unresolved bugs, key findings
- **Discard:** redundant tool outputs, intermediate reasoning
- Clearing tool call results = safest, lightest-touch form of compaction

### 2. Structured Note-Taking (Agentic Memory)
Agents write notes persisting **outside** the context window:
- To-do lists, NOTES.md, progress tracking files
- After context reset → read notes → continue coherently
- Notes are agent-authored artifacts, not conversation history

### 3. Sub-Agent Architectures
Specialized sub-agents handle focused tasks with clean context windows:
- Each explores extensively (tens of thousands of tokens)
- Returns only condensed summaries (1,000-2,000 tokens)
- Lead agent synthesizes results

**Rule of thumb:** if a sub-task generates >5K tokens of intermediate output that the main agent won't need, delegate to a sub-agent.
