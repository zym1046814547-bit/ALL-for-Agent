---
title: Agent Context Economics
page_shape: Concept
source_note: ../../Atoms/agent-context-economics.md
source_maturity: Stable
status: published
updated_at: 2026-03-12
---

# Agent Context Economics

> Context management is not only about compressing long chats; it begins before loading, continues during execution, and ends with selective forgetting.

## Core Idea

The learning sequence exposed three different ways context becomes wasteful:

- **context that never needed to be loaded** — idle knowledge wastes tokens and attention
- **context produced by a subtask that should not flow back to the parent** — dirty residue pollutes reasoning
- **context that was useful earlier but has already been metabolized into newer state** — stale history crowds out current work

## The Three Savings Modes

### 1. Save Idle Context (On-Demand Loading)

If a skill or body of guidance is not needed for the current problem, keeping it in every prompt wastes tokens and attention. On-demand loading keeps the desk clear.

### 2. Save Dirty Context (Subagent Isolation)

Subtasks often generate exploratory residue, false starts, and local reasoning that the parent task does not need. Isolated subagent context prevents this residue from polluting the main thread.

### 3. Save Stale Context (Layered Compression)

Old tool outputs can be downgraded or compressed once they have already contributed to newer decisions and outputs. The important insight is that older results are often not merely "sacrificed"; their value has already been transferred forward.

## Compression Rule

Only compress information that can be recovered or safely represented elsewhere.

**Safe candidates:**

- file contents that can be re-read
- old tool outputs whose essence has already been incorporated into newer work
- verbose logs that have a transcript escape hatch

**Unsafe candidates unless stabilized elsewhere:**

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

## Reader Goal

Use this note when deciding whether a new agent mechanism is really about "compression" or about a broader context-budgeting strategy.
