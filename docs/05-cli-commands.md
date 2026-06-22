# 05 — CLI 命令矩阵

## 服务管理

```
vega serve                          # 启动 Core 服务
vega stop                           # 停止服务
```

## AI 对话

```
vega chat "问题"                    # 终端流式对话
vega chat --session <id> "问题"     # 继续已有会话
vega chat --export <session-id>     # 导出会话记录
vega chat --list-sessions           # 列出所有会话
```

## 文档写入

```
vega ingest <file> [--yolo]        # 入库，--yolo 跳过审核
vega ingest --stdin [--yolo]       # 管道读入（原始二进制 + metadata）
```

## 三层检索（只读）

```
vega search "<q>" [--tag xy]       # Metadata + 摘要 FTS 搜索
  --limit 20 --offset 0            # 分页参数（默认 limit=20）
vega get-metadata <id>              # 单文档完整 metadata（JSON）
vega get-summary <id>               # AI 生成摘要
vega get-content <id> [--output f]  # 原文文件
```

## 审核

```
vega inbox                          # 待审列表
vega review <id>                    # 交互式审核
vega approve <id>                   # 快速通过
vega reject <id>                    # 拒绝
```

## 关系查询（只读）— 双链 / 知识图谱

```
vega links <id>                     # 出边：此文引用了谁
vega backlinks <id>                 # 入边：谁引用了此文
vega neighbors <id>                 # 出边 + 入边合并
vega path <id-a> <id-b>            # 两文档间最短路径
```

## OKF 互操作（导入 / 导出）

```
vega export --okf <path>
    [--include-inbox]              # 默认仅 reviewed；带上则含 new
    [--include-archived]           # 默认不含 archived
    [--id-strategy semantic|uuid]  # 默认 semantic（<tag>/<slug>.md）；uuid 用 documents/<uuid>.md
    [--no-attachments]             # 仅导 metadata + summary，不含原文

vega import --okf <path>
    [--yolo]                       # 跳过 inbox，直接 active
    [--merge-by uuid|title]        # uuid（默认）：vega_id 命中则更新；title：title 相同则更新
    [--allow-broken-links]         # 默认允许（OKF §5.3 强制）；--no-allow-broken-links 关闭则断链报错
```

详见 [okf-interop](./okf-interop.md)。

## 关键行为

| 行为 | 说明 |
|---|---|
| 自动拉起服务 | CLI 查询命令检测到 `vega serve` 未启动时，自动后台启动 `vega serve --daemon` |
| UUID 文档 ID | `vega ingest` 自动生成 UUID 作为文档 ID |
| 缺 metadata 自动提取 | 入库文档未提供 title/tags 时，LLM 自动分析内容生成 |
| 分页 | `vega search` 默认 `--limit 20`，支持 `--offset` |
| 标签平铺 | `tags: [python, fastapi, middleware]`，无层级 |