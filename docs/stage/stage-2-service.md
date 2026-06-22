# Stage 2 — 有服务的 Vega

## 是什么 Vega

从「单进程 CLI 直连 SQLite」升级为「Core 服务常驻 + CLI 通过 HTTP 调用」。CLI 变薄，所有读写都走 REST API。这是后续所有 stage（chat / 多客户端 / 异步任务）的架构基础。

## 新增能力

- FastAPI Core 服务（`vega serve`）
- 所有 stage 0 / 1 功能暴露为 REST API
- CLI 自动拉起后台 daemon
- `/health` 健康检查
- OpenAPI 自动文档（`/docs`）

## 前置依赖

[stage 1](./stage-1-searchable.md) 必须完成。

## 交付清单

### 目录重构

按 [10-project-structure](../10-project-structure.md) 拆分：

```
vega/
├── core/
│   ├── api/
│   │   ├── documents.py        # /api/documents/*
│   │   ├── search.py           # /api/search
│   │   └── health.py           # /health
│   ├── models/
│   │   └── document.py         # SQLModel
│   ├── engine/
│   │   └── search.py           # FTS5 查询逻辑（从 stage 1 迁入）
│   └── storage/
│       └── local.py
├── cli/
│   └── commands/
│       ├── ingest.py
│       ├── list.py
│       ├── get.py
│       ├── search.py
│       └── serve.py
├── shared/
│   └── config.py               # 配置加载
└── tests/
```

### Core 服务

- `vega.core.api.app` — FastAPI 实例
- 监听 `127.0.0.1:9487`（从 config 读取）
- 启动时初始化 SQLite + Alembic + 文件存储
- 优雅关闭（SIGTERM / SIGINT）

### REST API（对照 [08-api-routes](../08-api-routes.md)）

本 stage 仅实现已具备能力的路由：

```
GET  /health                            # { status: "ok", version: "..." }

POST /api/documents                     # multipart: file + metadata form 字段
GET  /api/documents?limit=20&offset=0   # 列表
GET  /api/documents/{id}                # 详情（含 metadata）
GET  /api/documents/{id}/content        # 原文下载

GET  /api/search?q=xxx&tag=yy&type=zz&limit=20&offset=0
```

> `/summary` / `/inbox` / `/links` / `/chat` / `/export` / `/import` 留给后续 stage。

### CLI 改造

- 所有命令从「直连 SQLite」改为「HTTP 调 Core」
- HTTP 客户端：`httpx`
- 自动拉起逻辑：
  - CLI 启动时先 GET `/health`
  - 连接失败 → 后台 fork `vega serve --daemon`
  - 等待最多 3 秒服务就绪
  - 仍失败 → 报错并提示手动启动
- `vega serve [--daemon]` — 前台 / 后台启动
- `vega stop` — 停止 daemon

### 配置扩展

```yaml
server:
  host: 127.0.0.1
  port: 9487

storage:
  type: local
  path: ~/.vega/files

search:
  default_limit: 20
```

### 文件清单

- `vega/core/` — 整个新建
- `vega/cli/` — 改造为 HTTP 客户端
- `vega/shared/config.py` — 加载 `~/.vega/config.yaml`，环境变量覆盖
- `vega/cli/commands/serve.py` — serve / stop / daemon 管理
- `tests/test_stage2_api.py` — FastAPI TestClient
- `tests/test_stage2_autolaunch.py` — 自动拉起行为

## 验收标准

### 服务生命周期

```bash
# 1. 启动服务
vega serve &
sleep 1
curl http://127.0.0.1:9487/health
# 期望: {"status":"ok","version":"0.2.0"}

# 2. 自动拉起（无显式 serve）
vega stop 2>/dev/null; sleep 1
vega list
# 期望: 自动后台启动 daemon 并返回结果

# 3. 停止
vega stop
curl http://127.0.0.1:9487/health || echo "stopped"
# 期望: 连接失败

# 4. OpenAPI 文档
vega serve &
curl http://127.0.0.1:9487/docs -o /dev/null -w "%{http_code}\n"
# 期望: 200
```

### API 功能（覆盖 stage 0/1）

```bash
# 5. 通过 API 入库
curl -X POST http://127.0.0.1:9487/api/documents \
    -F "file=@./readme.md" \
    -F "title=Vega README" \
    -F "type=Document" \
    -F "tags=[\"docs\"]"
# 期望: 200，返回 { id: "..." }

# 6. 列表 + 分页
curl 'http://127.0.0.1:9487/api/documents?limit=5&offset=0'

# 7. 搜索
curl 'http://127.0.0.1:9487/api/search?q=vega&limit=10'

# 8. 下载原文
curl http://127.0.0.1:9487/api/documents/<id>/content -o /tmp/out.md
```

### CLI 行为不变

stage 0 / 1 的所有 CLI 命令行为保持一致（用户视角无破坏）。改动是内部从 SQLite 直连切到 HTTP。

```bash
vega ingest ./file.md --tag python
vega search "fastapi"
vega get-metadata <id>
vega get <id> --output /tmp/out.md
# 全部仍正常工作
```

### 单元测试

- TestClient 跑通所有路由
- 自动拉起：mock `/health` 失败 → 触发 daemon 启动 → 二次 `/health` 成功
- 配置从环境变量覆盖（`VEGA_SERVER_PORT=9999` → 实际监听 9999）

## 不做的事

- ❌ 摘要相关路由 — stage 3
- ❌ AI 对话 / SSE — stage 7
- ❌ links / backlinks / neighbors / path — stage 5
- ❌ inbox / approve / reject — stage 6
- ❌ export / import — stage 8
- ❌ 异步任务队列（同步处理足够） — stage 8 如需要再引入
- ❌ 多 worker / 集群（uvicorn 单进程足够） — 不在路线图

## 完成判定

用户跑 `vega serve &` 后，可以从另一台机器（同账号 SSH）通过 `curl` 访问知识库；CLI 命令照常工作。Vega 现在是**服务化的**知识库，为后续所有网络化能力（对话、远程访问、多客户端）打地基。
