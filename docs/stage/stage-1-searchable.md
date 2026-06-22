# Stage 1 — 能搜的文件箱

## 是什么 Vega

加了 metadata 字段和 FTS5 全文搜索。用户能用 tag / type 过滤，用关键词搜摘要和文件名。Vega 从「文件归档器」升级为「可检索的知识库」。

## 新增能力

- `documents` 表扩展：`type` / `description` / `tags` / `updated` 字段
- Alembic 迁移工具上线
- FTS5 虚拟表 + 同步触发器
- `vega ingest` 新增 `--title` / `--type` / `--tag`（可多次）/ `--description` 参数
- `vega search "<q>"` — FTS 查询，返回 id + title + tags
- `vega get-metadata <id>` — 返回完整 metadata JSON

## 前置依赖

[stage 0](./stage-0-min-box.md) 必须完成。

## 交付清单

### 数据库变更

引入 Alembic。首次迁移 `0001_init` 重建 `documents` 表（stage 0 已建则 ALTER 添加新列）：

```sql
-- documents 表最终形态（本 stage 结束时）
CREATE TABLE documents (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL DEFAULT 'Document',
    title TEXT NOT NULL,
    description TEXT,               -- 可选，一句话摘要（LLM 在 stage 4）
    tags TEXT NOT NULL DEFAULT '[]', -- JSON 数组
    source_format TEXT,             -- 扩展名（md/pdf/docx/...）
    file_path TEXT NOT NULL,
    created TEXT NOT NULL,
    updated TEXT NOT NULL
);
```

> `summary` / `status` / `resource_uri` 留给 stage 3 / 6 / 8。

### FTS5 索引

按 [06-database](../06-database.md) 规格建：

```sql
CREATE VIRTUAL TABLE documents_fts USING fts5(
    title, description, tags, filename,
    content='documents',
    content_rowid='rowid'
);
-- + AFTER INSERT/UPDATE/DELETE 触发器
```

### CLI 新增

```
vega ingest <file>
    [--title <t>]                   # 默认 = 文件名
    [--type <t>]                    # 默认 Document
    [--tag <tag>]                   # 可多次：--tag python --tag fastapi
    [--description <text>]          # 可选

vega search "<query>"
    [--tag <tag>]                   # 过滤
    [--type <type>]
    [--limit 20] [--offset 0]

vega get-metadata <id>              # JSON 输出
```

### 配置文件

引入 `~/.vega/config.yaml`，本 stage 仅含：

```yaml
storage:
  type: local
  path: ~/.vega/files
search:
  default_limit: 20
```

### 文件清单

- `vega/db.py` — 改造为 Alembic 配套
- `vega/migrations/` — Alembic 目录
- `vega/cli.py` — 新增 search / get-metadata，扩展 ingest 参数
- `vega/search.py` — FTS5 查询逻辑（本 stage 仅基础 BM25，无 time_decay / tag_weight）
- `tests/test_stage1.py`

## 验收标准

### 功能测试

```bash
# 1. 带完整 metadata 入库
vega ingest ./fastapi-doc.md \
    --title "FastAPI 入门" \
    --type Document \
    --tag python --tag fastapi \
    --description "FastAPI 框架入门教程"

# 2. 搜索命中
vega search "fastapi"
# 期望: 含上述文档

# 3. 按标签过滤
vega search "" --tag python
# 期望: 所有 python 标签文档

# 4. 按类型过滤
vega search "" --type Document

# 5. metadata JSON
vega get-metadata <id> | jq .
# 期望: 含 type/title/description/tags/source_format/created/updated

# 6. 无匹配
vega search "xyz123nonexistent"
# 期望: 空结果，退出码 0

# 7. 多关键词 AND
vega search "fastapi middleware"
# 期望: 同时含两词的文档
```

### 迁移测试

```bash
# 从 stage 0 升级：已有数据库
vega list                          # stage 0 数据可见
vega ingest ./new.md               # 新数据带新字段
vega get-metadata <stage-0-id>     # 老文档 type=Document, tags=[]
```

### 单元测试

- ingest 带 / 不带 metadata → 字段正确填充
- search 命中 title / description / tags / filename 四个字段
- FTS 触发器在 UPDATE 时同步索引
- source_format 从文件扩展名自动探测

## 不做的事

- ❌ FastAPI 服务（仍 CLI 直连 SQLite） — stage 2
- ❌ 摘要字段（`summary`） — stage 3
- ❌ LLM 自动 metadata 提取（必须 `--tag` / `--title` 手动） — stage 4
- ❌ 三因子排序（BM25 内置排序即可） — stage 9
- ❌ `resource_uri` 字段 — stage 8（OKF 对齐时）
- ❌ `status` 字段（所有文档视为 active） — stage 6

## 完成判定

用户能给文档打 tag、搜关键词、用 `--type` / `--tag` 过滤、`get-metadata` 看 JSON。Vega 现在是一个**可检索**的知识库，而不只是文件归档器。
