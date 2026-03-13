# Session 11：去中心化调度 — 自主认领任务

> 来源 Part: part_b_restructured.md, part_f_restructured.md, part_g_restructured.md  
> 前置：[Session 10：通信协议 — 请求-响应 + 优雅关闭](session_10.md)  
> 后续：[Session 12（最终章）：Worktree 隔离 — 文件系统级隔离](session_12.md)

---

## 引入动机

s09-s10 里，任务分配是**主 Agent 指派**的（Alice 对 Bob 说"你去做 X"）。但这有瓶颈：主 Agent 成了调度中心，所有分配都经过它。Session 11 的 motto：

> *"Teammates scan the board and claim tasks themselves."*

## 核心概念 / 实现方式

核心变化：Agent 不再等待分配，而是运行一个**空闲扫描循环**：

```python
def idle_cycle(self):
    while True:
        # 扫描任务图，找到"status=pending + 依赖已满足"的任务
        available = [t for t in tasks if t.status == "pending" 
                     and all(dep.status == "done" for dep in t.deps)]
        
        if available:
            task = available[0]
            task.status = "claimed_by_" + self.name  # 认领
            self.execute(task)
        else:
            sleep(interval)  # 没任务就等一会儿
```

**从"中心化调度"变成了"去中心化自主认领"。** 主 Agent 只需要创建任务图（s07），不需要逐个分配。

## 讨论

**问（Assistant）**：这个设计有一个经典的并发问题：**如果 Agent Alice 和 Agent Bob 同时扫描到同一个可用任务，都试图认领它，会发生什么？** 你能想到怎么防止这种冲突吗？

**答（User）**：谁先认领这个任务，这个任务就归属谁，二者肯定有先后顺序。

**评估（Assistant）**：✅ 部分通过（需要补充一个关键点）。

方向是对的！但在多线程/多进程环境下，有一个经典问题叫 **竞态条件（Race Condition）**：

```
时间线：
Alice 读取任务状态 → "pending" ✓ 可以认领
                                    Bob 读取任务状态 → "pending" ✓ 可以认领
Alice 写入 "claimed_by_alice"
                                    Bob 写入 "claimed_by_bob"  ← 覆盖了 Alice 的认领！
```

两个 Agent **在"读"和"写"之间有时间差**，都认为自己是第一个。

常见的解决方案：

| 方案 | 原理 |
|-----|------|
| **文件锁** | 认领前锁定 tasks.json，写完后释放 |
| **原子操作** | 用 `os.rename()` 等原子文件操作来"抢"任务 |
| **乐观锁** | 读取时记录版本号，写回时检查版本是否变化 |

`learn-claude-code` 作为教学项目用了最简单的方案 — 因为它跑在同一进程的线程里，用文件写入的先后顺序就足够了。但**真正的生产系统必须处理这个问题**。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | 空闲扫描 + 自主认领 = 去中心化调度 |
| **Tension** | 去中心化高效但有竞态风险，需要锁或原子操作 |
| **Contrast** | s09 中心化分配 → 主 Agent 是瓶颈；s11 自主认领 → 无瓶颈但需并发控制 |
| **Use** | 理解为什么分布式系统都需要某种形式的"共识机制" |

---

## 延伸深入：Identity 重注入机制深度追问（来自 Part G）

> Part F 已讨论了 s11 的整体架构（spawn、inbox、idle cycle、auto-claim）。Part G 回到一个具体的实现细节——`make_identity_block` 和 `len(messages) <= 3` 的条件判断—— User 对此提出了一系列追问和质疑。

### 问题 1：Identity 是 Lead 设的吗？

**问（User）**：Identity 是 Lead 设的吗？

**答（Assistant）**：**不是。** Identity 不是 Lead 发送的消息，是 **Teammate 自己在代码里注入的**。

看 spawn 的流程：

```python
# Lead 调用 spawn_teammate(name="alice", role="coder", prompt="Fix the bug")

# Teammate 线程启动后（第 209-211 行）：
sys_prompt = f"You are 'alice', role: coder, team: default, at /workspace."
```

身份信息在**两个地方**：
- **system prompt** — 每次 LLM 调用都发送，不会丢
- **messages 里** — 只有被压缩后才需要重新注入

