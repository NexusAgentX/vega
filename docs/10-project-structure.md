# 10 — 项目目录结构 + 不做的事情

## 目录结构

```
vega/
├── core/                    # FastAPI 后端
│   ├── api/                 # REST 路由（search, ingest, chat, export/okf, import/okf...）
│   ├── models/              # SQLite ORM 模型
│   ├── engine/              # 文档关系引擎
│   ├── llm/                 # LLM Bridge（deepseek adapter 等）
│   ├── okf/                 # OKF v0.1 桥接层
│   │   ├── exporter.py      # Vega 知识库 → OKF bundle
│   │   ├── importer.py      # OKF bundle → Vega inbox
│   │   ├── mapper.py        # 字段映射 / concept id 策略 / 关系双写
│   │   └── validator.py     # OKF §9 合规性自检
│   └── storage/             # 文件存储（local + S3 backend）
├── cli/                     # CLI 客户端
│   └── commands/            # 各子命令实现
├── tests/
├── pyproject.toml
└── docs/
```

## 做的事

- **OKF v0.1 一等交换格式**：整个知识库可导出为合规 OKF bundle，任意 OKF bundle 可导入；metadata schema 对齐 OKF frontmatter（`type` / `resource`）。详见 [okf-interop](./okf-interop.md)
- **原文 BLOB 零改写**：metadata 在 SQLite，原文在文件系统（可按 tag/slug 生成 OKF concept，但 Vega 内部不修改原文）
- **结构化检索优先**：默认 metadata + 摘要 FTS + 文件名 + 关系图谱；embedding 仅作规模化后的可选、可删除、可重建的辅助索引

## 不做的事情

- 不做文档内容解析（`.docx`、`.xlsx` 等由 agent 自己的 skill 处理）
- 不把 embedding / 向量化作为默认知识组织方式；规模化后只作为可选、可删除、可重建的辅助索引
- 不做 OCR
- 不做默认 RAG 语义检索；检索入口优先使用 metadata、摘要 FTS、文件名和关系图谱
- 不做 MCP Server（只用 CLI 接口）
- 不做多用户 / 权限系统（单用户本地工具）
