---
title: "微博爬虫 Python 教程：官方 API、m.weibo.cn 接口与开源工具"
description: "用 Python 写微博爬虫的四种方法：微博开放平台 API、m.weibo.cn 接口、weiboSpider 等开源爬虫（多需 Cookie），以及无需登录的 weibo-scraper，附代码和限制对比。"
lang: zh-CN
---

# 微博爬虫 Python 教程：四种采集微博数据的方法

用 Python 采集微博数据，常见的路线有四条：微博开放平台的官方 API（要申请应用，能拿到的数据很少）、直接请求移动版 m.weibo.cn 背后的 JSON 接口（免费，但游客状态下翻不了几页，深度采集要带自己的 Cookie）、GitHub 上的开源微博爬虫（功能最全，多数需要登录 Cookie 并要自己维护），以及托管的按条计费服务，例如我们的 weibo-scraper（无需登录，按关键词或用户返回帖子）。下面按"能拿到什么、要准备什么、会在哪里卡住"逐一说明，并给出可以直接运行的代码。

## 先确定你要的数据

不同目标适合的方法差别很大，动手前先想清楚：

- **按关键词搜索帖子**（品牌监测、舆情、话题研究）：官方 API 基本不开放，免费路线要用网页版搜索接口或开源爬虫。
- **某个用户的全部微博**：游客只能看到最新一页，翻页需要登录，所以要带自己账号的 Cookie。
- **评论、转发链、粉丝列表**：几乎都需要登录状态，开源爬虫支持得最好。
- **热搜榜**：网页公开可见，但本文的托管工具不包含这一项。

## 方法一：微博开放平台官方 API