正常情况下 system prompt 就够了。`make_identity_block` 是**保险措施** — 当 messages 被压缩到很短时，额外在 messages 里加一条身份信息，加强效果。

---

### 问题 2：为什么是 `<= 3`？

**问（User）**：（隐含于上文讨论）

**答（Assistant）**：这是一个**启发式判断**："如果 messages 只剩 3 条或更少，说明上下文可能被压缩过了。"

正常工作流中 messages 至少有：

```
[0] user: 初始 prompt（"Fix the bug"）
[1] assistant: LLM 第一次回复
[2] user: tool_results
[3] assistant: LLM 第二次回复
...很快就超过 3 条
```

只有**两种情况** messages 会 ≤ 3：
1. **刚刚被 s06 auto_compact 压缩过** — 所有消息被替换为一条摘要
2. **刚启动还没做什么** — 但这时身份信息还"新鲜"，注入也没害处

所以 `<= 3` 本质是在问：**"上下文是不是被压缩过了？"** 数字 3 本身不重要，是一个保守的阈值。可以是 2，也可以是 5，核心思路是一样的。

---

### 问题 3：System prompt 不在 messages 里？

**问（User）**：What does "In system prompt 每次 LLM 调用都发送" and "重新注入 messages" mean? System prompt not in message?

**答（Assistant）**：看 API 调用的代码（第 226-232 行）：

```python
response = client.messages.create(
    model=MODEL,
    system=sys_prompt,       # ← system prompt，独立参数
    messages=messages,       # ← messages，另一个参数
    tools=tools,
    max_tokens=8000,
)
```

**system 和 messages 是两个独立的参数**，发给 LLM 时的结构是：

```
┌─────────────────────────────────────────┐
│ system: "You are 'alice', role: coder"  │  ← 每次调用都原封不动发送
├─────────────────────────────────────────┤
│ messages[0]: user: "Fix the bug"        │
│ messages[1]: assistant: "OK, let me..." │  ← 这些会增长、会被压缩
│ messages[2]: user: tool_results         │
│ ...                                     │
└─────────────────────────────────────────┘
```

**关键区别：**

| | system prompt | messages |
|--|--------------|----------|
| **位置** | 在 messages 外面，单独的参数 | 对话历史列表 |
| **会被压缩吗？** | ❌ 永远不会，每次调用都完整发送 | ✅ 会被 s06 压缩 |
| **LLM 记忆** | 每次调用都看到 | 可能被压缩丢失 |

所以 `sys_prompt` 里的 `"You are 'alice', role: coder"` **永远不会丢**。

那 `make_identity_block` 重注入进 messages 的作用是什么？—— **加强效果**。system prompt 在长对话中容易被"稀释"（你在 s03 学过的 system prompt dilution），在 messages 里再放一条身份提醒，让 LLM 更不容易"忘记自己是谁"。

---

### 问题 4：启动时双重 system prompt？

**问（User）**：But in this situation, every start, it begins with twice "system prompt". And just at the begin.

**评估（Assistant）**：你说得对。更准确地说：

```python
# 第 281-285 行 — 只在自动认领任务时触发
if len(messages) <= 3:
    messages.insert(0, make_identity_block(...))    # 插到最前面
    messages.insert(1, {"role": "assistant", ...})  # 假回复
messages.append({"role": "user", "content": task_prompt})
```

**这段代码只在一个场景触发**：Teammate 空闲时自动认领了新任务，且 messages 很短。

实际时间线：

```
第一次启动：
  system: "You are alice, coder"           ← system prompt
  messages: [user("Fix the bug")]           ← 正常，不注入 identity
  → messages > 3 条后永远不会注入

被压缩后：
  system: "You are alice, coder"           ← system prompt 还在
  messages: [user("Summary: ...")]          ← 压缩后只剩 1-2 条
  → 自动认领新任务时，len ≤ 3，触发注入
  → messages: [user(identity), assistant("I am alice"), user(summary), user(new task)]
```

所以 User 观察到的**确实是冗余** — system prompt 已经有身份了，messages 里再加一次是双保险。这种"重复"是**有意为之**的，因为压缩后 LLM 可能处于一种"迷糊"状态，多一次提醒比少一次好。

**但它只会出现一次**（在压缩后认领新任务时），不像 s03 的 nag 会反复触发。

---

