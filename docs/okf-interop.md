# 11 — OKF 互操作：导入 / 导出 / Schema 对齐

Vega 是一个运行时知识库（SQLite + 原文 BLOB + 服务 + CLI）。[OKF](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) v0.1 是一个交换格式（Markdown 目录 + YAML frontmatter）。Vega 不替换内部存储模型，而是把 OKF 作为**一等交换格式**：整个知识库可以导出为合规 OKF bundle，任意 OKF bundle 可以导入为 Vega 知识库。

## 设计立场

| 立场 | 说明 |
|---|---|
| **内部存储不变** | metadata 在 SQLite，原文在文件系统。这是 Vega 的性能、事务、检索优势来源 |
| **OKF 是交换层** | Vega ↔ Vega、Vega ↔ 其他工具、Vega ↔ git 仓库，都走 OKF bundle |
| **Schema 对齐** | Vega metadata 字段命名 / 语义尽量贴合 OKF frontmatter，减少映射损失 |
| **扩展字段保真** | OKF 不支持的 Vega 语义（UUID、5 种关系类型、非 md 原文格式）用 `vega_*` 扩展字段保留，回导无损 |
| **容忍缺失** | 导入时容忍 OKF bundle 不合规字段、断链、未知 type（OKF §9 强制要求） |

## 字段映射

| Vega metadata（SQLite） | OKF frontmatter | 方向 | 说明 |
|---|---|---|---|
| `id` (UUID) | `vega_id` | 扩展 | OKF concept id 是文件路径；UUID 走扩展字段保留 |
| `title` | `title` | 直接 | |
| `summary` | `description` | 直接 | 语义一致：一句话摘要 |
| `tags` | `tags` | 直接 | YAML list |
| `type` | `type` | 直接 | **Vega 新增字段**，默认 `Document` |
| `resource_uri` | `resource` | 直接 | **Vega 新增字段**，可选，绑定外部 URI |
| `updated` | `timestamp` | 直接 | ISO 8601 datetime |
| `source_format` | `vega_source_format` | 扩展 | OKF body 是 md，原格式走扩展字段 |
| `file_path` | `vega_attachment` | 扩展 | 指向 bundle 内 `attachments/<uuid>.<ext>` |
| `status` | `vega_status` | 扩展 | `new` / `reviewed` / `archived` |
| `links`（5 种关系） | `vega_links` + body `# Relations` | 双写 | 见下 |

**前两行直接命名的字段**（`title` / `description` / `tags` / `type` / `resource` / `timestamp`）是 OKF §4.1 定义的标准字段，任何 OKF 消费者都能识别。`vega_*` 是 §4.1 明确允许的 producer-defined 扩展，其他 OKF 工具会按规范保留、不会拒绝。

## Concept ID 策略

OKF concept id = 文件路径去掉 `.md`。Vega UUID 是稳定主键，但人难读。导出默认采用**语义化路径**：

```
<bundle-root>/
├── <top-tag>/<slug>.md      # 若文档有 tag，按首个 tag 分目录
├── untagged/<slug>.md        # 无 tag 文档
```

`<slug>` 由 title 生成（lowercase、kebab-case、截断 60 字符）。冲突时追加 `-{short-uuid前 8 位}`。

`vega_id: <full-uuid>` 始终写在 frontmatter，是回导时的真实主键 —— concept id 仅是 OKF 世界的本地路径，不稳定、可被用户重命名。

## 关系类型保真

OKF 链接无类型（关系靠 prose）。Vega 的 5 种显式有向类型不能仅靠 markdown 链接表达，否则导入时丢语义。双写策略：

**frontmatter 结构化保真（机器可读）**：

```yaml
vega_links:
  related: [<vega_id>, <vega_id>]
  extends: [<vega_id>]
  contradicts: []
  supersedes: []
  references: [<vega_id>, <vega_id>]
```

**body 章节人类可读**：

```markdown
# Relations

- Extends: [FastAPI 中间件原理](/python/fastapi-middleware.md)
- References: [ASGI 规范](/specs/asgi.md)
```

导出时双写；导入时优先解析 `vega_links`（无损），缺失则回退到 body markdown 链接解析为 `references` 关系（损失类型）。

