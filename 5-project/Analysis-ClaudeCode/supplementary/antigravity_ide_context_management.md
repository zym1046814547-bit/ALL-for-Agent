# Antigravity IDE 逆向分析：Context 管理机制

> 来源 Part: part_g_restructured.md  
> 相关 Session: [Session 6：Context 压缩](../session_06.md), [Session 12（最终章）](../session_12.md)

---

## 引入动机

User 完成 s03-s12 全部学习后，自然产生了一个应用问题：教学代码里讲的 `auto_compact`、`TOKEN_LIMIT`，**在真实的 Antigravity IDE 里是怎么实现的？** 这从理论学习转向了实际产品的逆向工程。

---

## 搜索 compact 相关功能代码

**问（User）**：你可以找到 Antigravity IDE 中触发 compact 的相关功能代码吗？

**答（Assistant）**：（经过多轮搜索尝试）

1. **main.js** — 11MB 的 minified bundle，直接 grep 搜关键词没命中
2. **extension.js** — 3MB minified，搜索 `truncated due to` 没有结果
3. **language_server_macos_arm** — Go 编译的二进制，用 `strings` 命令搜索

最终在 `language_server_macos_arm` 二进制中找到了关键字符串：`CHECKPOINT` 和 `earlier parts`。

---

## 发现的关键参数

从二进制中提取的截断相关字符串：

```
tokenThreshold %d is greater than MaxTokenLimit %d    ← 两个参数的关系
soft limit: %d, hard limit: %d                         ← 两级阈值
generation reservation: %d                             ← 回复预留空间
system prompt tokens: %d                               ← 系统提示预留
trajectory_truncation_multiplier                       ← 截断量控制因子
```

还有模板字符串：`Reference the following log files for the full, untruncated conversation:`

---

## 问题 1：Untruncated conversation 在哪里？

**问（User）**：untruncated conversation 在哪里？

**答（Assistant）**：**存储在 Antigravity 后端服务器上，不在本地。**

从逆向分析看：
- 二进制中有字符串 `Reference the following log files for the full, untruncated conversation:` — 这是写给 **LLM 阅读的提示**，告诉 LLM 如何找到完整对话
- 本地只存储了：
  - `~/.gemini/antigravity/brain/<conversation-id>/.system_generated/steps/<N>/output.txt` — **每步工具调用的输出**，不是完整对话
  - `~/Library/Application Support/Antigravity/logs/` — **IDE extension 的运行日志**（报错信息、编码检测），不是对话内容

**完整对话可能存储在哪里：**
1. **Antigravity 云端** — 最可能。`language_server` 与 `exa.api_server_pb` 和 `exa.language_server_pb` 通信，对话历史（trajectory）存在服务器
2. **本地 `.system_generated/steps/` 目录** — 保存了工具调用的输出片段，但不是完整对话文本

---

## 问题 2：tokenThreshold 可以修改吗？

**问（User）**：tokenThreshold 可以配置修改吗？

**答（Assistant）**：**目前没有暴露给用户的配置项。**

证据：
- 搜索了 `~/Library/Application Support/Antigravity/User/settings.json` — 没有任何 token/checkpoint/context 相关设置
- 搜索了 `package.json` 的 `contributes.configuration` — 没有暴露相关设置
- 该参数在**编译好的 Go 二进制**（`language_server_macos_arm`）中，是硬编码或从服务端下发的配置

这些参数看起来是**服务端配置**（通过 gRPC 的 `LanguageServerService` 下发），不是客户端可调的。这和在 s06 学到的教学代码不同——教学代码里 `TOKEN_LIMIT` 是一个本地常量，而真实产品中这是一个后端控制的参数。

**这也是一个设计决策**：不让用户修改截断阈值，避免用户把阈值调得太高导致超出模型的 context window 限制（二进制中明确警告了这一点）。

---

## User 质疑：完整对话应该在本地

**问（User）**：所以无法看到 Antigravity 完整的上下文管理方法吗？我这里还能看到完整的上下文，我猜应该是存在本地的。

**答（Assistant）**：你说得对！如果你在 UI 里能滚动看到完整的对话历史，那数据一定在本地。

经过进一步搜索：

1. **`chat.ChatSessionStore.index`** — 空的（26 bytes）
2. **`antigravityUnifiedStateSync.trajectorySummaries`** — ~320KB 的 base64 编码 protobuf 数据，是 **trajectory summary 索引**，不是完整对话
3. **`language_server_macos_arm` 二进制** — 包含 `trajectory manager not initialized`、`set up implicit trajectory saver`、`failed to load trajectory %v: %v` 等字符串

关键发现：trajectory 系统处理所有对话存储。

---

## 存储位置确认

**问（User）**：打开完整对话保存的地方。

**答（Assistant）**：已在 Finder 中打开 `globalStorage` 目录。里面有 `state.vscdb` 文件。

这是一个 **SQLite 数据库**，可以用以下方式查看：

```bash
sqlite3 "~/Library/Application Support/Antigravity/User/globalStorage/state.vscdb" \
  "SELECT substr(value, 1, 2000) FROM ItemTable WHERE key = 'antigravityUnifiedStateSync.trajectorySummaries';"
```

也可以用 GUI 工具 [DB Browser for SQLite](https://sqlitebrowser.org/) 打开。

注意：`trajectorySummaries` 的 value 是 **base64 编码的 protobuf 二进制格式**，不是人类可读的 JSON。
