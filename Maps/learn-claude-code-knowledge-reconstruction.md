---
title: Learn-Claude-Code Knowledge Reconstruction
note_type: Map
maturity: Draft
updated_at: 2026-03-12
reader_goal: "recover the full architectural knowledge taught in the source discussion as a coherent system, rather than as a session list or a checkpoint recap"
reasoning_backlinks:
  - Reasoning-Logs/2026-03-12-learning-agent-architecture-from-antigravity-context.md
---

# Learn-Claude-Code Knowledge Reconstruction

## What This Document Is
This note is not a summary of the conversation. It is a reconstruction of the architecture knowledge that the conversation built, using the source discussion as raw material.

The distinction matters:
- a summary says what was covered
- a reconstruction says what the system is, why each layer exists, how the layers relate, and where the boundaries are

Primary source windows:
- minimal loop to 12-layer architecture: `95-1047`
- `s03` deep reconstruction: `1108-1640`
- `s06` deep reconstruction: `1655-1928`
- `s04-s12` code-level caveats and boundaries: `1977-3309`

## 1. The Core Frame: Agent as Layered Loop
The most important reconstruction from the source is a change of perspective.

The discussion does not treat an agent as a bag of features such as tools, memory, planning, subagents, and worktrees. It treats an agent as a minimal loop that accumulates new layers because each earlier layer leaves a specific problem unresolved.

The base loop is:
- send `system + messages + tools` to the model
- let the model decide whether it needs a tool
- execute the requested tool
- feed the tool result back into the conversation
- continue until the model stops asking for tools

This is the first stable insight: the essence of the agent is not “many capabilities”, but “iterative closed-loop action whose continuation is model-directed” (`95-162`).

That frame changes how the rest of the architecture is read. Every later mechanism answers one question:
- what failure appears if the loop is left as-is?
- what is the smallest additional mechanism that addresses that failure?
- what new risk is created by adding that mechanism?

Once this frame is established, the sessions stop being a curriculum and become an architectural stack.

## 2. External Input Semantics: Why Tool Results Re-enter as `user`
One of the earliest pieces of knowledge in the source is small but foundational: tool results are fed back as `role: user`, not as `assistant` (`126-149`).

The reconstructed idea is not merely “because the API expects it.” The deeper point is:
- `assistant` is the model’s own side of the dialogue
- tool output is external world feedback
- therefore tool output belongs on the external-input side of the conversation

This pattern later generalizes:
- skill bodies are also injected as external input
- background task notifications are also injected as external input
- teammate mailbox messages are also treated as external input

So the durable knowledge is:
- the role label is not primarily a type system
- it is a semantic boundary about where information comes from
- the model is expected to distinguish specific kinds of external input by content and structure, not by inventing new roles for every source (`644-669`)

This is also where one important meta-correction appears later in the discussion: strong software-engineering instincts about explicit type labels do not always transfer directly to LLM-native systems (`733-769`).

## 3. Open Capability, Closed Invariants
The base loop becomes useful when tools are registered through a dispatch map (`165-226`).

That move creates an architecture with two desirable properties:
- the loop does not need to change when tools are added
- capabilities can scale by registration rather than by rewriting control flow

But the source is careful not to romanticize this. The same openness produces the first serious weakness:
- unknown tool names can crash the system
- wrong parameters can cause dangerous actions
- the model is carrying too much responsibility for structural correctness

This sets up one of the most important reconstructed ideas from the later `s03` deep dive: agent systems need both open capability surfaces and closed invariants.

Open capability means:
- let the model choose tools
- let the system grow by adding handlers

Closed invariants mean:
- some states should never be representable
- some transitions should never rely on prompt discipline
- some tool preconditions should be enforced before execution

This is the architectural reason the conversation later converges on layered control surfaces.

## 4. Control Surfaces: Steering, Reminder, Hook, Constraint
The source’s richest reconstructed knowledge lives in `s03` (`1108-1640`).

The final knowledge is not simply “TodoWrite is useful.” It is a full control theory for LLM agents:

### 4.1 Static steering
The system prompt gives persistent behavioral direction:
- who the agent is
- where it is operating
- which tools to prefer
- which workflow is preferred

Good prompt text in this model is concrete and behavior-facing, not ceremonial. The discussion explicitly downgrades generic “you are a very powerful AI” phrasing as low-value compared with direct operational instructions (`1197-1209`).

