# Session 3：Plan 机制 — TodoWrite 引入

> 来源 Part: part_a_restructured.md  
> 前置：[Session 2：多工具调度 — 循环不变，工具扩展](session_02.md)  
> 后续：[Session 4：Subagent — 隔离上下文](session_04.md)

---

## 引入动机

前两个 Session 的 Agent 虽然能调用工具、能循环，但有一个致命问题：**它没有计划**。LLM 每一步都是"看一步走一步"，没有全局规划。

> *"An agent without a plan drifts."*（没有计划的 Agent 会漂移）

这对应 Claude Code 里的 **TodoWrite 机制** — 在做事之前，先列出步骤清单，然后逐一执行并标记完成。

## 实现方式

Session 3 加入了一个新工具 `TodoWrite`，它不操作文件系统，而是**操作 Agent 自身的任务状态**：

```python
# Agent 现在的 system prompt 里多了一条规则：
# "在开始执行前，先用 TodoWrite 列出步骤清单"

TOOL_HANDLERS = {
    "bash":       run_bash,
    "read_file":  read_file,
    "write_file": write_file,
    "glob":       glob_search,
    "todo_write": todo_manager.write,   # ← 新增：写任务清单
}
```

有了 TodoWrite，Agent 的行为从 **"接到任务 → 直接干"** 变成了 **"接到任务 → 先列计划 → 逐步执行 → 标记完成"**。

关键数据：项目提到加了 TodoWrite 后，**任务完成率翻倍**。

## 讨论：强制 Plan 还是引导 Plan？

**问（Assistant）**：TodoWrite 是作为一个"工具"实现的，而不是作为循环逻辑的一部分硬编码进去。也就是说，"要不要列计划"这个决定，仍然是 LLM 自己做的（通过 system prompt 引导）。这样做的好处和风险分别是什么？

> 想想：如果你是架构师，你会选择"强制每次循环先 Plan"还是"靠 prompt 引导 LLM 自己决定要不要 Plan"？

**答（User）**：好处是，不用所有问题都列 TODO，风险是可能一直不列 TODO，（选择）前者（强制 Plan）。

**评估（Assistant）**：✅ 分析得很准确！而且选择的"强制 Plan"和 Claude Code 的实际做法非常接近。展开：

- **好处**：灵活性 — 简单问题（"今天几号？"）不需要列 TODO，省开销
- **风险**：LLM 可能"偷懒"跳过 Plan，导致复杂任务漂移
- **User 的选择（强制 Plan）**：正是 Claude Code 的策略 — 它在 system prompt 里用了 **nag reminder**（反复提醒），如果 Agent 没创建 TODO 就开始执行，prompt 会不断催促它补上

实际上 `learn-claude-code` 用了一个折中策略：**不是硬编码强制，而是用 nag reminder 持续施压**。这比硬编码更柔性，比纯靠自觉更可靠。

> 这就是 **Steering（转向）机制** 的雏形 — 不锁死行为，但用持续的 prompt 压力引导行为。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | TodoWrite = Agent 的自我规划工具 |
| **Tension** | 灵活性 vs 可靠性：靠 prompt 引导还是硬编码强制 |
| **Contrast** | 没 Plan 的 Agent 漫游，有 Plan 的 Agent 完成率翻倍 |
| **Use** | 理解 system prompt 作为"软性控制面"的设计哲学 |

---

## 深入学习背景（来自 Part C）

> 来源 Part: part_c_restructured.md

Part A–B 完成了从 s01 到 s12 的概念层学习（what + why），建立了完整的架构认知链路。Part C 从概念层下沉到**工程实现层（how）**，以 Session 3 为核心展开代码级深入分析。

> **本质区别**：概念层拿到了"地图"，深入代码拿到的是"施工图纸" — 你想自己造 Agent 时才需要后者。

---

## 延伸讨论：各 Session 深入的投入产出比分析

### 引入动机

12 个 Session 的概念层已经学完，接下来需要判断哪些值得深入代码。不是每个 Session 都值得逐行读代码 — 需要评估投入产出比。

### 讨论

**问（User）**：深入 session 能学习到什么？

