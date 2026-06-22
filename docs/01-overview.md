# 01 — 概述

Vega 是一个本地知识库应用，打包为单二进制。

- **默认结构化检索，规模化后可选语义召回** — 小规模时优先使用 metadata、摘要 FTS、文件名和关系图谱；数据量上来后，可启用 embedding 作为可删除、可重建的辅助索引
- **内置 AI Agent** — Core 支持 AI 对话，并在对话中动态更新 metadata 和文档关系
- **对外调用** — CLI 暴露 `get-metadata` / `get-summary` / `get-content` 等接口，供外部 coding agent 直接调用
- **OKF 一等交换格式** — 整个知识库可导出为合规 [OKF](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) v0.1 bundle，任意 OKF bundle 可导入；metadata schema 对齐 OKF frontmatter（`type` / `resource`），保证跨工具、跨组织、跨时间可移植。详见 [okf-interop](./okf-interop.md)

## 灵感来源

Vega 参考 [WeKnora](https://github.com/Tencent/WeKnora) 的 Core + CLI 分离架构，但不把 RAG/向量检索作为默认知识组织方式。Embedding 只作为规模化后的辅助召回层，不能替代原文、metadata、摘要和显式关系。