### 4.2 Dynamic reinforcement
The nag reminder appears when prompt steering is not enough.

The important reconstructed idea is not just “remind after 3 rounds.” It is:
- static guidance loses relative influence as fresh tool outputs fill the context
- reminders are a way to reassert control from the newest part of the context
- therefore reminders are an escalation of steering, not a separate category (`1229-1251`, `1606-1640`)

The discussion also corrects itself here:
- the exact threshold like `3` rounds is not theory-derived
- it is an engineering choice
- the right claim is not “the model becomes immune”, but that repeated reminders lose marginal influence when they become too frequent (`1213-1225`)

### 4.3 Hooks
Hooks appear when a rule is not just preferred, but attached to tool execution.

The reconstructed example is “read before write or edit” (`1275-1574`):
- a prompt can ask for this
- but a hook can detect whether the target file has actually been read
- the hook can then reject, auto-read, or auto-remediate with warning

This is the first place where the discussion moves from “tell the model” to “instrument the action boundary.”

### 4.4 Hard constraints
Some states should simply be impossible, such as multiple todo items marked `in_progress` in a design that assumes one active focus (`1113-1133`).

This yields a compact rule for future systems:
- preferences belong in prompts
- drift corrections belong in reminders
- tool preconditions belong in hooks
- correctness boundaries belong in code

That rule is more valuable than any single `s03` implementation detail.

## 5. Context Economics: Idle, Dirty, Stale
The strongest mid-stack reconstruction is that `s04`, `s05`, and `s06` are not separate tricks. They are one context-management system viewed from three timescales.

### 5.1 Dirty context: subagent isolation
`s04` solves pollution (`300-363`, `1977-2016`).

The insight is not merely “subagents are useful.” It is:
- a main thread accumulates local residue while solving a task
- if every subtask inherits all that residue, each subtask reasons in a contaminated room
- therefore subtasks need clean `messages[]`

The cost is equally important:
- clean context loses historical specifics
- bridging information must be passed explicitly in the subtask prompt
- durable memory can provide a second bridge when context should not be manually copied every time

### 5.2 Idle context: on-demand skills
`s05` solves preloaded irrelevance (`366-463`, `2028-2180`).

The reconstructed claim is:
- the problem is not only token cost
- it is also attention dilution
- irrelevant guidance competes with relevant reasoning

So skill systems are not only storage systems. They are selective attention systems.

### 5.3 Stale context: layered compression
`s06` solves accumulated history (`466-524`, `1655-1928`).

The source builds a layered model:
- micro-compaction removes or downgrades older long tool outputs while keeping the recent working set
- full compaction creates continuity state around completed work, current state, and key decisions
- transcript references and re-readable files act as recovery channels

The most important refinement happens late in the discussion (`1908-1928`):
- old tool outputs are not always being “thrown away despite still being valuable”
- often they have already done their job
- their useful content has been carried forward into newer actions, newer files, or summarized state

That changes the design stance from reluctant deletion to deliberate removal of already-consumed context.

### 5.4 The full principle
Together these three sessions yield one durable principle:
- do not load what is not needed
- do not return residue that the parent does not need
- do not keep old detail once its value exists elsewhere or can be regenerated

That is a knowledge architecture, not a compression trick.

## 6. State Placement: What Must Live Outside the Chat
Another strong reconstruction from the source is that not all important state should remain in `messages`.

`s07` makes this explicit by moving task state into files (`545-603`, `2189-2420`).

The architectural reason is deeper than persistence:
- in-chat todo state is tied to whatever the model still remembers
- compression can preserve some of it, but only through model-generated summary
- a task graph in files exists independently of what the current context window happens to contain

This produces a useful placement rule:
- short-lived reasoning can live in the conversation
- stable project constraints should live in durable project memory
- coordination state shared by multiple workers should live in external state, not in chat memory

The source also adds a second useful lesson:
- if code can maintain relationship consistency automatically, do not ask the model to maintain both sides

That is why `blocks` and `blockedBy` are kept consistent in code rather than by repeated LLM bookkeeping.

## 7. Async Work and Mailbox Thinking
`s08`, `s09`, and `s10` reconstruct one larger pattern: the agent should increasingly think in terms of pending external work rather than only immediate synchronous execution.

