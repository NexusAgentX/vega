# Stage 9 — 规模化的 Vega

## 是什么 Vega

知识库规模上来后的三件事：精细化排序、原文纯文本提取入 FTS（deep_extract）、S3 兼容远程存储。Vega 从「玩具规模（百级文档）」升级为「生产规模（万级文档）」。

## 新增能力

- 三因子加权排序：`BM25 × time_decay × tag_weight`（按 [07-search](../07-search.md) 规格）
- `search.deep_extract = true` 时入库调用解析库从原文提取纯文本写入 FTS（原文 BLOB 不变）
- S3 兼容存储后端（MinIO / AWS S3 / 阿里 OSS / 腾讯 COS）

## 前置依赖

[stage 8](./stage-8-okf.md) 必须完成。

## 交付清单

### 三因子排序

`vega/core/engine/ranking.py`：

```python
def score(doc, query, tag_weights, half_life_days):
    bm25_score = fts5_bm25(doc.rowid, query)         # SQLite 内置
    time_decay = 0.5 ** (days_since(doc.updated) / half_life_days)
    tag_weight = product(tag_weights.get(t, 1.0) for t in doc.tags)
    return bm25_score * time_decay * tag_weight
```

实现细节：
- `bm25()`：SQLite FTS5 内置函数
- `time_decay`：默认半衰期 90 天，配置可调
- `tag_weight`：默认 `{}`（不影响），用户可为特定 tag 配置权重（升权 / 降权）
- 排序在 Python 层做（FTS5 返回 BM25 raw score，后续两因子后处理）

> 注意：BM25 raw score 可能为负（FTS5 实现细节），需在排序前规范化或处理。

### Deep Extract

`vega/core/engine/extractor.py`：

| 文件类型 | 解析库 | 提取策略 |
|---|---|---|
| `.md` / `.txt` / `.markdown` | 无 | 直接读 utf-8 |
| `.pdf` | `pypdf` 或 `pdfplumber` | 逐页提取文本 |
| `.docx` | `python-docx` | 段落 + 表格文本 |
| `.xlsx` | `openpyxl` | 逐 sheet 逐行 |
| `.pptx` | `python-pptx` | 逐 slide 逐 shape |
| `.html` / `.htm` | `beautifulsoup4` | `<body>` 内文本 |
| `.epub` | `ebooklib` | 章节 + 段落 |
| 其他 | 无 | 不提取（FTS 仅索引 metadata） |

提取出的纯文本写入 FTS 的 `content_text` 列（需 ALTER documents_fts 增加该列）。原文 BLOB **绝不修改**。

Alembic 迁移 `0007_add_fts_content`：

```sql
-- 重建 FTS 表增加 content_text 列
DROP TABLE documents_fts;
CREATE VIRTUAL TABLE documents_fts USING fts5(
    title, description, tags, summary, filename, content_text,
    content='documents',
    content_rowid='rowid'
);
-- 触发器同步重建
```

已入库文档需要 `vega reindex` 重新提取（手动触发，不自动跑）。

### S3 存储

`vega/core/storage/s3.py`：

- 用 `boto3` 或 `aiobotocore`（后者支持 async）
- 兼容 S3 协议：AWS S3 / MinIO / 阿里 OSS / 腾讯 COS / Cloudflare R2
- 配置：

```yaml
storage:
  type: s3                           # local | s3
  s3:
    endpoint: https://s3.amazonaws.com
    bucket: vega-kb
    access_key: ${AWS_ACCESS_KEY}
    secret_key: ${AWS_SECRET_KEY}
    region: us-east-1
```

- LocalStorage 和 S3Storage 实现同一接口（`StorageBackend` 抽象基类），切换无感
- 文件路径：`vega://<uuid>`（逻辑路径），底层映射到 `~/.vega/files/<uuid>` 或 `s3://bucket/<uuid>`

### CLI 新增

```
vega reindex                        # 重新 deep_extract 所有已入库文档
vega reindex --since <date>         # 仅 reindex 某日期后更新的
vega reindex --dry-run              # 仅列出会处理的文档
```

### 配置扩展

