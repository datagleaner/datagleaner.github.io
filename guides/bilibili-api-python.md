---
title: "Bilibili API in Python: bilibili-api-python and wbi Signing"
description: "How to use the Bilibili API from Python: bilibili-api-python examples, wbi signing in 20 lines, public endpoints for video stats and search, and rate limits."
---

# Bilibili API in Python

Bilibili has no documented public API for reading video data, so "the Bilibili API" in Python means the JSON endpoints the bilibili.com website itself calls. You have two practical ways to reach them: the community library **bilibili-api-python** (`pip install bilibili-api-python`, by Nemo2011 on GitHub), which wraps hundreds of endpoints in async Python, or about 20 lines of your own code with `requests` that sign each call with Bilibili's **wbi** signature. Both are free. Both are unofficial, so they break when Bilibili changes something and they run into rate limits quickly. This page shows both, lists the endpoints for video stats, search, comments and danmaku, and ends with a hosted option for when you need volume without maintaining the code.

## Which route to pick

| Route | Good for | Cost | What you maintain |
|---|---|---|---|
| bilibili-api-python | Wide coverage: videos, users, search, comments, live rooms, logged-in actions | Free (GPL-3.0 license) | Library upgrades, your own pacing and retries |
| Your own `requests` code with wbi signing | A few endpoints, full control, no heavy dependency | Free | The signing code, parsers, retries, session cookies |
| Hosted scraper (for example our Bilibili Scraper on Apify) | Repeated or large pulls, no code to maintain, output as JSON, CSV or Excel | Pay per result | Nothing on Bilibili's side |

## Option 1: bilibili-api-python

The library is on PyPI as `bilibili-api-python` (version 17.4.2 when this page was written, Python 3.10 or newer). The import name is `bilibili_api`. It is async: every call is a coroutine, so you run it inside `asyncio.run()`. It needs an HTTP client installed alongside it; it uses `curl_cffi` or `aiohttp`, whichever is present.

```bash
pip install bilibili-api-python aiohttp
```

Search videos by keyword, sorted by views:

```python
import asyncio
from bilibili_api import search
from bilibili_api.search import SearchObjectType, OrderVideo

async def main():
    res = await search.search_by_type(
        "python 教程",
        search_type=SearchObjectType.VIDEO,
        order_type=OrderVideo.CLICK,   # TOTALRANK, CLICK, PUBDATE, DM, STOW
        page=1,
    )
    for v in res["result"]:
        print(v["bvid"], v["play"], v["author"], v["title"])

asyncio.run(main())
```

Video details and stats for one BV id (this call returned HTTP 412 in our test; see the note below):

```python
import asyncio
from bilibili_api import video

async def main():
    # Calls the unsigned /x/web-interface/view endpoint; if it returns 412, use the wbi/view call in Option 2
    info = await video.Video(bvid="BV1xx411c7mD").get_info()
    s = info["stat"]
    print(info["title"], s["view"], s["like"], s["coin"], s["favorite"], s["share"], s["reply"], s["danmaku"])

asyncio.run(main())
```

Most-liked comments for a video (comments are keyed by the numeric `aid`, not the BV id; the default order is newest first):

```python
import asyncio
from bilibili_api import comment

async def main():
    page = await comment.get_comments_lazy(170001, comment.CommentResourceType.VIDEO, order=comment.OrderType.LIKE)
    for r in page.get("replies") or []:
        print(r["member"]["uname"], r["like"], r["content"]["message"])

asyncio.run(main())
```

What we saw when testing these three snippets on 2026-10-09 from one residential connection: search and comments worked, but `get_info()` failed with HTTP 412. The library calls the unsigned `/x/web-interface/view` endpoint for video details, and that endpoint answered 412 to every client we tried, while the signed `/x/web-interface/wbi/view` endpoint (Option 2 below) returned the same video normally. Results vary by IP and by day, so if one call returns 412, try the `wbi` version of that endpoint before assuming you are blocked.

Other things worth knowing about the library:

- **Login is optional but changes what you get.** Many calls accept `credential=Credential(sessdata=..., bili_jct=..., buvid3=...)` built from your own browser cookies. Anonymous calls see what a logged-out visitor sees.
- **Read the license.** It is GPL-3.0-or-later, which matters if you ship it inside closed-source software.
- **The docs are in Chinese** at nemo2011.github.io/bilibili-api, with a module per area (`video`, `user`, `search`, `comment`, `live` and many more).

## Option 2: call the endpoints yourself, with wbi signing

Since 2023 many Bilibili web endpoints (the ones with `/wbi/` in the path) require two extra query parameters: `wts`, a Unix timestamp, and `w_rid`, an MD5 hash of the sorted query string plus a key. The key comes from two image file names that the `/x/web-interface/nav` endpoint returns even to logged-out visitors, shuffled through a fixed 64-entry table. The scheme is documented by the community in the bilibili-API-collect project on GitHub. Here it is in plain Python:

