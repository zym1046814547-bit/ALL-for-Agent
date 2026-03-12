# 1. 感知层 (Perception)

## 本章导航

感知层负责把原始文档、网页、表格和结构化数据转成 Agent 可消费的表示形式。

{% hint style="info" %}
如果你的 Agent 要读财报、研究报告、制度文件或网页，这一章决定了后面检索和推理的上限。
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
            <td>1.1 文档解析</td>
            <td><a href="document-parsing/README.md">比较规则、视觉和 XBRL 解析路线</a></td>
        </tr>
        <tr>
            <td>1.2 分块策略</td>
            <td><a href="chunking/README.md">决定信息如何拆分、存储和重建</a></td>
        </tr>
        <tr>
            <td>1.3 索引构建</td>
            <td><a href="indexing/README.md">从 BM25 到混合检索的索引策略</a></td>
        </tr>
        <tr>
            <td>1.4 Schema 定义</td>
            <td><a href="schema/README.md">用强结构约束后续推理与抽取</a></td>
        </tr>
    </tbody>
</table>

## 学习目标

- 理解解析、分块、索引、Schema 的关系。
- 明确文档到知识对象的转换链路。
- 为后续 RAG、抽取和验证打基础。

## 重点专题

- [Docling](document-parsing/docling.md)
- [RAGFlow 分块实现](chunking/ragflow-chunking.md)

## 页面清单

- [1.1 文档解析](document-parsing/README.md)
- [1.2 分块策略](chunking/README.md)
- [1.3 索引构建](indexing/README.md)
- [1.4 Schema 定义](schema/README.md)
