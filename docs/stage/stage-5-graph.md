# Stage 5 — 有知识图谱的 Vega

## 是什么 Vega

文档之间不再孤立。Vega 引入双链 / 知识图谱模型 —— 5 种显式有向关系类型，配合图遍历查询（links / backlinks / neighbors / path）。Vega 从「文档仓库」升级为「关联的知识网络」。

本 stage 的关系建议是**规则版**（tag 重合度 / title 相似度），LLM 版在 stage 6 的 review 流程里启用。

## 新增能力

- `links` 表（有向图）
- 5 种关系类型：`related` / `extends` / `contradicts` / `supersedes` / `references`
- `vega link` / `vega unlink` — 显式建立 / 删除关系
- `vega links` / `backlinks` / `neighbors` / `path` — 图查询
- `vega suggest-links <id>` — 规则版关系建议
- REST API 完整图路由

## 前置依赖

[stage 4](./stage-4-ai.md) 必须完成。

## 交付清单

### 数据库变更

Alembic 迁移 `0003_add_links`，按 [06-database](../06-database.md) 规格建表：

```sql
CREATE TABLE links (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id TEXT NOT NULL REFERENCES documents(id),
    target_id TEXT NOT NULL REFERENCES documents(id),
    relation_type TEXT NOT NULL,    -- related | extends | contradicts | supersedes | references
    created TEXT NOT NULL,
    updated TEXT NOT NULL,
    UNIQUE(source_id, target_id, relation_type)
);
CREATE INDEX idx_links_source ON links(source_id);
CREATE INDEX idx_links_target ON links(target_id);
```

### CLI 命令

```
# 写入
vega link <source-id> <target-id> --type extends
vega link <src> <tgt> --type references   # 默认 type=related
vega unlink <source-id> <target-id> [--type <t>]   # 不给 type 删所有

# 查询
vega links <id>                    # 出边：此文引用了谁
vega backlinks <id>                # 入边：谁引用了此文
vega neighbors <id>                # 出 + 入
vega path <id-a> <id-b>            # 最短路径（BFS）

# 建议
vega suggest-links <id>            # 规则版：返回建议的 (target, type, score)
vega suggest-links <id> --apply    # 直接应用建议（用户确认每条）
```

### REST API

```
POST   /api/documents/{id}/links           # body: { target_id, relation_type }
DELETE /api/documents/{id}/links/{target}  # ?type= 过滤

GET    /api/documents/{id}/links           # 出边
GET    /api/documents/{id}/backlinks       # 入边
GET    /api/documents/{id}/neighbors       # 出 + 入
GET    /api/path?from=<a>&to=<b>           # 最短路径

GET    /api/documents/{id}/suggestions     # 关系建议（规则版）
```

### 规则版关系建议

`suggest-links` 当前实现（不调 LLM）：

| 规则 | 建议类型 | 评分 |
|---|---|---|
| 共享 ≥ 2 个 tag | `related` | 0.6 + 0.1×共享 tag 数 |
| 标题包含另一文档标题 | `extends` 或 `references` | 0.5 |
| 标题前 5 个词完全相同 | `extends` | 0.7 |
| 同 type + 共享 ≥ 1 tag | `related` | 0.4 |

返回 top-10 建议，按 score 排序。`--apply` 进入交互式确认（y/n/skip）。

### 文件清单

- `vega/migrations/0003_add_links.py` — Alembic
- `vega/core/models/link.py` — SQLModel
- `vega/core/engine/graph.py` — 关系 CRUD + BFS 最短路径
- `vega/core/engine/suggester.py` — 规则版建议
- `vega/core/api/links.py` — 图相关路由
- `vega/cli/commands/link.py` / `links.py` / `suggest.py`
- `tests/test_stage5_graph.py`
- `tests/test_stage5_suggest.py`

## 验收标准

### 基础关系

```bash
# 1. 建立关系
A=$(vega ingest ./doc-a.md --title "FastAPI 基础" --tag python --tag fastapi | jq -r .id)
B=$(vega ingest ./doc-b.md --title "FastAPI 中间件" --tag python --tag fastapi | jq -r .id)
vega link $A $B --type extends

# 2. 查询
vega links $A           # 期望: 含 B（extends）
vega backlinks $B       # 期望: 含 A
vega neighbors $A       # 期望: 含 B（出边）

# 3. 重复关系去重
vega link $A $B --type extends
vega links $A | wc -l   # 期望: 1（UNIQUE 约束生效）
```

### 5 种关系类型

```bash
# 4. 每种类型独立
vega link $A $B --type related
vega link $A $B --type extends
vega link $A $B --type contradicts
vega links $A | jq 'length'
# 期望: 3（同 target 不同 type 各占一行）
```

### 最短路径

```bash
# 5. 三节点链 A → B → C
vega link $B $C --type references
vega path $A $C
# 期望: [A, B, C]

# 6. 无路径
vega path $A <unrelated-id>
# 期望: "no path" 退出码 0
```

### 规则版建议

```bash
# 7. tag 重合触发建议
vega suggest-links $A
# 期望: 含 B（共享 python + fastapi），type=related，score≥0.8

# 8. 建议不重复已存在的关系
vega suggest-links $A | grep $B || echo "filtered"
# 期望: 已 link 的目标被过滤或标记 "already linked"
```

### 单元测试

- 关系 CRUD + UNIQUE 约束
- 删除文档时级联清理 links（外键 ON DELETE CASCADE 或应用层）
- BFS 正确处理环（A→B→A 不死循环）
- `path` 返回节点序列，不包含权重
- 建议算法对 tag 重合数 / 标题相似度的边界值

## 不做的事

- ❌ LLM 关系建议（语义级「这两篇讲相关主题」） — stage 6 review 流程里
- ❌ 关系权重 / 置信度字段（规则版有 score，但不入库） — 未来
- ❌ 图可视化（`vega graph --render`） — 不在路线图
- ❌ 关系类型扩展（保持 5 种） — 未来按需
- ❌ inbox 流程（链接默认直接建立，不走审核） — stage 6 引入 inbox 时一并覆盖

## 完成判定

用户能在任意两文档间建立 5 种关系之一，查入度 / 出度 / 最短路径；规则版能给出「这两篇 tag 重合度高，建议 `related`」的建议。Vega 现在是**网络化的**知识库，不再是孤立文件堆。
