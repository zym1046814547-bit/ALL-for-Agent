# Session 5：Skills — 按需加载知识

> 来源 Part: part_a_restructured.md, part_e_restructured.md  
> 前置：[Session 4：Subagent — 隔离上下文](session_04.md)  
> 后续：[Session 6：上下文压缩 — 双层记忆架构](session_06.md)

---

## 引入动机

到目前为止，Agent 的所有知识都在 system prompt 里写死。但一个真实的 Agent 需要面对很多领域 — 你让它改 React 代码它需要 React 知识，让它写 Python 测试它需要 pytest 知识。问题是：

> **把所有知识塞进 system prompt，上下文窗口根本装不下。**

> *"Load knowledge when you need it, not upfront."*

## 实现方式

Skill 的实现方式出奇简单 — 它是一个**工具**，不是 system prompt 的一部分：

```python
def load_skill(skill_name):
    """读取 SKILL.md 文件内容，作为 tool_result 返回"""
    path = f"skills/{skill_name}/SKILL.md"
    return open(path).read()

TOOL_HANDLERS["load_skill"] = load_skill
```

关键洞察：Skill 内容通过 **`tool_result`**（即 `role: user`）注入，而不是塞进 system prompt。知识是 **按需加载、动态注入的**，而且注入后它在 messages 里的位置是"外部信息"，LLM 天然知道怎么处理。

## 讨论（第一轮）：全量加载 vs 按需加载的深层好处

**问（Assistant）**：除了"上下文窗口装不下"这个显而易见的原因，方案 B（按需加载）还有什么更深层的好处？提示：想想 LLM 的注意力机制。

**答（User）**：不太明白你的意思。

**再次解释（Assistant）**：换个角度 — 想象 system prompt 里塞了 20 个 Skill 的完整内容（React、Python、Docker、SQL…），总共 5 万字。现在用户问了一个**只和 Python 相关**的问题。LLM 在处理这个问题时，那 5 万字的内容都会参与计算。其中 4.5 万字是无关的 React、Docker、SQL 知识。这些无关内容会：

1. **稀释注意力** — LLM 需要从一大堆无关信息中找到 Python 相关的部分，容易"走神"
2. **浪费 token** — 每轮对话都重复发送 5 万字，速度慢、成本高
3. **噪音干扰** — 无关的 Skill 内容可能让 LLM 产生错误的关联

方案 B（按需加载）的好处就是：当 LLM 需要 Python 知识时，**只**加载 Python Skill，上下文里没有噪音。LLM 的"注意力"全部集中在有用的信息上。

> **System prompt 塞满 = 图书馆里上课，到处是书在分神**  
> **Tool_result 按需注入 = 只把需要的那本书打开放在桌上**

## 讨论（第二轮）：谁来决定加载哪个 Skill？

**问（Assistant）**：在 Session 5 的设计里，"谁来决定加载哪个 Skill"？是代码逻辑判断的，还是 LLM 自己决定的？回想一下我们之前讨论 TodoWrite 时的相同设计模式。

**答（User）**：还是 Agent decide。

**评估（Assistant）**：✅ 对！**和 TodoWrite 一模一样的设计模式** — LLM 自己决定调用 `load_skill("python")` 还是 `load_skill("react")`。

到这里可以看到一个贯穿整个项目的**核心设计哲学**：

> **代码提供能力（注册工具），LLM 做决策（选择何时调用哪个工具）。**

循环不变，工具越来越多，但**决策权始终在 LLM 手里**。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | Skill = 通过 tool_result 按需注入的知识，不是塞进 system prompt |
| **Tension** | 全量加载（噪音 + 浪费）vs 按需加载（依赖 LLM 判断准确性） |
| **Contrast** | RAG 是检索外部文档；Skill 是加载预写好的专家指南 |
| **Use** | Claude Code 的 CLAUDE.md、Skills 系统都是这个模式 |

---

## 延伸深入：代码级实现细节（来自 Part E）

### 🎯 深入点 1：两层加载策略 — s05 最核心的思想

```
Layer 1（System Prompt）: 只放名字和简短描述 → ~100 tokens/skill
Layer 2（tool_result）:  按需加载完整内容 → 只在需要时花 tokens
```

用数字感受：

| 方案 | 10 个 skill，每个 2000 tokens | 上下文开销 |
|------|------------------------------|-----------| 
| 全部放 system prompt | 10 × 2000 = **20,000 tokens** | 每轮都付出 |
| 两层加载 | 10 × 100 + 按需 2000 = **~3,000 tokens** | 用到才付出 |

