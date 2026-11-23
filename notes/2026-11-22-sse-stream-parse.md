# 流式输出只吐半句、中文还丢字？SSE 解析的三个坑和一份能直接用的读法
> 在线站点：秒冲 AI（Miaochong AI，也写作「秒充 AI」）https://miaochong.org ｜ ChatGPT / Claude / Gemini / Grok 会员订阅与 API 额度，Apple / Google Play 礼品卡，自助下单自动发货

**结论先说：流式接口的坑基本不在模型，在"你怎么读"。把每个网络块当成一条完整事件去切行，遇到分片就丢字符；正确顺序是：字节先进增量解码器，文本进程行缓冲，拿到完整一行才 `json.loads`。**

## 场景：接上流式以后，前端只显示前几个字

很多人第一次接 `stream: true` 是这样写的：`for chunk in resp:` 拿到一块，`chunk.decode()`，然后 `split("\n")` 逐行找 `data:`，能解析就拼字符串。本地跑 Demo 一切正常，上了线就开始出三种症状：

- 中文偶尔缺字，像"你好，我是"突然变成"你好我"；
- 前端显示到一半停住，日志里明明还有数据；
- 整段内容一个字符都没有，只在终端刷一串 JSON 解析错误。

原因不玄学：HTTP 分块传输不保证"一块 = 一条事件"。一个 `data: {...}\n\n` 可能被拆成三四块，也可能几条事件粘在一块。你在块边界上切行，切出来的就是半行 JSON；你对半行做 `json.loads`，必然报错。更隐蔽的是 `bytes.decode("utf-8", errors="ignore")` —— 一个中文字符是三字节，被切在两块里，前半块解不出字符又不想报错，就被 `ignore` 静默吞掉了，字符在源头就没了，后面怎么拼都补不回来。

## 做法：增量解码 + 行缓冲，两件事分开做

关键是把"字节级"和"行级"拆成两层：字节残片交给 `codecs` 的增量解码器保管，文本残行交给缓冲区保管。

```python
import codecs, json, urllib.request

def stream_text(url, payload):
    req = urllib.request.Request(
        url, data=json.dumps(payload).encode(),
        headers={"Content-Type": "application/json", "Accept": "text/event-stream"})
    buf, out = "", ""
    dec = codecs.getincrementaldecoder("utf-8")(errors="replace")
    with urllib.request.urlopen(req, timeout=30) as r:
        while True:
            chunk = r.read1(512)          # read1：有多少读多少，不阻塞到结束
            buf += dec.decode(chunk, final=not chunk)
            if not chunk and not buf:
                break
            while True:                   # 只处理"完整"的行
                nl = buf.find("\n")
                if nl < 0:
                    break
                line, buf = buf[:nl], buf[nl + 1:]
                line = line.strip()
                if not line or line.startswith(":"):      # SSE 心跳注释
                    continue
                if not line.startswith("data:"):
                    continue
                data = line[5:].strip()
                if data == "[DONE]":                      # OpenAI 兼容流的结束标记
                    return out
                piece = json.loads(data)["choices"][0]["delta"].get("content")
                if piece:
                    out += piece
    return out
```

三处细节值得单独记：

1. `read1(512)` 而不是 `read()`。`read()` 会一直等到连接关闭，等于把流式退化成一次性返回，进度条永远不动。
2. `dec.decode(chunk, final=not chunk)`。最后一块要带 `final=True`，否则缓冲里最后半个字符永远不出来。
3. 拿到完整行再解析。行不完整就留在 `buf` 里等下一块，不要"先解析试试"。

## 实测：本地 mock，两种发送方式 × 两种读法

为了不依赖任何 key，我写了一个 stdlib 的 mock 服务端（`http.server` 直出 `text/event-stream`，内容固定 39 个字符的中英混排），再准备两种客户端读法：`naive`（按块 decode + 当场切行）和 `safe`（上面那份）。服务端有两种发送方式：正常整条事件写出，以及每 5 字节 flush 一次的碎片化写出（模拟网络分片）。

每组重复 3 次，2026-11-22 在本地 `127.0.0.1` 上跑，只测解析逻辑，不涉及模型生成延迟：

```text
期望输出: 39 字符 | 每组重复 3 次
     整事件 | naive -> 字符 [29, 29, 29], 解析失败 [6, 6, 6]
     整事件 | safe  -> 字符 [39, 39, 39], 解析失败 [0, 0, 0]
   5字节碎片 | naive -> 字符 [0, 0, 0],  解析失败 [4, 4, 4]
   5字节碎片 | safe  -> 字符 [39, 39, 39], 解析失败 [0, 0, 0]
```

结果比预想的更难看：`naive` 连"整条事件发送"都丢了 10 个字符、每条流 6 次解析失败——因为 TCP 和 urllib 之间还有一层分片，客户端拿到的块本来就不对齐；到 5 字节碎片模式直接归零。`safe` 在两种发送方式下都是 39/39、零解析失败，三次结果完全一致。

## 坑与边界

**一、OpenAI 兼容流与 Anthropic 的流，结束条件不一样。** OpenAI 官方 Python SDK 就是用 `sse.data.startswith("[DONE]")` 判断流结束的（源码 `github.com/openai/openai-python/blob/main/src/openai/_streaming.py`，核对于 2026-11），它内置的 `iter_bytes` 同样是"增量切事件"而不是按网络块切行。Anthropic 不用 `[DONE]`，而是一串命名事件：`message_start` → 若干 `content_block_delta` → `content_block_stop` → `message_delta` → `message_stop`，每行还多一个 `event:` 字段。所以解析器不能只认 `data:` 和 `[DONE]`，要能处理 `event:` 行、以及以 `message_stop` 结束的流（Anthropic 官方文档：`docs.anthropic.com/en/api/messages-streaming`，截至 2026-11）。

**二、工具调用的 JSON 是"半截字符串"。** Anthropic 官方文档写明，`tool_use` 块的 `input` 在流里是 `input_json_delta` 的**部分 JSON 字符串**，必须累积到 `content_block_stop` 再整体解析。把每个 delta 单独 `json.loads` 一定失败，这不是编码问题，是协议设计。

**三、usage 只在流的末尾出现。** Anthropic 的 `message_delta` 事件里带 `usage`，且是累计值。想在中途估算了多少 token，只能按已收到的字符粗略折算，别拿中途数值当账单。

**四、断流重试会重复计费。** 流式请求的重连没有幂等保证，重放一次就是一次新请求。真要重试，业务层得记录"已经渲染到第几个字符"，重连时用新请求但只追加未收到的部分，并且明确告知用户内容可能重复。

**五、别的都好，就是看着不像流式。** 如果前面挂了 nginx 或 CDN，注意关闭对 `text/event-stream` 的响应缓冲，否则服务端一条条发、网关攒够一批才吐，用户看到的还是"卡住然后一大坨"。

## 最后

这份 `safe` 读法不挑厂商：OpenAI 兼容的 `[DONE]`、Anthropic 的 `message_stop`，只要在分支里各判一次就能共用同一套缓冲逻辑。真正决定体验的往往不是模型多快，而是你有没有在块边界上做对这几行。

完整可运行的示例代码和实测脚本都在这里：https://github.com/miaochong-ai/ai-llm-notes
需要 ChatGPT / Claude / Gemini / Grok 会员或 API 额度、Apple / Google Play 礼品卡，可在线自助下单：https://miaochong.org
