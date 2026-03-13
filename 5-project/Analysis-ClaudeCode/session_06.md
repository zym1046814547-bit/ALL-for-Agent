# Session 6：上下文压缩 — 双层记忆架构

> 来源 Part: part_a_restructured.md  
> 前置：[Session 5：Skills — 按需加载知识](session_05.md)  
> 后续：Session 7（待处理，来源于后续 part 文件）

---

## 引入动机

前 5 个 Session 一直在往 messages[] 里**加东西** — 对话历史、工具结果、Skill 内容。但上下文窗口有上限（比如 200K tokens）。对话越长，离爆满越近。

> *"Context will fill up; you need a way to make room."*

## 三层压缩策略

```
Layer 1: 保留最近的消息（最新 N 轮）
Layer 2: 中间消息压缩成摘要（LLM 总结要点）
Layer 3: 最早的消息丢弃（只保留 Layer 2 的摘要）
```

对应 Claude Code 逆向分析的发现：**当 token 使用率达到 92% 阈值时，自动触发压缩**。

关键点：压缩也是由 LLM 完成的 — 让 LLM 自己总结"哪些信息必须保留，哪些可以丢掉"。

## 讨论：压缩信息丢失的应对

**问（Assistant）**：压缩意味着信息损失。如果 LLM 在压缩时错误地丢弃了一条关键信息（比如用户说过"不要修改 config.yaml"），后续 Agent 就可能违反用户意图。你觉得有什么办法可以降低"压错"的风险？回忆一下 Session 5 里我们刚学过的机制，有没有可以复用的？

**答（User）**：用户意图可以沉淀为项目记忆？

**评估（Assistant）**：✅ 太棒了！自己把 Session 5 和 Session 6 串起来了。

User 说的"项目记忆"正是 **CLAUDE.md** 的设计意图：

> 关键约束、用户偏好、项目规则 → 写入 CLAUDE.md → 每次对话自动加载 → **不随压缩丢失**

这形成了一个**双层记忆架构**：

```
短期记忆：messages[]（会被压缩、会丢信息）
长期记忆：CLAUDE.md（持久化在磁盘上，永不丢失）
```

压缩只影响短期记忆。真正重要的约束活在长期记忆里，不参与压缩。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | 三层压缩 — 保留近期 + 摘要中期 + 丢弃远期 |
| **Tension** | 压缩必然丢信息，关键信息需要逃逸到压缩层之外 |
| **Contrast** | 没压缩 → 上下文爆满崩溃；有压缩无持久记忆 → 丢关键信息；双层记忆 → 兼得 |
| **Use** | Claude Code 的 92% 阈值自动压缩 + CLAUDE.md 长期记忆 |

---

## 贯穿 Session 1–6 的核心模式总结

### 设计哲学

> **代码提供能力（注册工具），LLM 做决策（选择何时调用哪个工具）。**  
> 循环不变，工具越来越多，但决策权始终在 LLM 手里。

### 架构演进路线

```
S1: 最小循环（while + stop_reason）
 ↓ 加工具
S2: 多工具调度（dispatch map）
 ↓ 加规划
S3: Plan 机制（TodoWrite）
 ↓ 加隔离
S4: Subagent（独立 messages[]）
 ↓ 加知识
S5: Skills（按需加载 SKILL.md）
 ↓ 加记忆管理
S6: 上下文压缩 + 双层记忆（messages[] + CLAUDE.md）
```

### 反复出现的核心张力

| 张力 | 体现 |
|------|------|
| **控制权归属** | 代码 vs LLM — 循环由 LLM 的 stop_reason 控制 |
| **灵活性 vs 可靠性** | 软性 prompt 引导 vs 硬编码强制（如 TodoWrite 的 nag reminder） |
| **隔离 vs 共享** | Subagent 的干净上下文 vs 信息丢失 |
| **全量 vs 按需** | System prompt 塞满 vs tool_result 动态注入 |
| **短期 vs 长期** | messages[] 会压缩丢失 vs CLAUDE.md 持久化不丢 |

