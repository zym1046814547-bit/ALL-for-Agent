---
title: "Agent Rounds, Messages, and System Prompt Boundaries"
note_type: Atom
maturity: Stable
updated_at: 2026-03-12
stable_summary: "an agent turn contains multiple rounds, and system is a separate input channel from messages, which matters for compression and message-role correctness"
reasoning_backlinks:
  - Reasoning-Logs/2026-03-12-learning-agent-architecture-from-antigravity-context.md
publish_targets:
  - mkdocs
---

# Agent Rounds, Messages, and System Prompt Boundaries

## Source Windows
- `1856-1899`: round / turn / message definitions
- `3075-3217`: distinction between `system` and `messages`, and the identity-reinjection debate
- `2788-2829`, `3187-3207`, `3224-3230`: role-ordering bugs discovered in the teaching code

## Definitions
- **Message**: one item in the conversation history, such as a `user` or `assistant` entry
- **Round**: one iteration of the agent loop, usually including one model call and the resulting tool handling
- **Turn**: one full user-to-agent interaction, which can contain many rounds

The key clarification from the discussion:
- one user request can trigger several rounds before the agent returns a final answer
- `rounds_since_todo >= 3` refers to loop iterations, not user interactions

Grounded in `1856-1899`.

## System Prompt Is Not a Message
In the studied API shape, `system` and `messages` are separate parameters:
- `system` is sent in full on every model call
- `messages` is the rolling dialogue history that grows, gets compacted, and can suffer role-order bugs

This matters because:
- compressing `messages` does not itself erase the system prompt
- explanations that rely on "the system prompt was compressed away" are inaccurate
- if identity or policy is duplicated inside `messages`, that duplication needs a stronger reason than simple persistence

Grounded in `3075-3217`, especially the correction at `3155-3177`.

## Role-Ordering Hazard
Once an agent starts injecting inbox notifications, compression summaries, or synthetic acknowledgements into `messages`, it becomes easy to violate strict role alternation.

The teaching discussion surfaced two risks:
- an injected synthetic `assistant` note can be followed by the model's real `assistant` response, creating consecutive assistant roles
- after compression, a synthetic identity block and a new auto-claimed task can still collide with existing `user` messages

These two bugs are explicitly acknowledged as bugs in the teaching code at `3224-3230`.

These are not abstract concerns. They are structural correctness issues in any system that manipulates chat history procedurally.

## Reader Goal
Use this note when inspecting an agent loop that mutates `messages` directly. It helps separate three questions that often get blurred:
- what channel is system policy actually traveling through
- what unit of time a counter like `rounds_since_*` refers to
- where message-order correctness can break under synthetic injections
