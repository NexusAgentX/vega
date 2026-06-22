# 06 — SQLite 数据库设计

文档关系采用**双链 / 知识图谱**模型：文档是节点，关系是有向边。

## 表结构

```sql
-- 文档表
CREATE TABLE documents (
    id TEXT PRIMARY KEY,            -- UUID
    type TEXT NOT NULL DEFAULT 'Document',  -- OKF type：Document / Playbook / Reference / ...
    title TEXT NOT NULL,
    description TEXT,               -- 一句话摘要（OKF description，对齐 summary 语义）
    tags TEXT NOT NULL DEFAULT '[]', -- JSON 数组
    status TEXT NOT NULL DEFAULT 'new',  -- new | reviewed | archived
    source_format TEXT,             -- md | docx | xlsx | pdf | ...
    summary TEXT,                   -- LLM 生成摘要（200 token 内）
    resource_uri TEXT,              -- 外部资源 URI（OKF resource，可选）
    file_path TEXT,                 -- 原文文件路径（本地或 S3 key）
    created TEXT NOT NULL,
    updated TEXT NOT NULL
);

-- 关系表（有向图）
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

-- 会话表
CREATE TABLE chat_sessions (
    id TEXT PRIMARY KEY,
    title TEXT,
    created TEXT NOT NULL,
    updated TEXT NOT NULL
);

-- 消息表
CREATE TABLE chat_messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES chat_sessions(id),
    role TEXT NOT NULL,             -- user | assistant | system
    content TEXT NOT NULL,
    metadata TEXT DEFAULT '{}',     -- JSON
    created TEXT NOT NULL
);
```

## FTS5 全文索引

```sql
-- 基础 FTS 索引（默认模式）
CREATE VIRTUAL TABLE documents_fts USING fts5(
    title, description, tags, summary, filename,
    content='documents',
    content_rowid='rowid'
);

-- 同步触发器（documents 变更时自动更新 FTS 索引）
CREATE TRIGGER documents_ai AFTER INSERT ON documents BEGIN
    INSERT INTO documents_fts(rowid, title, description, tags, summary, filename)
    VALUES (new.rowid, new.title, new.description, new.tags, new.summary, new.file_path);
END;
```

## 增强搜索（手动开启）

配置 `search.deep_extract: true` 后，入库时额外调用解析库从原文文件提取纯文本，写入 FTS 的 `content_text` 列，搜索时参与匹配。原文 BLOB 不做任何修改。

## 数据存储分布

| 数据 | 位置 | 格式 |
|---|---|---|
| Metadata + 摘要 + 关系 + 会话 | `~/.vega/vega.db` | SQLite |
| 原始文件 | `~/.vega/files/<uuid>` 或 S3 | 二进制原文 |