---

## Session 6 深入：三层上下文压缩的完整解析（来自 Part D）

> 来源 Part: part_d_restructured.md

### 引入动机

s06 是上下文管理的核心 Session。之前已经概览了三层压缩的架构，这里深入每一层的代码和设计精妙之处。

### 深入点 1: Layer 1 — 静默微压缩

这是**最精妙**的一层。它在后台静默运行，LLM 完全不知道它的存在：

```python
# 只保留最近 3 个工具结果的原文，其余替换为占位符
to_clear = tool_results[:-KEEP_RECENT]  # KEEP_RECENT = 3
for _, _, result in to_clear:
    if len(result["content"]) > 100:    # 只压短的不管
        result["content"] = f"[Previous: used {tool_name}]"
```

**设计精妙之处**：

| 细节 | 为什么 |
|------|-------|
| 只压缩 >100 字符的结果 | 短结果（如 "Wrote 128 bytes"）留着不占空间，保留信息 |
| 保留工具名 `[Previous: used bash]` | LLM 还能知道"之前用过什么"，只是不知道具体输出 |
| 保留最近 3 个 | 当前任务的工具结果不能丢，只丢"历史"结果 |
| **每轮都执行** | 不等阈值触发，持续保持上下文"瘦身" |

> **核心洞察**：Layer 1 不是"压缩"，是**信息降级** — 把完整信息降级为摘要，但保留"发生过什么"的线索。

### 深入点 2: Layer 2 — 压缩 Prompt 怎么写

这是最能**直接复用**的技巧：

```python
response = client.messages.create(
    model=MODEL,
    messages=[{"role": "user", "content":
        "Summarize this conversation for continuity. Include: "
        "1) What was accomplished, 2) Current state, 3) Key decisions made. "
        "Be concise but preserve critical details.\n\n" + conversation_text}],
    max_tokens=2000,
)
```

Prompt 只要求三件事：

| 要求 | 为什么 |
|------|-------|
| **What was accomplished** | 不重复已完成的工作 |
| **Current state** | 知道现在处于什么阶段 |
| **Key decisions made** | 不推翻已做的决定 |

**没有要求保留"所有细节"** — 这是关键。压缩的目的是让 Agent 能**继续工作**，不是完美复现历史。

### 深入点 3: 压缩后的消息结构

```python
return [
    {"role": "user", "content": f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
    {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."},
]
```

压缩后 messages 里只剩**两条消息**。两个设计细节：

**① 为什么要告诉 LLM transcript 的文件路径？**
— 如果 LLM 后来真的需要某个被压缩掉的细节，它可以用 `read_file` 去读原始记录！这是**逃生通道** — 压缩不是不可逆的。

**② 为什么要伪造一条 assistant 回复 "Understood"？**
— API 要求 messages 必须 user/assistant 交替。如果只有一条 user 消息，下一轮 LLM 回复后变成 user → assistant → assistant，格式会报错。所以加一条假的 assistant 回复来**维持轮次结构**。

### 讨论：Layer 1 和 Layer 2 的协作关系

**问（Assistant）**：Layer 1（微压缩）每轮都跑，Layer 2（全量压缩）到 50K token 才触发。如果没有 Layer 1，只靠 Layer 2，会有什么问题？换句话说，Layer 1 的存在让 Layer 2 被触发的频率发生了什么变化？

**答（User）**：layer 1 auto delete context working less efficient for current task, cause the used context may have worked to help llm reason and raise next step idea, and leave more context for maybe more useful docs and longer route for the future task in a more sufficient context. and layer 2 act less. question is in a real work, when the 2 layers act, show me

**评估（Assistant）**：你的分析很到位！Layer 1 确实有 tradeoff — 旧工具结果可能对推理有帮助，但移除它们为后续留出了空间。

### 实际工作场景演示：重构一个 Python 项目