```python
import hashlib, re, time, urllib.parse
import requests

MIXIN = [46, 47, 18, 2, 53, 8, 23, 32, 15, 50, 10, 31, 58, 3, 45, 35, 27, 43, 5, 49, 33, 9, 42, 19, 29, 28, 14, 39,
         12, 38, 41, 13, 37, 48, 7, 16, 24, 55, 40, 61, 26, 17, 0, 1, 60, 51, 30, 4, 22, 25, 54, 21, 56, 59, 6, 63,
         57, 62, 11, 36, 20, 34, 44, 52]

s = requests.Session()
s.headers.update({"User-Agent": "Mozilla/5.0 (research script)", "Referer": "https://www.bilibili.com/"})

def wbi_key():
    img = s.get("https://api.bilibili.com/x/web-interface/nav").json()["data"]["wbi_img"]
    stem = lambda u: u.rsplit("/", 1)[1].split(".")[0]
    raw = stem(img["img_url"]) + stem(img["sub_url"])
    return "".join(raw[i] for i in MIXIN)[:32]

def sign(params, key):
    p = dict(params, wts=int(time.time()))
    p = {k: re.sub(r"[!'()*]", "", str(v)) for k, v in sorted(p.items())}
    p["w_rid"] = hashlib.md5((urllib.parse.urlencode(p) + key).encode()).hexdigest()
    return p

key = wbi_key()

# Video details and stats
r = s.get("https://api.bilibili.com/x/web-interface/wbi/view",
          params=sign({"bvid": "BV1xx411c7mD"}, key)).json()
print(r["code"], r["data"]["title"], r["data"]["stat"]["view"])

time.sleep(2)

# Keyword search, most viewed first
r = s.get("https://api.bilibili.com/x/web-interface/wbi/search/type",
          params=sign({"search_type": "video", "keyword": "python 教程", "order": "click", "page": 1}, key)).json()
for v in (r.get("data") or {}).get("result", []):
    print(v["bvid"], v["play"], re.sub("<[^>]+>", "", v["title"]))   # titles contain <em> highlight tags
```

This ran as shown on 2026-10-09. Two details trip people up: the `nav` call returns `code: -101` (not logged in) but still includes `wbi_img`, so read the key from it anyway; and the key rotates, so fetch it again daily or whenever signed calls start failing.

## The endpoints for common tasks

All are `GET` requests on `https://api.bilibili.com` unless noted. Every JSON response has a `code` field: `0` means success.

| Task | Endpoint | Signed? | Notes |
|---|---|---|---|
| wbi key | `/x/web-interface/nav` | No | Read `data.wbi_img.img_url` and `sub_url` |
| Video details and stats | `/x/web-interface/wbi/view?bvid=...` | Yes | `data.stat` has views, danmaku, replies, likes, coins, favorites, shares. Also accepts `aid` |
| Video tags | `/x/tag/archive/tags?bvid=...` | No | List of tag objects with `tag_name` |
| Keyword search | `/x/web-interface/wbi/search/type?search_type=video&keyword=...&order=...&page=...` | Yes | `order`: `totalrank`, `click`, `pubdate`, `dm`, `stow`. 20 results per page, about 50 pages at most |
| Top-level comments | `/x/v2/reply/wbi/main?type=1&oid=<aid>&mode=3&next=<cursor>` | Yes | `mode=3` hottest, `mode=2` newest. Uses the numeric `aid` |
| Replies to one comment | `/x/v2/reply/reply?type=1&oid=<aid>&root=<rpid>&pn=1&ps=20` | No | Pages normally |
| Danmaku (bullet comments) | `https://comment.bilibili.com/<cid>.xml` | No | One XML file per video part. `cid` is in the video details under `pages` |

Search results lack coins and shares; call the video details endpoint for each hit if you need them. Search results also name fields differently from the details endpoint: views are `play`, the uploader is `author`, and titles contain `<em>` highlight tags.

## Rate limits and other caveats