**答（Assistant）**：刚才我们学的是**每个 Session 的设计思想**（what + why），深入代码能学到的是**工程实现细节**（how）。

各 Session 的深入价值评估：

| Session | 深入能额外学到什么 | 值得？ |
|---------|-------------------|--------|
| **s01** | Agent Loop 的完整错误处理、API 调用参数 | ⭐ 低 — 你已经理解了核心 |
| **s02** | 工具定义的 JSON Schema 怎么写、参数验证 | ⭐⭐ 中 — 如果你要自己写工具有用 |
| **s03** | nag reminder 的 prompt 工程、TodoManager 的状态机 | ⭐⭐⭐ **高** — prompt 引导行为的实战技巧 |
| **s04** | Subagent 的结果如何回传给主 Agent、错误传播 | ⭐⭐ 中 |
| **s05** | SKILL.md 的格式设计、Skill 发现机制 | ⭐⭐ 中 — 如果你要设计 Skill 系统 |
| **s06** | **压缩 prompt 怎么写、什么信息让 LLM 保留/丢弃** | ⭐⭐⭐ **高** — 直接可复用的技巧 |
| **s07** | 依赖图的拓扑排序、任务状态转换 | ⭐⭐ 中 |
| **s08** | Python 线程管理、通知注入时机的微妙 bug | ⭐⭐ 中 |
| **s09-s10** | **JSONL 邮箱协议的完整格式、协商 FSM 的状态图** | ⭐⭐⭐ **高** — 多 Agent 通信的可复用模式 |
| **s11** | 竞态处理的具体实现、空闲检测逻辑 | ⭐⭐ 中 |
| **s12** | git worktree 操作、任务-分支绑定逻辑 | ⭐⭐ 中 |
| **s_full** | **所有机制如何组合在一起、集成时的冲突和取舍** | ⭐⭐⭐⭐ **最高** |

**评估（Assistant）**：如果时间有限，**三个最值得深入的**：

1. **s_full.py**（集大成） — 看 12 个机制怎么在一个文件里协同工作，理解**集成的复杂度**
2. **s03**（Prompt 工程） — 看 nag reminder 具体怎么写，这是你立刻能复用到自己项目里的技巧
3. **s06**（压缩策略） — 看压缩 prompt 的措辞和保留/丢弃的决策逻辑

**答（User）**：进入 s03。

---

## Session 3 深入：三层控制机制的工程实现

### 引入动机

Part A 学到 Session 3 引入了 TodoWrite 机制（Plan 模式），以及 nag reminder 这个 prompt 引导手段。现在深入代码，看三个最有实战价值的设计点。

---

### 深入点 1：TodoManager 的硬约束设计

#### 核心概念

```python
if status == "in_progress":
    in_progress_count += 1
# ...
if in_progress_count > 1:
    raise ValueError("Only one task can be in_progress at a time")
```

**这不是普通的数据验证，而是对 LLM 行为的硬约束。** 它强制 Agent 一次只能做一件事，即使 LLM 试图把多个任务标记为 `in_progress`，代码层面直接拦截。

**评估（Assistant）**：这是项目中少见的**用代码强制控制 LLM 行为**的案例。回忆之前总结的设计哲学："代码提供能力，LLM 做决策" — 但这里代码说了"不行"。

**为什么这个约束是硬编码的，而不是像 TodoWrite 本身那样靠 prompt 引导？** 因为**多任务并行在单 Agent 场景下几乎必然失败** — LLM 的注意力只有一个焦点，同时推进两个任务会导致两个都做烂。这是一条不能容忍违反的规则，所以必须硬编码。

> **设计原则**：可以容忍偶尔违反的规则 → prompt 引导；绝不能违反的规则 → 代码强制。

---

### 深入点 2：Nag Reminder 机制

#### 核心概念

