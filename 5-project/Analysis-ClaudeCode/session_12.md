# Session 12（最终章）：Worktree 隔离 — 文件系统级隔离

> 来源 Part: part_b_restructured.md, part_c_restructured.md, part_g_restructured.md  
> 前置：[Session 11：去中心化调度 — 自主认领任务](session_11.md)  
> 后续：无（最终章）

---

## 引入动机

s11 解决了"谁做哪个任务"，但**所有 Agent 还在同一个目录里工作**。如果 Alice 在改 `main.py` 的第 10 行，Bob 也在改 `main.py` 的第 20 行 → 文件冲突。Session 12 的 motto：

> *"Each works in its own directory, no interference."*

## 核心概念 / 实现方式

解决方案借用了 Git 的 **worktree** 概念：

```python
# 每个 Agent 获得独立的工作目录
def create_worktree(task_id):
    os.system(f"git worktree add ./worktrees/{task_id} -b task_{task_id}")
    return f"./worktrees/{task_id}"

# Alice 在 ./worktrees/task_001/ 里干活
# Bob 在 ./worktrees/task_002/ 里干活
# 互不干扰，最后通过 git merge 合并
```

**任务管理（s07 的 tasks.json）绑定 worktree**：每个任务有自己的隔离目录，Agent 认领任务后自动切换到对应目录。

## 讨论

**问（Assistant）**：回顾整个 12 Session 的旅程 — Session 4 里我们讲到 Subagent 用"独立 messages[]"实现上下文隔离。Session 12 用"独立 worktree"实现文件系统隔离。这两种隔离解决的是同一个问题吗？还是不同层面的问题？

**答（User）**：原子操作是什么意思？一个是上下文隔离，一个是工作区隔离，前者是为了保持上下文干净，后者是为了方便审校回退。

**评估（Assistant）**：✅ 通过。对两种隔离的区分非常准确！

### 原子操作的解释

**原子（Atom）= 不可分割的。** 原子操作就是**要么完整执行，要么完全不执行，中间不会被打断**的操作。

```
❌ 非原子操作（两步，可能被打断）：
   1. 读取文件内容            ← Alice 做完这步
   2. 写入新内容              ← Bob 可能在这之间插入，覆盖了

✅ 原子操作（一步到位，不可打断）：
   os.rename("task_pending", "task_claimed_by_alice")
   # 文件重命名在操作系统层面是原子的
   # 要么成功改名，要么失败，不存在"改了一半"的状态
```

回到抢任务的场景：如果"认领任务"是把文件从 `pending/task_001` 移动到 `claimed/task_001`，那用 `os.rename()` 只有一个 Agent 能成功，另一个会得到"文件不存在"的错误 → 自然知道被别人抢了。

**简单记**：原子操作 = 不可能被别人插队的操作。

### 两种隔离的对比

| | s04 上下文隔离 | s12 工作区隔离 |
|---|---|---|
| **隔离什么** | messages[]（LLM 的思维空间） | 文件系统（代码工作目录） |
| **目的** | 保持上下文干净，子任务不污染主对话 | 防止文件冲突，方便审校回退 |
| **层面** | **认知层** — LLM 看到什么 | **物理层** — 代码在哪里 |

User 说的"方便审校回退"特别好 — 这正是用 git worktree 而不是简单复制目录的原因：**每个任务是一个 git branch，可以 review、merge、revert**。

## Memory Anchor

（Session 12 原文无独立 Memory Anchor 表格，此处综合讨论内容补充）

| 维度 | 内容 |
|------|------|
| **Essence** | git worktree = 每个任务独立目录 + 独立分支 |
| **Tension** | 共享目录简单但冲突；隔离目录安全但需要合并 |
| **Contrast** | s04 隔离认知层（messages[]）；s12 隔离物理层（文件系统） |
| **Use** | 完整的多 Agent 隔离需要两层都做：认知隔离 + 物理隔离 |

---

## 延伸深入：控制面 vs 执行面架构（来自 Part G）

> Part G 对 s12 的架构做了更深入的解读，补充了控制面/执行面分离、EventBus、以及 s12 与 s09-s11 的结构性对比。

s12 是最大的文件（781 行）。核心概念是**把 s07 的 Task 和 Git Worktree 绑定在一起**。

