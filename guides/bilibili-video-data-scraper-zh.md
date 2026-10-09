---
title: "B站视频数据爬取：按关键词搜索，播放量、点赞、投币导出 CSV"
description: "2026 年 B站视频爬取教程：用 wbi 签名的搜索接口按关键词抓视频，再用 view 接口补上投币和分享数，按播放量或最新排序，导出 CSV，附免费 Python 代码。"
lang: zh-CN
---

# B站视频数据爬取：按关键词搜索，抓播放量、点赞和投币

B站视频数据爬取的做法是：先请求 `api.bilibili.com/x/web-interface/wbi/search/type` 这个带 wbi 签名的搜索接口，用 `keyword` 传关键词，用 `order=click` 按播放量排序（`order=pubdate` 是按最新发布），每页返回 20 条视频；搜索结果里有播放量、点赞、收藏和弹幕数，但没有投币数，所以要对每个 BV 号再请求一次同样带签名的 `x/web-interface/wbi/view` 接口，从 `data.stat` 里读到 `coin`。不需要登录，不需要 Cookie，用 `requests` 写七十来行 Python 就能把结果存成 CSV。下面是可以直接运行的完整代码、字段含义和这条路线的限制。

Disclosure: 文末提到的 Bilibili Scraper 是我们（Data Gleaner）做的托管方案。前面的方法全部免费，不依赖我们的产品。

网上搜得到的 B站爬虫教程大多是几年前的 CSDN、博客园文章，代码里的接口已经失效，有的作者自己也在文中注明内容已不再有效，并提醒 B站会封 IP。原因主要有两个：搜索接口在 2023 年之后要求 wbi 签名，没有签名的老写法现在拿不到数据；而且搜索结果不含投币数，老教程也没讲怎么补。如果你要抓的是弹幕，请看[弹幕爬取教程](bilibili-danmaku-scraper-zh)，本文只讲视频搜索和视频数据。

## 先弄清三件事

**1. 搜索接口需要 wbi 签名。** 请求要多带两个参数：`wts`（当前 Unix 时间戳）和 `w_rid`（把排好序的参数串拼上一个密钥后算出的 MD5）。密钥来自 `x/web-interface/nav` 接口返回的两个图片文件名，经过一张固定的 64 位置换表打乱后取前 32 位。这个接口在未登录时会返回 `code: -101`，但 `data.wbi_img` 依然存在，所以照样读取就行。密钥会轮换，建议每次运行时重新获取。签名方案由社区项目 bilibili-API-collect 整理，更多细节也可以看[B站 API 的 Python 用法](bilibili-api-python)。

**2. 搜索结果里没有投币数。** 我们在 2026 年 10 月 9 日实测，搜索接口返回的每条结果有 `play`（播放）、`like`（点赞）、`favorites`（收藏）、`danmaku`（弹幕）、`review`（评论数）、`pubdate`（发布时间戳）、`author` 和 `tag` 等字段，没有投币字段。要拿到投币，必须再请求 view 接口。view 接口的 `stat` 里有播放、弹幕、评论、收藏、投币、分享、点赞等完整数据，代价是每个视频多一次请求。

**3. 排序参数。** `order` 常用的取值是：`totalrank`（综合排序，默认）、`click`（最多播放）、`pubdate`（最新发布）、`dm`（最多弹幕）、`stow`（最多收藏）。翻页用 `page`，从 1 开始。

## 方法一：免费，自己写 Python（requests）

下面的脚本按关键词搜索，对前 `LIMIT` 条结果补查 view 接口得到投币数，最后把所有结果存成 CSV。我们在 2026 年 10 月 9 日用关键词"露营装备"、按播放量排序实测可以运行；修改 `KEYWORD`、`ORDER` 和 `PAGES` 即可换成你自己的搜索。