### 7.1 Background tasks
`s08` introduces the minimal async pattern:
- start slow work in the background
- let the main loop continue
- drain notifications on later rounds (`2439-2638`)

This is the first time the architecture stops assuming “tool call means blocking wait.”

### 7.2 Persistent teammates
`s09` takes the same pattern and gives it identity:
- teammates are not throwaway subagents
- they have their own loops
- communication happens by mailbox (`2650-2839`)

The key reconstruction is that teammate messaging is structurally similar to background notifications:
- produce external result elsewhere
- ingest it later as external input

### 7.3 Protocol
`s10` adds the next missing layer: meaning must be trackable (`2841-3016`).

Once requests and responses carry `request_id`, the system can:
- correlate intent and resolution
- distinguish pending, approved, rejected states
- support graceful shutdown instead of abrupt kill semantics

This is where the architecture stops being “many loops talking” and starts becoming “coordinated work with explicit state transitions.”

## 8. Contention and Isolation
The source ultimately shows that coordination is not enough. Once multiple workers act, two different kinds of contention appear.

### 8.1 Logical contention
`s11` exposes contention over task ownership (`871-945`, `3026-3217`).

Autonomous claiming removes the bottleneck of central assignment, but creates:
- race conditions around task claiming
- ambiguity around who gets to act first
- pressure on shared task-state correctness

This is a logical contention problem: who owns the right to act.

### 8.2 Physical contention
`s12` then shifts the problem to the filesystem (`949-1047`, `3234-3309`).

Even if task ownership is resolved, workers can still collide physically if they edit in the same directory. Worktrees solve a different class of problem:
- they separate working directories
- they make review and merge explicit
- they turn rollback into a version-control operation instead of a manual repair job

This yields a critical distinction:
- context isolation is about what the model sees
- worktree isolation is about where code changes happen

They are both “isolation”, but they protect different failure surfaces.

## 9. What the Source Taught About Teaching Scaffolds
The discussion does not end by simply accepting the tutorial stack.

Late in the source, the conversation starts auditing the scaffold itself (`2788-2829`, `3155-3177`, `3224-3230`):
- some role-order injections appear structurally invalid
- lead-agent wake-up is incomplete
- identity reinjection is only partially justified
- concurrent task updates are recognized but not really solved

This creates an important final piece of knowledge:
- a teaching scaffold can be excellent for revealing the shape of a system
- that does not make each implementation detail production-worthy

So the correct use of `learn-claude-code` is:
- learn mechanism decomposition from it
- learn tradeoff vocabulary from it
- learn where new layers enter the design
- do not blindly cargo-cult later coordination code into production

## 10. Reconstruction Rule for Future Checkpoints
The failure you pointed out in the earlier notes can now be stated clearly.

Knowledge was lost because the processing path jumped too early into note containers:
- first extract summary
- then derive note headings
- then store compressed outputs

That order produces indexes and checkpoints, but not reconstructed knowledge.

The better order is:
1. split source into topic-bearing windows
2. reconstruct each window into full knowledge claims, mechanisms, tradeoffs, and limits
3. merge overlapping windows into a coherent domain note
4. only then create atoms, maps, and logs

In other words:
- checkpointing should preserve reasoning
- reconstruction should rebuild the knowledge object itself
- summarization, if any, should happen last

## Reader Use
Use this note as the content-bearing layer.

Then use the following notes as support:
- [learn-claude-code-agent-architecture-map.md](/Users/tsingbbby/Downloads/Agent-Vault/Maps/learn-claude-code-agent-architecture-map.md)
- [agent-control-surfaces-in-llm-systems.md](/Users/tsingbbby/Downloads/Agent-Vault/Atoms/agent-control-surfaces-in-llm-systems.md)
- [agent-context-economics.md](/Users/tsingbbby/Downloads/Agent-Vault/Atoms/agent-context-economics.md)
- [agent-rounds-messages-and-system-prompt-boundaries.md](/Users/tsingbbby/Downloads/Agent-Vault/Atoms/agent-rounds-messages-and-system-prompt-boundaries.md)
- [learn-claude-code-teaching-code-boundaries.md](/Users/tsingbbby/Downloads/Agent-Vault/Cases/learn-claude-code-teaching-code-boundaries.md)
