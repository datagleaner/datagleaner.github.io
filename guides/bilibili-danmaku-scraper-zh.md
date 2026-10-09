---
title: "B站弹幕爬取：Python 抓取弹幕和评论，导出 XML/ASS"
description: "B站弹幕爬取完整教程：用 BV 号查到每个分P的 cid，免费下载公开弹幕 XML 并用 Python 存成 CSV，再用 danmaku2ass 转成 ASS 字幕；同时讲清 bilibili-api-python 的用法、B站评论怎么爬，以及未登录只能拿到约 3 条评论、XML 只含最近一批弹幕等限制。"
lang: zh-CN
---

# B站弹幕爬取：用 Python 抓取弹幕和评论

B站弹幕爬取最简单的办法是两步：先用视频的 BV 号调用 `api.bilibili.com/x/player/pagelist` 拿到每个分P的 `cid`，再下载 `https://comment.bilibili.com/{cid}.xml`，这个 XML 文件里就是该分P的弹幕，每条带出现时间、颜色、发送时间和发送者哈希。不需要登录，也不需要 API Key，几十行 Python 就能存成 CSV，或用 danmaku2ass 转成 ASS 字幕。要注意的是，这个 XML 只包含最近的一批弹幕（一般几千条），不是热门视频的全部历史弹幕；评论则不同，未登录时 B站只返回约 3 条热门评论。下面逐一说明每种方法、代码和限制。

Disclosure: 文末提到的 Bilibili Scraper 是我们（Data Gleaner）做的。前面的方法全部免费，不依赖我们的产品。

## 先弄清三个 ID：BV 号、aid 和 cid

- **BV 号**（如 `BV1xx411c7mD`）：视频链接里那一串，对应整个视频。
- **aid**（av 号）：视频的数字 ID，抓评论时用它作为 `oid`。
- **cid**：每个分P一个。弹幕挂在 cid 上，不挂在视频上，所以多P视频要逐个分P下载。

`https://api.bilibili.com/x/player/pagelist?bvid=BV号` 返回该视频所有分P的 `cid`、`page`（第几P）和时长，不需要签名。

## 方法一：下载弹幕 XML（免费，不用登录）

下面的脚本读取一个视频的全部分P，把每个分P的原始 XML 存下来（方便之后转 ASS），同时解析成一张 CSV 表。我们在 2026 年 10 月用 B站最早的视频 `BV1xx411c7mD` 实测可以运行。

```python
# pip install requests
import csv
import requests
import xml.etree.ElementTree as ET

BVID = "BV1xx411c7mD"
HEADERS = {"User-Agent": "Mozilla/5.0", "Referer": "https://www.bilibili.com/"}

# 1. 用 BV 号查分P列表，拿到每个分P的 cid
pages = requests.get(
    "https://api.bilibili.com/x/player/pagelist",
    params={"bvid": BVID}, headers=HEADERS, timeout=15,
).json()["data"]

rows = []
for p in pages:
    # 2. 下载该分P的弹幕 XML，同时保存原文件，后面可以转成 ASS
    r = requests.get(f"https://comment.bilibili.com/{p['cid']}.xml", headers=HEADERS, timeout=15)
    with open(f"{BVID}_p{p['page']}.xml", "wb") as fh:
        fh.write(r.content)
    root = ET.fromstring(r.content)
    for d in root.iter("d"):
        f = d.get("p").split(",")
        rows.append({
            "part": p["page"],
            "time_in_video": float(f[0]),   # 出现在视频第几秒
            "mode": int(f[1]),              # 1 滚动，4 底部，5 顶部
            "font_size": int(f[2]),
            "color": f"#{int(f[3]):06x}",
            "sent_at": int(f[4]),           # 发送时间，Unix 时间戳
            "sender_hash": f[6],            # 发送者 ID 的哈希，不是 UID
            "text": d.text or "",
        })

rows.sort(key=lambda x: (x["part"], x["time_in_video"]))
with open(f"{BVID}_danmaku.csv", "w", newline="", encoding="utf-8-sig") as fh:
    w = csv.DictWriter(fh, fieldnames=list(rows[0].keys()))
    w.writeheader()
    w.writerows(rows)
print(len(rows), "条弹幕")
```

CSV 用 `utf-8-sig` 编码，Excel 直接打开不会乱码。

### XML 里 `p` 属性各字段的含义

每条弹幕是一个 `<d p="...">文本</d>` 元素，`p` 用逗号分成 9 段：

