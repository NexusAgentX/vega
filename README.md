# Vega

> 以原文 + 独立 metadata 为核心、默认不用 RAG/embedding、规模化后可选语义召回的**三层渐进式本地知识库**。单二进制，内置 AI Agent，CLI 对外供 coding agent 调用，OKF 一等交换格式。

**状态：设计中** — 待终审后进入实现阶段。当前仓库仅包含设计规格文档。

---

## 为什么造

现有 AI/LLM 知识库工具默认把 embedding 当作知识的载体，原文被切片、向量化、丢弃。这丢了三件事：

1. **原文本身** — 向量不是原文，无法复核、无法精读
2. **结构化 metadata** — tag/关系/类型被打平进语义空间，无法精确过滤
3. **可移植性** — 知识被锁进某个 vector store，跨工具、跨组织迁移成本极高

Vega 立场：**原文 BLOB 零改写**，metadata 独立存 SQLite 可检索可事务，embedding 仅作规模化后的**可选**辅助召回层（可删除、可重建）。整个知识库可一键导出为 [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) bundle，跨工具/组织/时间可移植。

## 七大设计支柱

| # | 支柱 | 详见 |
|---|---|---|
| 1 | 三层渐进加载 — metadata → 摘要 → 原文 | [03-data-model](./docs/03-data-model.md) |
| 2 | 零信息损失 — 原文 BLOB，不解析不改写 | [01-overview](./docs/01-overview.md) |
| 3 | Metadata 优先 — 结构化标签 + 摘要 FTS + 文件名 + 关系图谱 | [03-data-model](./docs/03-data-model.md) |
| 4 | 双链知识图谱 — 五种关系类型 + 图遍历 | [03-data-model](./docs/03-data-model.md) |
| 5 | Inbox 审核 — 默认人工审阅 / YOLO 直入 | [04-inbox-workflow](./docs/04-inbox-workflow.md) |
| 6 | Core + CLI 双模 — 服务常驻 + CLI 按需查询 | [02-architecture](./docs/02-architecture.md) |
| 7 | **OKF 一等交换格式** — 导入导出 + Schema 对齐 | [okf-interop](./docs/okf-interop.md) |

## 架构

```
┌─────────────────────────────────────────────────────┐
│                    vega (单二进制)                     │
│                                                       │
│  ┌──────────────┐        ┌─────────────────────────┐ │
│  │    CLI 层     │  HTTP  │       Core 层            │ │
│  │              │ ──────▶│                          │ │
│  │ vega serve   │        │  FastAPI Server          │ │
│  │ vega query   │        │  ├─ REST API             │ │
│  │ vega ingest  │        │  ├─ Chat / Agent         │ │
│  │ vega link    │        │  ├─ 文档关系引擎          │ │
│  │ vega review  │        │  ├─ OKF 桥接层            │ │
│  │ vega export  │        │  └─ LLM Bridge           │ │
│  │ vega import  │        │         │                │ │
│  └──────────────┘        │    SQLite                 │ │
│                          └─────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

## 技术栈

Python 3.12+ · uv · FastAPI · SQLModel · SQLite + FTS5 · typer · httpx · litellm · PyInstaller

## 三层数据模型

| 层 | 内容 | 加载时机 |
|---|---|---|
| **Metadata** | type / title / description / tags / resource_uri / 5 种关系 | 始终加载，极轻量 |
| **摘要** | LLM 生成的 ~200 token 总结 | 按需加载 |
| **原文** | 完整原始文件（md/pdf/docx/任意格式） | 显式 `get-content` 才拉 |

## OKF 互操作

Vega 内部存储保持 SQLite + 文件 BLOB（性能 / 事务 / 检索优势），[OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) 作为一等交换格式：

```bash
# 导出整个知识库为合规 OKF bundle
vega export --okf ./my-bundle/

# 导入任意 OKF bundle（走 inbox 审核流水线）
vega import --okf ./their-bundle/
```

字段映射、concept id 策略、关系类型保真、非 md 原文处理、合规性自检详见 [docs/okf-interop.md](./docs/okf-interop.md)。

## 阅读规格

从 [docs/INDEX.md](./docs/INDEX.md) 开始，有分级阅读路径（快速了解 / 深入设计 / 实现细节 / 工程落地）。

## 灵感来源

- [WeKnora](https://github.com/Tencent/WeKnora) — Core + CLI 分离架构（但不采用其 RAG 为默认知识组织方式）
- [Open Knowledge Format v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) — 一等交换格式

## License

待定。