回顾之前 User 的问题 — 多 Agent 编辑同一文件怎么办？s12 就是那个答案。

### 核心架构：控制面 vs 执行面

```
控制面 (Control Plane)          执行面 (Execution Plane)
.tasks/task_12.json             .worktrees/auth-refactor/
  status: "in_progress"            ← 一个独立的代码目录副本
  worktree: "auth-refactor"        ← 通过名字绑定
```

**Task 说"做什么"，Worktree 说"在哪做"。**

### Worktree 详解

Git worktree 就是**同一个 Git 仓库的另一个工作目录**。不需要 clone，不需要拷贝：

```
主目录:     /project/          ← main 分支
worktree:   /project/.worktrees/auth-refactor/  ← wt/auth-refactor 分支
```

两个目录是**完全隔离的文件系统**，但共享 git 历史。代码第 298 行：

```python
self._run_git(["worktree", "add", "-b", branch, str(path), base_ref])
```

### 完整生命周期

```
1. task_create("Implement auth")         → task_1.json (pending)
2. worktree_create("auth", task_id=1)    → 创建目录 + 绑定 task
3. worktree_run("auth", "pytest")        → 在 worktree 目录里跑命令
4. worktree_remove("auth", complete_task=True)
   → 删除 worktree + task 标记 completed
```

**`worktree_run`（第 367-391 行）是关键** — 它在 worktree 的目录里执行命令（`cwd=path`），不是在主目录。这就实现了 User 之前提到的**文件级隔离**。

### EventBus — 操作日志（第 82-117 行）

s12 新增了 `events.jsonl` — 每次创建/删除 worktree 都记录一条事件：

```json
{"event": "worktree.create.before", "ts": 1710000001, "task": {"id": 1}, "worktree": {"name": "auth"}}
{"event": "worktree.create.after", "ts": 1710000002, ...}
{"event": "worktree.remove.before", ...}
{"event": "task.completed", ...}
```

这是**可观测性（Observability）** — 事后可以查看完整的生命周期轨迹。`worktree_events` 工具让 LLM 能看到这些日志。

### s12 和 s09-s11 的区别

s12 **没有 Teammate 线程**。它回到了单 Agent 模式，但加了 worktree 工具。也就是说：

```
s09-s11:  多个 Agent 线程 → 通过 inbox 通信 → 共享文件系统（有冲突风险）
s12:      单个 Agent      → 管理多个 worktree → 每个任务有独立目录（无冲突）
```

**这两种模式可以组合** — 多个 Teammate，每个在自己的 worktree 里工作，完全隔离。

---

## 12 Session 全景回顾

```
s01  while + stop_reason           → 最小循环
s02  dispatch map                  → 开放式工具注册
s03  TodoWrite                     → 自我规划
s04  Subagent                      → 上下文隔离
s05  Skills                        → 按需加载知识
s06  Context压缩 + CLAUDE.md       → 双层记忆
s07  tasks.json + 依赖图            → 持久化任务管理
s08  后台线程 + 通知队列             → 异步执行
s09  JSONL 邮箱                    → 多 Agent 通信
s10  request-response + shutdown   → 协商协议
s11  空闲扫描 + 自主认领             → 去中心化调度
s12  git worktree                  → 文件系统隔离
```

---

## 贯穿 Part B（Session 7–12）的核心模式总结

### 架构演进：从单 Agent 到多 Agent 协作

Part A（s01-s06）构建了**单 Agent 的完整能力栈**。Part B 的前半段（s07-s08）解决了单 Agent 的**持久化和异步**问题，后半段（s09-s12）实现了**多 Agent 协作**的四个层次：

| 层次 | Session | 解决的问题 |
|------|---------|-----------| 
| 通信 | s09 | Agent 之间怎么传递消息 |
| 协商 | s10 | 怎么请求、接受、拒绝、确认完成 |
| 调度 | s11 | 谁做哪个任务（去中心化自主认领） |
| 隔离 | s12 | 各自的工作空间互不干扰 |

### 反复出现的核心张力