```python
# pip install requests
import csv
import hashlib
import re
import time
import urllib.parse

import requests

KEYWORD = "露营装备"
ORDER = "click"      # click 播放量 / pubdate 最新 / totalrank 综合 / dm 弹幕 / stow 收藏
PAGES = 1            # 每页 20 条
LIMIT = 5            # 前几条补查投币（每个视频多一次请求），其余行的 coins 为空

API = "https://api.bilibili.com"
MIXIN = [46, 47, 18, 2, 53, 8, 23, 32, 15, 50, 10, 31, 58, 3, 45, 35, 27, 43, 5, 49, 33, 9, 42, 19, 29, 28, 14, 39,
         12, 38, 41, 13, 37, 48, 7, 16, 24, 55, 40, 61, 26, 17, 0, 1, 60, 51, 30, 4, 22, 25, 54, 21, 56, 59, 6, 63,
         57, 62, 11, 36, 20, 34, 44, 52]

s = requests.Session()
s.headers.update({"User-Agent": "Mozilla/5.0", "Referer": "https://www.bilibili.com/"})


def wbi_key():
    # nav 未登录时 code 是 -101，但 wbi_img 仍然存在
    img = s.get(f"{API}/x/web-interface/nav", timeout=15).json()["data"]["wbi_img"]
    stem = lambda u: u.rsplit("/", 1)[1].split(".")[0]
    raw = stem(img["img_url"]) + stem(img["sub_url"])
    return "".join(raw[i] for i in MIXIN)[:32]


def sign(params, key):
    p = dict(params, wts=int(time.time()))
    p = {k: re.sub(r"[!'()*]", "", str(v)) for k, v in sorted(p.items())}
    p["w_rid"] = hashlib.md5((urllib.parse.urlencode(p) + key).encode()).hexdigest()
    return p


def search(keyword, key, order="click", pages=1):
    out = []
    for page in range(1, pages + 1):
        r = s.get(f"{API}/x/web-interface/wbi/search/type", timeout=15, params=sign(
            {"search_type": "video", "keyword": keyword, "order": order, "page": page}, key)).json()
        if r.get("code") != 0:
            print("stopped:", r.get("code"), r.get("message"))
            break
        items = (r.get("data") or {}).get("result") or []
        if not items:      # B站被限流时有时返回空结果而不是报错
            break
        out += items
        time.sleep(2)
    return out


def view_stat(bvid, key):
    r = s.get(f"{API}/x/web-interface/wbi/view", timeout=15, params=sign({"bvid": bvid}, key)).json()
    return r["data"]["stat"] if r.get("code") == 0 else None


key = wbi_key()
hits = search(KEYWORD, key, ORDER, PAGES)
rows = []
for i, v in enumerate(hits):
    stat = None
    if i < LIMIT:
        stat = view_stat(v["bvid"], key)
        time.sleep(2)
    rows.append({
        "bvid": v["bvid"],
        "title": re.sub(r"<[^>]+>", "", v["title"]),   # 标题里带 <em> 高亮标签，去掉
        "author": v["author"],
        "plays": v["play"],
        "likes": stat["like"] if stat else v.get("like"),
        "coins": stat["coin"] if stat else None,
        "favorites": stat["favorite"] if stat else v.get("favorites"),
        "published": time.strftime("%Y-%m-%d", time.gmtime(v["pubdate"])),
        "url": f"https://www.bilibili.com/video/{v['bvid']}",
    })

if rows:
    with open("bilibili_videos.csv", "w", newline="", encoding="utf-8-sig") as f:
        w = csv.DictWriter(f, fieldnames=list(rows[0]))
        w.writeheader()
        w.writerows(rows)
    print(f"saved {len(rows)} rows")
    for row in rows:
        print(row["plays"], row["likes"], row["coins"], row["title"])
```

几点说明：

- CSV 用 `utf-8-sig` 编码保存，是为了让 Excel 直接打开时中文不乱码。
- 标题里的 `<em class="keyword">` 是搜索高亮标签，脚本用正则去掉了。
- 发布时间用 UTC 日期显示；B站页面显示的是北京时间，临近午夜的视频可能差一天。
- 如果只需要播放、点赞、收藏数，把 `LIMIT` 设成 0，不请求 view 接口，速度快很多；要每条都有投币，把 `LIMIT` 设成 `PAGES * 20`。
- 想要最新发布的视频，把 `ORDER` 换成 `pubdate`。想要更多结果，增大 `PAGES`。

