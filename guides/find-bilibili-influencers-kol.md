---
title: "Find Bilibili Influencers (KOL) for Brand Marketing"
description: "How to find Bilibili KOLs without an agency: search niche keywords, rank uploaders by views, coins and favourites, check comments. Python code."
---

# How to find Bilibili influencers (KOL) for brand marketing

To find Bilibili influencers (KOLs, in Chinese marketing language) without an agency, search your niche keywords on Bilibili, sort the results by views, and group the videos by uploader. Then rank the uploaders by total views, by favourites and coins per view, and by how recently they posted, and read the comments under their videos to check that the audience matches your customer. The output is a ranked shortlist in a CSV file. This page gives a working Python script for the free route, the limits you will hit, and a hosted option that returns the same data (plus coins and comments) when you do not want to maintain code.

Disclosure: Data Gleaner, mentioned at the end as one option, is us. Every other method on this page is free and needs no account.

## What a shortlist needs, and what Bilibili shows

Searching for "Bilibili KOL" mostly returns agency pages and platform databases that you pay for, and older articles with statistics from years ago. Those are useful for context, but they do not let you start from your own keywords today. Bilibili's own search does, and it exposes enough per video to build a first shortlist:

- **Views, likes, favourites, danmaku and comment counts** for each video in the search results.
- **Coins** (a signal viewers give to videos they value) and shares, but only from the per-video details, not from the search list.
- **The uploader's name and id** (`mid`), which is what you group on.
- **The publish date**, which tells you whether an uploader is still active.

What you do not get without a login is the uploader's follower count or full video list. Treat the shortlist as a ranking of who is already getting attention for your topic, not as a complete census of creators.

## Step 1: choose keywords like a viewer, not like a marketer

Chinese viewers search in Chinese and with the words they use for the use case, not your product category. For a camping brand, `露营装备` (camping gear) and `露营` (camping) find different uploaders than your English product name would. Write 3 to 10 keywords: the category, the use case, and the problem your product solves. Search each one separately and merge the results, because the same uploader often appears under several keywords, which is itself a signal that they cover your niche.

Sort by views (`order=click`) to see who has proven reach, and run a second pass sorted by newest (`order=pubdate`) to find smaller uploaders who are active now. Bilibili's search ends at about 1,000 results per keyword.

## Step 2: pull the videos and aggregate by uploader (free, Python)

Bilibili has no documented public API, so this uses the search endpoint the website itself calls. Each request needs a `wbi` signature, which the script builds from a key that Bilibili serves to logged-out visitors. The signing scheme is explained in [Bilibili API in Python](bilibili-api-python). I ran this script on 2026-10-09 with the keywords `露营装备` and `露营`, 3 pages each, and it wrote a CSV and printed the top uploaders.

