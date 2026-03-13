# Session 4：Subagent — 隔离上下文

> 来源 Part: part_a_restructured.md  
> 前置：[Session 3：Plan 机制 — TodoWrite 引入](session_03.md)  
> 后续：[Session 5：Skills — 按需加载知识](session_05.md)

---

## 引入动机

有了 Plan 之后，Agent 能把大任务拆成步骤了。但有个新问题：**所有步骤共用同一个 messages[]，上下文互相污染**。

想象你在修一个 bug，Agent 的 messages 里已经积累了 50 轮对话，然后你让它"顺便格式化一下代码" — 它会带着那 50 轮修 bug 的上下文去做格式化，容易出错。

> *"Break big tasks down; each subtask gets a clean context."*

## 实现方式

Subagent 的实现出乎意料地简单：

```python
def run_subagent(task_description):
    # 关键：全新的 messages[]，不继承主对话的历史
    sub_messages = [{"role": "user", "content": task_description}]
    
    # 用同一个 agent_loop，但上下文是隔离的
    result = agent_loop(sub_messages)
    return result
```

**核心就是一行：`sub_messages = [...]`** — 给子任务一个全新的空白上下文。

循环还是同一个循环，工具还是同一套工具，唯一变的是 **messages 是独立的**。

## 讨论：隔离 vs 信息丢失

**问（Assistant）**：Subagent 获得了"干净上下文"，但同时也失去了主对话中的所有历史信息。如果子任务需要主对话中的某些关键信息（比如用户之前说的偏好），Subagent 怎么获取？你能想到什么解决方案？

**答（User）**：由主 Agent 传入，或者存档项目/用户记忆。

**评估（Assistant）**：✅ 准确！两个方案精确映射到了后续的机制：

| User 的方案 | 对应机制 | 对应 Session |
|------------|---------|-------------|
| **主 Agent 传入** | `task_description` 里附带关键上下文 | s04 就是这么做的 |
| **存档项目/用户记忆** | CLAUDE.md 文件 + Skills 系统 | s05（Skills）、s06（Context） |

第一种是 **显式传参**（短期），第二种是 **持久化记忆**（长期）。Claude Code 两个都用了 — 短期靠 Subagent 调用时注入，长期靠 CLAUDE.md 文件存储项目级知识。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | Subagent = 同一个循环 + 独立的 messages[] |
| **Tension** | 干净上下文 vs 信息丢失，需要显式传参或持久记忆来补 |
| **Contrast** | 没有 Subagent → 所有子任务共享污染的上下文；有 Subagent → 隔离但需要信息桥 |
| **Use** | 理解"隔离 vs 共享"这个贯穿整个 Agent 架构的核心张力 |

---

## Session 4 深入：Subagent 的隔离机制与 Worktree 引入（来自 Part D）

> 来源 Part: part_d_restructured.md

### 引入动机

s04 的概览已完成：Subagent = 独立 messages[] 实现上下文隔离。这里 User 在深入过程中，直接提出了一个跨 Session 的关联性问题。

### 讨论：文件编辑的并发隔离

**问（User）**：when father agent editing, lock on .py. Subagent only can see and edit when father done. If saying worktree, subagent may work in other worktree?

**答（Assistant）**：两个方案都对！

#### 方案 1: 文件锁

```
父 Agent 编辑 app.py → 加锁 🔒
子 Agent 想编辑 app.py → 被拒绝，等待
父 Agent 完成 → 解锁 🔓
子 Agent 现在可以编辑
```

这是**悲观锁**策略 — 先锁再做。简单有效，但有个前提：s04 里子 Agent 是**同步的**（父 Agent 调用 `run_subagent()` 后就在等它返回），所以父子不会同时编辑。文件冲突要到 **s07+（多 Agent 并行）** 才真正出现。

#### 方案 2: Worktree（User 自主推断，完全正确）

s12 的做法就是 User 猜的 — **每个 Agent/任务在独立的 Git worktree 里工作**：

```
主仓库 (main)
  ├── worktree-task-1/  ← Agent A 在这里改 app.py
  ├── worktree-task-2/  ← Agent B 在这里改 app.py
  └── worktree-task-3/  ← Agent C 在这里改 routes.py
```

每个 worktree 是同一个 Git 仓库的**独立副本**，互不干扰。做完后通过 Git merge 合并。

### 锁 vs Worktree 对比

| | 文件锁 | Git Worktree |
|--|-------|-------------|
| **隔离粒度** | 单个文件 | 整个工作目录 |
| **并行度** | 编辑同一文件时阻塞 | 完全并行，最后合并 |
| **复杂度** | 简单 | 需要 Git 支持 |
| **冲突处理** | 先到先得 | 合并时解决冲突 |
| **适用场景** | 单 Agent + 子 Agent | 多 Agent 并行 |

> **设计演进**：s04（同步子 Agent）不需要隔离 → s07-s08（多 Agent 并行）文件锁够用 → s12（生产级）用 worktree 彻底隔离。

### s04 深入完成，进入 s05

**问（User）**：anything remaining needs diving into? if not, next one

**答（Assistant）**：s04 核心已经覆盖完了。进 s05 — Skills 动态加载。