```python
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

三个精妙的设计选择：

**① 为什么是 3 轮而不是每轮？**
— 如果每轮都提醒，信噪比太低，LLM 会"免疫"。3 轮是给 LLM 足够的自主空间，但不至于漂移太远。

**② 为什么用 `<reminder>` 标签包裹？**
— 这是一种给 LLM 的**语义信号**。用 XML 风格标签让 LLM 理解这是系统层面的提醒，而不是用户的新输入。Claude 对这种标签格式特别敏感。

**③ 为什么用 `insert(0, ...)` 而不是 `append`？**
— 插入到 results 列表的**最前面**，确保 LLM 先看到提醒，再看到工具结果。如果放在最后，LLM 可能先处理工具结果就忘了提醒。

---

### 深入点 3：System Prompt 的极简设计

#### 核心概念

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use the todo tool to plan multi-step tasks. Mark in_progress before starting, completed when done.
Prefer tools over prose."""
```

只有三句话！每一句都有明确目的：

| 句子 | 目的 |
|------|------|
| `You are a coding agent at {WORKDIR}` | 身份 + 工作目录锚定 |
| `Use the todo tool to plan...` | **核心行为引导** — 告诉它有 todo 工具并描述使用流程 |
| `Prefer tools over prose` | 防止 LLM 只说话不行动（这是 Agent 的常见问题） |

注意没有任何"你是一个非常强大的 AI..."之类的废话。**每个词都在做功**。

---

### 三层控制机制总结

- **硬约束**（代码层 — `in_progress_count > 1` 报错）
- **软引导**（system prompt — "Use the todo tool"）
- **持续提醒**（nag reminder — `<reminder>` 注入）

---

## 延伸讨论：三层控制机制的深度辨析

### 引入动机

User 对三层控制机制提出了三个尖锐问题，每个都触及设计核心。

### 讨论

**问（User）**："你是一个非常强大的 AI..."的作用到底有多少？硬约束应当包含超出能力边界，出现就会对效果产生影响的事情。软引导应该是短时间发生问题不大，但是最好要执行的，能够产生正面影响的事情，但是怎么理解"如果每轮都提醒，信噪比太低，LLM 会'免疫'。3 轮是给 LLM 足够的自主空间，但不至于漂移太远。"有理论依据吗？软引导和持续提醒是并列关系吗？

---

#### 问题 1："你是一个非常强大的 AI..."到底有没有用？

**答（Assistant）**：**有用，但效果很小，且在缩小。**

早期的研究（2023 年左右）发现，给 LLM 加上"你是世界顶级专家"这类角色设定，在某些 benchmark 上有 1-5% 的提升。但这个效果有几个限制：

- **模型越强，效果越小** — GPT-4、Claude 3.5 级别的模型，这类 prompt 的边际收益已经接近零
- **机制是"采样偏移"** — 这类 prompt 把输出分布偏向"高质量回答"区域，但不会让模型"变聪明"
- **占用宝贵的上下文** — 每个废话 token 都是资源浪费

所以 `learn-claude-code` 项目的选择是**正确的工程判断**：不浪费 token 在虚假的"buff"上，每个词都用来传递**具体的行为指令**。

> **实战原则**：与其说"你是最强的编程专家"，不如说"Prefer tools over prose" — 后者直接指导行为，前者只是心理安慰。

---

#### 问题 2：3 轮提醒的"免疫"说法有理论依据吗？

**答（Assistant）**：**诚实回答：没有严格的理论依据。** 这是我表述不够准确的地方，让我修正。

**有依据的部分：**
- LLM 对 **system prompt 中重复出现的指令** 确实存在"衰减效应" — 在长上下文中，越靠近当前位置的信息权重越大（这和注意力机制的实现有关）
- 如果**每轮**都注入相同的 `<reminder>`，它会变成 messages 中的"常驻噪音"，LLM 可能学会忽略它（类似人类对反复弹窗的适应）

**没有严格依据的部分：**
- "3 轮"这个数字**是经验值，不是理论推导**。项目作者很可能是试了几个值（2、3、5），发现 3 在效果和开销之间达到了好的平衡
- "免疫"不是一个精确的技术术语，更准确的说法是：**高频重复的提醒信息在长上下文中的边际影响力递减**

> **修正后的判断**：3 轮是工程实践中的经验值，合理但不唯一正确。真正重要的设计原则是——**提醒不应该成为常驻信息，而应该在需要时出现**。

---

#### 问题 3：软引导和持续提醒是并列关系吗？