微博开放平台（[open.weibo.com](https://open.weibo.com)）提供 OAuth 2.0 授权的 REST API。流程是：注册开发者、创建应用、通过审核拿到 App Key 和 App Secret，再让用户授权得到 access_token。

它的问题在于数据范围。开放平台面向普通开发者的接口主要用于读写**授权用户自己**的内容（发微博、读取自己的时间线等），按关键词全站搜索、读取任意用户的历史微博这类能力属于商业数据接口，需要另行合作，不对个人开放。具体以开放平台当前文档为准。

适合：要做"用微博登录"或替用户发微博的应用。不适合：数据采集和舆情分析。

## 方法二：直接请求 m.weibo.cn 的 JSON 接口

微博移动版 m.weibo.cn 的页面是用 JSON 接口渲染的，在浏览器开发者工具的 Network 面板里就能看到。最常用的是用户时间线接口：

```
https://m.weibo.cn/api/container/getIndex?type=uid&value=<UID>&containerid=107603<UID>&page=<页码>
```

`containerid` 是固定前缀 `107603` 加上用户的数字 UID（主页链接 `https://weibo.com/u/1669879400` 里的数字就是 UID）。返回的 `data.cards` 里，`card_type` 为 9 的卡片就是微博，帖子内容在 `mblog` 字段。

下面是一个最小的 Python 示例。`COOKIE` 填你自己在浏览器登录 m.weibo.cn 后复制的 Cookie；不填时，接口可能返回跳转到访客验证的 HTML 而不是 JSON，或者只返回第一页。

```python
# pip install requests
import re
import time

import requests

UID = "1669879400"
COOKIE = ""  # 浏览器登录 m.weibo.cn 后，从开发者工具复制请求头里的 Cookie

headers = {
    "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 16_6 like Mac OS X) AppleWebKit/605.1.15 "
                  "(KHTML, like Gecko) Version/16.6 Mobile/15E148 Safari/604.1",
    "Referer": "https://m.weibo.cn/",
    "X-Requested-With": "XMLHttpRequest",
}
if COOKIE:
    headers["Cookie"] = COOKIE

for page in range(1, 4):
    r = requests.get(
        "https://m.weibo.cn/api/container/getIndex",
        params={"type": "uid", "value": UID, "containerid": f"107603{UID}", "page": page},
        headers=headers,
        timeout=20,
    )
    try:
        data = r.json()
    except ValueError:
        print("返回的不是 JSON（多半是访客验证页），请填写 Cookie")
        break
    if data.get("ok") != 1:
        print("第", page, "页没有数据：", data.get("msg"))  # ok:-100 表示需要登录
        break
    for card in data["data"]["cards"]:
        if card.get("card_type") != 9:
            continue
        mblog = card["mblog"]
        text = re.sub(r"<[^>]+>", "", mblog["text"])  # 去掉 HTML 标签
        print(mblog["created_at"], mblog["attitudes_count"], text[:60])
    time.sleep(3)  # 放慢请求，避免被限流
```

需要知道的几点：

- **翻页要登录。** 游客状态下第 2 页起接口返回 `ok: -100`，即需要登录。想拿一个用户的完整历史，必须带自己账号的 Cookie。
- **长微博会被截断。** `isLongText` 为 true 时，`text` 只有前一部分，全文要再请求 `https://m.weibo.cn/statuses/extend?id=<微博ID>`。
- **时间格式不统一。** `created_at` 可能是"刚刚""5分钟前"或英文日期格式，要自己统一转换。
- **关键词搜索不在这个接口里。** 网页版搜索由 weibo.com 的 `ajax/statuses/search` 接口提供，每个关键词大约只能翻到 1,000 条左右。
- **接口没有文档，随时会变。** 字段改名或访客机制调整后，代码就要跟着改。请求频率放低，Cookie 只用你自己的账号。

## 方法三：GitHub 上的开源微博爬虫

如果需要评论、转发、完整用户历史，开源项目比自己从零写省事得多。常被提到的有：

- **[dataabc/weiboSpider](https://github.com/dataabc/weiboSpider)**：抓取指定用户的微博和用户信息，可输出 CSV、JSON、MySQL 等，需要在配置里填 Cookie。
- **[dataabc/weibo-crawler](https://github.com/dataabc/weibo-crawler)**：同一作者基于 m.weibo.cn 接口的版本，按用户抓取微博、图片和视频，Cookie 是可选配置，但带 Cookie 才能翻过游客可见的范围。
- **[nghuyong/WeiboSpider](https://github.com/nghuyong/WeiboSpider)**：基于 Scrapy，覆盖用户、微博、评论、转发、关键词搜索等，同样需要 Cookie。

使用前的注意事项：

- **大多需要登录 Cookie。** Cookie 会过期，长期运行要定期更换；高频抓取可能影响你自己的微博账号。
- **看最近更新时间。** 微博接口一改，旧版本就会失效。选一个最近仍在维护的项目，并先读它的 issue 列表。
- **自己负责运行环境。** 部署、定时任务、存储、失败重试都要自己处理。

## 方法四：托管的微博爬虫服务（无需登录）

如果你只需要按关键词或用户拿公开帖子，不想维护 Cookie 和代码，可以用托管服务。Apify Store 上有多个微博爬虫，下面介绍我们的这一个。

Disclosure: Data Gleaner 就是我们。[Weibo Scraper](https://apify.com/datagleaner/weibo-scraper) 运行在 Apify 上，用任何首次访问者都会拿到的匿名访客会话请求公开数据，不需要微博账号、Cookie 或浏览器。每条帖子输出正文（已去 HTML）、ISO 8601 时间（北京时间 +08:00）、转发/评论/点赞数、图片、视频、发布地区、来源客户端、作者信息和被转发的原微博，长微博会自动补全全文。可导出 JSON、CSV 或 Excel。

价格是按条计费，每 1,000 条帖子 3 美元（每条 0.003 美元），只为实际保存的帖子付费。

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/weibo-scraper").call(
    run_input={
        "searchQueries": ["瑞幸"],
        "maxItemsPerQuery": 10,
        "sinceDate": "2026-10-01",
    }
)
if run is None:
    raise SystemExit("运行没有返回，请在 Apify Console 查看")

for item in client.dataset(run.default_dataset_id).iterate_items():
    author = (item.get("author") or {}).get("screenName")
    text = (item.get("text") or "").replace("\n", " ")[:80]
    print(item.get("createdAt"), item.get("likesCount"), author, text)
```

这次运行最多 10 条，约 0.03 美元。主要输入参数：

| 参数 | 作用 |
|---|---|
| `searchQueries` | 关键词列表，中英文均可 |
| `userIds` | 数字 UID 或主页链接，如 `https://weibo.com/u/1669879400` |
| `maxItemsPerQuery` | 每个关键词或用户的上限，默认 100，最多 1,000 |
| `sinceDate` | 只要此日期之后的帖子，如 `2026-10-01` |
| `includeUserProfile` | 补全作者粉丝数、简介、认证信息等，每个作者多一次请求 |

它的限制也要说清楚：每个关键词最多约 1,000 条（微博搜索本身的上限）；按用户抓取时，游客只能看到最新一页（约 10 条），要持续跟踪一个账号，需要定时运行并按 `id` 去重；不提供评论、粉丝列表和热搜榜。需要这些数据时，开源爬虫加自己的 Cookie 更合适。

## 四种方法对比

| 方法 | 需要登录 / Cookie | 关键词搜索 | 用户完整历史 | 评论 | 成本 | 维护 |
|---|---|---|---|---|---|---|
| 官方开放平台 API | 需要应用审核和用户授权 | 商业接口，不对个人开放 | 仅授权用户自己 | 有限 | 免费（基础接口） | 低 |
| m.weibo.cn 接口自写 | 翻页需要 Cookie | 需另用搜索接口 | 带 Cookie 可以 | 带 Cookie 可以 | 免费 | 高，接口会变 |
| 开源爬虫 | 大多需要 Cookie | 部分支持 | 可以 | 部分支持 | 免费 | 中，等作者更新 |
| weibo-scraper（我们） | 不需要 | 每词约 1,000 条 | 仅最新约 10 条 | 不支持 | 每 1,000 条 3 美元 | 无需自己维护 |

## 合规提醒

只采集公开可见的内容，遵守微博的用户协议，把请求频率控制在较低水平。微博帖子和用户资料属于个人信息，用于分析或存储时，你需要有合法依据并遵守《个人信息保护法》以及其他适用的法律。本文不构成法律意见。

## 常见问题

**微博爬虫一定要 Cookie 吗？**
不一定。游客可以看到关键词搜索结果和用户主页的第一页，所以只要公开帖子、不需要深翻用户历史时，可以不登录。翻用户第 2 页以后、抓评论和粉丝列表，基本都需要登录 Cookie。

**微博爬虫在 GitHub 上有哪些？**
常用的有 dataabc/weiboSpider、dataabc/weibo-crawler 和 nghuyong/WeiboSpider。选之前看一下最近的提交时间和 issue，微博接口变动后，长期没人维护的项目通常已经失效。

**m.weibo.cn 返回 ok:-100 是什么意思？**
表示这个请求需要登录。用户时间线在游客状态下第 2 页起就会返回这个值，换成你自己登录后的 Cookie 才能继续翻页。

**一个关键词最多能爬多少条微博？**
微博网页搜索每个关键词大约只返回 25 页左右，也就是 1,000 条上下，不管话题有多大。要拿更多，可以把话题拆成几个更具体的关键词，或者每天定时抓取新增内容。

**有没有不用写代码的微博爬虫工具？**
有。托管服务（例如 Apify Store 上的微博爬虫）可以在网页上填关键词直接运行，再导出 CSV 或 Excel。

## 相关指南

- [Weibo API in Python（英文）](weibo-api-python)：官方 API 与其他接口的英文说明
- [B 站弹幕爬虫](bilibili-danmaku-scraper-zh)
- [微信公众号文章爬虫](wechat-official-account-articles-scraper)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "微博爬虫一定要 Cookie 吗？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "不一定。游客可以看到关键词搜索结果和用户主页的第一页，所以只要公开帖子、不需要深翻用户历史时，可以不登录。翻用户第 2 页以后、抓评论和粉丝列表，基本都需要登录 Cookie。"
      }
    },
    {
      "@type": "Question",
      "name": "微博爬虫在 GitHub 上有哪些？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "常用的有 dataabc/weiboSpider、dataabc/weibo-crawler 和 nghuyong/WeiboSpider。选之前看一下最近的提交时间和 issue，微博接口变动后，长期没人维护的项目通常已经失效。"
      }
    },
    {
      "@type": "Question",
      "name": "m.weibo.cn 返回 ok:-100 是什么意思？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "表示这个请求需要登录。用户时间线在游客状态下第 2 页起就会返回这个值，换成你自己登录后的 Cookie 才能继续翻页。"
      }
    },
    {
      "@type": "Question",
      "name": "一个关键词最多能爬多少条微博？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "微博网页搜索每个关键词大约只返回 25 页左右，也就是 1,000 条上下，不管话题有多大。要拿更多，可以把话题拆成几个更具体的关键词，或者每天定时抓取新增内容。"
      }
    },
    {
      "@type": "Question",
      "name": "有没有不用写代码的微博爬虫工具？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "有。托管服务（例如 Apify Store 上的微博爬虫）可以在网页上填关键词直接运行，再导出 CSV 或 Excel。"
      }
    }
  ]
}
</script>