## 这条路线的限制

- **搜索结果有上限。** 同一个关键词最多翻到大约 50 页，也就是约 1,000 条。想覆盖更多视频，要换不同的关键词，或者换 `order` 各抓一遍后按 BV 号去重。
- **投币要多一次请求。** 抓 1,000 条视频的投币就是 1,000 次 view 请求。上面的脚本每次请求后停 2 秒，所以 1,000 条大约要半个多小时。
- **限流不一定报错。** B站有时对过快的请求返回空结果而不是错误码，脚本遇到空页就停下，你需要自己判断是真的没有更多结果，还是被限流了。也可能收到 HTTP 412 之类的拦截。出现这种情况就放慢速度，隔一阵再试；从机房或云服务器的 IP 发请求，比家用宽带更容易被拦。
- **接口随时可能变。** 这些都是网站自己用的非公开接口，B站改签名方式或字段时，代码就要跟着改。
- **只能拿公开数据。** 播放、点赞、投币、收藏这些公开数字可以拿，需要登录才能看的内容不行。

## 方法二：用现成的库

社区库 bilibili-api-python 封装了搜索和视频信息接口，免费，但它是异步写法，依赖较多，遇到接口变动要等库更新。具体用法和实测情况见[B站 API 的 Python 用法](bilibili-api-python)。

## 方法三：托管的 Bilibili Scraper（无需 Cookie）

