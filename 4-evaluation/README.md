# 4. 测评层 (Evaluation)

## 本章导航

测评层负责定义数据、指标、Gate 和可观测性，用于保证 Agent 的稳定性与可演进性。

{% hint style="info" %}
这一章负责回答一个更难的问题：Agent 不是“能跑”就行，而是要知道它什么时候可靠，什么时候该拦截。
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
            <td>4.1 评估数据构建</td>
            <td><a href="evaluation-data-construction.md">定义样本、难例和标注规范</a></td>
        </tr>
        <tr>
            <td>4.2 RAG Metric</td>
            <td><a href="rag-metrics/README.md">集中整理 RAG 指标与评测框架</a></td>
        </tr>
        <tr>
            <td>4.3 Gate 系统</td>
            <td><a href="gate-system-finsight-original.md">预留 FinSight 原创测评机制</a></td>
        </tr>
        <tr>
            <td>4.4 可观测性</td>
            <td><a href="observability.md">从日志到 Trace 的运行态观测</a></td>
        </tr>
    </tbody>
</table>

## 学习目标

- 建立从离线评估到在线观测的全链路测评视角。
- 明确 RAG 与 Agent 的常见指标体系。
- 为 FinSight 的原创 Gate 系统预留结构化位置。

## 重点专题

- [Ragas / DeepEval](rag-metrics/ragas-deepeval.md)
- [Gate 系统 [FinSight 原创]](gate-system-finsight-original.md)

## 页面清单

- [4.1 评估数据构建](evaluation-data-construction.md)
- [4.2 RAG Metric](rag-metrics/README.md)
- [4.3 Gate 系统 [FinSight 原创]](gate-system-finsight-original.md)
- [4.4 可观测性](observability.md)