| 位置 | 含义 | 示例 |
|---|---|---|
| 0 | 弹幕出现在视频中的时间（秒） | `26.43100` |
| 1 | 类型：1-3 滚动，4 底部，5 顶部，6 逆向，7 高级，8 代码 | `1` |
| 2 | 字号（25 为标准） | `25` |
| 3 | 颜色，十进制 RGB | `16777215`（白色） |
| 4 | 发送时间，Unix 时间戳 | `1552923069` |
| 5 | 弹幕池（0 普通，1 字幕，2 特殊） | `0` |
| 6 | 发送者 UID 的 CRC32 哈希 | `ecffc8da` |
| 7 | 弹幕 ID | `13516613578391554` |
| 8 | 屏蔽等级 | `10` |

XML 开头的 `<maxlimit>` 是该视频弹幕池的容量。第 6 段只是哈希，不能直接当用户 ID 用；网上有通过碰撞反查 UID 的做法，但这等于把匿名弹幕和具体用户关联起来，做研究时不建议这样处理。

### 这个方法的限制

- **只有最近的一批弹幕。** 弹幕池满了以后，旧弹幕会被新弹幕挤出 XML。热门视频累计几十万条弹幕，XML 里通常只有几千条。
- **历史弹幕要登录。** 网页上"查看历史弹幕"按日期读取，接口需要登录后的 Cookie，而且一次只能取一天。
- **新接口是 protobuf。** 网页播放器现在用 `api.bilibili.com/x/v2/dm/web/seg.so` 按每 6 分钟一段返回 protobuf 格式的弹幕，需要先用 `.proto` 定义解码。网上很多教程说"XML 接口已失效"，我们实测时 XML 接口仍然返回数据，但以后随时可能变化。
- **请求太快会被风控。** 返回 HTTP 412 说明触发了风控，放慢速度、隔一段时间再试即可。批量下载时每个请求间隔一两秒比较稳妥。

## 方法二：把弹幕 XML 转成 ASS 字幕