## 非 Markdown 原文处理

OKF concept 必须是 `.md`。Vega 原文可能是 pdf/docx/xlsx。策略：

```
<bundle-root>/
├── documents/
│   └── <slug>.md              # concept：frontmatter + summary + # Original 指针
└── attachments/
    └── <uuid>.pdf             # 原文文件原样保存
```

`<slug>.md` 的 frontmatter：

```yaml
---
type: Document
title: 2026 Q1 销售报告
description: 季度销售汇总，含区域拆分。
vega_id: 7c9f...
vega_source_format: pdf
vega_attachment: ../attachments/7c9f....pdf
resource: file://...            # 若 Vega 有 resource_uri 则放这
tags: [sales, quarterly]
timestamp: 2026-04-12T09:00:00Z
---
```

body：

```markdown
# Summary

（Vega 摘要内容）

# Original

原文为 PDF 格式，见 [附件](../attachments/7c9f....pdf)。
```

对于已经是 md 的原文，直接把 Vega 摘要拼 frontmatter 后作为 concept body，无需 attachments 副本。

## Bundle 顶层结构

导出整个 Vega 知识库为 OKF bundle：

```
<export-root>/
├── index.md                    # 根 index，OKF §6 格式，声明 okf_version
├── log.md                      # 可选，本次导出记录（不回导）
├── <top-tag>/                  # 按 tag 分组
│   ├── index.md                # 子目录 index
│   └── <slug>.md
├── untagged/
│   └── <slug>.md
└── attachments/                # 非 md 原文
    └── <uuid>.<ext>
```

根 `index.md` 示例：

```markdown
---
okf_version: "0.1"
vega_export:
  version: 0.1.0
  exported_at: 2026-06-22T10:00:00Z
  document_count: 142
---

# Vega Knowledge Bundle

# By Tag

* [python](python/) - Python 相关文档
* [sales](sales/) - 销售记录与季度报告

# Untagged

* [untagged](untagged/) - 未打标签文档
```

`okf_version` 是 OKF §11 规定的 bundle 声明字段，是 `index.md` 唯一允许的 frontmatter。`vega_export` 是扩展元数据，记录来源版本、时间、文档数。

## 导入流程

`vega import --okf <path>` 走标准 inbox 流水线（见 [04-inbox-workflow](./04-inbox-workflow.md)），不绕过审核：

```
扫描 <path> 下所有 .md
    │
    ├── index.md / log.md       → 解析为 bundle 元数据，不入库
    ├── <concept>.md            → 解析 frontmatter + body
    │
    ▼
对每个 concept：
    1. 提取标准字段 → Vega metadata（type / title / description / tags / timestamp）
    2. 提取 vega_* 扩展 → 还原 UUID / 关系 / 原文格式
    3. 若有 vega_attachment → 复制附件到 Vega 文件存储
    4. 若 body 含 # Summary 章节 → 作为 summary 候选
    5. status = 'new'，进入 inbox
    │
    ▼
按 vega_links 重建 links 表（若 vega_id 命中已有文档则更新；否则按 concept id 临时映射，全部入库后回填 UUID）
    │
    ▼
用户 vega review 逐条审核 / 或 --yolo 直接入库
```

**UUID 回填机制**：导入过程中，每个 concept 的 `vega_id`（若存在）直接作为 Vega 文档 UUID；不存在则生成新 UUID，并记一个 `concept_id_map.json` 在导入日志里，便于追溯。

**断链容忍**（OKF §5.3 强制）：bundle 内指向不存在 concept 的链接不视为错误，导入为 `references` 关系但 `target_id` 留空，待用户手动修复或后续导入补全。

## 导出流程

`vega export --okf <path>`：

```
1. 在 <path> 建空 bundle 骨架（index.md / log.md / attachments/）
2. 遍历 documents 表（默认仅 status='reviewed'，--include-inbox 带上 new）
3. 每篇文档：
    a. 选定 concept 路径（按 tag 分目录 / 无 tag 进 untagged/）
    b. 生成 frontmatter（标准字段 + vega_* 扩展）
    c. 拼 body（# Summary + # Original / md 原文 + # Relations）
    d. 非 md 原文复制到 attachments/
4. 按 concept 路径生成各层 index.md
5. 写根 index.md，含 okf_version 和 vega_export 元数据
6. 写 log.md，记本次导出（时间、文档数、过滤条件）
```

