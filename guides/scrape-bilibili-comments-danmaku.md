---
title: Download Bilibili Comments and Danmaku (XML, ASS, CSV)
description: "Bilibili comments and danmaku download, step by step: the free danmaku XML file, bilibili-api-python, danmaku to ASS, and a hosted scraper for many videos."
---

# How to download Bilibili comments and danmaku

Bilibili danmaku (弹幕, the bullet comments that fly across the video) are the easy part: every video part has a public XML file at `https://comment.bilibili.com/<cid>.xml`, which you can download without an account and convert to ASS subtitles with a free tool such as `biliass`. Comments under the video (评论) are harder: they come from a signed JSON endpoint, and Bilibili shows anonymous visitors only about 3 to 4 top comments per video, so a full comment list needs the `SESSDATA` cookie of a logged-in account. Below are the free steps for both, the Python library most people use, and a hosted option if you want comments and danmaku for many videos as one table.

## How Bilibili exposes comments and danmaku

| Data | Where it comes from | Login needed? | What you get |
|---|---|---|---|
| Danmaku (弹幕) | `comment.bilibili.com/<cid>.xml`, one file per video part (`cid`) | No | The most recent danmaku pool: text, time in the video, display mode, font size, color, send time, hashed sender id. Typically up to a few thousand per part, not the full history of a very popular video. |
| Danmaku, by segment or date | Protobuf endpoints the player uses | Partly; history by date needs a login | Same fields, in 6-minute segments. Libraries such as `bilibili-api-python` read these. |
| Comments (评论) | `api.bilibili.com/x/v2/reply/wbi/main`, which needs a `wbi` request signature | For more than about 3 to 4 top comments, yes | Comment text, likes, time, reply count, author, IP location (the province or country Bilibili shows). |
| Replies (楼中楼) | `api.bilibili.com/x/v2/reply/reply` | No | Replies under one comment, 20 per page. |

A video is identified by its BV id (`BV1xx411c7mD`) or av id. Each part of a multi-part video has its own `cid`, and danmaku belong to the part, not the video.

## Option 1: download the danmaku XML yourself (free)

This uses only Python's standard library. It looks up each part's `cid` and saves that part's danmaku XML. The file is sometimes served deflate-compressed, so the script inflates it when needed.

```python
import json
import urllib.request
import zlib

HEADERS = {"User-Agent": "Mozilla/5.0", "Referer": "https://www.bilibili.com/"}


def get(url):
    req = urllib.request.Request(url, headers=HEADERS)
    with urllib.request.urlopen(req, timeout=20) as r:
        return r.read()


bvid = "BV1xx411c7mD"
pages = json.loads(get(f"https://api.bilibili.com/x/player/pagelist?bvid={bvid}"))["data"]
for page in pages:
    raw = get(f"https://comment.bilibili.com/{page['cid']}.xml")
    if not raw.lstrip().startswith(b"<"):
        raw = zlib.decompress(raw, -zlib.MAX_WBITS)  # raw deflate
    path = f"{bvid}_p{page['page']}.xml"
    with open(path, "wb") as f:
        f.write(raw)
    print(path, raw.count(b"<d p="), "danmaku")
```

Each danmaku is one `<d>` element. Its `p` attribute is a comma-separated list:

```xml
<d p="0.00000,5,25,15138834,1587615763,0,9eeacb3b,31705576815722501,10">注意：此视频为B站已知最古老的视频</d>
```

| Position | Meaning |
|---|---|
| 0 | Time in the video, seconds |
| 1 | Mode: 1 scrolling, 4 bottom, 5 top, 6 reverse, 7 positioned, 8 code |
| 2 | Font size |
| 3 | Color as a decimal RGB integer (16777215 is white) |
| 4 | Send time, Unix timestamp |
| 5 | Pool |
| 6 | Sender hash (a hash of the user id, not the id itself) |
| 7 | Danmaku id |

Newer files carry a ninth field (a weight). To get a spreadsheet, parse the XML with `xml.etree.ElementTree`, split `p`, and write one row per `<d>`. Some danmaku contain control characters that XML forbids; strip characters below code point 32 (except tab and newlines) before parsing if the parser complains.

## Export danmaku to ASS subtitles

To watch the danmaku over a downloaded video in mpv, VLC or any player that reads ASS subtitles, convert the XML:

```bash
pip install biliass
biliass BV1xx411c7mD_p1.xml -s 1920x1080 -o BV1xx411c7mD_p1.ass
```

`-s` is the video size in pixels. `biliass` (an adaptation of Danmaku2ASS) can also hide modes it should not draw, for example `--block-top` or `--block-scroll`, and reads the protobuf format with `-f protobuf`. The older `danmaku2ass` script does the same job. Name the `.ass` file like the video and most players load it automatically.