假设用户说："把 app.py 拆分成 routes.py、models.py、utils.py，并添加测试"

```
Token 使用量
50K ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  THRESHOLD (Layer 2 触发线)
 │                                    ╱
 │                          没有 L1 ╱
 │                        的增长 ╱     有 Layer 1 时
 │                           ╱        的实际增长
 │                        ╱          ╱─────────
 │                     ╱          ╱            ↘ Layer 2 触发
 │                  ╱          ╱                 压缩回 ~2K
 │               ╱        ╱
 │            ╱        ╱
 │         ╱      ╱
 │      ╱    ╱
 │   ╱  ╱
 │ ╱╱
 ╱─────────────────────────────────────────────→ 轮次
 1   5   10   15   20   25   30   35   40
```

**逐轮详解**：

```
轮次 1:  user: "把 app.py 拆分..."                    ~200 tokens
轮次 2:  → todo: 创建 5 步计划                         ~500
轮次 3:  → read_file("app.py") → 返回 200 行代码      ~3,000
轮次 4:  → write_file("models.py")                    ~5,000
         ┌─ Layer 1: 轮次 3 的 read_file 结果还在（最近 3 个之一）
轮次 5:  → write_file("routes.py")                    ~7,500
轮次 6:  → write_file("utils.py")                     ~10,000
         ┌─ Layer 1 触发！轮次 3 的 read_file 结果已超出最近 3 个
         │  "200 行 app.py 代码" → "[Previous: used read_file]"
         │  释放约 2,500 tokens
         └─ 实际 token: ~7,500（而不是 10,000）
轮次 7:  → edit_file("app.py") 瘦身原文件              ~9,000
轮次 8:  → read_file("models.py") 验证                 ~11,000
         ┌─ Layer 1: 轮次 4 的 write_file 结果被压缩
         └─ 实际 token: ~9,000
轮次 9:  → bash("pytest") 运行测试                     ~10,500
轮次 10: → 修 bug, edit_file                           ~12,000
         ...
轮次 25: → 持续工作...                                 ~25,000
         ┌─ 如果没有 Layer 1，此时已经约 45,000，接近阈值
         └─ 有 Layer 1，实际只有约 25,000，还有很大空间
         ...
轮次 40: → 大量工作后，终于接近 50,000                   ~50,000
         ┌─ Layer 2 触发！
         │  1. 保存完整记录到 .transcripts/transcript_1710xxx.jsonl
         │  2. 让 LLM 总结: "已完成拆分，models/routes/utils 已创建，
         │     3 个测试通过，还剩 2 个 edge case 要修"
         │  3. messages 从 ~50,000 tokens 压缩到 ~2,000 tokens
         └─ Agent 继续工作，就像什么都没发生
```

**关键数据对比**：

| 指标 | 只有 Layer 2 | Layer 1 + Layer 2 |
|------|------------|-------------------|
| Layer 2 触发次数（100 轮内） | **约 3-4 次** | **约 1 次** |
| 每次 Layer 2 的成本 | 1 次额外 LLM 调用 | 同左 |
| 信息损失频率 | 高（频繁全量压缩） | 低（微压缩损失小，全量压缩很少发生） |

> **总结**：Layer 1 通过持续"减负"，**大幅延迟 Layer 2 的触发时机**。Layer 2 是一次性的"大手术"（丢弃所有历史），越少触发越好。

### 讨论续：KEEP_RECENT = 3 够不够？

**问（Assistant）**：代码里设定 `KEEP_RECENT = 3`，保留最近 3 个。如果你的 Agent 做的是需要"回头看"的任务（比如对比两个文件的差异），3 够不够？你会怎么调整这个策略？

**答（User）**：read again?

**评估（Assistant）**：对！就是这么简单 — **再读一次就行了**。

Layer 1 删掉的只是 messages 里的工具结果副本，**原始文件还在磁盘上**。LLM 需要的时候，再调用 `read_file` 就能拿回来。

