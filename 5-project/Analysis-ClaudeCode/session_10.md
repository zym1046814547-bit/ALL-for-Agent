# Session 10：通信协议 — 请求-响应 + 优雅关闭

> 来源 Part: part_b_restructured.md, part_f_restructured.md  
> 前置：[Session 9：多 Agent 通信 — JSONL 邮箱](session_09.md)  
> 后续：[Session 11：去中心化调度 — 自主认领任务](session_11.md)

---

## 引入动机

s09 解决了"能通信"，但没解决"怎么协商"。如果 Alice 发消息说"帮我写测试"，Bob 应该直接开始做？还是先确认？如果 Bob 觉得做不了呢？Session 10 的 motto：

> *"Teammates need shared communication rules."*

## 核心概念 / 实现方式

Session 10 引入了一个**请求-响应协议**（request-response pattern）：

```
Alice → Bob: { type: "request", task: "写单元测试", id: "req_001" }
Bob → Alice: { type: "accept", id: "req_001" }       ← 接受
         或: { type: "reject", id: "req_001", reason: "..." }  ← 拒绝
         
...Bob 完成后...
Bob → Alice: { type: "done", id: "req_001", result: "..." }
```

就这一个协议模式，驱动所有的多 Agent 协商。项目还加了一个 **plan approval FSM（有限状态机）**：Agent 提出计划 → 其他 Agent 审批 → 批准后才执行。

## 讨论

**问（Assistant）**：Session 10 还引入了一个重要机制：**shutdown（优雅关闭）**。**为什么多 Agent 系统需要"优雅关闭"？直接 kill 所有 Agent 进程不行吗？** 想想 s07 里任务状态持久化在磁盘上这个前提。

**答（User）**：先更新所有 Agent 进行的 task 状态更新后再关闭。

**评估（Assistant）**：✅ 通过。核心就是**状态一致性**。

如果直接 kill：
- `tasks.json` 里某个任务状态还是 `in_progress`，但实际执行线程已死 → **幽灵任务**
- 下次重启，Agent 看到这个任务"正在进行中"，不知道该接着做还是重新做

优雅关闭的流程：
```
1. 通知所有 Agent："准备关闭"
2. 每个 Agent 完成或挂起当前工具调用
3. 将进行中的任务标记为 suspended 或 pending
4. 所有状态刷盘
5. 退出
```

## Memory Anchor

| 维度 | 内容 |
|------|------|
| **Essence** | 一个 request-response 协议统一所有 Agent 协商 |
| **Tension** | 无协议 → 消息含义模糊、无法拒绝、无法追踪；有协议 → 增加通信开销 |
| **Contrast** | s09 只解决"能通信"；s10 解决"怎么协商"和"怎么安全退出" |
| **Use** | plan approval FSM = 多 Agent 版的 TodoWrite 审批 |

---

## 延伸深入：request_id 关联 + 优雅关闭的完整解析（来自 Part F）

> Part F 对 Session 10 进行了深入讨论，展开了两个协议的完整代码细节和 shutdown 流程示例。

### request_id 关联模式

s10 在 s09 基础上加了两个**协议（protocol）**。核心就一个模式：

```
发起方生成 request_id → 发请求 → 等待 → 收到带相同 request_id 的回复 → 匹配上了
```

#### 协议 1：Shutdown（关闭 Teammate）

```
Lead: "Alice 请关闭"
  → request_id = "abc123"
  → shutdown_requests["abc123"] = {target: "alice", status: "pending"}
  → 发送 shutdown_request 到 alice.jsonl

Alice 收到 → 决定同意 → 调用 shutdown_response(request_id="abc123", approve=true)
  → shutdown_requests["abc123"].status = "approved"
  → 发消息给 Lead
  → should_exit = True → 退出循环 → status = "shutdown"
```

#### 协议 2：Plan Approval（计划审批）

```
Alice: "我打算重构 models.py"
  → request_id = "xyz789"
  → plan_requests["xyz789"] = {from: "alice", plan: "...", status: "pending"}
  → 发消息给 Lead 的信箱

Lead 收到 → 审批 → plan_approval(request_id="xyz789", approve=true, feedback="可以")
  → plan_requests["xyz789"].status = "approved"
  → 发回复给 Alice
```

