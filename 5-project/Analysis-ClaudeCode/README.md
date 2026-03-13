# Agent 架构学习笔记 — Session 索引

> 来源项目：`learn-claude-code`  
> 学习方法：First Principles Learning Coach — Project Mode

---

## Session 一览

| Session | 标题 | Phase | 来源 Part | 文件 |
|---------|------|-------|-----------|------|
| 01 | 最小 Agent 循环 — stop_reason 驱动 | 1: Loop | A | [session_01.md](session_01.md) |
| 02 | 多工具调度 — 循环不变，工具扩展 | 1: Loop | A | [session_02.md](session_02.md) |
| 03 | Plan 机制 — TodoWrite 引入 | 2: Planning | A, **C**, **D** | [session_03.md](session_03.md) |
| 04 | Subagent — 隔离上下文 | 2: Planning | A, **D** | [session_04.md](session_04.md) |
| 05 | Skills — 按需加载知识 | 2: Knowledge | A, **E** | [session_05.md](session_05.md) |
| 06 | 上下文压缩 — 双层记忆架构 | 2: Knowledge | A, **D** | [session_06.md](session_06.md) |
| 07 | 文件级任务图 — 从内存到磁盘 | 3: Execution | B, **E** | [session_07.md](session_07.md) |
| 08 | 后台任务 — 非阻塞异步执行 | 3: Execution | B, **E**, **F** | [session_08.md](session_08.md) |
| 09 | 多 Agent 通信 — JSONL 邮箱 | 4: Teams | B, **F** | [session_09.md](session_09.md) |
| 10 | 通信协议 — 请求-响应 + 优雅关闭 | 4: Teams | B, **F** | [session_10.md](session_10.md) |
| 11 | 去中心化调度 — 自主认领任务 | 4: Teams | B, **F**, **G** | [session_11.md](session_11.md) |
| 12 | Worktree 隔离 — 文件系统级隔离（最终章） | 4: Teams | B, **C**, **G** | [session_12.md](session_12.md) |

## 非 Session 内容归属

| 内容 | 归入 | 来源 Part | 规则 |
|------|------|-----------|------|
| 学习背景 / 项目总览 / 起点诊断 | session_01.md 开头 | A | 前置背景归入首个 Session |
| 贯穿 Session 1–6 的核心模式总结 | session_06.md 末尾 | A | 范围总结归入范围内最后一个 Session |
| Phase 1-2 完成回顾 | session_07.md 开头 | B | Phase 回顾归入下一阶段首个 Session |
| Phase 3 完成过渡 | session_08.md 末尾 | B | Phase 回顾归入该 Phase 最后一个 Session |
| 跨章节讨论：传统工程直觉 vs LLM 原生设计 | session_08.md 内 | B | 内容关联 Session 8 的 role:user 讨论 |
| 12 Session 全景回顾 | session_12.md 末尾 | B | 全局回顾归入最终 Session |
| 贯穿 Part B 核心模式总结 | session_12.md 末尾 | B | 范围总结归入范围内最后一个 Session |
| 最终 Memory Anchor + 后续方向 | session_12.md 末尾 | **C** | 全局总结归入最终 Session |
| 深入学习背景 + ROI 分析 | session_03.md | **C** | 内容直接引导进入 s03 深入 |
| Session 3 深入：三层控制 + 代码实现 | session_03.md | **C** | 直接扩展 Session 3 |
| 延伸讨论：三层控制辨析 | session_03.md | **C** | 关联 Session 3 的控制机制 |
| 延伸讨论：Hook 模式 | session_03.md | **C** | 关联 Session 3 的工程实现 |
| 延伸讨论：Python 语法（lambda/**kwargs） | session_03.md | **C** | 关联 Session 3 代码阅读 |
| 延伸讨论：read_file 的 limit 参数 | session_03.md | **C** | 关联 Session 3 工具实现 |
| 延伸讨论：LLM 如何写文件 | session_03.md | **C** | 关联 Session 3 工具实现 |
| 贯穿 Part C 核心模式总结 | session_03.md 末尾 | **C** | Part C 全部围绕 Session 3 |
| 延伸讨论：write_file vs edit_file 工作原理 | session_03.md | **D** | 关联 Session 3 工具实现 |
| 延伸讨论：System Prompt 稀释效应 | session_03.md | **D** | 关联 Session 3 的 nag reminder 深层原因 |
| Session 6 深入：三层压缩代码解析 | session_06.md | **D** | 直接扩展 Session 6 |
| 延伸讨论：Round/Turn/Message 定义 | session_06.md | **D** | 关联 Session 6 的"每轮"概念 |
| 延伸讨论："重大内容"含义 + 学习路径 | session_06.md | **D** | s06 深入后的选路讨论 |
| 贯穿 Part D 核心模式总结 | session_06.md 末尾 | **D** | 主题围绕 s06 信息管理原则 |
| Session 4 深入：Subagent 隔离 + Worktree | session_04.md | **D** | 直接扩展 Session 4 |
| Session 5 深入：代码级实现 + 上下文经济学 | session_05.md | **E** | 直接扩展 Session 5 |
| Session 7 深入：双向同步 + JSON 完整演练 | session_07.md | **E** | 直接扩展 Session 7 |
| Session 8 深入：基础场景 + User 三问 | session_08.md | **E** | 直接扩展 Session 8 |
| 贯穿 Part E 核心模式总结 | session_08.md 末尾 | **E** | Part E 最后一个 Session 为 s08 |
| Session 8 深入：Lock + drain_notifications | session_08.md | **F** | 直接扩展 Session 8 |
| Session 8 深入：message 交替防御性设计 | session_08.md | **F** | 直接扩展 Session 8 |
| 贯穿 Part F 防御性设计模式总结 | session_08.md 末尾 | **F** | Part F 主要扩展 s08 |
| Session 9 深入：Subagent vs Teammate + JSONL + 通信 | session_09.md | **F** | 直接扩展 Session 9 |
| User 发现的代码缺陷 + 跨 Session 模式映射 | session_09.md 末尾 | **F** | 范围总结归入 s09 |
| Session 10 深入：request_id + should_exit + Shutdown 示例 | session_10.md | **F** | 直接扩展 Session 10 |
| Session 11 开端问题 | session_11.md | **F** | 直接扩展 Session 11 |
| Session 11 深入：Identity 重注入 7 问追问 | session_11.md | **G** | s11 的 make_identity_block 深度解析 |
| 教学代码 Bug 总结 + User 质疑模式 | session_11.md 末尾 | **G** | s11 相关 bug 和学习反思 |
| Session 12 深入：控制面 vs 执行面 + EventBus | session_12.md | **G** | 直接扩展 Session 12 |
| 延伸讨论：上下文估算 | session_12.md 末尾 | **G** | s12 完成后的 token 估算讨论 |
| 延伸讨论：从教学代码到真实产品 | session_12.md 末尾 | **G** | s12 后的对比分析表 |

## supplementary/

| 文件 | 主题 | 来源 Part |
|------|------|----------|
| [antigravity_ide_context_management.md](supplementary/antigravity_ide_context_management.md) | Antigravity IDE 逆向分析：Context 管理机制 | **G** |

