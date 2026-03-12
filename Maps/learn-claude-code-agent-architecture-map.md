---
title: Learn-Claude-Code Agent Architecture Map
note_type: Map
maturity: Draft
updated_at: 2026-03-12
reader_goal: "regain the architecture map of the learning sequence without losing the main tensions introduced by each layer"
reasoning_backlinks:
  - Reasoning-Logs/2026-03-12-learning-agent-architecture-from-antigravity-context.md
aliases:
  - Agent Architecture Map
---

# Learn-Claude-Code Agent Architecture Map

- Full Reconstruction: [[learn-claude-code-knowledge-reconstruction]]

## Source Spine
- `95-226`: minimal loop and dispatch extensibility
- `230-524`: planning, subagent isolation, skill loading, and memory/compression framing
- `545-1047`: task graphs, async work, multi-agent coordination, claiming, and worktree isolation
- `1108-1928`: deep dives that sharpen `s03` and `s06`
- `1977-3309`: deep dives that sharpen `s04-s12` and expose implementation boundaries

## Overview
This map treats `learn-claude-code` as a staged teaching scaffold. The important unit is not a file or an API call, but the question each session answers:
- what problem did the previous layer leave unresolved
- what mechanism was added
- what new risk or tension did that mechanism introduce

## Phase 1 - The Loop

| Session | Mechanism Added | Problem Addressed | New Tension Introduced | Source |
| --- | --- | --- | --- |
| `s01` | `while` loop + `stop_reason` | a single LLM call cannot decide and act iteratively | loop control is delegated to the model rather than being explicit in code | `95-162` |
| `s02` | dispatch map for tools | one tool does not scale to a real agent surface | open registration is elegant, but unsafe if tool names or arguments are wrong | `165-226` |

## Phase 2 - Planning and Knowledge

| Session | Mechanism Added | Problem Addressed | New Tension Introduced | Source |
| --- | --- | --- | --- |
| `s03` | todo tool + nag reminder | tool use without planning drifts | prompting is flexible, but unreliable unless reinforced or backed by hard constraints | `230-297`, refined by `1113-1251`, `1606-1640` |
| `s04` | subagent with isolated `messages[]` | one conversation history pollutes unrelated subtasks | isolated context is clean, but must be bridged by explicit task description or persistent memory | `300-363`, deepened by `1977-2016` |
| `s05` | on-demand skill loading | static prompts cannot carry every domain guide | savings depend on the model deciding when a skill is needed | `366-463`, deepened by `2028-2180` |
| `s06` | layered compression + transcript escape hatch | context fills and old tool results crowd out current work | compression is lossy unless durable constraints live outside the conversation | `466-524`, deepened by `1655-1928` |

## Phase 3 - Externalized State and Async Work

| Session | Mechanism Added | Problem Addressed | New Tension Introduced | Source |
| --- | --- | --- | --- |
| `s07` | file-backed task graph | in-chat todo state is fragile and does not support dependencies | shared state survives compression, but now consistency and concurrent updates matter | `545-603`, deepened by `2189-2420` |
| `s08` | background tasks + notification queue | slow tools block the whole agent loop | async execution increases coordination complexity and requires safe notification draining | `606-680`, deepened by `2439-2638` |

## Phase 4 - Teams and Isolation

| Session | Mechanism Added | Problem Addressed | New Tension Introduced | Source |
| --- | --- | --- | --- |
| `s09` | persistent teammates + JSONL mailboxes | temporary subagents do not model real team coordination | mailbox communication is durable but introduces wake-up and ordering issues | `692-802`, deepened by `2650-2839` |
| `s10` | request-response protocols + graceful shutdown | communication without explicit protocol is ambiguous and brittle | protocol state must be tracked and cleanedly closed | `808-867`, deepened by `2841-3016` |
| `s11` | autonomous task claiming | central assignment bottlenecks the lead agent | decentralized claiming creates race conditions and identity/context edge cases | `871-945`, deepened by `3026-3217` |
| `s12` | Git worktree isolation | multiple workers in one directory collide at the filesystem level | isolation improves safety but pushes complexity into branch/worktree lifecycle and merge flow | `949-1047`, deepened by `3234-3309` |

## Cross-Cutting Structure

### 1. Control Surface Escalation
The teaching path repeatedly converges on the same pattern:
- code exposes a capability
- prompt steers toward preferred use
- dynamic reminders reinforce that steering when drift appears
- hooks and hard constraints protect invariants that should not be trusted to model discipline alone

This is clearest in `s03` (`1113-1251`, `1275-1350`), but the same design instinct appears again in task consistency and protocol handling.

### 2. Context Economics
The single-agent middle phase can be read as one coherent strategy:
- `s05` avoids loading idle knowledge
- `s04` prevents dirty subtask residue from returning
- `s06` clears stale history once it has served its purpose

Taken together, these sessions define a context economy rather than three isolated tricks (`2028-2107`, `1908-1928`).

### 3. State Placement
The map also clarifies where different kinds of state should live:
- conversational reasoning and current tool outputs can live in `messages`
- project constraints and durable knowledge should live outside the compressible chat
- task coordination state needs a shared external representation if more than one worker is involved

### 4. Isolation Has Multiple Layers
The course distinguishes at least two different kinds of isolation:
- context isolation: what the model sees and carries mentally
- filesystem isolation: where edits happen physically

They solve different classes of interference and should not be collapsed into the same idea.

## Where the Teaching Scaffold Became Fragile
The later sessions yielded durable concepts, but the implementation examples showed cracks:
- some mailbox/inbox paths appear to generate invalid consecutive message roles (`2788-2829`, `3187-3207`, `3224-3230`)
- lead-agent wake-up is incomplete when teammate mail arrives after the lead loop has exited (`2773-2781`)
- identity reinjection after compression may be over-defensive or at least insufficiently justified (`3155-3177`)
- concurrent task graph updates point toward race conditions that the tutorial does not truly solve (`2273-2275`)

These flaws do not erase the map. They clarify which parts belong in durable architecture notes and which parts belong in a project-bound case file.

## Stable Summary
`learn-claude-code` is most useful as a decomposition of agent architecture into solvable tensions. Its strongest contribution is not a production-ready implementation, but a sequence of questions that teaches where planning, context management, persistence, concurrency, protocol, and isolation each enter the design.