想在本地播放器（PotPlayer、mpv、VLC 等）里看到带弹幕的视频，需要把 XML 转成 ASS 字幕文件。最常用的工具是开源的 [danmaku2ass](https://github.com/m13253/danmaku2ass)，它是一个单文件 Python 脚本：

```bash
python danmaku2ass.py -s 1920x1080 -o BV1xx411c7mD_p1.ass BV1xx411c7mD_p1.xml
```

`-s` 是视频分辨率，`-o` 是输出文件。字体、字号、透明度和弹幕停留时间也都可以调，具体参数以项目 README 为准。把生成的 `.ass` 和视频放在同一个文件夹、改成同名，播放器一般会自动加载。

如果不想写代码，一些 B站下载工具（如 yutto）在下载视频时也能一并保存弹幕并转换成 ASS，适合连视频一起保存的场景。

## 方法三：用 bilibili-api-python 抓弹幕和评论

[bilibili-api-python](https://pypi.org/project/bilibili-api-python/) 是社区维护的 B站接口封装（PyPI 上的包名是 `bilibili-api-python`，导入名是 `bilibili_api`），覆盖视频、评论、弹幕、直播、动态等，并帮你处理 wbi 签名和 protobuf 解码。它是异步库，还需要另外装一个 HTTP 客户端：

```python
# pip install bilibili-api-python aiohttp
import asyncio
from bilibili_api import video, comment

async def main():
    v = video.Video(bvid="BV1xx411c7mD")
    danmakus = await v.get_danmakus(page_index=0)   # 第 1 个分P
    for dm in danmakus[:10]:
        print(dm.dm_time, dm.text)

    info = await v.get_info()
    c = await comment.get_comments(
        oid=info["aid"], type_=comment.CommentResourceType.VIDEO, page_index=1
    )
    for r in c.get("replies") or []:
        print(r["member"]["uname"], r["content"]["message"])

asyncio.run(main())
```

我们在 2026 年 10 月用 17.4 版、未登录测试时，`get_danmakus` 直接返回了 412 风控页面；同一时间上面的 XML 方法正常。这个库更适合登录后使用：用自己账号的 Cookie 构造 `Credential(sessdata=..., bili_jct=...)` 传给 `Video(..., credential=...)` 和 `get_comments(..., credential=...)`。库的文档注明，评论第 2 页及以后需要 `credential`；历史弹幕（`get_danmakus` 的 `date` 参数）同样需要登录。库更新很快，函数签名在不同大版本之间会变，以官方文档为准。

## B站评论怎么爬

评论和弹幕是两套接口：

- 评论列表：`api.bilibili.com/x/v2/reply/wbi/main`（需要 wbi 签名），参数 `type=1`、`oid=aid`，按游标翻页。
- 楼中楼回复：`api.bilibili.com/x/v2/reply/reply`，参数 `oid=aid`、`root=` 主评论的 `rpid`。
- 旧接口 `api.bilibili.com/x/v2/reply?type=1&oid=aid&pn=1&sort=1` 也还能访问。

关键限制在登录状态，而不是接口：**未登录时，B站对每个视频只返回约 3 条精选评论。** 我们用 aid=2 的视频测试，第 1 页返回 3 条，第 2 页起返回空列表，而该视频显示的评论总数超过 8 万。想拿完整评论，必须带上自己账号的 `SESSDATA` Cookie。建议用小号：登录状态下大量抓取，账号可能被限制。

每条评论里有 `content.message`（正文）、`like`（点赞）、`ctime`（时间戳）、`rcount`（回复数）、`member.uname`（昵称）和 `reply_control.location`（IP 属地，如"IP属地：上海"）。

## 方法四：托管的 Bilibili Scraper（按条付费）

Disclosure: Bilibili Scraper 是我们做的。

如果你需要定期、批量地抓取（比如几百个视频的弹幕加评论），又不想自己维护签名、翻页、重试和分P处理，可以用 Apify Store 上的 [Bilibili Scraper](https://apify.com/datagleaner/bilibili-scraper)。它按关键词搜索视频，或接受视频链接、BV 号、av 号，返回视频数据（播放、点赞、投币、收藏、标签、UP 主），可选抓取评论（含 IP 属地）、楼中楼回复和弹幕。结果可导出 JSON、CSV 或 Excel。

它读取的弹幕来源就是上面的公开 XML，所以同样只有最近的一批弹幕，不含历史弹幕。评论同样受登录限制：不提供 `sessionCookie` 时每个视频约 3 到 4 条；填入你自己账号的 `SESSDATA` 才能拿到完整列表（这一路径我们自己还没有完整验证）。

价格按实际返回的条数计：

| 数据 | 价格 |
|---|---|
| 视频 | $8 / 1000 条 |
| 评论（含回复） | $2 / 1000 条 |
| 弹幕 | $0.50 / 1000 条 |

例如 20 个视频、每个最多 1000 条弹幕：20 x $0.008 + 20,000 x $0.0005 = $10.16。

用 Python 调用（抓指定视频的弹幕）：

```python
# pip install apify-client
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/bilibili-scraper").call(
    run_input={
        "videoUrls": ["BV1xx411c7mD"],
        "includeDanmaku": True,
        "maxDanmakuPerVideo": 1000,
    }
)
for item in client.dataset(run.default_dataset_id).iterate_items():
    if item.get("type") == "danmaku":
        print(item["timeInVideoSeconds"], item["text"])
```

结果数据集里视频、评论、弹幕三类记录用 `type` 字段区分。每条弹幕带 `bvid`、`cid`、`part`、`timeInVideoSeconds`、`mode`、`fontSize`、`color`、`sentAt`、`senderHash` 和 `text`。按关键词批量抓时，把 `videoUrls` 换成 `searchKeywords`（如 `["露营装备"]`），用 `searchOrder: "dm"` 可以按弹幕数排序。它不下载视频文件，也不抓 UP 主的视频列表。

## 怎么选

| 需求 | 合适的方法 |
|---|---|
| 一两个视频的弹幕，存成表格 | 方法一的 XML 脚本 |
| 在本地播放器里看弹幕 | XML 加 danmaku2ass 转 ASS |
| 完整评论、历史弹幕，自己有账号 | bilibili-api-python 加 Cookie |
| 几百个视频的弹幕和评论，定期跑，不想维护代码 | 托管的 Bilibili Scraper |

## 常见问题

**B站弹幕接口还能用吗？** 我们在 2026 年 10 月测试时，`comment.bilibili.com/{cid}.xml` 不登录也能返回弹幕，只是限于最近的一批。网页播放器自己用的是 protobuf 格式的 `x/v2/dm/web/seg.so` 分段接口。两者都是非公开接口，随时可能调整。

**怎么爬取 B站全部历史弹幕？** 公开 XML 拿不到全部。历史弹幕要用登录后的 Cookie 按日期逐天请求，热门视频的数据量很大，请求要放慢。即使这样，被删除或被屏蔽的弹幕也拿不到。

**B站评论爬取为什么只有 3 条？** 这是 B站对未登录访问的限制，不是代码问题。带上自己账号的 `SESSDATA` Cookie 后，评论接口才会正常翻页。

**弹幕能看到是谁发的吗？** XML 里只有发送者 UID 的 CRC32 哈希，不是 UID 本身。做统计分析时，用哈希区分"同一个发送者"就够了。

**爬 B站弹幕合法吗？** 取决于所在地区和用途。请阅读 B站用户协议，控制请求频率，只收集公开数据，并注意弹幕和评论的作者是真实用户，在中国《个人信息保护法》和欧盟 GDPR 下都可能属于个人信息。本文不构成法律意见。

## 相关文章

- [Bilibili API in Python（英文）](bilibili-api-python)
- [微博爬虫：用 Python 抓取微博内容](weibo-scraper-zh)
- [Web scraping for AI agents with MCP（英文）](web-scraping-for-ai-agents-mcp)
