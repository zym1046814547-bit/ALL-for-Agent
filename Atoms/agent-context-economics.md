---
title: Agent Context Economics
note_type: Atom
maturity: Stable
updated_at: 2026-03-12
stable_summary: "context management is not only about compressing long chats; it begins before loading, continues during execution, and ends with selective forgetting"
reasoning_backlinks:
  - Reasoning-Logs/2026-03-12-learning-agent-architecture-from-antigravity-context.md
publish_targets:
  - mkdocs
---

# Agent Context Economics

## Source Windows
- `300-363`: `s04` isolated subagent context
- `366-463`: `s05` on-demand skill loading
- `466-524`: first compression and dual-memory framing
- `1606-1640`: prompt dilution as the deeper reason for reminder reinforcement
- `1655-1840`: `s06` compression layers and transcript escape hatch
- `1908-1928`: refinement that old outputs may already have transferred value into newer results
- `2028-2107`: explicit synthesis of `s04`, `s05`, and `s06` as one context-economics model

## Core Idea
The learning sequence exposed three different ways context becomes wasteful:
- context that never needed to be loaded
- context produced by a subtask that should not flow back to the parent
- context that was useful earlier but has already been metabolized into newer state

These correspond to three different mechanisms:
- `s05`: do not preload idle knowledge; load skills only when needed
- `s04`: give subtasks clean contexts so their internal residue does not dirty the main conversation
- `s06`: compress stale history once its value has been carried forward

Line grounding:
- idle context: `380-451`, synthesized again at `2028-2107`
- dirty context: `314-352`, synthesized again at `2088-2107`
- stale context: `1655-1840`, refined at `1908-1928`

## The Three Savings Modes

### 1. Save Idle Context
If a skill or body of guidance is not needed for the current problem, keeping it in every prompt wastes tokens and attention. On-demand loading keeps the desk clear.

### 2. Save Dirty Context
Subtasks often generate exploratory residue, false starts, and local reasoning that the parent task does not need. Isolated subagent context prevents this residue from polluting the main thread.

### 3. Save Stale Context
Old tool outputs can be downgraded or compressed once they have already contributed to newer decisions and outputs. The important insight from the discussion was that older results are often not merely "sacrificed"; their value has already been transferred forward.

This refinement matters because it changes the framing from reluctant loss to deliberate removal of already-consumed context (`1908-1928`).

## Compression Rule
Only compress information that can be recovered or safely represented elsewhere.

Safe candidates:
- file contents that can be re-read
- old tool outputs whose essence has already been incorporated into newer work
- verbose logs that have a transcript escape hatch

Unsafe candidates unless stabilized elsewhere:
- user constraints
- project rules
- key design decisions
- current task state that does not exist outside the conversation

## Persistent Escape Hatches
Context economics works because not all state is trapped inside the chat:
- transcripts allow a compressed conversation to be revisited
- files can be read again
- durable project memory can preserve instructions that must survive compression
- external task state can keep coordination facts available even after message history shrinks

Grounded in:
- transcript path and synthetic compressed summary shape: `1684-1723`
- durable project memory: `498-524`
- re-reading files after micro-compaction: `1819-1840`

## Reader Goal
Use this note when deciding whether a new agent mechanism is really about "compression" or about a broader context-budgeting strategy.