**答（Assistant）**：**不是并列，是递进关系。** 之前的表述确实模糊了。重新梳理：

```
三个层次是一个从弱到强的连续谱：

弱 ─────────────────────────────────────────── 强
│                                                │
system prompt 引导     动态 reminder 注入       代码硬约束
(写一次，被动影响)    (条件触发，主动干预)     (直接拒绝执行)
│                     │                         │
"Use todo tool"       "<reminder>Update"        raise ValueError
│                     │                         │
LLM 可以选择忽略      LLM 可以忽略但有压力      LLM 无法绕过
```

关系是：
- **System prompt 引导** = 一次性声明期望行为（"你应该用 todo"）
- **Nag reminder** = 当 system prompt 失效时的**升级手段**（"你已经 3 轮没更新 todo 了，该更新了"）
- **硬约束** = 某些规则不允许违反时的**最终保障**

所以 nag reminder 不是和 system prompt 并列的，而是 **system prompt 的加强版** — 当静态引导不够用时，动态注入更强的提醒。

用一个类比：

> **System prompt** = 公司规章制度（员工应该自觉遵守）
> **Nag reminder** = 经理隔几天提醒一下（"记得更新周报"）
> **硬约束** = 系统不允许提交空白周报（技术层面阻止）

---

## 延伸讨论：Hook 模式 — 用代码实现"修改前必须先读"

### 引入动机

上一节讨论了三层控制机制后，提出了一个应用层面的问题：**如果你要为自己的 Agent 设计一个新规则（比如"每次修改文件前必须先读取文件内容"），你会把它放在哪个层？**

### 讨论

**问（Assistant）**：如果你要为自己的 Agent 设计一个新规则（比如"每次修改文件前必须先读取文件内容"），你会把它放在哪个层？为什么？

**答（User）**：（因输入法原因用英语回答）i ll prefer make a hook, before read, how to enforce using script

**评估（Assistant）**：很好的想法！你选择了 hook（钩子）— 在工具执行前拦截检查。这其实是 Claude Code 逆向分析中发现的 **PreToolUse hook** 的雏形。

你提出的方案比 system prompt 引导更可靠，比直接 raise 报错更优雅。Hook 的思路是：**不阻止 LLM 的决策，但在执行前检查是否满足前置条件**。

### 核心概念 / 实现方式

```python
# Hook 实现：修改文件前必须先读取过该文件

# 记录已读取过的文件集合
files_read = set()

# 原始的 handler
original_read = run_read
original_write = run_write
original_edit = run_edit

def hooked_read(path, limit=None):
    """读取文件时，记录到 files_read"""
    files_read.add(str(Path(path).resolve()))
    return original_read(path, limit)

def hooked_write(path, content):
    """写入前检查：是否读取过这个文件（新文件除外）"""
    resolved = str(safe_path(path).resolve())
    if Path(resolved).exists() and resolved not in files_read:
        # 不报错，而是自动先读取，然后告知 LLM
        current = Path(resolved).read_text()
        files_read.add(resolved)
        result = original_write(path, content)
        return f"⚠️ Auto-read triggered (file existed but wasn't read first):\n{current[:2000]}\n\n{result}"
    return original_write(path, content)

def hooked_edit(path, old_text, new_text):
    """编辑前检查：是否读取过这个文件"""
    resolved = str(safe_path(path).resolve())
    if resolved not in files_read:
        current = original_read(path)
        files_read.add(resolved)
        result = original_edit(path, old_text, new_text)
        return f"⚠️ Auto-read triggered:\n{current[:2000]}\n\n{result}"
    return original_edit(path, old_text, new_text)

# 注册 hook 后的 handler（循环代码完全不变）
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: hooked_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: hooked_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: hooked_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "todo":       lambda **kw: TODO.update(kw["items"]),
}
```

### 设计选择的关键点

注意没有用 `raise ValueError` 直接拒绝，而是选择了**自动补救 + 警告**：

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| **硬拒绝** | `raise ValueError("Must read first")` | LLM 拿到错误信息，可能重试也可能困惑 |
| **自动补救** | 自动读取 + 返回警告 + 继续执行 | ✅ 不中断流程，LLM 下次会学会先读 |
| **静默补救** | 自动读取但不告知 LLM | ❌ LLM 不知道发生了什么，无法学习 |

