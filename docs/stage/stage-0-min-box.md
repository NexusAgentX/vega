# Stage 0 — 最小文件箱

## 是什么 Vega

一个能存、能列、能取的本地文件归档器。没有搜索、没有服务、没有 LLM、没有关系、没有审核。这是 Vega 能称为「知识库」的最低门槛 —— 原文不丢，随时拿回。

## 新增能力

- SQLite 数据库初始化（`~/.vega/vega.db`）
- 文件存储初始化（`~/.vega/files/`）
- `vega ingest <file>` — 复制文件到存储，记一行 metadata
- `vega list` — 列出所有文档
- `vega get <id>` — 取回原文件到 stdout 或 `--output`

## 前置依赖

无。本 stage 是起点。

## 交付清单

### 项目骨架

- `pyproject.toml` — uv 管理，Python 3.12+，依赖：`typer`、`sqlmodel`、`pydantic`
- `vega/__init__.py`、`vega/__main__.py` — 包入口
- `vega/cli.py` — typer CLI，注册 `ingest` / `list` / `get` 三个命令
- `vega/db.py` — SQLite 连接 + 表初始化
- `vega/storage.py` — 本地文件存储（复制到 `~/.vega/files/<uuid>`）
- `tests/test_stage0.py` — 端到端测试

### 数据模型（极简）

`documents` 表仅 4 列：

```sql
CREATE TABLE documents (
    id TEXT PRIMARY KEY,        -- UUID
    title TEXT NOT NULL,        -- 默认 = 文件名（去扩展名）
    file_path TEXT NOT NULL,    -- ~/.vega/files/<uuid> 的绝对路径
    created TEXT NOT NULL       -- ISO 8601
);
```

本 stage 不引入 Alembic（schema 直接 `CREATE TABLE IF NOT EXISTS`，后续 stage 1 再上迁移工具）。

### CLI 行为

```
vega ingest <file>                     # 复制文件，打印新文档 ID
vega list [--limit 50]                 # 列出最近文档（id + title + created）
vega get <id> [--output <path>]        # 默认输出到 stdout；--output 写文件
```

无 `--tag`、`--title`、`--type` 等参数（留给 stage 1）。无服务（留给 stage 2）。

### 配置

固定路径 `~/.vega/`，无配置文件（留给 stage 1）。若 `~/.vega/` 不存在自动创建。

## 验收标准

### 功能测试（必须全过）

```bash
# 1. 首次运行，自动建库
vega ingest ./README.md
# 期望: 打印 UUID，如 "ingested: 7c9f..."

# 2. 列表
vega list
# 期望: 表格含刚才的文档

# 3. 取回（stdout）
vega get 7c9f... > /tmp/recovered.md
diff ./README.md /tmp/recovered.md && echo "OK"
# 期望: 无差异

# 4. 取回（写文件）
vega get 7c9f... --output /tmp/out.md
# 期望: 文件存在且内容一致

# 5. 二进制文件（pdf）
vega ingest ./paper.pdf
vega get <pdf-id> --output /tmp/out.pdf
diff ./paper.pdf /tmp/out.pdf && echo "OK"
# 期望: 二进制原样恢复
```

### 边界测试

```bash
# 6. 不存在的 ID
vega get nonexistent-id
# 期望: 错误退出码 != 0，stderr 打印 "document not found"

# 7. 重复入库同一文件 → 两个独立文档
vega ingest ./README.md
vega ingest ./README.md
vega list | wc -l
# 期望: 2 行（两次入库产生两个 UUID）
```

### 单元测试

```bash
pytest tests/test_stage0.py -v
```

至少覆盖：ingest 创建文件 + 行、list 返回正确数量、get 内容字节一致、不存在的 ID 抛错。

## 不做的事

- ❌ metadata 字段（tags / type / description / summary / resource_uri） — stage 1 / 3
- ❌ FTS5 全文索引 — stage 1
- ❌ search 命令 — stage 1
- ❌ FastAPI 服务 — stage 2
- ❌ CLI 自动拉起服务 — stage 2
- ❌ 摘要生成 — stage 3 / 4
- ❌ LLM 集成 — stage 4
- ❌ links 关系表 — stage 5
- ❌ inbox / status / 审核 — stage 6
- ❌ Alembic 迁移 — stage 1
- ❌ PyInstaller 打包 — stage 10

## 估算文件量

约 5 个源文件，300-500 行代码（含 CLI 框架 + 测试）。

## 完成判定

用户能跑 `uv run vega ingest <file>` → `vega list` → `vega get <id> > recovered` 三步，文件无损往返。这就是 Vega 的起点 —— **零信息损失**从 stage 0 就成立。