```yaml
search:
  deep_extract: false               # 入库时是否自动 deep_extract（默认关，性能考虑）
  default_limit: 20
  ranking: bm25                     # 保留位（未来可能加 tfidf / vector）
  time_decay:
    enabled: true
    half_life_days: 90
  tag_weight:
    enabled: true
    weights:
      python: 2.0
      deprecated: 0.3
```

### 文件清单

- `vega/core/engine/ranking.py` — 三因子排序
- `vega/core/engine/extractor.py` — 文本提取
- `vega/core/storage/base.py` — `StorageBackend` 抽象
- `vega/core/storage/local.py` — 改造为继承抽象
- `vega/core/storage/s3.py` — 新建
- `vega/cli/commands/reindex.py`
- `vega/migrations/0007_add_fts_content.py`
- `tests/test_stage9_ranking.py`
- `tests/test_stage9_extractor.py`
- `tests/test_stage9_s3.py`（用 moto mock 或本地 MinIO）

## 验收标准

### 三因子排序

```bash
# 1. 时间衰减
vega ingest ./old.md --yolo; sleep 1
# 手动改 updated 为 1 年前
vega search "test"
# 期望: old.md 排在最近入库的同分文档之后

# 2. Tag 权重
# config.yaml: search.tag_weight.weights.python: 2.0
vega ingest ./a-python.md --tag python --yolo
vega ingest ./b-other.md --tag other --yolo
vega search "<both-match>" 
# 期望: a-python 排在 b-other 之前

# 3. 降权
# config: deprecated: 0.3
vega ingest ./old-approach.md --tag deprecated --yolo
vega search "<match>"
# 期望: old-approach 排在末尾
```

### Deep Extract

```bash
# 4. PDF 文本进入 FTS
# config: search.deep_extract: true
vega ingest ./paper.pdf --yolo
vega search "论文中的特殊关键词"
# 期望: 命中（说明 PDF 正文进了 FTS）

# 5. 原文未变
vega get-content <pdf-id> --output /tmp/out.pdf
diff ./paper.pdf /tmp/out.pdf && echo "OK"

# 6. reindex 旧文档
vega reindex --since 2026-01-01
# 期望: 列出处理数量 + 实际重建索引
```

### S3 存储

```bash
# 7. 切换 S3 后端（本地 MinIO）
# 启动 MinIO: docker run -p 9000:9000 minio/minio server /data
# config: storage.type: s3, endpoint: http://localhost:9000
vega ingest ./doc.md --yolo
aws --endpoint-url http://localhost:9000 s3 ls s3://vega-kb/
# 期望: 含 <uuid> 文件

vega get-content <id> --output /tmp/out.md
# 期望: 正确下载

# 8. 从 local 迁移到 s3
# 提供迁移命令（或文档说明手动步骤）
```

### 性能基线（非硬性）

```bash
# 9. 万级文档搜索延迟
# 准备: 批量 ingest 10000 篇随机 md
time vega search "random-term" --limit 20
# 期望: < 500ms（不含网络往返）

# 10. deep_extract 性能
time vega ingest ./large.pdf   # 100 页 PDF
# 期望: < 30s（取决于 PDF 复杂度）
```

### 单元测试

- 三因子排序公式正确性（边界值：BM25 为负 / time_decay 趋零 / tag_weight 缺省 1.0）
- 各 extractor 格式覆盖（md/pdf/docx/xlsx/html）
- StorageBackend 抽象：local 与 s3 行为一致（同样的 put/get/list/delete）
- S3 mock（moto）跑通完整 CRUD
- reindex 增量逻辑（--since 过滤）

## 不做的事

- ❌ Embedding / 向量检索（仍保持「可选」立场，不在路线图强制实现） — 未来按需
- ❌ 自定义解析库切换（用户不能替换 pypdf 为 pdfplumber） — 未来按需
- ❌ OCR（图片型 PDF 不识别） — 明确不做（[10-project-structure](../10-project-structure.md)）
- ❌ 多 bucket 并行存储 — 不在路线图
- ❌ CDN / 缓存层 — 不在路线图

## 完成判定

用户的知识库长到 5000+ 篇文档时，Vega 仍能：快速搜索（三因子排序让相关结果靠前）、搜到 PDF/DOCX 的正文内容（deep_extract）、把原文放到对象存储（S3 后端）。**规模化不掉链子** —— Vega 是生产工具，不是玩具。