**自动补救 + 警告**是最优解 — 既保证了安全（文件一定被读过），又给 LLM 反馈让它改进行为。

### 与 Claude Code 真实设计的对比

逆向分析发现 Claude Code 的 Edit 工具有一个类似的硬约束：

> **Edit 工具强制读取机制** — 如果尝试编辑一个未读取的文件，系统会自动触发读取并验证内容一致性。

User 独立想到的 hook 方案，和 Claude Code 的实际实现思路是一致的。

### Guided Prompt：Hook 的泛化

**问（Assistant）**：这个 hook 模式其实可以泛化。想象一下：

**如果你要给 Agent 加更多的 hook（比如"执行 bash 前检查是否危险命令"、"写文件后自动更新 todo 状态"），你会怎么组织这些 hook？是每个工具都写 if-else，还是有更优雅的架构？**

> 提示：想想 Web 框架里的 middleware（中间件）模式。

（User 没有回答这个问题，而是转向追问代码语法细节。）

---

## 延伸讨论：Python 语法深入 — lambda 与 **kwargs

### 引入动机

在阅读 Hook 实现代码时，User 对 `lambda **kw: hooked_read(kw["path"], kw.get("limit"))` 这行语法提出了疑问。这涉及 Python 的两个基础语法点。

### 讨论

**问（User）**：`lambda **kw: hooked_read(kw["path"], kw.get("limit"))`，wat does it mean, lambda, kw?

**答（Assistant）**：

#### `lambda` — 匿名函数

`lambda` 就是一个**没有名字的简短函数**。它和 `def` 做同一件事，只是更紧凑：

```python
# 用 def 写：
def add(a, b):
    return a + b

# 用 lambda 写（完全等价）：
add = lambda a, b: a + b

# 两种写法调用方式一样：
add(1, 2)  # → 3
```

**规则**：`lambda 参数: 返回值` — 冒号左边是参数，右边是返回的表达式。

#### `**kw` — 接收所有关键字参数

`**kw` 的意思是：**把调用时传入的所有 `key=value` 参数打包成一个字典**。

```python
def show(**kw):
    print(kw)

show(name="Alice", age=30)
# 输出: {"name": "Alice", "age": 30}

show(path="main.py", limit=100)
# 输出: {"path": "main.py", "limit": 100}
```

`kw` 只是变量名（keyword 的缩写），你也可以叫 `**args`、`**params`，任意名字都行。关键是 `**` 这两个星号。

#### 合在一起看

```python
lambda **kw: hooked_read(kw["path"], kw.get("limit"))
```

拆解：

```python
# 等价于：
def anonymous_function(**kw):
    return hooked_read(kw["path"], kw.get("limit"))
```

| 部分 | 含义 |
|------|------|
| `**kw` | 接收所有参数，打包成字典 |
| `kw["path"]` | 从字典里取 path（必须存在，否则报错） |
| `kw.get("limit")` | 从字典里取 `limit`（不存在则返回 `None`，不报错） |

#### 为什么项目要这么写？

因为 Agent 循环里的工具调用是这样的：

```python
output = TOOL_HANDLERS[block.name](**block.input)
#                                  ↑
#        block.input 是一个字典，比如 {"path": "main.py", "limit": 100}
#        **block.input 会展开成 path="main.py", limit=100
#        然后 lambda **kw 再把它接回字典
```

**本质上就是在做参数转发** — Agent 循环不关心每个工具具体需要什么参数，统一用 `**` 展开和收集，让每个 handler 自己去取需要的参数。

---

## 延伸讨论：`read_file` 的 `limit` 参数

### 引入动机

User 继续追问代码细节，想理解 `limit` 参数的作用。

### 讨论

**问（User）**：what is limit?

**答（Assistant）**：`limit` 是 `read_file` 工具的一个**可选参数**，用来限制读取的行数。

```python
def run_read(path: str, limit: int = None) -> str:
    lines = safe_path(path).read_text().splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit] + [f"... ({len(lines) - limit} more)"]
    return "\n".join(lines)[:50000]
```