| 张力 | 体现 |
|------|------|
| **中心化 vs 去中心化** | s09 主 Agent 指派 → s11 自主认领，去掉瓶颈但引入并发问题 |
| **同步 vs 异步** | s08 后台线程 → s09 邮箱通信，异步高效但需要处理时序问题 |
| **共享 vs 隔离** | s04 认知隔离 + s12 物理隔离，两层隔离解决不同层面的污染问题 |
| **传统工程直觉 vs LLM 原生设计** | role: user 的三种用途之争 — 信任 LLM 的语义理解，不加显式标签 |
| **简单 vs 健壮** | 文件锁/原子操作在教学项目中省略，但生产系统必须处理竞态条件 |

### 设计哲学的延续

> **代码提供能力（注册工具），LLM 做决策（选择何时调用哪个工具）。**

这条哲学在 Part B 中进一步深化为：**代码提供基础设施（任务图、邮箱、worktree），LLM 做运行时决策（认领任务、发送消息、选择协作策略）。**

---

## 最终 Memory Anchor（全 12 Session 总结）

> 来源 Part: part_c_restructured.md

```
s04  独立 messages[]               → 上下文隔离
s05  Skill via tool_result         → 按需知识加载
s06  三层压缩 + CLAUDE.md          → 双层记忆
s07  tasks.json + 依赖图           → 持久化任务管理
s08  守护线程 + 通知队列            → 异步执行
s09  JSONL 邮箱                    → 多 Agent 通信
s10  request-response 协议         → 协商与优雅关闭
s11  空闲扫描 + 自主认领            → 去中心化调度
s12  git worktree                  → 文件系统隔离
```

| 维度 | 内容 |
|------|------|
| **Essence** | 一个 while 循环 + 12 层机制 = 完整的类 Claude Code Agent |
| **Tension** | 每一层都是在"简单/脆弱"和"强大/复杂"之间做选择 |
| **Contrast** | 传统软件用硬编码控制流；Agent 系统用 LLM 决策 + 工具注册 + prompt 引导 |
| **Use** | 设计自己的 Agent 时，可以按需选择加哪些层，不需要全部 |

**评估（Assistant）**：你现在对整个项目的架构有了完整的认知链路。接下来的方向有三个：

1. **深入某个 Session** — 打开实际代码逐行阅读
2. **做 checkpoint** — 把今天学的内容沉淀成笔记
3. **对比分析** — 和 ThreeFish-AI 的逆向分析文档做交叉验证

---

## 延伸讨论：上下文估算（来自 Part G）

> User 完成 s12 学习后要求进行 checkpoint。之后追问"当前上下文大概有多少了"。

**问（User）**：当前上下文大概有多少了？

**答（Assistant）**：我没有直接测量 token 数的工具，但可以估算：

**当前 context 大致构成：**

| 来源 | 估计 tokens |
|------|------------|
| System prompt（工具定义、skills 列表等） | ~15,000-20,000 |
| 截断前的对话摘要（CHECKPOINT 3） | ~5,000 |
| 截断后的对话（s11 讨论 → checkpoint） | ~8,000-10,000 |
| 最近几轮的文件 view（s12 全文 781 行） | ~5,000 |
| 写入的 checkpoint 文件内容 | ~6,000 |
| MCP 工具定义 | ~10,000 |

**估计总量: ~50,000-55,000 tokens**

这已经是一个比较大的 context 了。注意到系统已经触发了 **CHECKPOINT 3**（对话前半部分被截断并替换为摘要），说明之前的 context 已经超过阈值被压缩过了。

> 你现在体验的就是 s06 讲的东西——**你的对话正在被 auto_compact** 😄

如果想继续高质量的讨论，可以考虑开一个新 session，把 checkpoint 文件作为起点。

---

## 延伸讨论：从教学代码到真实产品（来自 Part G）

> Part G 中 User 完成全部 12 Session 学习后，将教学代码与真实 Antigravity IDE 进行了对比分析。

| 维度 | 教学代码 (learn-claude-code) | 真实产品 (Antigravity IDE) |
|------|-------|-------|
| 截断阈值 | `TOKEN_LIMIT` 本地常量 | 服务端下发，用户不可调 |
| 压缩实现 | Python 代码可读 | Go 编译二进制 + protobuf |
| 对话存储 | messages[] 在内存中 | SQLite (state.vscdb) + protobuf |
| CHECKPOINT | 教学中的概念 | 真实系统的截断注入机制 |
| 安全机制 | 无/很少 | `tokenThreshold > MaxTokenLimit` 校验 |
