# 03 — 三层数据模型 + 关系图谱

## 第一层：Metadata

独立存储在 SQLite 中的结构化元数据，始终加载，极轻量。来源为 CLI 参数或 LLM 自动提取。

```json
{
  "id": "doc-001",
  "type": "Document",
  "title": "FastAPI 中间件实现原理",
  "description": "FastAPI 中间件机制与 ASGI 请求生命周期的深度解析。",
  "tags": ["python", "fastapi", "middleware"],
  "status": "reviewed",
  "source_format": "md",
  "resource_uri": null,
  "created": "2026-05-20",
  "updated": "2026-05-22",
  "links": {
    "related": ["doc-042", "doc-107"],
    "extends": ["doc-003"],
    "contradicts": [],
    "supersedes": [],
    "references": ["doc-015", "doc-088"]
  }
}
```

**字段说明**：

| 字段 | 必填 | 默认值 | 来源 / 用途 |
|---|---|---|---|
| `id` | 是 | UUID | 主键 |
| `type` | 是 | `Document` | 概念类型，对齐 OKF `type`；用于路由 / 过滤 / 呈现（如 `Document` / `Playbook` / `Reference` / `Metric`） |
| `title` | 否 | 文件名推导 | 展示名 |
| `description` | 否 | LLM 生成 | 一句话摘要，对齐 OKF `description`；与 `summary` 语义统一 |
| `tags` | 否 | LLM 提取 | 跨维度分类，扁平无层级 |
| `resource_uri` | 否 | null | 外部资源 URI（Jira issue / GitHub PR / 网页 URL），对齐 OKF `resource` |
| `summary` | 否 | LLM 生成 | 200 token 摘要，导出 OKF 时写入 `description` 字段 |
| `source_format` | 是 | 自动探测 | 原文扩展名（md/docx/pdf/...） |
| `status` | 是 | `new` | `new` / `reviewed` / `archived` |
| `links` | 否 | 空 | 5 种有向关系图谱，见下文 |

> **OKF 对齐**：`type` / `title` / `description` / `tags` / `resource_uri` / `updated` 六个字段与 OKF v0.1 frontmatter 标准字段一一映射，保证导入导出无损。详见 [okf-interop](./okf-interop.md)。

## 第二层：摘要

LLM 生成的文档摘要，约 200 token，按需加载。

## 第三层：原文

完整原始文件，本地或 S3 存储，零解析、零改写。

## 对外接口映射（三层渐进加载）

| CLI 命令 | 加载层 | 返回内容 |
|---|---|---|
| `vega search "<q>" [--tag xy]` | Metadata + 摘要 FTS | 匹配的文档 ID + title + tags 列表 |
| `vega get-metadata <id>` | Metadata | 单文档完整 metadata（JSON） |
| `vega get-summary <id>` | 摘要 | 200 token 内 AI 生成摘要 |
| `vega get-content <id>` | 原文 | 完整原始文件 |

搜索范围：所有 metadata 字段 + 摘要全文索引（SQLite FTS） + 文件名，三合一。

## 文档关系类型

双链 / 知识图谱模型：文档是节点，关系是有向边。

| 类型 | 含义 | 示例 |
|---|---|---|
| `related` | 泛关联 | 两篇讲同类话题 |
| `extends` | 继承/扩展 | B 是在 A 基础上继续深入 |
| `contradicts` | 矛盾/冲突 | 两篇对同一问题给出相反结论 |
| `supersedes` | 替代 | B 完全替代 A（旧版本、废弃方案） |
| `references` | 引用 | 文中引用了另一篇文档 |

## 图查询命令

| 命令 | 说明 |
|---|---|
| `vega links <id>` | 出边：此文引用了谁 |
| `vega backlinks <id>` | 入边：谁引用了此文 |
| `vega neighbors <id>` | 出边 + 入边合并 |
| `vega path <a> <b>` | 两文档间最短路径 |