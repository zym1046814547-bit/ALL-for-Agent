# Session 7：文件级任务图 — 从内存到磁盘

> 来源 Part: part_b_restructured.md, part_e_restructured.md  
> 前置：[Session 6：上下文压缩 — 双层记忆架构](session_06.md)  
> 后续：[Session 8：后台任务 — 非阻塞异步执行](session_08.md)

---

## Phase 1-2 完成回顾

在进入 Phase 3-4 之前，对前 6 个 Session 做了一次结构化回顾：

| Phase | Session | 核心机制 |
|-------|---------|---------| 
| Loop | s01 | while + stop_reason = Agent |
| Loop | s02 | dispatch map 注册工具，循环不变 |
| Planning | s03 | TodoWrite 实现自我规划 |
| Planning | s04 | Subagent = 独立 messages[] 实现隔离 |
| Knowledge | s05 | Skill 按需加载，通过 tool_result 注入 |
| Knowledge | s06 | 三层压缩 + CLAUDE.md 双层记忆 |

**一个核心设计哲学贯穿始终**：代码只提供能力，决策权在 LLM。

---

## 引入动机

Session 3 的 TodoWrite 把计划存在 messages 里，但它有两个问题：
1. **重启就没了** — Agent 退出后计划消失
2. **没有依赖关系** — 不知道"任务 B 必须等任务 A 完成才能开始"

Session 7 的 motto：

> *"Break big goals into small tasks, order them, persist to disk."*

## 核心概念 / 实现方式

Session 7 把 TodoWrite 升级为一个**文件级任务系统**：

```python
# tasks.json — 持久化在磁盘上
{
  "task_001": {"desc": "读取需求文件", "status": "done", "deps": []},
  "task_002": {"desc": "设计数据模型", "status": "in_progress", "deps": ["task_001"]},
  "task_003": {"desc": "写单元测试", "status": "pending", "deps": ["task_002"]}
}
```

工具从 1 个 `todo_write` 变成了 CRUD 四件套：`task_create`、`task_read`、`task_update`、`task_list`。增加到 **8 个工具**。

## 讨论

**问（Assistant）**：Session 3 的 TodoWrite 是一个"平面清单"（第1步、第2步、第3步…），Session 7 变成了一个"有依赖关系的图"（B 依赖 A，C 依赖 B）。这个升级为什么是 Session 9（多 Agent 协作）的前置条件？

> 提示：如果多个 Agent 同时工作，没有依赖关系会发生什么？

**答（User）**：没有依赖关系，一是可能后者工作根本无法进行，二是后者可能会新定义一个前置依赖，导致最终结果前后不对应。

**评估（Assistant）**：✅ 通过。把两个关键风险都说到了。用更精确的术语来说：

1. **执行阻塞** — Agent B 需要 Agent A 的输出作为输入，但不知道 A 还没做完，直接开始 → 报错或用空数据
2. **状态冲突** — Agent B 自行决定了前置条件（比如重新定义数据模型），和 Agent A 已经做出的结果矛盾 → 最终集成时崩溃

所以**依赖图不是优化，是多 Agent 协作的硬性前提**。没有它，多个 Agent 就是一群各干各的、互相踩脚的工人。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | 文件级任务图：CRUD + 依赖关系 + 持久化 |
| **Tension** | 平面清单够单 Agent 用，但多 Agent 必须有依赖图 |
| **Contrast** | s03 TodoWrite = 内存中的平面清单；s07 Tasks = 磁盘上的有向图 |
| **Use** | 为 s09-s12 的多 Agent 协作奠定基础 |

---

## 延伸深入：代码级实现细节（来自 Part E）

### 🎯 深入点 1：从 messages 内存到文件系统持久化

**s03 的 TodoManager vs s07 的 TaskManager 最关键的区别：**

