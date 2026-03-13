# Session 2：多工具调度 — 循环不变，工具扩展

> 来源 Part: part_a_restructured.md  
> 前置：[Session 1：最小 Agent 循环 — `stop_reason` 驱动](session_01.md)  
> 后续：[Session 3：Plan 机制 — TodoWrite 引入](session_03.md)

---

## 引入动机

Session 1 只有一个工具（Bash）。但 Claude Code 有几十个工具（读文件、写文件、搜索、浏览器…）。一个自然的问题是：**加工具会不会让这个循环变复杂？**

## 核心洞察

> **"Adding a tool means adding one handler — the loop stays the same."**

加工具 = 往一个 dispatch map（字典）里注册一个函数。循环体**完全不变**。

```python
TOOL_HANDLERS = {
    "bash":       run_bash,       # s01 就有
    "read_file":  read_file,      # 新增
    "write_file": write_file,     # 新增
    "glob":       glob_search,    # 新增
}

# 循环里的这行代码，从来不用改：
output = TOOL_HANDLERS[block.name](**block.input)
```

## 讨论：dispatch map 的脆弱性

**问（Assistant）**：这个设计非常优雅，但有一个边界问题：如果 LLM 返回了一个 `block.name` 不在 `TOOL_HANDLERS` 字典里的工具名（比如 `"delete_database"`），会发生什么？这个最小循环有没有防御机制？这暴露了什么问题？

> 这个问题会自然地引出后面 Session 的一个重要主题。

**答（User）**：就会无法调用工具，没有防御机制，全靠模型能不能正确输出格式内容。

**评估（Assistant）**：✅ 没错！发现了这个最小循环的**第一个真实缺陷**。

准确地说，会有两种情况：
1. **KeyError 崩溃** — `TOOL_HANDLERS[block.name]` 直接报错，Agent 挂掉
2. **更隐蔽的风险** — 如果工具名正确但参数错误，可能执行出危险操作（比如 `bash` 里传了 `rm -rf /`）

这个循环**零防御**，完全依赖 LLM 的输出质量。这就引出了 Claude Code 逆向分析中的一个关键发现 ——

> Claude Code 有 **6 层权限验证**，从 UI 到工具执行形成完整安全链。

但在 `learn-claude-code` 这个项目里，防御能力是**后面的 Session 逐步加上的**，这也是为什么项目采用渐进式设计。

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | 加工具 = 加一个 handler，循环不变 |
| **Tension** | dispatch map 优雅但脆弱，没有权限/验证层 |
| **Contrast** | 和硬编码 if-else 比，字典 dispatch 是开放注册模式，扩展性好但安全性差 |
| **Use** | 理解为什么生产系统必须在循环之外加防护层 |
