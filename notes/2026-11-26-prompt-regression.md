# Prompt 一改就出事？给提示词做版本管理和回归测试，附可跑的 Python 代码

> 在线站点：秒冲 AI（Miaochong AI，也写作「秒充 AI」）https://miaochong.org ｜ ChatGPT / Claude / Gemini / Grok 会员订阅与 API 额度，Apple / Google Play 礼品卡，自助下单自动发货

一句话结论：prompt 不是文案，是生产代码。只要做三件事——把 prompt 落成带哈希的文件、备一组写死预期结果的回归用例、让脚本用退出码表达「过/不过」，你就能在改 prompt 之前知道它会让什么变坏。

## 问题：改提示词最容易"修一个坏三个"

我的场景很土：客服工单分类，把人一句话分到 退款 / 开通 / 额度 / 封号 / 其它 五类之一，下游按类别走不同话术。

第一版 prompt 上线两周后，运营说"付了钱没到账"这种问法老被分错，要求补规则。问题在于：这次补完，原来判对的会不会被弄错？如果只靠"上线后盯两天"来验证，坏的那批工单已经回给用户了。

所以我改成三步：prompt 存文件、每次改动记哈希、改完先跑 20 条固定用例，不全过就不上线。

## 做法：三个文件加一段脚本

目录就长这样，没有框架：

```
prompt-versioning/
├── prompt_regression.py
├── cases.jsonl
└── prompts/
    ├── v1.md
    ├── v2.md
    └── v3.md
```

用例是一行一条 JSON，预期结果写死，谁改用例谁就得在评审里解释：

```json
{"text": "付了钱会员没到账，怎么办", "expect": "开通"}
{"text": "刚买的能退款吗", "expect": "退款"}
{"text": "会员冻结了怎么恢复", "expect": "封号"}
```

脚本核心只有三块：读 prompt（顺手算 sha1）、跑用例、用退出码给结论。

```python
def load_rules(prompt_path):
    rules = {}
    for line in prompt_path.read_text(encoding="utf-8").splitlines():
        line = line.strip()
        if line.startswith("- ") and "=>" in line:
            kws, label = line[2:].split("=>", 1)
            rules[label.strip()] = [k.strip() for k in kws.split("|") if k.strip()]
    return rules

def sha8(p):
    return hashlib.sha1(p.read_bytes()).hexdigest()[:8]

# 模型调用层单独一层，换成真 API 时只动这里
def mock_model(rules, text):
    best, best_hits = "其它", 0
    for label, kws in rules.items():
        hits = sum(1 for kw in kws if kw in text)
        if hits > best_hits:
            best, best_hits = label, hits
    return best

ok = len(cases) - len(fails)
print(f"prompt: {args.prompt}  sha1:{sha8(prompt_path)}  cases:{len(cases)}")
print(f"PASS {ok}  FAIL {len(fails)}  pass_rate {ok/len(cases):.1%}")
return 0 if not fails else 1   # 0 全过 / 1 有失败，CI 靠这个卡
```

关键是最后那行 return。人看报表容易漏，流水线只认退出码：非 0 就不合并，这一条比任何"上线后观察"都硬。

## 实测结果

环境：2026-11-26，Windows 本机，Python 3.13.18，20 条用例。模型这一层用的是本地 mock——按 prompt 里声明的关键词表打分，**不是真实模型调用**，所以下面的数字验证的是"版本管理 + 回归门禁"这条流水线，不是模型准确率。

v1（原线上版）：

```
prompt: prompts/v1.md  sha1:c7687979  rules:4  cases:20
PASS 14  FAIL 6  pass_rate 70.0%  elapsed 0.0ms
  FAIL #1 expect=开通 got=其它 | 付了钱会员没到账，怎么办
  FAIL #5 expect=退款 got=其它 | 自动续费扣了钱，我想取消
  FAIL #13 expect=开通 got=其它 | 付款成功但一直未到账
```

v2（补了"没到账/升级/多少钱/下单/冻结"等问法）：

```
prompt: prompts/v2.md  sha1:602e6592  rules:4  cases:20
PASS 19  FAIL 1  pass_rate 95.0%  elapsed 0.1ms
  FAIL #5 expect=退款 got=其它 | 自动续费扣了钱，我想取消
```

v3（再补"取消/自动续费"两个词）：

```
prompt: prompts/v3.md  sha1:607a492f  rules:4  cases:20
PASS 20  FAIL 0  pass_rate 100.0%  elapsed 0.1ms
```

三次改动全程不到十分钟，而且每一次的分数和哈希都留了痕：谁想知道"三天前那次改 prompt 到底影响了什么"，翻日志就能对上 sha1。耗时 0.1ms 说明卡点根本不在脚本，在真实模型调用上——真跑 API 的时候，20 条用例的成本和延迟按厂商定价另算，不要拿这个 0.1ms 当参考。

## 坑与边界

1. 100% 只是"没退化"，不是"变好了"。20 条样本内全过，只能说明这次改动没打破旧行为。要谈泛化，得留一份 holdout 用例，或者每周从真实工单里抽新样本补进集子，别让用例集跟线上话术一起老化。
2. mock 会骗人。它按关键词打分，对否定句、反问句几乎没感觉（"我没想退款，只是想改地址"这种），真实模型在这类句子上更容易翻车。mock 的作用是让流水线可复现、可 CI，接真模型时把 mock_model 换成一次 API 调用即可，别拿 mock 的分数对外说模型效果。
3. 只哈希 prompt 不够。同一个 prompt 换模型、换温度、换系统消息，结果都可能变。版本信息里至少要把 prompt 哈希 + 模型名 + 关键参数一起记，否则复现不了。
4. 用例的预期本身也会错。上面 #5「自动续费扣了钱，我想取消」两版都判成"其它"，跑完才发现是我们的类别定义里压根没有"续费/取消"这一档——回归测试的第一份产出经常不是修 prompt，而是发现自己的分类标准有坑。
5. 别把 prompt 写成规则表然后跟真实模型比。规则表是为了让本地 harness 可复现；真实 prompt 应该写判定标准和边界例子，剩下的交给模型判断。

一句话收尾：prompt 的价值不在于写得多漂亮，而在于每次改动都能说出"这次动了什么、坏了没有"。能做到这点，就已经比大多数把 prompt 存在数据库字段里的项目靠前了。

---

本文的实测脚本和完整示例都在这个仓库里：https://github.com/miaochong-ai/ai-llm-notes
需要 ChatGPT / Claude / Gemini / Grok 会员订阅、API 额度或 Apple / Google Play 礼品卡，可在线自助下单：https://miaochong.org
