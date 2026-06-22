# Vega 实施路线图 — MVP 成长 Stage

Vega 按 **MVP pyramid** 拆分：每个 stage 结束时，用户都能下载、运行、得到真实价值。后一个 stage 不破坏前一个 stage 的功能，只是让 Vega 更强大。

每个 stage 文档统一结构：
- **是什么 Vega**：一句话定位这个 stage 的产品形态
- **新增能力**：相比上一个 stage 多了什么
- **前置依赖**：必须先完成哪个 stage
- **交付清单**：具体的代码/文件/行为
- **验收标准**：可运行的命令 + 期望行为 + 测试用例
- **不做的事**：本 stage 范围外，留给后续 stage

## Stage 索引

| Stage | 是什么 Vega | 新增关键能力 | 文档 |
|---|---|---|---|
| 0 | 能存能取的本地文件箱 | SQLite + 文件 BLOB + ingest/list/get | [stage-0](./stage-0-min-box.md) |
| 1 | 能搜的文件箱 | metadata 字段 + FTS5 + `search` | [stage-1](./stage-1-searchable.md) |
| 2 | 有服务的 Vega | FastAPI Core + CLI→HTTP + 自动拉起 | [stage-2](./stage-2-service.md) |
| 3 | 三层渐进的 Vega | 摘要层 + `get-metadata/summary/content` | [stage-3](./stage-3-progressive.md) |
| 4 | AI 增强的 Vega | LLM 自动摘要 / metadata 提取 | [stage-4](./stage-4-ai.md) |
| 5 | 有知识图谱的 Vega | links 表 + 5 种关系 + 图查询 | [stage-5](./stage-5-graph.md) |
| 6 | 有审核流水线的 Vega | inbox / review / approve / YOLO | [stage-6](./stage-6-inbox.md) |
| 7 | 能对话的 Vega | chat SSE + 会话 + 热更新 | [stage-7](./stage-7-chat.md) |
| 8 | 可互通的 Vega | OKF 导入 / 导出 + Schema 对齐 | [stage-8](./stage-8-okf.md) |
| 9 | 规模化的 Vega | 三因子排序 + deep_extract + S3 | [stage-9](./stage-9-scale.md) |
| 10 | 可分发的 Vega | PyInstaller 单二进制 + E2E | [stage-10](./stage-10-ship.md) |

## 依赖图

```
stage 0 (能存能取)
   │
   ▼
stage 1 (能搜) ─────────────┐
   │                         │
   ▼                         │
stage 2 (有服务)             │
   │                         │
   ▼                         │
stage 3 (三层渐进)           │
   │                         │
   ├─────────────┐           │
   ▼             ▼           │
stage 4 (AI)   stage 5 (图谱)│
   │             │           │
   └──────┬──────┘           │
          ▼                  │
stage 6 (审核) ◀─────────────┘
          │
          ▼
stage 7 (对话)
          │
          ▼
stage 8 (OKF 互通)
          │
          ▼
stage 9 (规模化)
          │
          ▼
stage 10 (打包分发)
```

stage 5（图谱）和 stage 4（AI）可以并行做，但都依赖 stage 3。stage 6 依赖前 5 个全部完成（审核流水线需要能操作 metadata、摘要、关系）。

## 时间估算

不做。每个 stage 的复杂度差异大，估算意义有限。看交付清单和验收标准自己判断。

## 参考规格

实施时对照设计规格：
- [01-overview](../01-overview.md) — 核心理念
- [02-architecture](../02-architecture.md) — 架构图 + 技术栈
- [03-data-model](../03-data-model.md) — 三层数据 + 关系图谱
- [04-inbox-workflow](../04-inbox-workflow.md) — 审核流水线
- [05-cli-commands](../05-cli-commands.md) — CLI 矩阵
- [06-database](../06-database.md) — SQLite 表结构 + FTS5
- [07-search](../07-search.md) — 排序算法
- [08-api-routes](../08-api-routes.md) — REST API
- [09-config](../09-config.md) — 配置
- [10-project-structure](../10-project-structure.md) — 目录
- [okf-interop](../okf-interop.md) — OKF 桥接