**两个协议的唯一区别是方向**：
- Shutdown: Lead → Teammate → Lead（Lead 发起）
- Plan Approval: Teammate → Lead → Teammate（Teammate 发起）

### `should_exit` 的巧妙设计（第 183, 213–214, 188–189 行）

```python
should_exit = False

for _ in range(50):
    inbox = BUS.read_inbox(name)      # 先读信箱
    if should_exit:                    # 然后才检查是否退出
        break
    
    # ... 处理工具 ...
    if block.name == "shutdown_response" and block.input.get("approve"):
        should_exit = True            # 标记，但不立刻退出
```

**为什么不在 `approve` 的时候立刻 break？**

因为要先完成当前轮的工具处理（可能还有其他同轮的工具调用），回到循环顶部还要**最后读一次信箱**（可能有最后的消息需要处理），然后才干净地退出。这就是**优雅关闭（graceful shutdown）**。

s10 的新知识就这些：**request_id 关联** + **优雅关闭**。其他（MessageBus、TeammateManager、agent_loop）和 s09 完全一样。

### Shutdown 完整流程示例

**问（User）**：sample

**答（Assistant）**：

#### 第 1 步：Lead 发起关闭请求

Lead 调用 `shutdown_request(teammate="alice")`

```python
# handle_shutdown_request 执行：
req_id = "a1b2c3d4"
shutdown_requests["a1b2c3d4"] = {"target": "alice", "status": "pending"}
```

写入 `.team/inbox/alice.jsonl`：
```json
{"type":"shutdown_request","from":"lead","content":"Please shut down gracefully.","timestamp":1710000001,"request_id":"a1b2c3d4"}
```

#### 第 2 步：Alice 收到并回复

Alice 的循环读到信箱 → LLM 决定同意 → 调用 `shutdown_response`

```python
# _exec 里执行：
shutdown_requests["a1b2c3d4"]["status"] = "approved"    # tracker 更新
should_exit = True                                       # 标记退出
```

写入 `.team/inbox/lead.jsonl`：
```json
{"type":"shutdown_response","from":"alice","content":"Work saved, ready to shut down.","timestamp":1710000005,"request_id":"a1b2c3d4","approve":true}
```

#### 第 3 步：Alice 优雅退出

```python
# 回到循环顶部
inbox = BUS.read_inbox("alice")    # 最后读一次信箱
if should_exit:                     # True
    break                           # 退出循环

# 循环结束后
member["status"] = "shutdown"       # config.json 更新
```

`.team/config.json` 变化：
```diff
- {"name": "alice", "role": "coder", "status": "working"}
+ {"name": "alice", "role": "coder", "status": "shutdown"}
```

#### 第 4 步：Lead 得知结果

Lead 下一轮循环读到 `lead.jsonl` → 看到 Alice 说 "Work saved, ready to shut down." + `approve: true`。

Lead 也可以主动查：`shutdown_response(request_id="a1b2c3d4")` → 返回 `{"target":"alice","status":"approved"}`

**整个过程有 3 处状态变化：**

| 位置 | 变化 |
|------|------|
| `shutdown_requests` 字典 | `pending` → `approved` |
| `config.json` | `working` → `shutdown` |
| `alice.jsonl` / `lead.jsonl` | 消息传递的载体 |

### 讨论：拒绝后 Lead 会自动重试吗？

**问（User）**：If this round, Alice refuses, will Lead auto-request again when thinking appropriate?

**答（Assistant）**：**代码不会自动重试。** 看拒绝后发生什么：

```python
# Alice 拒绝
shutdown_response(request_id="a1b2c3d4", approve=False, reason="Still running tests")

# 结果：
shutdown_requests["a1b2c3d4"]["status"] = "rejected"
should_exit = False   # 不设 True → Alice 继续工作
```

Lead 收到回复后，是否再次请求完全取决于 **LLM 自己判断**。代码没有重试逻辑。

```
Lead 收到: "Alice rejected: Still running tests"
Lead (LLM 思考): "好的，她还在跑测试，等一下再说"
  → 可能过一会儿再发 shutdown_request（新的 request_id）
  → 也可能算了，不再关闭她
```

这和 s03 的设计思路一致 — **代码不做决策，LLM 做决策**。代码只提供机制（发请求、收回复），策略（何时重试）交给 LLM。
