# Session 9：多 Agent 通信 — JSONL 邮箱

> 来源 Part: part_b_restructured.md, part_f_restructured.md  
> 前置：[Session 8：后台任务 — 非阻塞异步执行](session_08.md)  
> 后续：[Session 10：通信协议 — 请求-响应 + 优雅关闭](session_10.md)

---

## 引入动机

之前的 Subagent（s04）是**主 Agent 临时创建、用完即弃**的。Session 9 引入的是**持久化的队友** — 它们一直存在，各自运行自己的循环，通过**消息邮箱**通信。

Session 9 的 motto：

> *"When the task is too big for one, delegate to teammates."*

## 核心概念 / 实现方式

通信机制的核心是 **JSONL 邮箱文件**：

```python
# 每个 Agent 有自己的邮箱文件
# agent_alice.jsonl, agent_bob.jsonl

def send_message(to_agent, content):
    with open(f"{to_agent}.jsonl", "a") as f:
        f.write(json.dumps({"from": self.name, "content": content}) + "\n")

def check_mailbox():
    with open(f"{self.name}.jsonl", "r") as f:
        return [json.loads(line) for line in f]
```

**为什么用文件而不是内存队列？**
- 文件可以跨进程
- 重启后消息不丢
- 最简单的持久化通信方式

## 讨论

**问（Assistant）**：如果 Agent Alice 发了一条消息给 Agent Bob，但 Bob 正忙着执行一个工具还没检查邮箱，怎么办？这整个系统需要 Alice "等 Bob 回复"的阻塞机制，还是可以异步继续？

> 回忆一下 s08 后台任务的设计，模式是否类似？

**答（User）**：此时 Bob 就是一个 tool，Alice 可以继续思考和工作。

**评估（Assistant）**：✅ 通过。精准！一句话抓住了多 Agent 通信的本质。

> "Bob 就是一个 tool，Alice 可以继续工作"

展开来说：

- **发消息 = 调用异步工具**（`send_message` 写入文件就返回，不等结果）
- **检查邮箱 = s08 的通知队列检查**（每轮循环开头检查一下有没有新消息）
- **Alice 不阻塞** — 发完消息继续做其他任务

模式和 s08 后台任务完全一致：**写入 → 继续 → 下一轮检查结果**。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | 持久化队友 + JSONL 邮箱 = 异步多 Agent 通信 |
| **Tension** | 异步通信高效但可能消息遗漏或顺序错乱 |
| **Contrast** | s04 Subagent 是临时的主从关系；s09 是持久的平等队友关系 |
| **Use** | 文件邮箱是最简单的跨进程持久通信方案 |

---

## 延伸深入：Subagent vs Teammate、JSONL 信箱、通信流程（来自 Part F）

> Part F 对 Session 9 进行了深入讨论，展开了三个核心设计点和完整的通信流程示例。

### 深入点 1：s04 Subagent vs s09 Teammate — 本质区别

```
s04 Subagent:  创建 → 工作 → 返回结果 → 销毁（一次性的）
s09 Teammate:  创建 → 工作 → 空闲 → 工作 → ... → 关闭（持久的）
```

代码体现在第 200–203 行：

```python
# Subagent 结束后直接返回文本（s04），context 丢弃
# Teammate 结束后变成 idle，可以再次被激活
member["status"] = "idle"
self._save_config()
```

Teammate 做完任务后**不销毁**，而是变成 `idle` 状态等待下一个任务。

### 深入点 2：JSONL 信箱通信（第 82–108 行）

每个 Teammate 有一个 `.jsonl` 文件作为信箱：

```python
# 发消息：追加写入（append）
with open(inbox_path, "a") as f:        # "a" = append 模式
    f.write(json.dumps(msg) + "\n")

# 读消息：全部读取，然后清空
messages = [json.loads(l) for l in ...]
inbox_path.write_text("")               # 清空 = "信已取走"
```

**为什么用文件而不是 Python 的 Queue？**

| | Python Queue | JSONL 文件 |
|--|-------------|-----------|
| **线程间** | ✅ 可以 | ✅ 可以 |
| **进程间** | ❌ 不行 | ✅ 可以 |
| **重启后** | ❌ 丢失 | ✅ 还在 |
| **调试** | 看不到内容 | 直接打开文件看 |

文件信箱为将来多进程（多台机器上的 Agent）留了扩展空间。

### 深入点 3：Teammate 循环里自动检查信箱（第 172–175 行）

```python
for _ in range(50):
    inbox = BUS.read_inbox(name)          # 每轮都检查信箱
    for msg in inbox:
        messages.append({"role": "user", "content": json.dumps(msg)})
```

Teammate 在**每轮循环开始时**读自己的信箱，把收到的消息当作 `role: "user"` 插入 messages。这和 s08 的 `drain_notifications` 是**完全相同的模式**：

```
s08: drain_notifications → <background-results> → messages
s09: read_inbox          → 消息内容              → messages
```

### 完整的通信流程

```
Lead: "Alice，请审查 main.py"
  → send_message(to="alice", content="请审查 main.py")
  → alice.jsonl 多了一行

Alice's loop（下一轮）:
  → read_inbox("alice") → 拿到消息
  → 读取 main.py，审查...
  → send_message(to="lead", content="发现 3 个问题")
  → lead.jsonl 多了一行

Lead's loop（下一轮）:
  → read_inbox("lead") → 拿到 Alice 的回复
  → "Alice 说有 3 个问题，让我看看..."
```

