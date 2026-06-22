# 07 — 搜索排序

三因子加权排序：

```
score = BM25 × time_decay × tag_weight
```

## BM25（FTS5 内置）

SQLite FTS5 从 3.31.0 起内置 `bm25()` 排序函数。

## 时间衰减

新文档权重高，旧文档自然衰减：

```
decay = 0.5 ^ (days_elapsed / half_life_days)
```

默认半衰期 90 天。

## Tag 权重

用户可为特定 tag 设置权重，匹配该 tag 的文档加权或降权：

```yaml
search:
  tag_weight:
    enabled: true
    weights:
      python: 2.0       # 高优先级
      fastapi: 1.5
      deprecated: 0.3   # 低优先级
```