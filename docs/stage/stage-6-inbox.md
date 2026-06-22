# Stage 6 — 有审核流水线的 Vega

## 是什么 Vega

ingest 默认不再直接进 active 库，而是先进 inbox 待人工审核。review 流程里能看到 LLM 生成的 metadata / 摘要 / 关系建议，人工拍板是否通过、如何调整。这是「agent 自动抓取 + 人把关」模式的核心 —— YOLO 模式保留给信任来源。

## 新增能力

- `documents.status` 列（`new` / `reviewed` / `archived`）
- ingest 默认 `status=new` 进 inbox
- `--yolo` 标志跳过 inbox，直接 active
- inbox / review / approve / reject CLI 与 API
- review 时显示 LLM 关系建议（升级 stage 5 的规则版）

## 前置依赖

[stage 5](./stage-5-graph.md) 必须完成。

## 交付清单

### 数据库变更

Alembic 迁移 `0004_add_status`：

```sql
ALTER TABLE documents ADD COLUMN status TEXT NOT NULL DEFAULT 'new';
-- 已有文档（stage 0-5 入库的）默认 'reviewed'
UPDATE documents SET status = 'reviewed' WHERE status = 'new';
```

### CLI 命令

```
# 入库（默认走 inbox）
vega ingest <file> [--yolo]          # --yolo 跳过审核，直接 reviewed

# 审核
vega inbox                          # 待审列表（status=new）
vega inbox --tag python             # 按 tag 过滤
vega inbox --limit 20

vega review <id>                    # 交互式：显示 metadata + 摘要 + 关系建议
                                    #   [A] 通过  [R] 拒绝  [E] 编辑  [S] 跳过

vega approve <id>                   # 快速通过
vega reject <id>                    # 拒绝（删除文件 + documents 行 + 相关 links）
vega archive <id>                   # 已 reviewed 文档归档（不删，仅标记）
```

### `vega review` 交互流程

```
┌─────────────────────────────────────────┐
│  Reviewing: 7c9f...                      │
│                                          │
│  Title:       FastAPI 教程                │
│  Type:        Document                    │
│  Tags:        [python, fastapi]           │
│  Description: FastAPI 入门与核心特性...    │
│                                          │
│  Summary:                                │
│  本文介绍 FastAPI 的路由、依赖注入、       │
│  ORM 集成和异步支持...                    │
│                                          │
│  Suggested Links (LLM):                  │
│  → extends:   doc-042 (ASGI 规范)  ★0.92 │
│  → related:   doc-107 (Starlette)  ★0.85 │
│  → references: doc-015 (Pydantic) ★0.78 │
│                                          │
│  [A]pprove  [R]eject  [E]dit  [S]kip    │
└─────────────────────────────────────────┘
```

- **A**: status → reviewed，应用选中的 suggested links
- **R**: 删除文档 + 文件 + 相关 links
- **E**: 打开 `$EDITOR` 编辑 metadata JSON，保存后回流程
- **S**: 留在 inbox，下一个

### REST API

```
GET  /api/inbox                         # 列表（?tag=&type=&limit=）
POST /api/documents/{id}/approve        # body: { apply_links?: [link-id...] }
POST /api/documents/{id}/reject
POST /api/documents/{id}/archive

GET  /api/documents/{id}/suggestions    # LLM 关系建议（stage 5 的升级版）
```

### LLM 关系建议（升级规则版）

`stage 5` 的规则版基础上，新增 LLM 路径：

```
suggest_links(id):
    candidates = rule_based_suggest(id)        # stage 5 规则
    if llm_enabled:
        llm_suggestions = llm.analyze(doc, candidates)  # 语义判断 + 重排
        candidates = merge(candidates, llm_suggestions)  # 去重 + 取最高 score
    return candidates[:10]
```

LLM 调用输入：原文 metadata + 摘要 + top-20 候选文档的 metadata + 摘要。LLM 输出每条候选的关系类型 + score。

### 文件清单

- `vega/migrations/0004_add_status.py`
- `vega/core/engine/inbox.py` — 状态机
- `vega/core/engine/suggester.py` — 加 LLM 升级路径
- `vega/core/api/inbox.py` — inbox 路由
- `vega/cli/commands/inbox.py` / `review.py` / `approve.py` / `reject.py`
- `vega/cli/tui/` — review 交互式 TUI（推荐 prompt_toolkit 或 rich）
- `tests/test_stage6_inbox.py`
- `tests/test_stage6_review_flow.py`

## 验收标准

### 状态机

```bash
# 1. 默认进 inbox
vega ingest ./doc.md
vega inbox | grep <id>                 # 期望: 出现
vega list | grep <id>                  # 期望: 默认 list 也显示（除非加 --status reviewed）

# 2. YOLO 直入 active
vega ingest ./doc.md --yolo
vega inbox | grep <id> || echo "not in inbox"   # 期望: not in inbox

# 3. 审核通过
ID=$(vega inbox --limit 1 | jq -r .[0].id)
vega approve $ID
vega inbox | grep $ID || echo "removed from inbox"   # 期望: removed

# 4. 拒绝（删除）
vega ingest ./temp.md
ID=$(...)
vega reject $ID
vega get-metadata $ID                  # 期望: 404
ls ~/.vega/files/ | grep $ID || echo "file removed"   # 期望: removed
```

### 关系建议（LLM 升级）

```bash
# 5. 配置 LLM 后，建议含语义判断
vega suggest-links <id> --llm
# 期望: 返回 top-10，含 LLM 判定的 relation_type 和 score

# 6. review 流程内能选择性应用
vega review <id>
# 期望: 列出 LLM 建议，用户可勾选哪些应用

# 7. LLM 失败降级到规则版
unset DEEPSEEK_API_KEY
vega suggest-links <id>
# 期望: 仅规则版结果
```

### 交互式 review

```bash
# 8. 完整审核闭环
vega ingest ./new-doc.md
ID=$(vega inbox --limit 1 | jq -r .[0].id)
echo -e "e\n{}\na\n" | vega review $ID   # 模拟键入：编辑 → 保存 → 通过
vega inbox | grep $ID || echo "approved"
```

### 单元测试

- 状态迁移：new → reviewed / archived / 删除，不允许反向（reviewed 不能回 new）
- reject 删除时级联清理 links（stage 5 的 ON DELETE CASCADE 触发）
- review 的 `[E]` 编辑后字段实际更新
- LLM 建议去重（同 target 多来源时取最高 score）

## 不做的事

- ❌ 批量审核（`vega approve --all`） — 不在路线图（违背「人把关」初衷）
- ❌ 审核工作流（多人协作 / 评论 / 指派） — 不在路线图（单用户工具）
- ❌ YOLO 模式的来源白名单（仅 `--yolo` 标志位） — 未来按需
- ❌ 撤销审核（reviewed → new） — 不支持，重新入库即可

## 完成判定

用户能放心让 agent 自动 ingest 大量文档（默认进 inbox，不污染 active），定期 `vega inbox` 逐条审核，review 时看到 LLM 建议的 tag / 关系，一键通过或编辑调整。**信任边界清晰** —— 自动化与人工把关共存。