```python
# pip install requests pandas
import hashlib, re, time, urllib.parse
import pandas as pd
import requests

MIXIN = [46, 47, 18, 2, 53, 8, 23, 32, 15, 50, 10, 31, 58, 3, 45, 35, 27, 43, 5, 49, 33, 9, 42, 19, 29, 28, 14, 39,
         12, 38, 41, 13, 37, 48, 7, 16, 24, 55, 40, 61, 26, 17, 0, 1, 60, 51, 30, 4, 22, 25, 54, 21, 56, 59, 6, 63,
         57, 62, 11, 36, 20, 34, 44, 52]
KEYWORDS = ["露营装备", "露营"]   # your niche keywords, Chinese works best
PAGES = 3                         # 20 videos per page per keyword

s = requests.Session()
s.headers.update({"User-Agent": "Mozilla/5.0 (research script)", "Referer": "https://www.bilibili.com/"})

def wbi_key():
    img = s.get("https://api.bilibili.com/x/web-interface/nav", timeout=20).json()["data"]["wbi_img"]
    stem = lambda u: u.rsplit("/", 1)[1].split(".")[0]
    raw = stem(img["img_url"]) + stem(img["sub_url"])
    return "".join(raw[i] for i in MIXIN)[:32]

def sign(params, key):
    p = dict(params, wts=int(time.time()))
    p = {k: re.sub(r"[!'()*]", "", str(v)) for k, v in sorted(p.items())}
    p["w_rid"] = hashlib.md5((urllib.parse.urlencode(p) + key).encode()).hexdigest()
    return p

key = wbi_key()
rows = {}
for kw in KEYWORDS:
    for page in range(1, PAGES + 1):
        r = s.get("https://api.bilibili.com/x/web-interface/wbi/search/type", timeout=20,
                  params=sign({"search_type": "video", "keyword": kw, "order": "click", "page": page}, key)).json()
        for v in (r.get("data") or {}).get("result") or []:
            rows[v["bvid"]] = {"mid": v["mid"], "uploader": v["author"], "bvid": v["bvid"],
                               "views": int(v["play"]), "favorites": int(v["favorites"]),
                               "likes": int(v["like"]), "pubdate": v["pubdate"]}
        time.sleep(2)

df = pd.DataFrame(rows.values())
df["pub"] = pd.to_datetime(df["pubdate"], unit="s")
g = df.groupby(["mid", "uploader"]).agg(
    videos=("bvid", "count"), total_views=("views", "sum"),
    avg_views=("views", "mean"), favorites=("favorites", "sum"),
    likes=("likes", "sum"), first=("pub", "min"), last=("pub", "max")).reset_index()
g["avg_views"] = g["avg_views"].round().astype(int)
g["favs_per_1k_views"] = (g["favorites"] / g["total_views"] * 1000).round(1)
g = g.sort_values("total_views", ascending=False)
g.to_csv("kol_shortlist.csv", index=False, encoding="utf-8-sig")
print(g.head(10).to_string(index=False))
```

The CSV has one row per uploader: how many of their videos appeared in your search (`videos`), total and average views, favourites, likes, the dates of their first and last video in the sample, and `favs_per_1k_views`. The last column is a rough engagement measure. A video that people favourite is one they want to come back to, which is often a better sign for product-led content than raw views. In my run, one uploader had about 36 favourites per 1,000 views and another about 5, on similar-sized audiences, so ranking by views alone would have hidden a big difference in how viewers used the videos.

The file is written with the `utf-8-sig` encoding so Excel shows the Chinese names correctly.

## Step 3: rank, and read the posting dates

Sort the table three ways before you decide:

1. **By total views**, to see who has reach on your topic.
2. **By `favs_per_1k_views`**, to see whose viewers save content. Ignore uploaders with a single video here, since one video proves little.
3. **By `last`**, to drop creators whose most recent relevant video is years old. In my run, one of the top ten by views had a single relevant video, from 2020.

Also scan the titles of each top video. Keyword search returns anything that matches, so some high-view results will be off-topic for your brand.

Posting frequency is the weak spot of the free route. The script only sees the videos that matched your keywords, so `videos` and the first and last dates describe your niche, not the uploader's whole channel. A creator who posts every week about other things looks inactive here. For a real posting rhythm, open the shortlisted channels and look at their video lists yourself.

## Step 4: read the comments to check audience fit

Views do not tell you who is watching. Comments do, in two ways: what viewers say (do they ask about products, prices and brands, or only joke?) and where they say it from, because Bilibili shows the province or country next to each comment. Open the top few videos of each shortlisted uploader and read the hottest comments. Look for questions about buying, mentions of competitor brands, and whether the tone fits your product. Anonymous visitors only see about 3 to 4 curated top comments per video, so this is a sample, not a count. Our guide on [downloading Bilibili comments and danmaku](scrape-bilibili-comments-danmaku) covers the comment endpoints and the login cookie that unlocks more.

## Limits of the free route

