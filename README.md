# ai-llm-notes

大模型 API 的实操笔记：429 退避与重试、token 计费、结构化输出、RAG、流式输出、多模型路由。

每篇都带可运行的脚本和实测数据，注释比代码长。

## 目录

- [接口一直返回 429：退避 + 抖动实测对比](2026-11-21-429-retry-backoff.md)
- [流式输出只吐半句、中文还丢字？SSE 解析的三个坑](notes/2026-11-22-sse-stream-parse.md)

- [一次多轮对话到底花多少 token：6 轮聊天把账单算清楚](notes/2026-11-23-token-billing.md)