**路径冲突**：同名 slug 追加 `-{uuid 前 8 位}`，保证幂等。

**增量导出**（可选 `--incremental`）：对比上次导出的 `vega_export.exported_at`，仅写出 `updated > last_export` 的文档。当前版本不实现，预留配置位。

## CLI 命令

```
vega export --okf <path>
    [--include-inbox]              # 默认仅 reviewed；加上则含 new
    [--include-archived]           # 默认不含 archived
    [--id-strategy semantic|uuid]  # 默认 semantic（<tag>/<slug>.md）；uuid 用 documents/<uuid>.md
    [--no-attachments]             # 不导出原文，仅 metadata + summary

vega import --okf <path>
    [--yolo]                       # 跳过 inbox，直接 active
    [--merge-by uuid|title]        # uuid（默认）：vega_id 命中则更新；title：title 相同则更新
    [--allow-broken-links]         # 默认允许（OKF 强制）；显式关闭则断链报错
```

## API 路由

```
POST /api/export/okf              # body: { path, include_inbox?, include_archived?, id_strategy? }
                                  # 返回: { path, document_count, attachment_count, bytes }
POST /api/import/okf              # multipart: bundle path 或 zip
                                  # body: { yolo?, merge_by?, allow_broken_links? }
                                  # 返回: { imported: N, updated: N, skipped: N, broken_links: N }
GET  /api/export/okf/status/<id>  # 异步导出进度
```

导出/导入大 bundle 时走异步任务，立即返回 task id，客户端轮询 `/status/<id>`。

## 合规性

导出 bundle 自检 OKF §9 三条硬约束：

1. ✅ 每个非保留 `.md` 都有可解析 YAML frontmatter
2. ✅ 每个 frontmatter 都有非空 `type` 字段（Vega 强制写入，默认 `Document`）
3. ✅ `index.md` / `log.md` 遵循 §6 / §7 结构

Vega 导入器对未知 `type`、未知扩展字段、断链按 OKF §9 要求**MUST NOT** 拒绝 bundle。

## 吸收的 OKF 设计经验

Vega 借鉴 OKF 的几条设计原则，内化到 metadata schema：

| OKF 经验 | Vega 落地 |
|---|---|
| `type` 字段做路由 / 过滤 | metadata 新增 `type`，默认 `Document`，可按 type 筛选检索 |
| `resource` 绑定外部 URI | metadata 新增 `resource_uri`，把 Vega 文档和外部系统（Jira / GitHub issue / 网页 URL）关联 |
| `description` 是一句话摘要 | 与 Vega 已有 `summary` 字段语义统一，导入导出直接互译 |
| 绝对 bundle-relative 链接 | 导出时关系链接全部用 `/<path>.md` 绝对形式，文件在 bundle 内移动不影响 |
| `# Citations` 章节引用外部源 | 导入时 `# Citations` 里的 URL 自动转为 `references` 关系，target 存 `resource_uri` 占位文档 |
| `index.md` 渐进式披露 | 导出 bundle 每层目录都生成 index，Vega 自身 browse 命令可选 `vega browse` 渲染 |
| `log.md` 演进历史 | 启发 Vega 后续可加 `vega log` 命令读 SQLite 审计表 |
| 容忍未知字段 / type / 断链 | 导入器鲁棒性原则 |

## 不吸收的 OKF 立场

| OKF 立场 | Vega 保留立场 | 原因 |
|---|---|---|
| YAML frontmatter 是 metadata 唯一载体 | metadata 主存 SQLite | SQLite 支持 FTS5、事务、关系查询、索引；frontmatter 无法高效检索 |
| 一切皆 `.md` | 原文支持任意格式 | 「零信息损失」是 Vega 核心价值；pdf/docx 不应被强制转 md |
| 关系靠 prose，无类型 | 5 种显式有向关系 | 结构化关系是图谱查询（path / neighbors）的基础 |

## 参考

- [OKF v0.1 SPEC](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)
- [Vega 数据模型](./03-data-model.md)
- [Vega CLI 矩阵](./05-cli-commands.md)
