---
description: Agent 学习、设计、实现与评估的一体化知识库首页。
layout:
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
---

# 🎯 Agent All-in-One 手册

## 手册定位

这是一套面向 Agent 设计、实现、评估与项目落地的 GitBook 知识库骨架。内容按感知、大脑、工具、测评四层展开，并补充类型学、项目案例与附录索引。

{% hint style="info" %}
阅读建议：先从首页卡片进入一级章节，再在每个章节页继续下钻到 L2 和 L3 专题。
{% endhint %}

## 一级导航

<table data-view="cards">
    <thead>
        <tr>
            <th>章节</th>
            <th data-card-target data-type="content-ref">入口</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>0. 认知架构总论</td>
            <td><a href="0-agent-cognition/README.md">建立 Agent 总体认知与类型学</a></td>
        </tr>
        <tr>
            <td>1. 感知层</td>
            <td><a href="1-perception/README.md">从文档解析到 Schema 建模</a></td>
        </tr>
        <tr>
            <td>2. 大脑</td>
            <td><a href="2-brain/README.md">模型、Prompt、推理与编排</a></td>
        </tr>
        <tr>
            <td>3. 工具层</td>
            <td><a href="3-tools/README.md">RAG、抽取、验证与工具协议</a></td>
        </tr>
        <tr>
            <td>4. 测评层</td>
            <td><a href="4-evaluation/README.md">评估、Gate 与可观测性</a></td>
        </tr>
        <tr>
            <td>附录</td>
            <td><a href="appendix/README.md">术语、索引与版本记录</a></td>
        </tr>
    </tbody>
</table>

## 快速开始

- [0. Agent 认知架构总论](0-agent-cognition/README.md)
- [1. 感知层 (Perception)](1-perception/README.md)
- [2. 大脑 (Brain)](2-brain/README.md)
- [3. 工具层 (Tools)](3-tools/README.md)
- [4. 测评层 (Evaluation)](4-evaluation/README.md)
- [附录](appendix/README.md)

## 重点专题

<table data-view="cards">
    <thead>
        <tr>
            <th>专题类型</th>
            <th data-card-target data-type="content-ref">页面</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>项目深潜</td>
            <td><a href="1-perception/document-parsing/docling.md">Docling</a></td>
        </tr>
        <tr>
            <td>项目深潜</td>
            <td><a href="1-perception/chunking/ragflow-chunking.md">RAGFlow 分块实现</a></td>
        </tr>
        <tr>
            <td>项目深潜</td>
            <td><a href="2-brain/orchestration/langgraph-stategraph.md">LangGraph StateGraph</a></td>
        </tr>
        <tr>
            <td>项目深潜</td>
            <td><a href="2-brain/orchestration/crewai.md">CrewAI 编排</a></td>
        </tr>
        <tr>
            <td>项目深潜</td>
            <td><a href="3-tools/rag/ragflow.md">RAGFlow</a></td>
        </tr>
        <tr>
            <td>进阶专题</td>
            <td><a href="3-tools/tool-calling/mcp.md">MCP 协议</a></td>
        </tr>
    </tbody>
</table>

## 使用方式

1. 从左侧层级目录按主题进入。
2. 在每个页面右侧使用章节标题目录快速跳转。
3. 先阅读 L1 和 L2 导览页，再进入具体 L3 专题。

## 建议阅读路径

- 初次建立认知：从「认知架构总论」开始。
- 设计具体系统：依次阅读「感知层」「大脑」「工具层」。
- 做项目验收或回顾：阅读「测评层」与附录。