| 调用方式 | 行为 |
|---------|------|
| `read_file(path="main.py")` | 读取**全部内容**（limit 默认 None） |
| `read_file(path="main.py", limit=20)` | 只返回**前 20 行**，末尾附 "... (剩余 N 行)" |

**为什么需要 limit？** 如果一个文件有 5000 行，全部塞进 messages 会浪费大量上下文窗口。LLM 可以先读前 20 行了解文件结构，再决定是否需要看更多。

这也是为什么用 `kw.get("limit")` 而不是 `kw["limit"]` — `.get()` 在没传这个参数时返回 `None`，不报错。因为 limit 是可选的。

---

## 延伸讨论：LLM 如何写文件 — 格式、覆盖与编辑

### 引入动机

User 在理解了 hook 和语法之后，提出了一个非常实际的问题链：LLM 怎么写出格式正确的 md 文件？write 是覆盖还是追加？edit 又是怎么工作的？

### 讨论

**问（User）**：got it. means get right word controlled lines. and with hook: 1. read, filed. 2. write, read first, write. 3. edit, read first, edit. But, how llm could write docs/md? send words with change line character? in md, they show well edited. but in other ways, just word+character? and write is along with text before? how edit works?

---

#### 问题 1：LLM 怎么写出格式正确的 md 文件？

**答（Assistant）**：**LLM 就是发送带换行符 `\n` 的纯文本字符串。**

```python
# LLM 实际发出的 tool_use 请求长这样：
{
  "name": "write_file",
  "input": {
    "path": "README.md",
    "content": "# My Project\n\nThis is a description.\n\n## Features\n\n- Feature A\n- Feature B\n"
  }
}
```

`\n` 就是换行。写入文件后，文件内容变成：

```markdown
# My Project

This is a description.

## Features

- Feature A
- Feature B
```

**对 LLM 来说，md 文件就是一个字符串。** LLM 天然知道 Markdown 语法，所以它会在正确的地方放 `#`、`-`、`\n`。没有任何特殊魔法。

---

#### 问题 2：write_file 是覆盖还是追加？

**答（Assistant）**：**完全覆盖。**

```python
def run_write(path: str, content: str) -> str:
    fp = safe_path(path)
    fp.write_text(content)  # ← 这一行会把文件原有内容全部替换
    return f"Wrote {len(content)} bytes"
```

（注：原文在此处被截断，但从代码可知 write_file 使用 `write_text(content)` 执行完全覆盖写入。）

---

## 贯穿 Part C 的核心模式总结

### 从概念到实战的学习路径

Part C 的学习模式与 Part A–B 不同：从"每个 Session 学一个概念"变为"**选择性深入一个 Session 的代码**"。这体现了学习的自然进阶：

```
Part A–B：地图（12 个 Session 的设计思想）
    ↓
Part C：施工图纸（选择性深入代码实现）
```

### 三层控制机制（本部分核心知识）

```
弱 ─────────────────────────────────────────── 强
│                                                │
System Prompt 引导     动态 Reminder 注入       代码硬约束
(写一次，被动影响)    (条件触发，主动干预)     (直接拒绝执行)
│                     │                         │
递进关系：静态引导不够 → 动态提醒升级 → 代码兜底
```

决策规则：
- **可以容忍偶尔违反** → System Prompt 引导
- **需要但偶尔忘记** → Nag Reminder 动态注入
- **绝不能违反** → 代码硬约束（`raise ValueError` 或 Hook 自动补救）

### Prompt 工程实战原则

| 原则 | 说明 |
|------|------|
| **每个词都在做功** | System Prompt 不放废话，不放"你是最强的AI" |
| **提醒不应成为常驻信息** | 3 轮经验值，条件触发而非每轮注入 |
| **XML 标签作语义信号** | `<reminder>` 让 LLM 区分系统提醒和用户输入 |
| **位置决定优先级** | `insert(0, ...)` 确保 LLM 先看到提醒 |

### Hook 模式的设计智慧

