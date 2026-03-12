---
title: Agent Control Surfaces in LLM Systems
note_type: Atom
maturity: Stable
updated_at: 2026-03-12
stable_summary: "agent behavior should be shaped through an escalation ladder of control surfaces rather than a single prompt-only strategy"
reasoning_backlinks:
  - Reasoning-Logs/2026-03-12-learning-agent-architecture-from-antigravity-context.md
publish_targets:
  - mkdocs
---

# Agent Control Surfaces in LLM Systems

## Source Windows
- `242-285`: planning introduced as a tool-plus-steering decision rather than hardcoded flow
- `1113-1187`: hard constraint, minimal prompt, and reminder mechanics in the `s03` deep dive
- `1197-1251`: correction of prompt-theory claims and clarification that reminder is an escalation, not a parallel category
- `1275-1350`: hook-style enforcement proposal around read-before-write/edit

## Definition
An LLM agent does not have one control mechanism. It has a stack of control surfaces with different strengths, costs, and failure modes.

The useful distinction surfaced in the `s03` deep dive:
- **system prompt guidance**: state the preferred behavior once
- **dynamic reminder**: restate the preference when drift becomes visible
- **hook**: intercept a tool call and verify a prerequisite before execution
- **hard code constraint**: reject or prevent invalid state transitions directly in code

Line grounding:
- hard constraint example: `1113-1133`
- reminder example: `1139-1157`
- minimal prompt example: `1160-1178`
- escalation clarification: `1229-1251`
- hook proposal: `1275-1350`

## Why This Matters
Treating every rule as "just put it in the prompt" hides an important design choice. Some rules are preferences; some are invariants. The former can tolerate occasional drift. The latter should not be entrusted to model discipline.

## Practical Decision Rule
Use the weakest layer that still reliably enforces the rule:
- if occasional violation is acceptable and the main goal is better behavior, start with system prompt guidance
- if the model reliably drifts after some time or under long-context pressure, add a dynamic reminder
- if the rule is a precondition on tool execution, use a hook
- if violating the rule corrupts state, breaks correctness, or crosses a safety boundary, enforce it in code

## Example
The discussion used "read the file before editing or overwriting it" as a concrete example.

Good placement:
- prompt: encourage reading before editing
- hook: auto-read or reject when `write` or `edit` targets an unread existing file
- hard constraint: prevent impossible or inconsistent task states, such as multiple items marked `in_progress` when the design requires one active focus

## Important Clarification
Dynamic reminders are not parallel to system prompts in the abstract. They are an escalation of the same steering strategy. Their job is to restore pressure when the original instruction has been diluted by later context.

The source also corrects an earlier overstatement:
- the exact threshold such as `3` rounds is not theory-backed in the discussion; it is treated as an engineering choice (`1213-1225`)
- the right claim is weaker than "the model becomes immune"; repeated reminders merely lose marginal influence when they become frequent background noise (`1217-1225`)

## Reader Goal
Use this note when deciding whether a new agent rule belongs in prompt text, a middleware-like hook, or hard execution logic.
