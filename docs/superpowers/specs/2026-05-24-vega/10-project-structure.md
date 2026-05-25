# 10 — 项目目录结构 + 不做的事情

## 目录结构

```
vega/
├── core/                    # FastAPI 后端
│   ├── api/                 # REST 路由（search, ingest, chat...）
│   ├── models/              # SQLite ORM 模型
│   ├── engine/              # 文档关系引擎
│   ├── llm/                 # LLM Bridge（deepseek adapter 等）
│   └── storage/             # 文件存储（local + S3 backend）
├── cli/                     # CLI 客户端
│   └── commands/            # 各子命令实现
├── tests/
├── pyproject.toml
└── docs/
    └── superpowers/
        └── specs/
```

## 不做的事情

- 不做文档内容解析（`.docx`、`.xlsx` 等由 agent 自己的 skill 处理）
- 不做 embedding / 向量化
- 不做 OCR
- 不做 RAG 语义检索
- 不做 MCP Server（只用 CLI 接口）
- 不做多用户 / 权限系统（单用户本地工具）