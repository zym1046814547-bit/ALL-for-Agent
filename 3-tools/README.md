# 3. 工具层 (Tools)

## 本章导航

工具层负责把检索、抽取、验证、执行与外部协议接到 Agent 推理回路里。

{% hint style="info" %}
如果说大脑负责决定“怎么想”，工具层负责决定“能做什么”。这里直接影响 Agent 的行动半径。
{% endhint %}

## 章节入口

<table data-view="cards">
    <thead>
        <tr>
            <th>模块</th>
            <th data-card-target data-type="content-ref">说明</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>3.1 信息检索 (RAG)</td>
            <td><a href="rag/README.md">从基础 RAG 到图增强检索</a></td>
        </tr>
        <tr>
            <td>3.2 结构化提取</td>
            <td><a href="structured-extraction/README.md">把非结构化内容稳定转成字段对象</a></td>
        </tr>
        <tr>
            <td>3.3 验证与回查</td>
            <td><a href="verification-and-backcheck.md">补齐事实核验和来源回查链路</a></td>
        </tr>
        <tr>
            <td>3.4 计算与代码执行</td>
            <td><a href="computing-and-code-execution.md">引入计算器、解释器和沙箱</a></td>
        </tr>
        <tr>
            <td>3.5 Tool-Calling 协议</td>
            <td><a href="tool-calling/README.md">理解工具描述、参数与调用边界</a></td>
        </tr>
        <tr>
            <td>3.6 外部信息获取</td>
            <td><a href="external-information-access.md">网页、数据库和 API 信息接入</a></td>
        </tr>
    </tbody>
</table>

## 学习目标

- 建立常见工具能力地图。
- 明确工具调用和模型推理的分工。
- 为系统集成和工程化做准备。

## 重点专题

- [RAGFlow](rag/ragflow.md)
- [Instructor](structured-extraction/instructor.md)
- [SGLang FSM](structured-extraction/sglang-fsm.md)
- [MCP 协议](tool-calling/mcp.md)

## 页面清单

- [3.1 信息检索 (RAG)](rag/README.md)
- [3.2 结构化提取](structured-extraction/README.md)
- [3.3 验证与回查](verification-and-backcheck.md)
- [3.4 计算与代码执行](computing-and-code-execution.md)
- [3.5 Tool-Calling 协议](tool-calling/README.md)
- [3.6 外部信息获取](external-information-access.md)
