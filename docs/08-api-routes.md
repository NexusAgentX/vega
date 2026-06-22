# 08 — REST API 路由

所有响应使用 FastAPI + Pydantic model 定义 schema，自动生成 OpenAPI 文档，不加统一信封包装。

```
GET  /health                            # 健康检查

# 文档
POST /api/documents                     # ingest 文档（multipart/form-data）
GET  /api/documents                     # 文档列表（分页：?limit=&offset=）
GET  /api/documents/{id}                # 文档详情（含 metadata）
GET  /api/documents/{id}/metadata       # 仅 metadata
GET  /api/documents/{id}/summary        # 仅摘要
GET  /api/documents/{id}/content        # 下载原文文件

# 搜索
GET  /api/search?q=xxx&tag=yy&limit=20&offset=0

# 审核
GET  /api/inbox                         # 待审列表
POST /api/documents/{id}/approve        # 通过审核
POST /api/documents/{id}/reject         # 拒绝

# 关系图谱
GET  /api/documents/{id}/links          # 出边
GET  /api/documents/{id}/backlinks      # 入边
GET  /api/documents/{id}/neighbors      # 出边 + 入边
GET  /api/path?from=x&to=y              # 最短路径

# AI 对话
POST /api/chat                          # 发送消息（流式 SSE 响应）
GET  /api/chat/sessions                 # 会话列表
GET  /api/chat/sessions/{id}            # 会话历史
GET  /api/chat/sessions/{id}/export     # 导出会话

# OKF 互操作（导入 / 导出）
POST /api/export/okf              # body: { path, include_inbox?, include_archived?, id_strategy? }
                                  # 返回: { path, document_count, attachment_count, bytes }
POST /api/import/okf              # multipart: bundle path 或 zip 上传
                                  # body: { yolo?, merge_by?, allow_broken_links? }
                                  # 返回: { imported: N, updated: N, skipped: N, broken_links: N }
GET  /api/export/okf/status/<id>  # 异步导出进度（大 bundle 轮询）
```