---
title: Agent Control Surfaces in LLM Systems
page_shape: Concept
source_note: ../../Atoms/agent-control-surfaces-in-llm-systems.md
source_maturity: Stable
status: published
updated_at: 2026-03-12
---

# Agent Control Surfaces in LLM Systems

> Agent behavior should be shaped through an escalation ladder of control surfaces rather than a single prompt-only strategy.

## Definition

An LLM agent does not have one control mechanism. It has a stack of control surfaces with different strengths, costs, and failure modes:

- **System prompt guidance** — state the preferred behavior once
- **Dynamic reminder** — restate the preference when drift becomes visible
- **Hook** — intercept a tool call and verify a prerequisite before execution
- **Hard code constraint** — reject or prevent invalid state transitions directly in code

## Why This Matters

Treating every rule as "just put it in the prompt" hides an important design choice. Some rules are preferences; some are invariants. The former can tolerate occasional drift. The latter should not be entrusted to model discipline.

## Practical Decision Rule

Use the weakest layer that still reliably enforces the rule:

- if occasional violation is acceptable and the main goal is better behavior, start with **system prompt guidance**
- if the model reliably drifts after some time or under long-context pressure, add a **dynamic reminder**
- if the rule is a precondition on tool execution, use a **hook**
- if violating the rule corrupts state, breaks correctness, or crosses a safety boundary, **enforce it in code**

## Example

The discussion used "read the file before editing or overwriting it" as a concrete example:

- **prompt**: encourage reading before editing
- **hook**: auto-read or reject when `write` or `edit` targets an unread existing file
- **hard constraint**: prevent impossible or inconsistent task states

## Important Clarification

Dynamic reminders are not parallel to system prompts in the abstract. They are an escalation of the same steering strategy. Their job is to restore pressure when the original instruction has been diluted by later context.

## Reader Goal

Use this note when deciding whether a new agent rule belongs in prompt text, a middleware-like hook, or hard execution logic.