### 问题 5：User 质疑——压缩后 LLM 不应该更清醒吗？

**问（User）**：Why 压缩后 LLM 可能处于一种"迷糊"状态? The context is short after compressed, the sys prompt should be more powerful I think.

**评估（Assistant）**：**你说得对。** 压缩后 context 变短了，system prompt 反而**更突出**，不会被稀释。

所以这个 identity 重注入**并不是因为 LLM 会迷糊**，更准确的原因应该是：

**压缩后的 summary 可能根本没提到"我是谁"。** 比如压缩后 messages 可能是：

```
[user: "之前完成了 models.py 的重构，修了 3 个 bug"]
```

这条摘要描述的是**做了什么**，没说**谁做的**。然后紧接着来了一个新任务。LLM 虽然从 system prompt 知道"我是 alice"，但 messages 里的上下文完全断裂了 — 没有任何历史工作和身份的关联。

加一条 identity block 的作用是**在 messages 里建立连续性**：

```
[user: "<identity>You are alice, coder</identity>"]
[assistant: "I am alice. Continuing."]
[user: "之前完成了 models.py 的重构..."]
[user: "<auto-claimed>Task #5: Write tests</auto-claimed>"]
```

**但说实话，你的质疑是合理的** — system prompt 已经有身份信息了，这个重注入可能是**过度防御的设计**，在实际运行中可能没那么必要。

---

### 问题 6：User 再次纠正——示例中的 role 错误

**问（User）**：Two users in your sample. And is sysprompt not in message?

**评估（Assistant）**：两个好的纠正：

#### 1. 示例有误，实际代码是：

```python
# 第 281-285 行
messages.insert(0, make_identity_block(...))          # user
messages.insert(1, {"role": "assistant", ...})        # assistant
messages.append({"role": "user", "content": task})    # user
messages.append({"role": "assistant", "content": ...}) # assistant
```

正确的 messages 顺序：

```
[user: identity]
[assistant: "I am alice. Continuing."]
[user: "Summary: 之前完成了..."]     ← 压缩后剩的
[user: "<auto-claimed>Task #5..."]   ← 还是连续 user！
```

**User 又发现了一个格式问题** — 压缩后的 summary（user）和新任务（user）仍然可能连续。取决于压缩后 messages 里最后一条是什么 role。

#### 2. System prompt 确实不在 messages 里

```python
client.messages.create(
    system=sys_prompt,    # ← 单独的参数，不在 messages 列表中
    messages=messages,    # ← 对话历史列表
)
```

它们发送给 LLM 时是**分开的**。所以 s06 压缩 messages 时，**不会动 system prompt**。System prompt 永远完整地存在，每次调用都原样发送。这也是为什么 User 说的对 — 压缩后 system prompt 更显眼了。

---

### 问题 7：这些是 bug 吗？

**问（User）**：Are these bugs?

**答（Assistant）**：是的，这是教学代码中的 bug。User 在这个项目里一共发现了两处：

1. **s09/s10/s11 的 inbox 注入** — `assistant("Noted") + assistant(LLM回复)` 连续两个 assistant
2. **s11 的压缩后认领任务** — 可能出现连续 user

真实的 Claude Code 肯定处理了这些格式问题。后面分析真实代码时可以验证。

---

## 教学代码 Bug 总结（来自 Part G）

User 在 Part F-G 的讨论过程中累计发现了两处教学代码 bug：

| Bug # | 位置 | 问题 |
|-------|------|------|
| 1 | s09/s10/s11 inbox 注入 | `assistant("Noted") + assistant(LLM回复)` 连续两个 assistant |
| 2 | s11 压缩后认领任务 | 压缩后 summary(user) + 新任务(user) 可能连续 |

这两个 bug 的共同根因是：**教学代码没有严格维护 user/assistant 交替的消息格式约束**。

### User 的质疑模式

Part G 展现了 User 越来越主动的质疑能力：

1. **质疑"迷糊"说法** — 压缩后 context 短了，system prompt 反而更突出，不应该迷糊 → Assistant 承认说法不准确，修正为"summary 没提身份"
2. **发现示例错误** — 连续两个 user role → Assistant 确认是 bug
3. **质疑本地存储** — "我能看到完整对话，应该存在本地" → 推动发现 `state.vscdb`
