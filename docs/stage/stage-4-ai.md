# Stage 4 — AI 增强的 Vega

## 是什么 Vega

接入 LLM（litellm），ingest 时自动生成摘要、自动提取 metadata（title/tags/type）。Vega 从「手动填字段」升级为「丢文件进去，自动整理好」。这是用户体验的第一个质变 —— 入库成本骤降，知识库才能滚起来。

## 新增能力

- litellm 集成（DeepSeek / OpenAI / Ollama / 其他 provider）
- ingest 时未提供 `summary` / `title` / `tags` / `description` 时，LLM 自动分析原文生成
- 配置 LLM provider / model / api_key
- LLM 失败时降级到 stage 3 的截取版（核心功能零 LLM 依赖）

## 前置依赖

[stage 3](./stage-3-progressive.md) 必须完成。

## 交付清单

### LLM Bridge 模块

新建 `vega/core/llm/`：

```
vega/core/llm/
├── __init__.py
├── bridge.py               # litellm 统一调用入口
├── summarizer.py           # 替换 stage 3 的截取版
├── metadata_extractor.py   # 从原文提取 title/tags/type/description
└── prompts/                # 模板（system prompts）
    ├── summarize.md
    ├── extract_metadata.md
    └── infer_type.md
```

### 调用流程

ingest 时的自动填充优先级：

```
字段 X 缺失？
  ├─ 用户显式提供（--tag/--title/...）→ 用提供的值
  ├─ 未提供 + LLM 启用 → LLM 分析原文生成
  └─ 未提供 + LLM 未配置/失败 → 降级（stage 3 截取版 / 文件名推导）
```

降级路径**必须**存在，保证未配置 LLM 的环境 Vega 仍可用（对照 [09-config](../09-config.md) 降级表）。

### 配置扩展

```yaml
llm:
  provider: deepseek               # deepseek | openai | ollama | ...
  api_key: ${DEEPSEEK_API_KEY}
  model: deepseek-chat
  base_url: https://api.deepseek.com

storage:
  type: local
  path: ~/.vega/files

server:
  host: 127.0.0.1
  port: 9487

search:
  default_limit: 20

okf:
  default_type: Document
```

环境变量 `DEEPSEEK_API_KEY` / `OPENAI_API_KEY` 优先于配置文件。

### CLI 行为

`vega ingest` 默认启用 LLM 自动提取（若配置了 LLM）。新增开关：

```
vega ingest <file>
    [--no-llm]                     # 跳过 LLM，走截取版（stage 3 行为）
    [--llm-only summary|metadata]  # 仅生成摘要 / 仅提取 metadata（调试用）
```

### 文件清单

- `vega/core/llm/` — 整个新建
- `vega/core/engine/summarizer.py` — 改为「LLM 优先，失败降级到截取」
- `vega/cli/commands/ingest.py` — 加 `--no-llm` / `--llm-only`
- `vega/shared/config.py` — 加 llm 段加载 + 环境变量替换 `${VAR}`
- `tests/test_stage4_llm.py` — mock litellm 调用
- `tests/test_stage4_fallback.py` — LLM 失败降级

## 验收标准

### LLM 摘要

```bash
# 1. 未配置 LLM，走截取
unset DEEPSEEK_API_KEY
vega ingest ./doc.md
# 期望: summary 是截取版（原文字符）

# 2. 配置 LLM，自动生成
export DEEPSEEK_API_KEY=sk-...
vega ingest ./doc.md
# 期望: summary 是 LLM 生成（不同于原文前缀，语义总结）

# 3. 显式跳过
vega ingest ./doc.md --no-llm
# 期望: 走截取版

# 4. LLM 失败降级（断网）
vega ingest ./doc.md
# 期望: 自动降级到截取版，stderr 打印警告，不报错
```

### Metadata 自动提取

```bash
# 5. 完全自动（无任何 --tag/--title）
vega ingest ./fastapi-tutorial.md
vega get-metadata <id> | jq .
# 期望: title 是 LLM 从内容提炼（"FastAPI 教程"之类）
#       tags 含 ["python", "fastapi", "web"] 之类
#       type 是 "Document" 或 LLM 推断（如 "Tutorial"）

# 6. 用户提供的字段不被 LLM 覆盖
vega ingest ./doc.md --tag my-custom-tag --title "My Title"
vega get-metadata <id> | jq '.tags, .title'
# 期望: ["my-custom-tag"], "My Title"
```

### 多 provider 切换

```bash
# 7. OpenAI provider
export OPENAI_API_KEY=sk-...
# 修改 config.yaml: provider: openai, model: gpt-4o-mini
vega ingest ./doc.md
# 期望: 正常生成摘要

# 8. Ollama 本地
# config: provider: ollama, base_url: http://localhost:11434
vega ingest ./doc.md
# 期望: 本地模型生成
```

### 单元测试

- mock litellm 侧 LLM 返回固定文本 → ingest 后 summary 等于该文本
- LLM 抛异常 → 降级路径触发、summary 是截取版、不抛错
- 用户显式字段优先于 LLM 输出
- 多 provider 配置（deepseek/openai/ollama）参数正确传递给 litellm

## 不做的事

- ❌ LLM 关系建议（`related` / `extends` 等推荐） — stage 5 的规则版 + stage 6 的 LLM 版
- ❌ AI 对话 / chat SSE — stage 7
- ❌ 热更新（对话中改 metadata） — stage 7
- ❌ 自定义 prompt 模板（用内置模板） — 未来路线
- ❌ streaming 摘要生成（ingest 是后台任务，streaming 无意义） — N/A
- ❌ `type` / `resource_uri` 字段（`type` 已在 stage 1 引入，`resource_uri` 留 stage 8） — N/A

## 完成判定

用户 `vega ingest <file>` 一条命令（无任何 metadata 参数），Vega 自动整理好 title / tags / type / summary。入箱成本从「填表」降到「丢文件」。**这是 Vega 知识库能持续滚起来的关键** —— 用户不用每次手动整理。
