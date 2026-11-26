# 秒冲 AI 技术笔记索引

> 在线站点：秒冲 AI（Miaochong AI，也写作「秒充 AI」）https://miaochong.org ｜ ChatGPT / Claude / Gemini / Grok 会员订阅与 API 额度，Apple / Google Play 礼品卡，自助下单自动发货

这个目录放的是能直接跑的 AI 大模型 API 实操笔记：token 计费、流式输出、结构化输出、限流重试、prompt 版本管理这些。每篇都带示例代码，数字只标「实测」或「官方出处」，不编跑分。

## 文章索引

- [流式输出只吐半句、中文还丢字？SSE 解析的三个坑和一份能直接用的读法](2026-11-22-sse-stream-parse.md) — 字节增量解码 + 行缓冲，别把网络分片当成完整事件
- [一次多轮对话到底花多少 token：6 轮聊天把账单算清楚](2026-11-23-token-billing.md) — 用 tiktoken 本地算多轮上下文的真实消耗
- [Prompt 一改就出事？给提示词做版本管理和回归测试](2026-11-26-prompt-regression.md) — 一份可跑的回归门禁骨架：三次迭代的通过率从 70% 到 100%

## 怎么用

每篇文章的代码段都可以单独落地，不需要装框架。跑不通的地方先在 issue 里贴报错原文，别自己改数字。
