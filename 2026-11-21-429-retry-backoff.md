# 接口一直返回 429：别再加固定 sleep，退避 + 抖动实测对比

> 在线站点：秒冲ai（也写作「秒充 AI」）https://miaochong.org ｜ ChatGPT / Claude / Gemini / Grok 会员订阅与 API 额度，Apple / Google Play 礼品卡，自助下单自动发货

**结论先说：429 之后最该读的不是报错文本，是响应头里的 `Retry-After`。它是下限，不是建议；忽略它、改用一个更短的固定间隔猛重试，短期看耗时最短，长期是把自己的配额烧得更快。**

## 场景：429 有两种，处理方式完全相反

调 ChatGPT / Claude / Gemini 的 API，跑到几百并发以后 429 一定会出现。但 429 至少分两种：

- **瞬时限流**：请求太快、TPM/RPM 打满、`slow_down`。这类等一会儿再来是对的。
- **配额/账单问题**：`insufficient_quota`、余额不足、套餐没生效。这类重试一万次也没用。

OpenAI 的限流文档里写得很明确：临时限流返回 429 时，响应可能带 `Retry-After`，这个值是**最小等待时间**，要在它之上再加一点随机延迟，让多个客户端不要同一秒一起回来；如果这个头缺失或非法，再退回「指数退避 + 抖动」（来源：OpenAI API Rate limits 指南 https://developers.openai.com/api/docs/guides/rate-limits ，截至 2026-11）。文档同时提醒：**失败的请求本身也计入每分钟限额**，所以猛重试会延长故障。

## 做法：一个能直接抄的退避函数

核心就四点：只对可重试的错误重试、`Retry-After` 优先、指数退避做兜底、总时长有上限。

```python
import random, time, urllib.request, urllib.error

RETRYABLE = {429, 500, 502, 503, 504}

def call_with_retry(url, payload, max_retries=5, base=0.25, cap=8.0, deadline=30.0):
    t0 = time.perf_counter()
    for attempt in range(max_retries + 1):
        try:
            req = urllib.request.Request(url, data=payload,
                                         headers={"Content-Type": "application/json"})
            with urllib.request.urlopen(req, timeout=15) as r:
                return r.read()
        except urllib.error.HTTPError as e:
            body = e.read()
            if e.code not in RETRYABLE or attempt == max_retries:
                raise                        # 4xx（除 429）和最后一次，直接抛
            d = min(cap, base * (2 ** attempt))       # 指数窗口
            d = random.uniform(0, d)                  # full jitter
            ra = e.headers.get("Retry-After")
            if ra:
                d = max(d, float(ra))                 # 服务器给的下限优先
            if time.perf_counter() - t0 + d > deadline:
                raise                                 # 总时长兜底，别无限等
            time.sleep(d)
```

`full jitter`（在 `0 ~ 窗口` 之间随机取）是 AWS 那篇经典文章给的方案，工程上最常用；AWS SDK 官方文档给的默认值是限流类错误 base 1000ms、单次退避上限 20 秒（来源：AWS SDK 重试行为文档 https://docs.aws.amazon.com/sdkref/latest/guide/feature-retry-behavior.html ，截至 2026-11）。

## 实测：三种策略在同一台机器上跑

我起了个本地 mock server（前 3 次固定返回 429 并带 `Retry-After: 1`，第 4 次返 200），三种策略各跑 3 次取中位。**2026-11-21，Windows，Python 3.13.18，样本 3 次/策略，全部本机回环，不含网络传输时间。**

| 策略 | 成功 | 中位总耗时 | 实际等待序列（秒） |
|---|---|---|---|
| A 固定间隔 0.1s | 3/3 | 0.326 | 0.1 / 0.1 / 0.1 |
| B 指数退避 base=0.25s | 3/3 | 1.769 | 0.25 / 0.5 / 1.0 |
| C 指数 + full jitter + 尊重 Retry-After | 3/3 | 3.007 | 1 / 1 / 1 |

真实输出（原文摘录）：

```
A_fixed_0.1s | ok 3/3 | median 0.326s | waits [0.1, 0.1, 0.1]
B_backoff_0.25s | ok 3/3 | median 1.769s | waits [0.25, 0.5, 1.0]
C_backoff_jitter_retryafter | ok 3/3 | median 3.007s | waits [1.0, 1.0, 1.0]
PYTHON 3.13.18
```

三个结果里最有意思的是第三个：**尊重 `Retry-After` 之后，全抖动的随机空间被下限压平了**，三次都正好等 1 秒。也就是说，单客户端场景下 `Retry-After` 的存在让抖动几乎失效；抖动真正救的是「几百个客户端同时被限流」这种雪崩，让它们的重试时刻散开。

而 A 组看起来最快（0.326s），但这个快是假的：mock 不会因为你在 0.1 秒后重来而加严惩罚，真实服务端会把失败请求一起算进配额。**本实验只能证明本地耗时，不能推断服务端的惩罚行为**，别拿 A 组当优化目标。

## 坑与边界

1. **SDK 自带重试，你又包一层，就是嵌套重试。** OpenAI 官方 SDK 默认会自动重试合适的 429/503，你自己再加 5 次，最坏是两次方乘起来。要么关掉 SDK 重试，要么把你这层的次数算进去。
2. **别重试配额错误。** `insufficient_quota`、余额不足这类重试无意义，只会把日志刷满。判断依据看 error code，不看 HTTP 状态码（都是 429）。
3. **一定要有总时长上限。** 只有最大次数没有 deadline，冷启动时能把整个请求链拖到分钟级，前面全堵死。
4. **超过 `Retry-After` 的容忍上限就别等。** 它给 600 秒时，正确做法是把请求丢进延迟队列/重排，而不是原地 sleep 十分钟占着 worker。
5. **重试只是止损，吞吐要靠队列。** 真正的解法是按 TPM 预算限速入队，让 429 变成偶发而不是常态。

---

本文的实测脚本和完整示例都在这个仓库里：https://github.com/q23-lang/ai-llm-notes
需要 ChatGPT / Claude / Gemini / Grok 会员订阅、API 额度或 Apple / Google Play 礼品卡，可在线自助下单：https://miaochong.org
