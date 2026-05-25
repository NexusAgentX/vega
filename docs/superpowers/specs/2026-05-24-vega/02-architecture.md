# 02 — 架构 + 技术栈

## 技术栈

| 维度 | 选择 |
|---|---|
| 语言 | Python 3.12+ |
| 包管理 | uv |
| Web 框架 | FastAPI |
| ORM | SQLModel |
| 数据库 | SQLite（`~/.vega/vega.db`），迁移用 Alembic |
| 文件存储 | 默认本地 `~/.vega/files/`，可选 S3 兼容协议 |
| CLI 框架 | typer |
| HTTP 客户端 | httpx |
| LLM | litellm（DeepSeek、OpenAI 等），可配置 |
| 测试 | pytest + httpx（FastAPI TestClient） |
| 打包 | PyInstaller，单二进制 |

## 架构图

```
┌─────────────────────────────────────────────────────┐
│                    vega (单二进制)                     │
│                                                       │
│  ┌──────────────┐        ┌─────────────────────────┐ │
│  │    CLI 层     │  HTTP  │       Core 层            │ │
│  │              │ ──────▶│                          │ │
│  │ vega serve   │        │  FastAPI Server          │ │
│  │ vega query   │        │  ├─ REST API             │ │
│  │ vega ingest  │        │  ├─ Chat / Agent         │ │
│  │ vega link    │        │  ├─ 文档关系引擎          │ │
│  │ vega review  │        │  └─ LLM Bridge           │ │
│  └──────────────┘        │         │                │ │
│                          │    SQLite                 │ │
│                          └─────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

## 架构原则

- **单二进制**：用户下载一个文件即可使用
- **代码分离**：`core/` 和 `cli/` 目录独立，未来可拆为双二进制
- **CLI 分两类**：
  - 查询命令（`search`、`get-*`、`links`、`related`）：只读，供外部 coding agent 使用
  - 管理命令（`ingest`、`review`、`approve`、`reject`）：写入，供人管理知识库
- **热更新仅在 Core**：AI 对话过程中动态调整 metadata 和文档关系，CLI 不触发