```
s03 TodoManager:
  self.items = [...]     ← 存在 Python 内存里
  ↓ s06 压缩时
  💥 丢失！TodoManager 的状态还在，但 LLM 不记得有哪些 todo 了

s07 TaskManager:
  .tasks/task_1.json     ← 存在文件系统里
  .tasks/task_2.json
  ↓ s06 压缩时
  ✅ 安全！LLM 可以调用 task_list 重新读取所有任务状态
```

这就是文档里说的 **"State that survives compression — because it's outside the conversation"**。

### 🎯 深入点 2：双向依赖自动同步（第 88-100 行）

```python
if add_blocks:
    task["blocks"] = list(set(task["blocks"] + add_blocks))
    # Bidirectional: also update the blocked tasks' blockedBy lists
    for blocked_id in add_blocks:
        blocked = self._load(blocked_id)
        if task_id not in blocked["blockedBy"]:
            blocked["blockedBy"].append(task_id)
            self._save(blocked)
```

当你说"Task 1 blocks Task 2"时，代码**自动**做两件事：
1. Task 1 的 `blocks` 列表加入 2
2. Task 2 的 `blockedBy` 列表加入 1

LLM 只需要声明一次关系，代码负责维护双向一致性。这防止了"Task 1 说它 blocks Task 2，但 Task 2 不知道"的不一致。

### 🎯 深入点 3：完成任务时自动解锁依赖（第 104-110 行）

```python
def _clear_dependency(self, completed_id: int):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

当 Task 1 标记为 `completed` 时，代码**自动扫描所有任务**，把 Task 1 从它们的 `blockedBy` 列表中移除。

```
完成前:  Task 2 blockedBy: [1, 3]
Task 1 完成后: Task 2 blockedBy: [3]     ← 自动更新
Task 3 完成后: Task 2 blockedBy: []      ← Task 2 可以开始了！
```

LLM 不需要手动去解锁，**代码自动维护依赖图的一致性**。

### s03 → s07 进化对比

| | s03 TodoManager | s07 TaskManager |
|--|----------------|-----------------|
| **存储** | Python 内存 | JSON 文件 |
| **生存** | 压缩后丢失 | 永远存在 |
| **关系** | 无（平铺列表） | 有依赖图（blockedBy/blocks） |
| **完成后** | 手动标记 | 自动解锁下游任务 |
| **多 Agent** | 不支持（单 Agent 内存） | 支持（文件系统是共享的） |

> **核心进化**：从"LLM 脑子里的笔记"变成了"团队共享的项目管理板"。

---

### 讨论：理解双向同步机制

**问（Assistant）**：注意 `_clear_dependency` 扫描**所有任务文件**来移除依赖。如果有 1000 个任务文件，每完成一个就要读写所有文件。**在多 Agent 并行的场景下，两个 Agent 同时完成各自的任务并同时调用 `_clear_dependency`，会发生什么问题？**

**答（User）**：dont quite understand point 2（关于双向同步）

**答（Assistant）**：让我用一个具体例子解释。

#### 场景：重构项目，3 个任务

```
Task 1: 创建 models.py
Task 2: 创建 routes.py（需要 models.py 先完成）
Task 3: 写测试（需要 Task 1 和 Task 2 都完成）
```

#### 没有双向同步时，LLM 需要调用两次

```python
# LLM 必须分别告诉两边：
task_update(task_id=1, addBlocks=[2])      # 告诉 Task 1："你 blocks Task 2"
task_update(task_id=2, addBlockedBy=[1])    # 告诉 Task 2："你被 Task 1 block"
```

如果 LLM 只做了第一步忘了第二步 → **Task 1 知道它 blocks Task 2，但 Task 2 不知道它被 block** → 不一致。

#### 有双向同步后，LLM 只需要调用一次

```python
# LLM 只说一句：
task_update(task_id=1, addBlocks=[2])

