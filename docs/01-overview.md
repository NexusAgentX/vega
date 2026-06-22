# 01 — 概述

Vega 是一个本地知识库应用，打包为单二进制。

## 核心理念

- **零信息损失** — 原文原样存储，不解析、不改写，不把向量当作原文替代
- **三层渐进加载** — metadata → 摘要 → 原文，按需拉取，尽量不浪费 context window
- **不依赖 YAML front matter** — metadata 独立存储在 SQLite 中，可由 CLI 参数提供，也可由 LLM 自动提取
- **默认结构化检索，规模化后可选语义召回** — 小规模时优先使用 metadata、摘要 FTS、文件名和关系图谱；数据量上来后，可启用 embedding 作为可删除、可重建的辅助索引
- **内置 AI Agent** — Core 支持 AI 对话，并在对话中动态更新 metadata 和文档关系
- **对外调用** — CLI 暴露 `get-metadata` / `get-summary` / `get-content` 等接口，供外部 coding agent 直接调用

## 灵感来源

Vega 参考 [WeKnora](https://github.com/Tencent/WeKnora) 的 Core + CLI 分离架构，但不把 RAG/向量检索作为默认知识组织方式。Embedding 只作为规模化后的辅助召回层，不能替代原文、metadata、摘要和显式关系。