**节省 85% 的上下文空间。** 而且这和 s06 的压缩是互补的 — s05 减少输入，s06 压缩历史。

### 🎯 深入点 2：`<skill>` 标签包裹（第 103 行）

```python
return f'<skill name="{name}">\n{skill["body"]}\n</skill>'
```

Skill 内容用 `<skill>` XML 标签包裹后作为 `tool_result` 返回。这和 s03 的 `<reminder>` 标签是同一个思路 — **用标签告诉 LLM 这段内容的性质**。

LLM 看到 `<skill>` 会理解："这是专业知识指导，我应该遵循它来完成任务"，而不是当成普通的用户输入或文件内容。

### 🎯 深入点 3：LLM 自己决定什么时候加载（第 109-113 行）

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
{SKILL_LOADER.get_descriptions()}"""
```

System prompt 说的是 **"before tackling unfamiliar topics"** — 不是每次都加载，而是 LLM 自己判断"我不熟悉这个"时才调用 `load_skill`。

这又是**"LLM 做决策"**的设计哲学。代码给能力（load_skill 工具），prompt 给引导（何时使用），LLM 自己判断要不要用。

---

### 讨论：s05 Skill 加载 vs s04 Subagent 的共同问题

**问（Assistant）**：s05 的 skill 加载和 s04 的 subagent，解决的其实是同一个问题的两个维度。你能看出来是哪个问题吗？

> 提示：想想它们各自在"节省"什么。

**答（User）**：Saving context. S04 saves dirty processing context; S05 saves unnecessary context.

**评估（Assistant）**：✅ 完美！总结得非常精准：

```
上下文管理的两个维度：

s04 Subagent → 节省"脏"上下文（过程垃圾不回传父 Agent）
s05 Skills   → 节省"闲"上下文（不用的知识不预加载）
s06 Compress → 节省"旧"上下文（已消化的信息降级或压缩）
```

三者共同构成了**完整的上下文经济学**：

| 策略 | 目标 | 时间点 |
|------|------|--------|
| s05 | 别加载不需要的 | **进入前** |
| s04 | 别让子任务的垃圾回来 | **产生时** |
| s06 | 把旧的清理掉 | **积累后** |

这就是 s01-s06（单 Agent 篇）的完整设计体系。

---

### 补充深入：Frontmatter 格式解析

**问（User）**：（要求深入）frontmatter 格式解析

**答（Assistant）**：看第 72-82 行的代码：

```python
def _parse_frontmatter(self, text: str) -> tuple:
    match = re.match(r"^---\n(.*?)\n---\n(.*)", text, re.DOTALL)
    if not match:
        return {}, text
    meta = {}
    for line in match.group(1).strip().splitlines():
        if ":" in line:
            key, val = line.split(":", 1)
            meta[key.strip()] = val.strip()
    return meta, match.group(2).strip()
```

#### 它在解析什么？

一个 SKILL.md 文件长这样：

```markdown
---
name: pdf
description: Process PDF files using Python libraries
tags: document, conversion
---
# PDF Processing Skill

Step 1: Install PyPDF2...
Step 2: Extract text...
```

`---` 之间的部分叫 **frontmatter**（元数据），`---` 之后的部分叫 **body**（正文）。

#### 逐行解析

| 代码 | 作用 |
|------|------|
| `re.match(r"^---\n(.*?)\n---\n(.*)", text, re.DOTALL)` | 用正则找出 `---` 之间的内容（group 1）和之后的内容（group 2） |
| `re.DOTALL` | 让 `.` 也匹配换行符，否则多行内容匹配不到 |
| `(.*?)` | `?` 是懒惰匹配 — 尽可能少匹配，不会贪心吃掉第二个 `---` |
| `line.split(":", 1)` | 按**第一个**冒号分割。`1` 表示只分一次，防止 description 里有冒号被切断 |

#### 解析结果

```python
# 输入文件内容
text = "---\nname: pdf\ndescription: Process PDF files\n---\nStep 1: ..."

# 解析后
meta = {"name": "pdf", "description": "Process PDF files"}
body = "Step 1: ..."
```

- `meta` → Layer 1 用，放进 system prompt（只有名字和描述）
- `body` → Layer 2 用，`load_skill` 被调用时返回完整正文

#### 为什么不用 YAML 库？

代码用手写解析，没有 `import yaml`。因为 frontmatter 格式很简单（只有 `key: value`），手写解析**零依赖**，不需要安装额外的库。这是工程上的实用选择 — 够用就行。