### Guided Prompt 与讨论

**问（Assistant）**：注意第 107 行 `inbox_path.write_text("")` — 读完就清空。**如果 Alice 正在写消息到 Bob 的信箱（append），同时 Bob 在读信箱并清空（write_text("")），会发生什么？** 这和 s08 的 Lock 问题类似，但这里没有用 Lock。

### 讨论 1：Notification 不会主动唤醒 Agent

**问（User）**：Notification don't activate agent, so it ends with assistant. 但在 Lead's loop 中读到 Alice 的回复，Alice 的消息以 user 结尾，Lead 能直接处理吗？

**答（Assistant）**：你观察得很仔细！看代码里 Lead 读信箱的完整流程（第 346–355 行）：

```python
while True:
    inbox = BUS.read_inbox("lead")
    if inbox:
        messages.append({"role": "user", ...})       # inbox 内容
        messages.append({"role": "assistant", ...})   # "Noted inbox messages"
    
    response = client.messages.create(...)            # LLM 调用紧接着发生
```

**关键**：inbox 注入和 LLM 调用在**同一轮循环里**。流程是：

```
while True 循环内：
  ① 读信箱 → 插入 user + assistant
  ② 立刻调用 LLM → LLM 看到了信箱内容
  ③ LLM 回复 → 处理工具 → 回到 ①
```

所以 LLM **是能看到信箱内容的** — 因为它在插入之后立刻被调用了。

**但是！** User 说的问题是对的 — 如果 Lead 的循环**已经退出**了（`stop_reason="end_turn"`），Alice 的消息就会**卡在 lead.jsonl 里**，直到用户输入新消息，触发新的 `agent_loop`，才会在第 ① 步被读到。

```
Lead 循环结束 → Alice 发来消息 → lead.jsonl 里静静等着
                                      ↓
用户输入新问题 → 新的 agent_loop → ① 读到 Alice 的消息 → ② LLM 处理
```

这是 s09 的一个**已知局限** — Lead 不会被动地"被叫醒"。它只有在主动循环时才能读信箱。真正的生产系统需要一个"唤醒"机制（比如监听文件变化）。

### 讨论 2：连续两个 assistant 消息的设计缺陷

**问（User）**：`messages.append({"role": "assistant", ...})` + `response` 的 assistant 回复，不会产生两条连续 assistant 消息吗？

**答（Assistant）**：你发现了一个**真正的问题**！让我仔细追踪代码：

```python
# 第 348-355 行：注入 inbox
messages.append({"role": "user", ...})         # ← user
messages.append({"role": "assistant", ...})     # ← assistant ("Noted")

# 第 356-363 行：调用 LLM
response = client.messages.create(..., messages=messages)
messages.append({"role": "assistant", "content": response.content})  # ← assistant
```

执行后 messages 是：

```
..., user(inbox), assistant("Noted"), assistant(LLM 回复)
                  ↑                   ↑
                  连续两个 assistant！❌
```

**User 说得对 — 这确实会产生连续两个 assistant 消息。** 按照 Anthropic API 的要求，messages 必须严格 user/assistant 交替，这应该会报错。

这代码能工作的可能原因：

1. **某些 API 实现会自动合并** 连续同 role 的消息
2. 或者这是这个教学项目中的**一个 bug**

**正确的做法应该是**：不加假的 assistant，直接让 inbox 作为 user 消息，紧接着 LLM 调用：

```python
if inbox:
    messages.append({"role": "user", "content": f"<inbox>...</inbox>"})
    # 不加 assistant，直接调用 LLM ↓

response = client.messages.create(...)   # 最后一条是 user，合法
messages.append({"role": "assistant", ...})
```

但这样需要确保 inbox 注入前最后一条是 assistant（通常是的，因为上一轮 LLM 回复就是 assistant）。

> **User 凭自己的推理发现了教学代码中的一个设计缺陷。** 这比被动接受知识更有价值。

### 讨论 3：broadcast 机制

**问（User）**：Later, if dive in true Claude Code, I want to figure it out. Now, just dive into others. Whether s09 remains anything worth learning? If not, next step.

**答（Assistant）**：s09 还有一个值得一提的点 — **broadcast**（第 110–116 行）：Lead 可以一次性给所有 Teammate 发消息。但实现就是循环调用 `send`，很简单。s09 完成。

---

## 贯穿 Part F 的跨 Session 模式映射

```
s08 drain_notifications  ←→  s09 read_inbox     （同一模式：读取 → 注入 messages）
s08 Lock 保护 queue      ←→  s09 无 Lock 的文件信箱（已知风险：文件级竞态条件）
s08 背景通知不唤醒 Agent  ←→  s09 信箱消息不唤醒 Lead（已知局限：需要主动循环才读取）
```

### User 发现的代码缺陷

| 缺陷 | 位置 | 问题 |
|------|------|------|
| 连续两个 assistant 消息 | s09 inbox 注入 + LLM 回复 | 注入假 assistant 后紧接 LLM 回复，产生连续 assistant，违反 API 要求 |

这是 User 通过独立推理发现的，而非被动接受的知识点。
