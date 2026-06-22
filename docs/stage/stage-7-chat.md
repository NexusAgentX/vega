# Stage 7 — 能对话的 Vega

## 是什么 Vega

用户能与 Vega 直接对话：「这篇文档讲了什么？」「帮我找出所有关于 FastAPI 中间件的文档，整理成表格」「给这篇文档打个 python 标签」。对话中 AI 能动态更新 metadata 和文档关系（热更新）。这是 Vega 从「工具」升级为「AI 知识伙伴」的一步。

## 新增能力

- `chat_sessions` / `chat_messages` 表
- `vega chat` 流式对话（SSE）
- 会话管理（list / export / continue）
- **热更新**：对话中 AI 可调整 metadata、建立/断开 links、改 summary
- LLM 工具调用（function calling）：LLM 显式调用 Vega 内部 API

## 前置依赖

[stage 6](./stage-6-inbox.md) 必须完成。

## 交付清单

### 数据库变更

Alembic 迁移 `0005_add_chat`，按 [06-database](../06-database.md) 规格建表：

```sql
CREATE TABLE chat_sessions (
    id TEXT PRIMARY KEY,
    title TEXT,
    created TEXT NOT NULL,
    updated TEXT NOT NULL
);

CREATE TABLE chat_messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES chat_sessions(id),
    role TEXT NOT NULL,             -- user | assistant | system | tool
    content TEXT NOT NULL,
    metadata TEXT DEFAULT '{}',     -- JSON：tool_calls / tool_results / tokens
    created TEXT NOT NULL
);
```

### CLI 命令

```
# 对话
vega chat "问题"                       # 单轮，默认 session
vega chat "问题" --session <id>        # 续已有 session
vega chat --interactive                # 交互式 REPL

# 会话管理
vega chat --list-sessions              # 列出所有会话
vega chat --session <id> --show        # 显示某会话全部消息
vega chat --export <session-id>        # 导出为 markdown
vega chat --delete-session <id>
```

### REST API

```
POST /api/chat                         # body: { message, session_id? }
                                       # 响应：SSE 流（text/event-stream）
                                       #   data: {"type":"token","content":"..."}
                                       #   data: {"type":"tool_call","name":"update_metadata",...}
                                       #   data: {"type":"tool_result",...}
                                       #   data: {"type":"done"}

GET  /api/chat/sessions                # 会话列表
GET  /api/chat/sessions/{id}           # 会话消息
GET  /api/chat/sessions/{id}/export    # 导出 markdown
DELETE /api/chat/sessions/{id}
```

### 热更新工具集

LLM 通过 function calling 调用以下工具（LLM 侧用 OpenAI tools / deepseek tools 格式）：

| 工具 | 作用 |
|---|---|
| `search_documents(query, tag?, type?)` | 搜索文档 |
| `get_metadata(doc_id)` | 读取 metadata |
| `get_summary(doc_id)` | 读取摘要 |
| `update_metadata(doc_id, title?, tags?, type?, description?)` | **热更新** metadata |
| `update_summary(doc_id, new_summary)` | **热更新** 摘要 |
| `add_link(source, target, relation_type)` | **热更新** 加关系 |
| `remove_link(source, target, relation_type?)` | **热更新** 删关系 |
| `create_document(title, content, tags?)` | 在对话中创建新文档 |

热更新**仅在 chat 流程中允许**；CLI 单独的 `vega update-metadata` 命令也加（stage 7 之前 stage 缺失这个 CRUD 能力），但不走 chat 的工具调用路径。

### 文件清单

- `vega/migrations/0005_add_chat.py`
- `vega/core/models/chat.py` — SQLModel for sessions/messages
- `vega/core/engine/chat.py` — SSE 流生成 + LLM 工具调用循环
- `vega/core/engine/tools.py` — function calling 工具定义与实现
- `vega/core/api/chat.py` — SSE 路由
- `vega/cli/commands/chat.py` — CLI 流式渲染（rich.live）
- `vega/cli/commands/update.py` — 非对话场景的 metadata/summary/link 更新命令
- `tests/test_stage7_chat.py`
- `tests/test_stage7_tools.py`
- `tests/test_stage7_hotupdate.py`

## 验收标准

### 基础对话

```bash
# 1. 单轮
vega chat "你好"
# 期望: 流式输出回复

# 2. 交互式
vega chat --interactive
> 第一句
<回复>
> 第二句（同 session）
<带上下文的回复>

# 3. 续会话
SID=$(vega chat --list-sessions | jq -r .[0].id)
vega chat "接着说" --session $SID

# 4. 导出
vega chat --export $SID > conversation.md
```

### SSE 流

```bash
# 5. 直接 curl SSE
curl -N -X POST http://127.0.0.1:9487/api/chat \
    -H "Content-Type: application/json" \
    -d '{"message":"hello"}'
# 期望: 一系列 data: {...}\n\n
```

### 工具调用

```bash
# 6. 触发 search_documents
vega ingest ./fastapi-tutorial.md --yolo
vega chat "知识库里有没有讲 FastAPI 的文档？"
# 期望: 回复中 LLM 调用了 search_documents 并引用结果

# 7. 热更新 metadata
vega chat "给 <doc-id> 这篇加一个 'web' 标签"
vega get-metadata <doc-id> | jq '.tags'
# 期望: 含 "web"（LLM 调用了 update_metadata）
```

### 单元测试

- SSE 流：每个 token 作为一个 event，最后 `done`
- 工具调用循环：LLM 返回 tool_call → 执行 → 把结果作为 tool_result 回喂 LLM → 继续
- 会话上下文：第二轮 LLM 能看到第一轮的 user/assistant 消息
- 热更新实际写入 SQLite（chat 结束后 `get-metadata` 能查到）
- 工具调用失败时（如 doc_id 不存在）LLM 收到错误信息并自行处理

## 不做的事

- ❌ Web UI（Phase 2 才上，本 stage 仅 CLI + API） — 路线图外
- ❌ 多模态（图像 / 语音输入） — 不在路线图
- ❌ RAG / 向量检索增强对话（Vega 立场是结构化检索优先） — stage 9 仍是可选
- ❌ 对话中创建的文档自动进 inbox（chat 内 create_document 默认 reviewed，因为是用户主导） — 设计决策
- ❌ Token 计费 / 配额管理 — 不在路线图（单用户工具）

## 完成判定

用户能在终端 `vega chat "..."` 自然语言问知识库问题，AI 流式回答、必要时调内部工具搜索 / 改 metadata / 改关系。**对话即操作** —— Vega 成为有记忆、能动手的 AI 知识伙伴，不再是被动检索工具。
