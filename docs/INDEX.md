# Vega 设计规格 — 渐进式索引导读

**日期**: 2026-05-24
**版本**: v0.1.0

---

## 一句话概括

Vega 是一个**以原文 + 独立 metadata 为核心、默认不用 RAG/embedding、规模化后可选语义召回的三层渐进式知识库**，分装为单一二进制，内置 AI Agent，CLI 对外供 coding agent 调用。

## 六大设计支柱

| # | 支柱 | 详见 |
|---|---|---|
| 1 | 三层渐进加载 — metadata → 摘要 → 原文 | [03-data-model](./03-data-model.md) |
| 2 | 零信息损失 — 原文 BLOB，不解析不改写 | [01-overview](./01-overview.md) |
| 3 | Metadata 优先 — 默认用结构化标签、摘要 FTS 和文件名检索，embedding 仅作规模化后的可选辅助召回 | [03-data-model](./03-data-model.md) |
| 4 | 双链知识图谱 — 五关系类型 + 图遍历 | [03-data-model](./03-data-model.md) |
| 5 | Inbox 审核 — 默认人工审阅 / YOLO 直入 | [04-inbox-workflow](./04-inbox-workflow.md) |
| 6 | Core + CLI 双模 — 服务常驻 + CLI 按需查询 | [02-architecture](./02-architecture.md) |

## 阅读路径

### 快速了解（~5 分钟）
1. [01-overview](./01-overview.md) — 是什么、为什么、不做的事
2. [02-architecture](./02-architecture.md) — 架构图 + 技术栈

### 深入设计（~15 分钟）
3. [03-data-model](./03-data-model.md) — 三层数据 + 关系图谱
4. [04-inbox-workflow](./04-inbox-workflow.md) — 审核流水线
5. [05-cli-commands](./05-cli-commands.md) — 命令矩阵

### 实现细节（~20 分钟）
6. [06-database](./06-database.md) — SQLite 表结构 + FTS5
7. [07-search](./07-search.md) — 排序算法
8. [08-api-routes](./08-api-routes.md) — REST API
9. [09-config](./09-config.md) — 配置 + LLM + 降级

### 工程落地
10. [10-project-structure](./10-project-structure.md) — 目录 + 约束

## 状态

设计中 — 待用户终审后进入实现计划阶段。
