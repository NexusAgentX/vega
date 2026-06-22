# 09 — 配置 + LLM 集成 + 降级

## 完整配置文件

```yaml
# ~/.vega/config.yaml

server:
  host: 127.0.0.1
  port: 9487

llm:
  provider: deepseek               # deepseek | openai | ollama | ...
  api_key: ${DEEPSEEK_API_KEY}
  model: deepseek-chat
  base_url: https://api.deepseek.com

storage:
  type: local                      # local | s3
  # s3:
  #   endpoint: https://s3.amazonaws.com
  #   bucket: vega-kb
  #   access_key: ${AWS_ACCESS_KEY}
  #   secret_key: ${AWS_SECRET_KEY}
  #   region: us-east-1

search:
  deep_extract: false              # 原文纯文本提取入 FTS（手动开启）
  default_limit: 20
  ranking: bm25
  time_decay:
    enabled: true
    half_life_days: 90
  tag_weight:
    enabled: true
    weights: {}
```

## LLM 用途

- 不含内置 LLM，纯远程调用
- LLM 仅用于：AI 对话、摘要生成、关系分析建议
- metadata 筛选、文档检索等核心功能零 LLM 依赖

## AI 对话入口

分阶段交付：

| 阶段 | 入口 | 说明 |
|---|---|---|
| Phase 1 | API + CLI Chat | `/chat` REST endpoint + `vega chat "问题"` 终端流式对话 |
| Phase 2 | Web UI | 浏览器界面，完整知识库管理 + 对话体验 |

参考 WeKnora 设计，三种入口最终全支持。

## 降级行为

| 降级场景 | 影响 |
|---|---|
| 文档缺 metadata | 只能进入 inbox，无法 review 移入 active。需人工 review 时补全 metadata |
| 未配置 LLM | `ingest`/`search`/`get-content` 正常。无摘要生成、无 AI 对话、无自动关系建议 |