| 设计选择 | 原因 |
|---------|------|
| **自动补救 + 警告**（而非硬拒绝） | 不中断流程，同时给 LLM 反馈以改进行为 |
| **`**kw` 统一参数转发** | Agent 循环不关心工具具体参数，实现解耦 |
| **`.get()` vs `[]`** | 可选参数用 `.get()` 防 KeyError，必需参数用 `[]` |

---

## 延伸讨论：write_file vs edit_file 的工作原理（来自 Part D）

> 来源 Part: part_d_restructured.md

### 引入动机

在 Part C 中已经讨论了 Hook 模式（"修改前必须先读"），但还没看底层的 `write_file` 和 `edit_file` 具体怎么工作。这一节补上这个细节，解释了为什么 Hook 的存在是技术上必要的。

### 核心概念

#### write_file — 全量覆盖

| 情况 | 行为 |
|------|------|
| 文件不存在 | 创建新文件，写入 content |
| 文件已存在 | **整个文件内容被 content 替换** |

> 所以如果你有一个 100 行的文件，LLM 调用 `write_file` 只写了 5 行 → 原来的 100 行全没了，只剩这 5 行。**这就是为什么 "修改前必须先读" 很重要** — 不读就不知道原来有什么。

#### edit_file — 精确替换

```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    content = fp.read_text()
    if old_text not in content:
        return f"Error: Text not found in {path}"
    fp.write_text(content.replace(old_text, new_text, 1))  # 只替换第一次出现
```

举例：

```python
# 文件原始内容：
# "Hello World\nGoodbye World\n"

edit_file(
    path="test.txt",
    old_text="Hello World",    # ← 要替换的精确文本
    new_text="Hi Universe"     # ← 替换成什么
)

# 文件变成：
# "Hi Universe\nGoodbye World\n"
```

#### 两者对比

| write_file | edit_file |
|-----------|-----------|
| 整个文件被替换 | 只替换匹配的那段文本 |
| 适合写新文件 | 适合改现有文件的一小部分 |
| 不需要知道原内容 | **必须精确知道 old_text 是什么**（所以必须先 read） |

> **这就回到了 Hook 设计** — `edit_file` 如果不先 `read_file`，LLM 根本不知道 `old_text` 应该填什么，很可能填错导致 "Text not found" 报错。

### s03 完整收获总结

| 层次 | 学到的 |
|------|---------| 
| **表面** | TodoManager 的 CRUD 实现 |
| **中层** | 三层控制机制 + Hook 模式 |
| **深层** | System prompt 稀释效应 → nag reminder 是**注意力衰减的对策** |

---

## 延伸讨论：System Prompt 稀释效应（来自 Part D）

> 来源 Part: part_d_restructured.md

### 引入动机

s03 的配套文档中有一个之前没展开的**关键洞察**：

> *"对话越长越严重: 工具结果不断填满上下文, 系统提示的影响力逐渐被稀释。一个 10 步重构可能做完 1-3 步就开始即兴发挥, 因为 4-10 步已经被挤出注意力了。"*

### 核心概念

这解释了**为什么 TodoWrite 的存在是必要的**，比"更好地组织工作"更深一层：

```
对话开始时：
  [system prompt] ████████████████  ← 占比高，影响力强
  [messages]      ██

对话 50 轮后：
  [system prompt] ██                ← 占比极低，被淹没
  [messages]      ████████████████████████████████████████
```

System prompt 里说的"Use the todo tool" — **随着对话越来越长，这条指令的影响力在衰减**。LLM 的注意力被大量的工具结果吸走，逐渐"忘记"自己应该更新 todo。

**这就是 nag reminder 存在的真正原因** — 不是因为 LLM "偷懒"，而是因为 system prompt 在长上下文中**物理性地被稀释了**。Nag reminder 通过在**最新位置**注入提醒，对抗这种衰减。

> **这是 s03 最深层的设计洞察**：TodoWrite + nag reminder 本质上是在解决 **LLM 注意力随上下文长度衰减** 的问题，而不只是"提醒 LLM 写 todo"。

### 讨论

**问（User）**：so may llm know todo in first two round, to avoid forget in third, remind it.

**评估（Assistant）**：没错！前几轮 system prompt 影响力还强，LLM 自然会用 todo。3 轮后开始衰减，reminder 就接上。