> **这揭示了压缩策略的一个重要原则：只压缩可以重新获取的信息，不压缩不可恢复的信息。**

文件内容可以重新读 → 安全压缩。用户说的话、LLM 做的决定 → 不能重新获取 → 这就是为什么 Layer 2 要用 LLM 做总结来**保留决定和状态**，而不是直接删掉。

### 讨论续：User 对 Layer 1 价值的更精确理解

**问（User）**：rather than "旧工具结果可能对推理有帮助", i mean old tool result may have worked building a new step or result, so a new tool result may act more possibility helping than old.

**答（Assistant）**：明白了，你说的比我理解的更精确。

**你的意思是**：旧工具结果的价值**已经被 LLM 消费过了** — 它已经基于这些结果做出了下一步决策。这些决策本身（体现为后续的工具调用和结果）已经"继承"了旧结果的信息价值。

```
旧工具结果 → 已被 LLM 消化 → 产出了新的决策/结果
                              ↓
                         新结果承载了旧结果的"精华"
                              ↓
                         旧结果可以安全删除
                         腾出空间给未来更有价值的新结果
```

**这不是"旧的可能有用但忍痛删掉"，而是"旧的已经完成使命了，删掉是合理的"。** 就像你读完一本书做了笔记，书可以放回书架（压缩），笔记（新的工具结果和决策）留在手边就够了。

> User 的理解比 tradeoff 更深 — 这其实是 **信息流的单向性**：旧 → 新的价值传递已经发生，保留旧的是冗余而非保险。

### s06 完整收获总结

| 层次 | 学到的 |
|------|---------| 
| **Layer 1** | 微压缩 = 信息降级，每轮静默运行，延迟 Layer 2 触发 |
| **Layer 2** | 全量压缩 = LLM 总结三件事（完成了什么、当前状态、关键决定） |
| **Layer 3** | 和 Layer 2 一样，但由 LLM 主动触发（又是"LLM 做决策"的模式） |
| **逃生通道** | transcript 存磁盘 + 文件可重新读取 → 压缩不是真正的遗忘 |
| **核心原则** | 只压缩可重新获取的，保留不可恢复的 |

---

## 延伸讨论：Round / Turn / Message 的精确定义（来自 Part D）

> 来源 Part: part_d_restructured.md

### 引入动机

在讨论 Layer 1 的"每轮都跑"时，User 追问了一个关键的定义问题：**"round" 到底是什么？怎么定义？** 这个定义对理解 nag reminder 的 `rounds_since_todo >= 3` 至关重要。

### 核心概念

一个 round = **agent_loop 里 while 循环的一次迭代**：

```
一个 Round:
  ┌─────────────────────────────────────────┐
  │ 1. micro_compact (压缩旧结果)            │
  │ 2. 发送 messages 给 LLM                  │
  │ 3. 收到 response                         │
  │ 4. LLM 说要用工具 → 执行工具             │
  │ 5. 把工具结果加入 messages                │
  └─────────────────────────────────────────┘
       ↓ 回到第 1 步 = 下一个 round
```

**注意：一个 round ≠ 一次用户输入。** 用户输入一次后，Agent 可能循环很多轮：

```
用户: "重构 app.py"
  │
  ├─ Round 1: LLM → todo(创建计划)
  ├─ Round 2: LLM → read_file("app.py")
  ├─ Round 3: LLM → write_file("models.py")
  ├─ Round 4: LLM → write_file("routes.py")
  ├─ Round 5: LLM → bash("pytest")
  └─ Round 6: LLM → stop_reason="end_turn" → 退出循环，回复用户
```

用户只说了一句话，但产生了 **6 个 rounds**。

**所以 `rounds_since_todo >= 3`（s03 的 nag reminder）的意思是**：LLM 连续调用了 3 次工具都没更新 todo，而不是用户等了 3 轮。

### Round vs Turn vs Message 对照表

