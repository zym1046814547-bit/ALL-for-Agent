---
title: "Case - Learn-Claude-Code Teaching Code Boundaries"
note_type: Case
maturity: Draft
updated_at: 2026-03-12
reader_goal: "separate the durable lessons of the teaching project from the places where its implementation should not be copied blindly"
reasoning_backlinks:
  - Reasoning-Logs/2026-03-12-learning-agent-architecture-from-antigravity-context.md
---

# Case - Learn-Claude-Code Teaching Code Boundaries

## Source Windows
- `1977-2016`: `s04` locking vs worktree, plus the clarification that subagents there are synchronous
- `2189-2420`: task graph persistence and dependency maintenance
- `2439-2638`: background tasks, locks, and notification draining
- `2650-3016`: teammate mailboxes, protocols, wake-up gaps, and retry policy left to the LLM
- `3026-3217`: identity reinjection and `system` vs `messages`
- `3224-3309`: explicit bug acknowledgment and `s12` worktree architecture framing

## Context
The discussion eventually moved from learning mechanisms to auditing the tutorial code itself. This shift mattered because the same project was serving two roles:
- architecture explainer
- implementation example

It remained strong in the first role and weaker in the second.

## What Held Up Well
- the staged decomposition from loop to planning, context management, persistence, async work, team protocols, and worktree isolation
- the distinction between soft steering and hard enforcement
- the idea that state needed for continuity should move out of compressible chat history
- the separation between context isolation and filesystem isolation

## Where Confidence Dropped

### 1. Message-role normalization
The discussion identified probable invalid message sequences in later-session inbox injection flows:
- synthetic `assistant` acknowledgment followed by the model's real `assistant` response
- compressed or injected `user` messages colliding with new `user` content

If the API requires strict role alternation, these are real bugs rather than cosmetic quirks.

Grounded in `2788-2829`, `3187-3207`, and the explicit acknowledgment at `3224-3230`.

### 2. Wake-up semantics
Teammate messages can remain parked in the lead inbox when the lead loop has already exited. This means mailbox durability exists, but passive wake-up does not.

The concept still teaches asynchronous coordination, but the implementation does not yet model a complete event-driven lead agent.

Grounded in `2773-2781`.

### 3. Identity reinjection after compression
The explanation for identity reinjection shifted during the discussion. The initial claim was that the model might be "confused" after compression. That was challenged successfully: the system prompt remains present, and compression shortens context rather than diluting the system prompt further.

The more defensible explanation is continuity inside `messages`, but even that remains only partially convincing. This suggests over-defensive design or at least insufficient justification.

Grounded in `3075-3217`, with the key correction at `3155-3177`.

### 4. Concurrent task-state updates
The task graph teaches dependency management well, but concurrent completion and dependency clearing point toward race conditions once multiple workers update shared files.

The tutorial surfaces the right problem but does not fully solve the production version of it.

Grounded in `2236-2245` and the unresolved question at `2273-2275`.

## Why This Case Matters
Without this case file, the durable architecture notes would overstate the trustworthiness of the sample code. The project is worth keeping as a learning scaffold precisely because its boundaries became visible during close reading.

## Current Position
Use `learn-claude-code` to learn mechanism decomposition, tradeoff framing, and useful mental models. Do not copy later-session coordination code into a production agent without verification against a stronger reference.

## Open Follow-up
- Verify the exact message-role handling in true Claude Code or a higher-fidelity reverse-engineering source
- Check whether later production systems use an internal normalized transcript representation rather than direct user/assistant mutation
- Revisit identity reinjection only after seeing a real implementation where the motivation is explicit