# 代码自动完成：
# 1. Task 1 的 blocks: [2]        ← LLM 说的
# 2. Task 2 的 blockedBy: [1]     ← 代码自动加的！
```

**一句话总结**：LLM 只需要声明"A blocks B"，代码自动在 B 那边记录"我被 A block 了"。减少 LLM 犯错的机会。

和 s03 的硬约束是同样的思路 — **能用代码保证一致性的，不要靠 LLM 记住**。

---

### 讨论：JSON 文件的完整变化过程

**问（User）**：what real task json changes in the 3 steps (any others?)

**答（Assistant）**：完整展示 JSON 文件在每一步的变化：

#### Step 1: 创建三个任务

```python
task_create(subject="创建 models.py")
task_create(subject="创建 routes.py")
task_create(subject="写测试")
```

```
.tasks/task_1.json                .tasks/task_2.json                .tasks/task_3.json
{                                 {                                 {
  "id": 1,                          "id": 2,                          "id": 3,
  "subject": "创建 models.py",      "subject": "创建 routes.py",      "subject": "写测试",
  "status": "pending",              "status": "pending",              "status": "pending",
  "blockedBy": [],                  "blockedBy": [],                  "blockedBy": [],
  "blocks": [],                     "blocks": [],                     "blocks": [],
  "owner": ""                       "owner": ""                       "owner": ""
}                                 }                                 }
```

#### Step 2: 设置依赖关系

```python
task_update(task_id=1, addBlocks=[2, 3])   # Task 1 blocks Task 2 和 Task 3
task_update(task_id=2, addBlocks=[3])       # Task 2 也 blocks Task 3
```

第一条命令后（代码自动双向同步）：

```
task_1.json                       task_2.json                       task_3.json
{                                 {                                 {
  "blocks": [2, 3],  ← LLM设的     "blockedBy": [1],  ← 自动加的     "blockedBy": [1],  ← 自动加的
  ...                               ...                               ...
}                                 }                                 }
```

第二条命令后：

```
task_1.json                       task_2.json                       task_3.json
{                                 {                                 {
  "blocks": [2, 3],                 "blocks": [3],     ← LLM设的      "blockedBy": [1, 2], ← 自动加了2
  ...                               "blockedBy": [1],                  ...
}                                   ...                              }
                                  }
```

#### Step 3: 完成 Task 1

```python
task_update(task_id=1, status="completed")
```

代码自动调用 `_clear_dependency(1)` — 扫描所有文件，移除 blockedBy 里的 1：

```
task_1.json                       task_2.json                       task_3.json
{                                 {                                 {
  "status": "completed", ← 改了    "blockedBy": [],    ← 自动移除1    "blockedBy": [2],   ← 自动移除1
  "blocks": [2, 3],                "blocks": [3],                      ...
  ...                               ...                              }
}                                 }
```

Task 2 的 blockedBy 变成 `[]` → **Task 2 可以开始了！**
Task 3 的 blockedBy 还有 `[2]` → **Task 3 还得等。**

#### Step 4: 完成 Task 2

```python
task_update(task_id=2, status="completed")
```

```
task_1.json                       task_2.json                       task_3.json
{                                 {                                 {
  "status": "completed",            "status": "completed", ← 改了    "blockedBy": [],    ← 自动移除2
  ...                               ...                               ...
}                                 }                                 }
```

Task 3 的 blockedBy 变成 `[]` → **Task 3 终于可以开始了！**

整个流程就是一个**依赖图的自动拓扑解锁**。LLM 只需要声明关系和标记完成，代码负责所有一致性维护。

---

### 讨论：User 总结 blocks/blockedBy 的分工

**问（User）**：got it. Using `blocks` is very easy to set, and obvious, perhaps never wrong. And what works truly is `blockedBy`, blocking task being processed. Finally, done task will delete all the number in other tasks. Anything more? Or next.

**评估（Assistant）**：✅ 完全正确：

- `blocks` → 声明方，容易设，方向明确
- `blockedBy` → 执行方，真正控制"能不能开始"
- 完成后 → 自动清理所有引用
