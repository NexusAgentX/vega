# Stage 8 — 可互通的 Vega（OKF 一等公民）

## 是什么 Vega

Vega 知识库可以整体导出为合规 [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) bundle，任意 OKF bundle 可以导入。Vega 不再是孤岛 —— 知识可以跨工具、跨组织、跨时间迁移，可以用 git 管理、可以 diff、可以分享。

本 stage 同时补齐 stage 0-7 没引入的 `resource_uri` 字段（OKF 对齐要求）。

## 新增能力

- `documents.resource_uri` 列（对齐 OKF `resource`）
- `core/okf/` 桥接模块（exporter / importer / mapper / validator）
- `vega export --okf <path>` / `vega import --okf <path>` CLI
- OKF 相关 REST API
- OKF §9 合规性自检

## 前置依赖

[stage 7](./stage-7-chat.md) 必须完成（因为 export 要包含所有已有能力产生的数据）。

## 交付清单

### 数据库变更

Alembic 迁移 `0006_add_resource_uri`：

```sql
ALTER TABLE documents ADD COLUMN resource_uri TEXT;
```

`resource_uri` 可选，用于绑定外部 URI（Jira issue / GitHub PR / 网页 URL）。OKF `resource` 字段的 Vega 对应。

### `core/okf/` 模块

按 [10-project-structure](../10-project-structure.md) 规格：

```
vega/core/okf/
├── __init__.py
├── mapper.py            # 字段映射 / concept id 策略 / 关系双写
├── exporter.py          # Vega 知识库 → OKF bundle
├── importer.py          # OKF bundle → Vega inbox
├── validator.py         # OKF §9 合规性自检
└── templates/           # index.md / log.md / concept.md 模板
```

详细映射规则、concept id 策略、关系双写规则、非 md 原文处理见 [okf-interop](../okf-interop.md) 规格。本 stage 严格按该规格实现。

### CLI 命令

```
vega export --okf <path>
    [--include-inbox]              # 默认仅 reviewed
    [--include-archived]
    [--id-strategy semantic|uuid]  # 默认 semantic（<tag>/<slug>.md）
    [--no-attachments]             # 仅 metadata + summary

vega import --okf <path>
    [--yolo]                       # 跳过 inbox，直接 active
    [--merge-by uuid|title]        # uuid（默认）| title
    [--no-allow-broken-links]      # 默认容忍断链（OKF §5.3 强制）
```

### REST API

```
POST /api/export/okf                # body: { path, include_inbox?, ... }
                                    # 返回: { task_id }（大 bundle 异步）
POST /api/import/okf                # multipart: bundle zip 或 path
                                    # 返回: { task_id }
GET  /api/export/okf/status/<id>    # 异步进度
GET  /api/import/okf/status/<id>
```

### Validator

OKF §9 三条硬约束自检：

1. 每个非保留 `.md` 有可解析 YAML frontmatter
2. 每个 frontmatter 有非空 `type` 字段
3. `index.md` / `log.md` 结构合规

输出合规报告（json）：通过 / 警告 / 错误列表。

### 文件清单

- `vega/migrations/0006_add_resource_uri.py`
- `vega/core/okf/mapper.py`
- `vega/core/okf/exporter.py`
- `vega/core/okf/importer.py`
- `vega/core/okf/validator.py`
- `vega/core/api/okf.py` — export/import/status 路由
- `vega/cli/commands/export.py` / `import.py`
- `tests/test_stage8_export.py`
- `tests/test_stage8_import.py`
- `tests/test_stage8_roundtrip.py` — 往返无损测试
- `tests/fixtures/okf-bundles/` — 测试用合规 / 不合规 bundle

## 验收标准

### 字段映射

```bash
# 1. resource_uri 入库
vega ingest ./issue.md \
    --title "Bug 142" \
    --resource-uri "https://jira.example.com/BUG-142"

vega get-metadata <id> | jq .resource_uri
# 期望: "https://jira.example.com/BUG-142"
```

### 导出

```bash
# 2. 基础导出
vega export --okf ./my-bundle/
ls ./my-bundle/
# 期望: index.md / attachments/ / <tag>/<slug>.md / untagged/...

# 3. 合规自检
vega export --okf ./my-bundle/ --validate
# 期望: "OKF v0.1 conformant: PASS"

# 4. 仅导 reviewed
vega ingest ./temp.md                    # 进 inbox
vega export --okf ./bundle/ --include-inbox=false
grep <temp-id> -r ./bundle/ || echo "excluded"
# 期望: excluded

# 5. 非 md 原文走 attachments
vega ingest ./report.pdf --yolo
vega export --okf ./bundle/
ls ./bundle/attachments/
# 期望: <uuid>.pdf
cat ./bundle/untagged/<slug>.md           # 期望: frontmatter 含 vega_attachment
```

### 导入

```bash
# 6. 导入合规 bundle
vega import --okf ./their-bundle/
vega inbox --limit 100 | wc -l           # 期望: 增加（走 inbox）

# 7. YOLO 导入
vega import --okf ./their-bundle/ --yolo
vega inbox --limit 100 | wc -l           # 期望: 不变

# 8. UUID 命中合并
vega export --okf ./bundle/              # 假设导出包含 id=X
# 修改 X 的内容
vega import --okf ./bundle/ --merge-by uuid
vega get-metadata <X> | jq .title        # 期望: 是被导入的版本（合并）
```

### 往返无损

```bash
# 9. roundtrip
vega export --okf ./bundle-1/
vega import --okf ./bundle-1/ --yolo --merge-by uuid
vega export --okf ./bundle-2/
diff -r ./bundle-1/ ./bundle-2/
# 期望: 仅 log.md 时间戳差异（或完全一致）
```

### 断链容忍

```bash
# 10. 导入含断链的 bundle
# fixture: okf-bundles/broken-links/
vega import --okf ./tests/fixtures/okf-bundles/broken-links/
# 期望: 成功导入，broken_links 计数 > 0，无报错

# 11. 关闭容忍
vega import --okf ./tests/fixtures/okf-bundles/broken-links/ --no-allow-broken-links
# 期望: 退出码 != 0，错误列出断链
```

### 单元测试

- mapper 字段往返：所有标准字段（type/title/description/tags/resource/timestamp）+ 扩展字段（vega_id/vega_source_format/...）双向无损
- concept id 策略：semantic 模式按 tag 分目录、slug 冲突追加 uuid 前 8 位
- exporter 生成的 bundle 通过 validator
- importer 容忍未知 type / 未知扩展字段 / 断链
- merge-by uuid 命中则 UPDATE，否则 INSERT
- merge-by title 命中则 UPDATE，否则 INSERT

## 不做的事

- ❌ 增量导出（`--incremental`） — 配置位预留，未实现
- ❌ OKF bundle 的 git push/pull（用户自己 git） — 不在路线图
- ❌ 其他交换格式（Avro / Protobuf / OpenAPI 等） — 不在路线图
- ❌ 可视化 bundle 浏览器 — 不在路线图（用户随便用 markdown viewer）
- ❌ OKF v0.2+ 兼容 — 待 spec 演进

## 完成判定

用户能把整个 Vega 知识库 `git clone` 给同事，对方 `vega import --okf` 即可继续工作；或导出 bundle 用任意 OKF 消费工具读取；或反之，从其他 OKF 生产工具迁移到 Vega。**Vega 成为 OKF 生态的一等公民**，知识不再被锁。