## Option 2: bilibili-api-python (free, more endpoints)

[bilibili-api-python](https://github.com/Nemo2011/bilibili-api) is a community library that wraps many of Bilibili's web endpoints, including the `wbi` signing, comments and protobuf danmaku. It needs an HTTP backend installed alongside it, or it refuses to make requests:

```bash
pip install bilibili-api-python httpx
```

```python
import asyncio
from bilibili_api import Credential, comment, video

# Optional, but anonymous calls are limited: copy SESSDATA from your own logged-in browser.
cred = Credential(sessdata="YOUR_SESSDATA")


async def main():
    v = video.Video(bvid="BV1xx411c7mD", credential=cred)
    info = await v.get_info()

    danmakus = await v.get_danmakus(page_index=0)  # first part
    for d in danmakus[:10]:
        print(d.dm_time, d.text)

    page = await comment.get_comments(
        info["aid"], comment.CommentResourceType.VIDEO, page_index=1, credential=cred
    )
    for c in page.get("replies") or []:
        print(c["content"]["message"])


asyncio.run(main())
```

Caveats from our own test: without a credential, a request from an ordinary connection can come back as HTTP 412 (Bilibili's risk control) or as an empty comment list, so plan on a logged-in cookie, a slow pace, and retries. The library changes its function names between major versions, so check its docs for the version you install. Use an account you can afford to lose; heavy automated use while logged in can get it restricted.

## Option 3: browser tools

If you only need one video, browser add-ons (for example "ASS Danmaku" on Firefox) save the danmaku of the video you are watching as ASS, and command-line downloaders for Bilibili videos often have a flag to save danmaku next to the video. These are the quickest route for watching, not for analysis across many videos.

## Option 4: a hosted scraper (comments and danmaku as one dataset)

Disclosure: Data Gleaner is us. Our [Bilibili Scraper](https://apify.com/datagleaner/bilibili-scraper) on the Apify Store takes keywords or video links and returns videos, comments (with replies and IP location) and danmaku as one dataset, told apart by a `type` field. It handles the request signing and pacing, needs no browser, and exports JSON, CSV or Excel.

| | DIY XML | bilibili-api-python | Bilibili Scraper (ours) |
|---|---|---|---|
| Cost | Free | Free | $8 per 1,000 videos, $2 per 1,000 comments, $0.50 per 1,000 danmaku |
| Danmaku | Latest pool per part | Latest pool, segments, history with login | Latest pool (public XML), up to `maxDanmakuPerVideo` |
| Comments without login | No | About 3 to 4 per video, if not blocked | About 3 to 4 per video |
| Full comment lists | No | With your `SESSDATA` | With your `SESSDATA` in `sessionCookie` (not yet verified by us) |
| Search videos by keyword | No | Yes | Yes, up to about 1,000 per keyword |
| You maintain the code | Yes | Partly | No |

This script downloads the comments and danmaku of two videos, writes the comments to CSV, and rebuilds each video part's danmaku as Bilibili-style XML so `biliass` can turn it into ASS:

```python
# pip install apify-client
import csv
import os
import sys
from collections import defaultdict
from datetime import datetime
from xml.sax.saxutils import escape

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/bilibili-scraper").call(
    run_input={
        "videoUrls": ["https://www.bilibili.com/video/BV1xx411c7mD", "BV1rpWjevEip"],
        "includeComments": True,
        "maxCommentsPerVideo": 20,
        "commentSort": "hot",
        "includeReplies": True,
        "maxRepliesPerComment": 10,
        "includeDanmaku": True,
        "maxDanmakuPerVideo": 1000,
    }
)
if run is None:
    sys.exit("Run did not return.")

comments = []
danmaku = defaultdict(list)  # (bvid, part) -> items
for item in client.dataset(run.default_dataset_id).iterate_items():
    if item.get("type") == "comment":
        comments.append(item)
    elif item.get("type") == "danmaku":
        danmaku[(item["bvid"], item["part"])].append(item)

with open("comments.csv", "w", newline="", encoding="utf-8-sig") as f:
    w = csv.writer(f)
    w.writerow(["videoBvid", "rpid", "parentRpid", "createdAt", "likes", "ipLocation", "author", "text"])
    for c in comments:
        w.writerow([c["videoBvid"], c["rpid"], c.get("parentRpid"), c["createdAt"], c["likes"],
                    c.get("ipLocation"), (c.get("author") or {}).get("name"), c["text"]])

for (bvid, part), items in danmaku.items():
    rows = []
    for d in items:
        if d.get("timeInVideoSeconds") is None or d.get("mode") is None:
            continue
        color = int((d.get("color") or "#ffffff").lstrip("#"), 16)
        sent = d.get("sentAt")
        sent_ts = int(datetime.fromisoformat(sent.replace("Z", "+00:00")).timestamp()) if sent else 0
        p = f'{d["timeInVideoSeconds"]},{d["mode"]},{d.get("fontSize") or 25},{color},{sent_ts},0,{d.get("senderHash") or ""},{d.get("danmakuId") or ""}'
        rows.append(f'<d p="{p}">{escape(d["text"])}</d>')
    with open(f"{bvid}_p{part}.xml", "w", encoding="utf-8") as f:
        f.write('<?xml version="1.0" encoding="UTF-8"?><i>' + "".join(rows) + "</i>")

print(len(comments), "comments;", sum(map(len, danmaku.values())), "danmaku")
```

Then `biliass BV1xx411c7mD_p1.xml -s 1920x1080 -o BV1xx411c7mD_p1.ass` as above. At the default 1,000 danmaku per video, two videos cost about 2 x $0.008 + 2,000 x $0.0005 = $1.02, plus $0.002 per comment. You pay only for items delivered, and a run stops at the maximum cost you set.

What it does not do: it does not download video files, does not list an uploader's videos, does not resolve short `b23.tv` links (use the full URL or BV id), and returns only the danmaku in the public XML, not a video's full danmaku history. Bilibili rate-limits heavily; if a large run is slowed down, raise `requestDelaySecs`.

## Caveats for any method

- **Danmaku history is not complete.** The public XML holds the latest pool, capped per video part (the file's `maxlimit`). Older danmaku of a busy video are only reachable through the logged-in history endpoints.
- **Anonymous comment depth is about 3 to 4 per video.** That is Bilibili's choice, the same for every tool. Replies under a comment page normally.
- **Respect the people in the data.** Comment authors are real users. Use the data for analysis, not to profile or contact individuals, and follow Bilibili's terms and the privacy law that applies to you.

## FAQ

**How do I download danmaku from a Bilibili video?** Find the part's `cid` (from `api.bilibili.com/x/player/pagelist?bvid=<BV id>`), then download `https://comment.bilibili.com/<cid>.xml`. The script in Option 1 does both for every part of a video.

**How do I convert Bilibili danmaku XML to ASS?** Install `biliass` with pip and run `biliass file.xml -s 1920x1080 -o file.ass`, matching `-s` to the video's resolution. The ASS file plays as a subtitle track in mpv, VLC and most desktop players.

**Can I download all Bilibili comments without logging in?** No. Anonymous visitors get about 3 to 4 top comments per video. For full lists, every tool (including ours) needs the `SESSDATA` cookie of an account you own.

**Is there a Bilibili danmaku downloader on GitHub?** Yes, several: `bilibili-api-python` covers danmaku and comments in Python, and Danmaku2ASS and `biliass` convert the XML to subtitles. Check when a project was last updated, because Bilibili changes its endpoints.

**Does the danmaku XML include who sent each message?** It includes a hash of the sender's user id (field 6 of `p`), not the user id or name.

## Related

- [Weibo, Bilibili and Xiaohongshu scraper APIs compared](../chinese-social-media-scrapers)
- [Weibo API in Python: get Weibo posts from Python](weibo-api-python)
- [How to scrape Xiaohongshu (RedNote)](scrape-xiaohongshu-rednote)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I download danmaku from a Bilibili video?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Find the part's cid (from api.bilibili.com/x/player/pagelist?bvid=<BV id>), then download https://comment.bilibili.com/<cid>.xml. The script in Option 1 does both for every part of a video."
      }
    },
    {
      "@type": "Question",
      "name": "How do I convert Bilibili danmaku XML to ASS?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Install biliass with pip and run biliass file.xml -s 1920x1080 -o file.ass, matching -s to the video's resolution. The ASS file plays as a subtitle track in mpv, VLC and most desktop players."
      }
    },
    {
      "@type": "Question",
      "name": "Can I download all Bilibili comments without logging in?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. Anonymous visitors get about 3 to 4 top comments per video. For full lists, every tool (including ours) needs the SESSDATA cookie of an account you own."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a Bilibili danmaku downloader on GitHub?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, several: bilibili-api-python covers danmaku and comments in Python, and Danmaku2ASS and biliass convert the XML to subtitles. Check when a project was last updated, because Bilibili changes its endpoints."
      }
    },
    {
      "@type": "Question",
      "name": "Does the danmaku XML include who sent each message?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It includes a hash of the sender's user id (field 6 of p), not the user id or name."
      }
    }
  ]
}
</script>