| 术语 | 定义 | 例子 |
|------|------|------|
| **Message** | messages 列表里的一条记录 | `{"role": "user", "content": "..."}` |
| **Turn** | 一次完整的用户-Agent 交互 | 用户提问 → Agent 做完所有事 → 回答 |
| **Round** | while 循环一次迭代 | 一次 LLM 调用 + 一次工具执行 |

> 一个 Turn 包含多个 Rounds，一个 Round 产生多条 Messages。

---

## 延伸讨论："重大内容"的含义 + 完整学习路径选项（来自 Part D）

> 来源 Part: part_d_restructured.md

### 引入动机

User 在 s06 深入完成后追问了两件事：(1) Assistant 说"没有遗漏的重大内容"，到底什么算"重大内容"？(2) 除了之前给的 4 个选项，还有什么可以学？

### 讨论

**问（User）**：what is "重大内容"? and any other choice?

**答（Assistant）**：我查 s06 文档时，在找是否有**我们还没讨论过的新机制或设计决策**。比如：
- 有没有第四层压缩我们没看到？→ 没有
- 有没有特殊的 edge case 处理？→ 没有
- 文档是否提到了代码里没实现的思路？→ 没有

所以"没有遗漏的重大内容" = 文档和代码是一致的，核心我们已经全部覆盖了。

### 完整学习路径选项

当时的学习进度是 s03 ✅ s06 ✅，完整选项如下：

| 选项 | 内容 | 会学到什么 |
|------|------|-------------|
| **A. s04 — Subagent** | 看代码怎么创建子 Agent | 上下文隔离的具体实现 |
| **B. s05 — Skills** | 动态加载技能的代码 | on-demand 注入 tool_result 的写法 |
| **C. s07 — Task Graph** | tasks.json 持久化 | 多 Agent 依赖管理 |
| **D. s08 — Background** | 守护线程 + 通知队列 | 异步任务不阻塞 Agent |
| **E. s09-s10 — 多 Agent 通信** | Mailbox + 协议 | Agent 间如何协作 |
| **F. s11-s12 — 自主认领 + Worktree** | 竞争任务 + Git 隔离 | 多 Agent 生产级架构 |
| **G. s_full.py** | 12 个机制集成到一起 | 看完整 Agent 的真实架构 |
| **H. 对比逆向分析** | learn-claude-code vs 真实 Claude Code | 教学版和生产版的差距 |
| **I. Checkpoint** | 把已学内容沉淀为结构化笔记 | 方便复习和后续深入 |

User 选了 **A（s04 Subagent）**。

---

## 贯穿 Part D 的核心模式总结

> 来源 Part: part_d_restructured.md

### 深入学习的节奏模式

Part D 展现了一种高效的深入学习节奏：
1. **先吃透代码细节**（write_file vs edit_file 的具体实现）
2. **再上升到设计洞察**（system prompt 稀释效应）
3. **通过真实场景验证理解**（重构 Python 项目的逐轮 token 演示）
4. **然后用一个简洁回答检验深度**（"read again?" — 最简单的答案就是最好的答案）

### 信息管理的核心原则

本部分反复出现的一条主线：

> **只压缩可以重新获取的信息，不压缩不可恢复的信息。**

这条原则贯穿了：
- Layer 1 微压缩（文件内容可重读 → 安全压缩）
- Layer 2 全量压缩（保留决定和状态 → 不可恢复的信息用 LLM 总结保留）
- transcript 文件（原始记录存磁盘 → 逃生通道）

### User 的独立洞察：信息流的单向性

User 对 Layer 1 价值的理解超越了 "tradeoff" 框架，提出了更精确的模型：

> 旧工具结果的价值已经被 LLM 消费并转化为新的决策和结果。删除旧结果不是"忍痛割爱"，而是"使命完成，合理退场"。新结果比旧结果对未来更有帮助。

这是**信息流的单向性**洞察 — 价值已经向前传递，保留源头是冗余而非保险。