- **Unofficial endpoints.** They can change or start returning errors without notice, and the `wbi` key rotates, so fetch it again whenever signed calls fail.
- **Search ends near 1,000 results per keyword**, and each page is 20 videos.
- **No coins or shares in search results.** To rank by coins you need one more signed request per video to the video details endpoint, covered in the Chinese-language guide [B站视频数据爬取](bilibili-video-data-scraper-zh).
- **No follower counts, no full channel list, no audience demographics.** Those need a logged-in account or a paid influencer platform.
- **Rate limits.** Bilibili can answer with errors or empty results if you go fast. Keep the pause between requests and run long jobs slowly.
- **Not a measure of sales.** High views do not mean a creator converts for your product, and creators differ in whether they take brand work. Contacting them, agreeing terms and checking disclosure rules for sponsored content are outside this method.
- **Terms and privacy.** Check Bilibili's terms of service, and treat commenters as real people when you store their words.

## Option: Data Gleaner Bilibili Scraper

If you want this for many keywords, on a schedule, or with coins and comments included, the [Bilibili Scraper](https://apify.com/datagleaner/bilibili-scraper) on the Apify Store returns the same search as structured data without the signing code.

What it does, from its documentation:

- Searches videos by keyword (`searchKeywords`), ordered by relevance, most views, newest, most danmaku, most favourites or most comments (`searchOrder`: `totalrank`, `click`, `pubdate`, `dm`, `stow`, `scores`), up to 1,000 videos per keyword (`maxItems`).
- Can limit results by length (`durationFilter`) and by publish date (`publishedAfter`, `publishedBefore`, including a relative window such as `7 days`), which helps to find creators who are active now.
- Returns per video the title, tags, publish time, `stats` (views, likes, favorites, danmaku, comments, plus `coins` and `shares` when `enrichSearchResults` is on) and the `owner` (`mid` and `name`), so you can group by uploader as in the script above.
- Can also fetch comments for every video (`includeComments`, `maxCommentsPerVideo`, `commentSort`), each with its text, likes and `ipLocation`.
- Costs US$8 per 1,000 videos and US$2 per 1,000 comments, charged only for items delivered. For example, 300 videos is US$2.40, and 30 videos with the roughly 4 top comments an anonymous request returns is about US$0.24 more.
- Uses plain HTTP, no browser, and needs no login. It does not include an uploader's own video list, so the posting-frequency limit above applies here too.

```python
# pip install apify-client pandas
import os

import pandas as pd
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/bilibili-scraper").call(run_input={
    "searchKeywords": ["露营装备", "露营"],
    "searchOrder": "click",
    "maxItems": 100,
    "enrichSearchResults": True,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

rows = []
for item in client.dataset(run.default_dataset_id).iterate_items():
    if item.get("type") != "video":
        continue
    st = item["stats"]
    rows.append({"mid": item["owner"]["mid"], "uploader": item["owner"]["name"],
                 "views": st["views"] or 0, "favorites": st["favorites"] or 0,
                 "coins": st["coins"] or 0, "published": item["publishedAt"]})

g = pd.DataFrame(rows).groupby(["mid", "uploader"]).agg(
    videos=("views", "count"), total_views=("views", "sum"),
    favorites=("favorites", "sum"), coins=("coins", "sum"), last=("published", "max"))
g["coins_per_1k_views"] = (g["coins"] / g["total_views"] * 1000).round(1)
g.sort_values("total_views", ascending=False).to_csv("kol_shortlist.csv", encoding="utf-8-sig")
```

Add `"includeComments": True` and `"maxCommentsPerVideo": 20` to `run_input` to read comments in the same run, or pass the BV ids of your shortlisted videos in `videoUrls` for a second, comments-only pass. You can also export the dataset as CSV or Excel from the Apify Console instead of writing code.

## FAQ

**How do I find Bilibili influencers for my brand?**
Search your niche keywords in Chinese on Bilibili, sort by views, collect the videos and group them by uploader. Rank the uploaders by total views, favourites per view and the date of their latest relevant video, then read the comments under their top videos to check that the audience fits your customer. This can be done with a short Python script or by hand for a small list.

**What is a KOL on Bilibili?**
KOL stands for key opinion leader, the term Chinese marketing uses for an influencer whose audience trusts their recommendations. On Bilibili these creators are called uploaders (UP主). The platform has no official public KOL list, so a shortlist comes from who is already getting views and engagement on your topic.

**Which metrics should I rank Bilibili creators by?**
Total and average views show reach. Favourites and coins per 1,000 views show how much viewers valued the content, which is more telling than views alone. The latest relevant video date shows whether the creator is active. Comments show whether the audience matches your customer. No single number replaces watching a few of their videos.

**Can I get a creator's follower count from Bilibili search?**
Not from the search results used here, and the Data Gleaner Actor does not return follower counts either. Followers are shown on the creator's profile page, which you can open yourself for each shortlisted name. Views on recent videos show current reach better than a follower count does.

**How much does it cost to build a shortlist this way?**
The Python route is free apart from your time. With the Actor above, videos cost US$8 per 1,000 and comments US$2 per 1,000, so a 300-video search costs US$2.40.

**Can I contact the creators from this data?**
The data identifies each creator by name and user id, not by email or other contact details. Reach them through their Bilibili profile and messages, or through the contact method they list there. Sponsored content in China carries its own disclosure and advertising rules, so check those before you commission anything. This page is not legal advice.

## Related guides

- [Bilibili video data scraper: keyword search, views, likes and coins to CSV](bilibili-video-data-scraper-zh): the Chinese-language guide that adds coins with the video details call.
- [Download Bilibili comments and danmaku](scrape-bilibili-comments-danmaku): read audience comments and where they come from.
- [Bilibili API in Python](bilibili-api-python): how the `wbi` signing and the common endpoints work.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I find Bilibili influencers for my brand?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Search your niche keywords in Chinese on Bilibili, sort by views, collect the videos and group them by uploader. Rank the uploaders by total views, favourites per view and the date of their latest relevant video, then read the comments under their top videos to check that the audience fits your customer. This can be done with a short Python script or by hand for a small list."
      }
    },
    {
      "@type": "Question",
      "name": "What is a KOL on Bilibili?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "KOL stands for key opinion leader, the term Chinese marketing uses for an influencer whose audience trusts their recommendations. On Bilibili these creators are called uploaders (UP主). The platform has no official public KOL list, so a shortlist comes from who is already getting views and engagement on your topic."
      }
    },
    {
      "@type": "Question",
      "name": "Which metrics should I rank Bilibili creators by?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Total and average views show reach. Favourites and coins per 1,000 views show how much viewers valued the content, which is more telling than views alone. The latest relevant video date shows whether the creator is active. Comments show whether the audience matches your customer. No single number replaces watching a few of their videos."
      }
    },
    {
      "@type": "Question",
      "name": "Can I get a creator's follower count from Bilibili search?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not from the search results used here, and the Data Gleaner Actor does not return follower counts either. Followers are shown on the creator's profile page, which you can open yourself for each shortlisted name. Views on recent videos show current reach better than a follower count does."
      }
    },
    {
      "@type": "Question",
      "name": "How much does it cost to build a shortlist this way?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The Python route is free apart from your time. With the Actor above, videos cost US$8 per 1,000 and comments US$2 per 1,000, so a 300-video search costs US$2.40."
      }
    },
    {
      "@type": "Question",
      "name": "Can I contact the creators from this data?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The data identifies each creator by name and user id, not by email or other contact details. Reach them through their Bilibili profile and messages, or through the contact method they list there. Sponsored content in China carries its own disclosure and advertising rules, so check those before you commission anything. This page is not legal advice."
      }
    }
  ]
}
</script>
