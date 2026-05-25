# 01 — 概述

Vega 是一个本地知识库应用，分装为单一二进制。

## 核心理念

- **不做信息损失** — 原文原样存储，不解析、不改写、不向量化
- **三层渐进加载** — metadata → 摘要 → 原文，按需拉取，context window 零浪费
- **不需要 YAML front matter** — metadata 独立存储在 SQLite 中，来源为 CLI 参数或 LLM 自动提取
- **内置 AI Agent** — Core 本体支持 AI 对话，对话中动态更新 metadata 和文档关系
- **外部分发** — CLI 暴露 `get-metadata` / `get-summary` / `get-content` 等接口，供外部 coding agent 直接调用

## 灵感来源

参考 [WeKnora](https://github.com/Tencent/WeKnora) 的 Core + CLI 分离架构，但不使用 RAG/向量检索方案。