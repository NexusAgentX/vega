# Stage 3 — 三层渐进的 Vega

## 是什么 Vega

Vega 的核心特色 —— **三层渐进加载（metadata → 摘要 → 原文）** —— 在本 stage 落地。CLI 和 API 明确区分三层，按需拉取，尽量不浪费 context window。这是 Vega 区别于「全文检索引擎」和「向量数据库」的关键。

摘要本 stage 可以是手动提供的字符串或从原文截取的前 N 字。真正的 LLM 生成在 stage 4。

## 新增能力

- `documents.summary` 列（LLM 生成的 200 token 摘要，本 stage 暂用截取 / 手动填充）
- `vega get-summary <id>` — 仅取摘要
- `GET /api/documents/{id}/summary` 路由
- 三层检索的命令分层清晰（stage 2 的 get-content 已存在，本 stage 加 get-summary）
- `vega ingest --summary <text>` 手动提供摘要

## 前置依赖

[stage 2](./stage-2-service.md) 必须完成。

## 交付清单

### 数据库变更

Alembic 迁移 `0002_add_summary`：

```sql
ALTER TABLE documents ADD COLUMN summary TEXT;
```

`summary` 与 `description` 共存（详见 [03-data-model](../03-data-model.md) 字段说明）：
- `description` — 一句话摘要（≤ 80 字），OKF 对齐字段，检索和列表预览用
- `summary` — 较长摘要（≤ 200 token，约 400-800 字），三层渐进加载的第二层

### FTS5 索引扩展

把 `summary` 加入 FTS 索引列（按 [06-database](../06-database.md) 规格更新触发器）。

### CLI 命令

```
vega ingest <file>
    [--summary <text>]              # 手动提供摘要
    [--description <text>]          # 一句话摘要
    [--auto-summary]                # 启用「截取前 N 字」填充（stage 4 后改走 LLM）
    ...

vega get-summary <id>               # 仅摘要（第二层）
vega get-metadata <id>              # 仅 metadata（第一层）
vega get-content <id>               # 原文（第三层）
```

三层命令对应三层加载，用户/coding agent 可以按精度选择。

### REST API

```
GET /api/documents/{id}/metadata    # 仅 metadata（轻量）
GET /api/documents/{id}/summary     # 仅 summary（中等）
GET /api/documents/{id}/content     # 原文下载（最重）
```

明确分层。`GET /api/documents/{id}` 返回 metadata + summary（不含原文 BLOB），用于列表预览。

### 自动摘要（截取版）

`--auto-summary` 启用时：
- md / txt：截取前 500 字符（按段落切，不截断句子）
- pdf / docx / 其他：`summary = "[Binary content, type=<ext>, size=<n> bytes]"`
- 不调用任何 LLM

这是 stage 4 LLM 摘要的占位实现，保证 stage 3 的所有流程能跑通。

### 文件清单

- `vega/migrations/0002_add_summary.py` — Alembic
- `vega/core/api/documents.py` — 加 summary 路由，改 detail 路由不含 BLOB
- `vega/cli/commands/get.py` — 拆分为 get-metadata / get-summary / get-content
- `vega/cli/commands/ingest.py` — 加 `--summary` / `--auto-summary`
- `vega/core/engine/summarizer.py` — 截取版摘要逻辑（stage 4 替换为 LLM）
- `tests/test_stage3.py`

## 验收标准

### 三层分离

```bash
# 1. 手动摘要入库
vega ingest ./long-doc.md \
    --title "深度学习综述" \
    --description "深度学习各分支综述" \
    --summary "本文综述深度学习的发展历史、核心架构（CNN/RNN/Transformer）及未来方向。"

# 2. 三层大小递增
vega get-metadata <id> | wc -c       # 最小，~200-500 字节 JSON
vega get-summary <id> | wc -c        # 中等，~500-2000 字节
vega get-content <id> | wc -c        # 最大，原文大小

# 3. /api/documents/{id} 不含 BLOB
curl http://127.0.0.1:9487/api/documents/<id> | jq 'has("content")'
# 期望: false

# 4. 三层路由各自独立
curl http://127.0.0.1:9487/api/documents/<id>/metadata | jq .
curl http://127.0.0.1:9487/api/documents/<id>/summary
curl http://127.0.0.1:9487/api/documents/<id>/content -o /tmp/out.md
```

### 自动摘要（截取版）

```bash
# 5. md 文件自动截取
vega ingest ./long.md --auto-summary
vega get-summary <id>
# 期望: 原文前 ~500 字符

# 6. 二进制文件占位
vega ingest ./report.pdf --auto-summary
vega get-summary <id>
# 期望: "[Binary content, type=pdf, size=N bytes]"
```

### 搜索扩展

```bash
# 7. summary 参与 FTS
vega ingest ./doc.md --summary "特别的关键词 xyz123abc"
vega search "xyz123abc"
# 期望: 命中（说明 summary 进了 FTS）
```

### 单元测试

- 三层命令返回不同 content-length
- `description` 与 `summary` 字段语义独立、不互相覆盖
- 截取摘要按段落边界切，不截断句子
- Alembic 迁移在已有数据上跑通

## 不做的事

- ❌ LLM 生成摘要 / metadata 自动提取 — stage 4
- ❌ `resource_uri` 字段 — stage 8
- ❌ `status` / inbox — stage 6
- ❌ 关系图谱 — stage 5
- ❌ 三因子排序 — stage 9

## 完成判定

用户（或 coding agent）能按精度选择「先看 metadata 决定要不要、再看 summary 决定值不值得、最后才取原文」的三步渐进流程，每层流量递增。**context window 友好**是 Vega 的核心承诺，本 stage 兑现。