- **No published limits.** Bilibili does not document rate limits for these endpoints. In practice, rapid requests get refused with HTTP 412, or JSON codes such as `-412`, `-352` or `-799`. Sometimes the answer is HTTP 200 with `code: 0` and no real data, only a `v_voucher` field (a captcha challenge), so check for that too.
- **Go slowly.** A pause of one to two seconds between requests, with some random jitter, and backing off for longer after a refusal, keeps a small script working. Hammering an endpoint after a 412 tends to extend the block.
- **Anonymous comment depth is shallow.** Logged out, Bilibili returns only about 3 to 4 curated top comments per video. Full comment lists need a logged-in session (your own account's `SESSDATA` cookie), and an account used for heavy automated reading can be restricted.
- **Search tops out near 1,000 results per keyword.** To collect more, split the query into narrower keywords.
- **Danmaku XML is the recent pool, not full history.** It typically holds up to a few thousand bullet comments per part, not the full history of a popular video.
- **Everything is undocumented.** Endpoint paths, field names and the signing scheme have changed before and will again. Pin your library version and re-test when results look wrong.
- **Terms and privacy.** Read Bilibili's terms of service before scraping. Comment authors are real people, so do not use the data to profile or harass anyone.

## Option 3: a hosted Bilibili scraper (no code to maintain)

Disclosure: Data Gleaner is us. Our [Bilibili Scraper](https://apify.com/datagleaner/bilibili-scraper) on the Apify Store wraps the endpoints above, with the wbi signing, session cookies, pacing and retries handled, and returns video stats, comments (with the commenter's IP location) and danmaku as JSON, CSV or Excel. It needs no login. It costs $8 per 1,000 videos, $2 per 1,000 comments and $0.50 per 1,000 danmaku, charged only for items delivered. It does not return an uploader's video list or download video files.

Run it from Python with the `apify-client` package (you need an Apify account and API token):

```python
# pip install apify-client
import os
import sys

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/bilibili-scraper").call(
    run_input={
        "searchKeywords": ["python 教程"],
        "searchOrder": "click",
        "maxItems": 5,
    }
)
if run is None:
    sys.exit("Run did not return.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    if item.get("type") != "video":
        continue
    stats = item.get("stats") or {}
    owner = item.get("owner") or {}
    print(f'{item["title"]} | {stats.get("views")} views | {owner.get("name")} | {item["url"]}')
```

That run returns five videos and costs about $0.04. To fetch specific videos instead, pass `"videoUrls": ["BV1xx411c7mD"]`. To add comments or danmaku, set `"includeComments": true` or `"includeDanmaku": true`. The same limits apply as for your own code: anonymous comment depth is about 3 to 4 per video unless you supply your own `sessionCookie`, and search stops near 1,000 results per keyword.

## FAQ

**Is there an official Bilibili API for Python?**
No. Bilibili does not publish a documented API for reading public video, search or comment data, and there is no official Python SDK for it. Python tools such as bilibili-api-python call the website's own undocumented endpoints.

**Where is bilibili-api-python on GitHub?**
At github.com/Nemo2011/bilibili-api. The PyPI package is `bilibili-api-python`, the import is `bilibili_api`. Other PyPI packages with similar names (`bilibili-api`, `bilibili-api-dev`) are different releases, so check which one a tutorial installs.

**Do I need a Bilibili API key?**
No key exists for these endpoints. The wbi "key" is derived from the `nav` endpoint as shown above, not issued to you. A login cookie (`SESSDATA`) is optional and only needed for data a logged-out visitor cannot see, such as full comment lists.

**Why do I get error 412 or -352?**
Both are Bilibili's risk control refusing the request, usually because of request speed, a missing or stale wbi signature, or missing browser-like cookies (`buvid3`). Slow down, refresh the wbi key, use the signed `/wbi/` version of the endpoint, and wait before retrying.

**How do I get danmaku (bullet comments) in Python?**
Get the video's `cid` from the details endpoint (each part has its own), then download `https://comment.bilibili.com/<cid>.xml`. Each `<d>` element is one bullet comment; its `p` attribute holds the time in the video, display mode, font size, color and send time. Our Chinese guide on [scraping Bilibili danmaku](bilibili-danmaku-scraper-zh) goes further.

## Related guides

- [Bilibili danmaku scraper (Chinese)](bilibili-danmaku-scraper-zh)
- [Weibo API in Python](weibo-api-python)
- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there an official Bilibili API for Python?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. Bilibili does not publish a documented API for reading public video, search or comment data, and there is no official Python SDK for it. Python tools such as bilibili-api-python call the website's own undocumented endpoints."
      }
    },
    {
      "@type": "Question",
      "name": "Where is bilibili-api-python on GitHub?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "At github.com/Nemo2011/bilibili-api. The PyPI package is bilibili-api-python, the import is bilibili_api. Other PyPI packages with similar names (bilibili-api, bilibili-api-dev) are different releases, so check which one a tutorial installs."
      }
    },
    {
      "@type": "Question",
      "name": "Do I need a Bilibili API key?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No key exists for these endpoints. The wbi \"key\" is derived from the nav endpoint as shown above, not issued to you. A login cookie (SESSDATA) is optional and only needed for data a logged-out visitor cannot see, such as full comment lists."
      }
    },
    {
      "@type": "Question",
      "name": "Why do I get error 412 or -352?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Both are Bilibili's risk control refusing the request, usually because of request speed, a missing or stale wbi signature, or missing browser-like cookies (buvid3). Slow down, refresh the wbi key, use the signed /wbi/ version of the endpoint, and wait before retrying."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get danmaku (bullet comments) in Python?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Get the video's cid from the details endpoint (each part has its own), then download https://comment.bilibili.com/<cid>.xml. Each <d> element is one bullet comment; its p attribute holds the time in the video, display mode, font size, color and send time. Our Chinese guide on scraping Bilibili danmaku goes further."
      }
    }
  ]
}
</script>
