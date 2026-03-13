# Session 1：最小 Agent 循环 — `stop_reason` 驱动

> 来源 Part: part_a_restructured.md  
> 前置：无（本 Session 为起点）  
> 后续：[Session 2：多工具调度 — 循环不变，工具扩展](session_02.md)

---

## 前置背景

### 学习背景

**User 请求**：分析 analysis_claude_code 及其 fork 项目，选择最好的项目下载。随后请求使用 coach skill 进行学习。

Assistant 选择了 **First Principles Learning Coach** 的 **Project Mode**，引导学习 `learn-claude-code` 项目。

### 项目总览

#### 核心命题

> *"Bash is all you need — 一个 while 循环 + 一个工具 = 一个 Agent"*

整个 12 个 Session 围绕这个最小循环做一件事 — **逐步添加一个机制，让简陋的循环逐渐具备 Claude Code 级别的能力**。

#### 项目解决的核心问题

> Claude Code 看起来能力很强（多工具调度、Plan 模式、子 Agent、上下文压缩、多 Agent 协作…），但**这些能力的本质是什么？它们是如何一层一层叠加到一个最小循环上的？**

回答：**从最小可行 Agent（一个 while + stop_reason 检测）出发，每次只加一个机制，12 步抵达完整多 Agent 系统**。

### 起点诊断：认知锚点测试

#### 为什么先诊断

在进入项目学习之前，需要知道学习者的认知锚点在哪里。这会帮助决定从哪个深度开始。

#### 讨论

**问（Assistant）**：你能用自己的话描述一下，一个最基本的 AI Agent（比如 Claude Code）和普通的 LLM API 调用之间，最本质的区别是什么？

**答（User）**：Agent 拥有 tool 和 memory，能够产生交互，返回结果，通过分析结果沉淀经验，以此为基础指导下一步乃至之后的动作。

**评估（Assistant）**：✅ 通过。抓住了三个关键点：
1. **Tool** — Agent 能调用外部工具（不只是生成文字）
2. **Memory** — 能积累上下文和经验
3. **循环决策** — 分析结果 → 指导下一步动作

这正是 Agent 和普通 API 调用的本质区别。普通 API 是 **单次问答**，Agent 是 **带决策的循环**。

---

## 引入动机

User 说的"分析结果指导下一步"非常对，但这里有个关键的细节需要弄清楚：**Agent 怎么知道该继续循环还是停下来？** 这是整个 Agent 循环里最小但最关键的判断点。

## 核心机制：`stop_reason` 驱动循环

Agent 循环的最精简版本：

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
        )
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason != "tool_use":  # ← 这一行是整个循环的开关
            return
        
        # 执行工具，把结果追加到 messages
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})
```

整个循环的控制权**不在我们手里，而是在 LLM 手里**。LLM 通过 `stop_reason` 告诉我们：
- `"tool_use"` → "我想用工具，别停"
- `"end_turn"` → "我说完了，可以停了"

## 讨论：为什么工具结果用 `role: user`？

**问（Assistant）**：在这个循环里，工具执行的结果被包装成了 `"role": "user"` 的消息追加回去。为什么不是 `"role": "assistant"` 或者某个新的 role（比如 `"role": "tool"`）？这样做的设计意图是什么？

> 提示：想想 LLM 的对话模型是怎么理解 user 和 assistant 轮次的。

**答（User）**：因为在训练时，就是一轮 user 一轮 assistant，前者是外部消息，后者是自己推理的消息。

**评估（Assistant）**：✅ 完全正确！精炼一下：

> LLM 训练时只见过两种角色：**user（外部输入）** 和 **assistant（自身推理）**。工具执行结果是**来自外部世界的信息**，所以自然应该放在 `user` 位置。这不是"欺骗"模型，而是**符合模型对信息来源的认知结构**。

这也解释了为什么 Agent 循环如此简洁 — 它复用了 LLM 已有的对话范式，没有发明新协议。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | Agent = while True + stop_reason 判断 |
| **Tension** | 循环控制权在 LLM 手里，不在代码手里 |
| **Contrast** | 普通 API 调用是单射（一问一答），Agent 循环是**由 LLM 自主决定何时结束** |
| **Use** | 这个最小循环是所有后续 11 个 Session 的地基 |