如果不想自己维护签名、重试和限流处理，或者要一次抓很多关键词，可以用 Apify Store 上的 [Bilibili Scraper](https://apify.com/datagleaner/bilibili-scraper) 跑成托管任务。Disclosure: Data Gleaner 是我们。

根据它的文档：

- 按关键词搜索视频，`searchOrder` 可选 `totalrank`（综合）、`click`（最多播放）、`pubdate`（最新）、`dm`（最多弹幕）、`stow`（最多收藏）和 `scores`（最多评论）；`maxItems` 设置每个关键词抓多少条，上限 1,000，默认 30。
- 搜索结果本身不带投币和分享数。打开 `enrichSearchResults`，它会为每个搜索结果补查完整记录（每个视频多 2 次请求），投币、分享、准确分区和完整标签就都有了；也可以把链接、BV 号或 av 号放进 `videoUrls`，直接得到完整记录。
- 还可以按视频时长（`durationFilter`）和发布时间范围（`publishedAfter`、`publishedBefore`，支持 `7 days` 这样的相对时间）筛选，也能一并抓评论、回复和弹幕。
- 已内置 wbi 签名、会话刷新和限流重试，不需要登录或 Cookie。结果可导出 JSON、CSV 或 Excel。
- 计费按事件：视频每 1,000 条 8 美元（每条 0.008 美元），评论每 1,000 条 2 美元，弹幕每 1,000 条 0.50 美元。只为实际交付的数据付费，达到你设的最高费用时会正常停止。文档里的例子：5 个视频的搜索花费 0.04 美元。

下面的例子按播放量排序抓 50 个视频，并补全投币数：

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/bilibili-scraper").call(run_input={
    "searchKeywords": ["露营装备"],
    "searchOrder": "click",
    "maxItems": 50,
    "enrichSearchResults": True,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    s = item["stats"]
    print(s["views"], s["likes"], s["coins"], item["title"], item["url"])
```

每个视频记录里 `stats` 包含 `views`、`danmaku`、`comments`、`likes`、`coins`、`favorites` 和 `shares`，另有 `bvid`、`title`、`tags`、`category`、`durationSec`、`publishedAt` 和 `owner`。不开 `enrichSearchResults` 时，`coins` 和 `shares` 是 `null`。

它的限制和上面的自写脚本一样，来自 B站本身：同一关键词最多约 1,000 条结果；B站限流很严，从数据中心 IP 发起的超大规模任务仍可能被拦，遇到时可以调大 `requestDelaySecs` 或加代理，已抓到的数据会保留；已删除或有地区限制的视频会被跳过；暂不支持抓取某个 UP 主的全部视频列表。

## FAQ

**B站视频数据怎么爬？**
调用带 wbi 签名的搜索接口 `x/web-interface/wbi/search/type`，用 `keyword` 传关键词，拿到视频列表，再对每个 BV 号调用 `x/web-interface/wbi/view` 补全点赞、投币、收藏和分享数。完整代码见上文方法一。

**为什么搜索结果里没有投币数？**
B站的搜索接口只返回播放、点赞、收藏、弹幕和评论数，不含投币和分享。投币在视频详情（view）接口的 `stat.coin` 里，所以要对每个视频多请求一次。托管的 Bilibili Scraper 里对应的开关是 `enrichSearchResults`。

**怎么按播放量或最新发布排序？**
搜索接口的 `order` 参数：`click` 是按播放量从高到低，`pubdate` 是按发布时间从新到旧，`totalrank` 是默认的综合排序，`dm` 按弹幕数，`stow` 按收藏数。

**抓 B站数据需要登录或 Cookie 吗？**
视频搜索和视频详情都不需要。需要登录的主要是评论：未登录时 B站只返回每个视频约 3 到 4 条热门评论，要拿完整评论列表得用自己账号的 SESSDATA Cookie。

**一个关键词最多能抓多少个视频？**
大约 1,000 个，也就是 50 页、每页 20 条。想要更多，就用不同的关键词或不同的排序方式分别抓，再按 BV 号去重。

**爬 B站数据合法吗？**
取决于你所在的地区和用途。请阅读 B站的用户协议，控制请求频率，只收集公开数据，并注意视频版权，评论和弹幕的作者是真实用户，可能属于个人信息。本文不构成法律意见。

## Related guides

- [B站 API 的 Python 用法：bilibili-api-python 与 wbi 签名](bilibili-api-python): 这里用到的签名代码的详细说明，以及更多接口。
- [B站弹幕爬取：Python 抓取弹幕和评论](bilibili-danmaku-scraper-zh): 找到视频之后，下载它的弹幕。
- [Download Bilibili comments and danmaku](scrape-bilibili-comments-danmaku): 英文版，讲评论和弹幕。

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "B站视频数据怎么爬？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "调用带 wbi 签名的搜索接口 x/web-interface/wbi/search/type，用 keyword 传关键词，拿到视频列表，再对每个 BV 号调用 x/web-interface/wbi/view 补全点赞、投币、收藏和分享数。完整代码见上文方法一。"
      }
    },
    {
      "@type": "Question",
      "name": "为什么搜索结果里没有投币数？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "B站的搜索接口只返回播放、点赞、收藏、弹幕和评论数，不含投币和分享。投币在视频详情（view）接口的 stat.coin 里，所以要对每个视频多请求一次。托管的 Bilibili Scraper 里对应的开关是 enrichSearchResults。"
      }
    },
    {
      "@type": "Question",
      "name": "怎么按播放量或最新发布排序？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "搜索接口的 order 参数：click 是按播放量从高到低，pubdate 是按发布时间从新到旧，totalrank 是默认的综合排序，dm 按弹幕数，stow 按收藏数。"
      }
    },
    {
      "@type": "Question",
      "name": "抓 B站数据需要登录或 Cookie 吗？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "视频搜索和视频详情都不需要。需要登录的主要是评论：未登录时 B站只返回每个视频约 3 到 4 条热门评论，要拿完整评论列表得用自己账号的 SESSDATA Cookie。"
      }
    },
    {
      "@type": "Question",
      "name": "一个关键词最多能抓多少个视频？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "大约 1,000 个，也就是 50 页、每页 20 条。想要更多，就用不同的关键词或不同的排序方式分别抓，再按 BV 号去重。"
      }
    },
    {
      "@type": "Question",
      "name": "爬 B站数据合法吗？",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "取决于你所在的地区和用途。请阅读 B站的用户协议，控制请求频率，只收集公开数据，并注意视频版权，评论和弹幕的作者是真实用户，可能属于个人信息。本文不构成法律意见。"
      }
    }
  ]
}
</script>
