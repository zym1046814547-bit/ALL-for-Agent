---
title: "Agent Rounds, Messages, and System Prompt Boundaries"
page_shape: Concept
source_note: ../../Atoms/agent-rounds-messages-and-system-prompt-boundaries.md
source_maturity: Stable
status: published
updated_at: 2026-03-12
---

# Agent Rounds, Messages, and System Prompt Boundaries

> An agent turn contains multiple rounds, and `system` is a separate input channel from `messages`, which matters for compression and message-role correctness.

## Definitions

- **Message** — one item in the conversation history, such as a `user` or `assistant` entry
- **Round** — one iteration of the agent loop, usually including one model call and the resulting tool handling
- **Turn** — one full user-to-agent interaction, which can contain many rounds

Key clarification: one user request can trigger several rounds before the agent returns a final answer.

## System Prompt Is Not a Message

In the studied API shape, `system` and `messages` are separate parameters:

- `system` is sent in full on every model call
- `messages` is the rolling dialogue history that grows, gets compacted, and can suffer role-order bugs

This matters because:

- compressing `messages` does not itself erase the system prompt
- explanations that rely on "the system prompt was compressed away" are inaccurate
- if identity or policy is duplicated inside `messages`, that duplication needs a stronger reason than simple persistence

## Role-Ordering Hazard

Once an agent starts injecting inbox notifications, compression summaries, or synthetic acknowledgements into `messages`, it becomes easy to violate strict role alternation.

Two risks surfaced:

1. An injected synthetic `assistant` note can be followed by the model's real `assistant` response, creating consecutive assistant roles
2. After compression, a synthetic identity block and a new auto-claimed task can still collide with existing `user` messages

These are structural correctness issues in any system that manipulates chat history procedurally.

## Reader Goal

Use this note when inspecting an agent loop that mutates `messages` directly. It helps separate three questions that often get blurred:

- what channel is system policy actually traveling through
- what unit of time a counter like `rounds_since_*` refers to
- where message-order correctness can break under synthetic injections
