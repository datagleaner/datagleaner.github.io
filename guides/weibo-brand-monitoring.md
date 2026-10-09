---
title: "Weibo Brand Monitoring: A Low-Cost Social Listening Setup"
description: "Monitor your brand on Weibo for a few dollars: Chinese keyword lists, a daily sinceDate run, engagement and region triage, an LLM sentiment pass, with Python code."
---

# Weibo brand monitoring: a low-cost social listening setup

To monitor your brand on Weibo without an enterprise contract, run a daily keyword search for your brand's Chinese names, keep only the posts newer than your last run, rank them by likes, reposts and comments, and send the top ones through an LLM that labels sentiment and topic. The pieces are a keyword list, a scheduled scraper, a dedupe step and a short prompt. With a pay-per-post scraper at $3 per 1,000 posts, a daily run that collects 100 new posts costs about $0.30. This guide gives the setup, the code for the parts you can do for free, and the limits you should know before you rely on it.

Disclosure: Data Gleaner, mentioned in the middle as one way to collect the posts, is us. The keyword method, the triage script and the sentiment pass work with any source of Weibo data.

## Why general social listening tools miss Weibo

Most social listening products are built around English-language platforms. Weibo needs three things they handle poorly:

- **Chinese names.** People write a foreign brand in Chinese characters, in abbreviations and in nicknames, and rarely in the Latin spelling you registered. A search for "Luckin Coffee" misses a conversation that says 瑞幸.
- **Region.** Weibo shows where a post was sent from. For a brand it matters whether a complaint comes from your core city market or from somewhere you do not sell.
- **Price transparency.** Enterprise suites rarely publish prices. At the time of writing, a priced option in the Apify Store, [zhorex/chinese-brand-monitor](https://apify.com/zhorex/chinese-brand-monitor), charges $0.06 per mention across several Chinese platforms, with optional sentiment tags. At $0.003 per post, the Weibo-only setup below is 20 times cheaper per item, in exchange for doing the keyword work and the analysis yourself.

## Step 1: build the Chinese keyword list

The keyword list decides what you see, so spend your time here. Build it in four groups:

1. **Brand names in every form.** The official Chinese name, the English name, the pinyin or common abbreviation, and the nicknames fans use. Ask someone on your China team, or check how the brand appears in Weibo's own search suggestions.
2. **Product names.** Each product line and each launch, in Chinese and English.
3. **Campaign hashtags.** Weibo topics are written as `#name#`. A campaign you run is its own keyword.
4. **Competitors.** The same three groups for two or three rivals, so you can compare volume and tone.

Keep one search per keyword, because the scraper returns at most about 1,000 posts per keyword (a cap set by Weibo). Narrow keywords stay under the cap and return cleaner results than one broad term. Two traps:

- **Ambiguous words.** A brand name that is also a common word (an apple, a coffee, a lucky number) pulls in unrelated posts. Pair it with a product word, for example `苹果手机` instead of `苹果`, or filter the results afterwards with a rule such as "must also contain one of these words".
- **Typos and variants.** Add the two or three misspellings you see in the first week of results.

Start with 10 to 30 keywords. Read a week of results and cut the ones that return noise.

## Free option: watch your own brand account

Weibo's official API is built for apps that users authorize, and it does not give an ordinary developer keyword search over all public posts. For free, the approach that works without a login is to read the first page of a public account's timeline. That is enough to watch your own brand account or a competitor's, though not to search for mentions by other people. This script prints the newest posts of one account with a simple engagement score:

```python
# pip install requests
import re

import requests


def to_int(v):
    # Weibo sends very large counts as text such as "100万+" (万 = 10,000).
    if isinstance(v, int):
        return v
    v = str(v or "0").rstrip("+")
    return int(float(v[:-1]) * 10_000) if v.endswith("万") else int(float(v))

UA = ("Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 "
      "(KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1")

session = requests.Session()
session.headers["User-Agent"] = UA

# Anonymous visitor cookie that any first-time visitor receives.
session.post(
    "https://visitor.passport.weibo.cn/visitor/genvisitor2",
    data={"cb": "visitor_gray_callback", "tid": "", "from": "weibo"},
    headers={"Referer": "https://visitor.passport.weibo.cn/visitor/visitor"},
    timeout=20,
)

uid = "1669879400"  # the number in https://weibo.com/u/<uid>
resp = session.get(
    "https://m.weibo.cn/api/container/getIndex",
    params={"type": "uid", "value": uid, "containerid": f"107603{uid}"},
    headers={"Referer": "https://m.weibo.cn/", "X-Requested-With": "XMLHttpRequest"},
    timeout=20,
)
data = resp.json()
if data.get("ok") != 1:
    raise SystemExit(f"Weibo did not return posts: {data}")

for card in data["data"]["cards"]:
    post = card.get("mblog")
    if not post:
        continue
    text = re.sub(r"<[^>]+>", "", post["text"])
    score = sum(to_int(post.get(k)) for k in ("attitudes_count", "reposts_count", "comments_count"))
    print(score, post["id"], post["created_at"], text[:60])
```

Limits: a visitor without a login gets about 10 posts and no page 2, the endpoints are undocumented and can change without notice, and keyword search is not covered by this script. For free keyword search over a long range, open-source crawlers such as `weibo-search` exist, but they need the cookie of a logged-in Weibo account, which puts that account at risk and means refreshing the cookie by hand. Our [Weibo API in Python](weibo-api-python) guide covers those options in detail.

## Step 2: collect mentions with a daily sinceDate run

The cheap way to monitor is to run the same keyword list every day and ask only for what is new. The `sinceDate` field skips older posts, and the run stops once a page holds nothing newer, so you pay only for fresh posts.

Data Gleaner's [Weibo Scraper](https://apify.com/datagleaner/weibo-scraper) is a hosted Apify Actor that does this without a Weibo login, cookie or browser. Facts from its listing:

- It takes keywords in `searchQueries` (Chinese or English) and returns public posts newest first, up to about 1,000 per keyword.
- Each post has the full text with HTML removed, an ISO 8601 timestamp, `likesCount`, `repostsCount`, `commentsCount`, images, video, the poster's `region`, the sending client in `source`, the `author` and the original post when it is a repost.
- It costs $3 per 1,000 posts ($0.003 each), charged only for posts saved to the dataset, with platform usage included. You can set a maximum total charge and the run stops when it is reached.
- Keyword search does not carry the author's follower count, bio or verified reason, so those are `null` unless you turn on `includeUserProfile`, which makes runs slower. Posts found through `userIds` include them.

```python
# pip install apify-client
import os
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

KEYWORDS = ["瑞幸", "瑞幸咖啡", "luckin coffee", "生椰拿铁"]  # replace with your list
# Weibo dates are Beijing dates, so "yesterday" is computed in Beijing time.
since = (datetime.now(ZoneInfo("Asia/Shanghai")).date() - timedelta(days=1)).isoformat()

run = client.actor("datagleaner/weibo-scraper").call(run_input={
    "searchQueries": KEYWORDS,
    "maxItemsPerQuery": 200,
    "sinceDate": since,
})
if run is None:
    raise SystemExit("The Actor run did not return; check it in the Apify Console.")

posts = list(client.dataset(run.default_dataset_id).iterate_items())
print(len(posts), "posts since", since)
```

Schedule it with cron, GitHub Actions or Apify's own scheduler. Two points on correctness: Weibo shows time in Beijing time (`+08:00`), so a date without a timezone is read as a Beijing date; and starting each daily run from yesterday overlaps the previous run by a day, so keep the `id`s you have already stored and drop repeats. Within one run the Actor already returns each post once, even when several keywords match it. Weibo can also throttle guest access, so a run may return fewer posts than you asked for; the run log says why, and raising `requestDelaySeconds` or enabling a proxy helps.

## Step 3: triage by engagement and region

Not every mention deserves a human. A post with no likes from an account with no readers matters less than one that is being reposted. A simple score and a region count give you a shortlist:

```python
from collections import Counter

seen = set()
unique = []
for p in posts:
    if p["id"] not in seen:
        seen.add(p["id"])
        unique.append(p)


def score(p):
    # Reposts spread a post, comments show a reaction, likes are the weakest signal.
    return p["repostsCount"] * 3 + p["commentsCount"] * 2 + p["likesCount"]


ranked = sorted(unique, key=score, reverse=True)
shortlist = ranked[:20]

print("Posts by region:", Counter(p.get("region") or "unknown" for p in unique).most_common(5))
for p in shortlist:
    print(score(p), p.get("region"), p["url"], p["text"][:60].replace("\n", " "))
```

The weights (3, 2, 1) are a judgment call, not a standard: change them to fit what your brand cares about. Two refinements are worth adding:

- **Reposts.** When `retweetedStatus` is not empty, the post is a repost of an older one. Count how many posts point at the same original to see what is spreading.
- **Large accounts.** Keyword results leave follower counts `null`. Rather than turning on `includeUserProfile` for every daily run, which slows it down, put the authors of your top posts in `userIds` in a separate run: posts found that way carry the author's follower count and verification.

## Step 4: add an LLM sentiment pass

Sentiment tools tuned for English often handle Chinese slang, sarcasm and emoji such as `[doge]` badly. A general LLM reads Chinese well enough to label each post, and you can ask for exactly the fields you act on. Run it on the shortlist, or on everything if volume is small. This example uses the Anthropic Python SDK; the model name is read from an environment variable so you can choose it:

```python
# pip install anthropic
import json
import os

import anthropic

llm = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY
MODEL = os.environ["ANTHROPIC_MODEL"]

PROMPT = """You label Weibo posts about the brand {brand}.
For each post return JSON with: id, sentiment (positive, neutral or negative about the brand),
topic (one of: product, price, service, ad_or_celebrity, other), needs_reply (true or false),
and summary_en (one short English sentence).
Reply with a JSON list only.

Posts:
{posts}
"""


def label(batch, brand="Luckin Coffee"):
    items = [{"id": p["id"], "text": p["text"][:500]} for p in batch]
    msg = llm.messages.create(
        model=MODEL,
        max_tokens=2000,
        messages=[{"role": "user", "content": PROMPT.format(
            brand=brand, posts=json.dumps(items, ensure_ascii=False))}],
    )
    return json.loads(msg.content[0].text)


labels = []
for i in range(0, len(shortlist), 10):
    labels += label(shortlist[i:i + 10])

for row in labels:
    print(row["sentiment"], row["topic"], row["needs_reply"], row["summary_en"])
```

Treat the output as a first pass. Check a sample of 30 to 50 labels against your own reading before you trust the numbers, add the brand's slang to the prompt if the model misses it, and handle a reply that does not parse as JSON, since a model can return extra text. The LLM bill depends on the provider and model you pick and is separate from the scraping cost.

## What this setup does not do

- **Comments cost extra.** The Actor has an `includeComments` switch that adds top-level comments to each post, at the price of extra requests; check the Actor page for its current pricing before you turn it on.
- **No full history.** Each keyword stops at about 1,000 posts. Daily runs build your own archive going forward, but you cannot reconstruct last year from one run.
- **No private or follower-only posts.** Only what a logged-out visitor can see.
- **Gaps are possible.** Results come from Weibo's search, which can be throttled or incomplete, so treat counts as a sample of public posts, not a census.
- **Weibo only.** Xiaohongshu, Douyin and WeChat need their own sources. See our [Xiaohongshu guide](scrape-xiaohongshu-rednote) and the [WeChat articles guide](wechat-official-account-articles-scraper).
- **Not a crisis alarm.** A daily batch is fine for tracking sentiment. If a fast-moving crisis is your main risk, run it more often or use a service with alerting.

Posts and profiles are personal data. You are responsible for a lawful basis and for obligations under laws such as GDPR and China's PIPL, including retention and deletion. This is not legal advice.

## FAQ

**How do I monitor my brand on Weibo?**
Make a list of Chinese names, product names and campaign hashtags, search each one daily, keep the posts newer than your last run, and review the ones with the most reposts, comments and likes. The code above does the collection, ranking and sentiment labelling.

**How much does Weibo brand monitoring cost?**
With a pay-per-post scraper at $3 per 1,000 posts, 100 new posts a day is about $0.30 a day, plus your own LLM cost for the sentiment pass. Enterprise social listening suites generally do not publish prices, so ask for a quote and compare it with your actual post volume.

**Can I monitor Weibo mentions for free?**
Partly. Without a login you can read the first page of a public account's timeline, which suits watching your own or a competitor's account. Free keyword search across all posts needs an open-source crawler with a logged-in account's cookie, which carries account risk and manual upkeep.

**What keywords should I track for a foreign brand?**
Track the official Chinese name, any abbreviation or nickname that appears in real posts, each product in Chinese and English, and your campaign hashtags. Avoid single common words; pair them with a product word, or filter the results afterwards.

**Can an LLM do sentiment analysis on Chinese Weibo posts?**
Yes, a general LLM can label Chinese posts as positive, neutral or negative about a brand and summarise them in English. Check a sample of its labels by hand first, because slang, sarcasm and in-jokes can be misread.

**Does it work for competitor monitoring?**
Yes. Add a keyword group for each competitor and compare post counts, engagement and sentiment week to week. Each post carries a `query` field with the keyword that found it, so you can tell which results belong to which brand. A post that names two brands appears once per run, under whichever keyword found it first, so check the text when you count head-to-head mentions.

## Related guides

- [Weibo API in Python: official API, m.weibo.cn and options](weibo-api-python): what Weibo gives developers and what a visitor without a login can reach.
- [How to scrape Weibo posts with Python](scrape-weibo-posts-python): the three routes in more detail, with limits.
- [Scrape Xiaohongshu (RedNote) notes and profiles](scrape-xiaohongshu-rednote): the next platform to cover for Chinese consumer sentiment.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I monitor my brand on Weibo?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Make a list of Chinese names, product names and campaign hashtags, search each one daily, keep the posts newer than your last run, and review the ones with the most reposts, comments and likes. The code above does the collection, ranking and sentiment labelling."
      }
    },
    {
      "@type": "Question",
      "name": "How much does Weibo brand monitoring cost?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "With a pay-per-post scraper at $3 per 1,000 posts, 100 new posts a day is about $0.30 a day, plus your own LLM cost for the sentiment pass. Enterprise social listening suites generally do not publish prices, so ask for a quote and compare it with your actual post volume."
      }
    },
    {
      "@type": "Question",
      "name": "Can I monitor Weibo mentions for free?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Partly. Without a login you can read the first page of a public account's timeline, which suits watching your own or a competitor's account. Free keyword search across all posts needs an open-source crawler with a logged-in account's cookie, which carries account risk and manual upkeep."
      }
    },
    {
      "@type": "Question",
      "name": "What keywords should I track for a foreign brand?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Track the official Chinese name, any abbreviation or nickname that appears in real posts, each product in Chinese and English, and your campaign hashtags. Avoid single common words; pair them with a product word, or filter the results afterwards."
      }
    },
    {
      "@type": "Question",
      "name": "Can an LLM do sentiment analysis on Chinese Weibo posts?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, a general LLM can label Chinese posts as positive, neutral or negative about a brand and summarise them in English. Check a sample of its labels by hand first, because slang, sarcasm and in-jokes can be misread."
      }
    },
    {
      "@type": "Question",
      "name": "Does it work for competitor monitoring?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. Add a keyword group for each competitor and compare post counts, engagement and sentiment week to week. Each post carries a query field with the keyword that found it, so you can tell which results belong to which brand. A post that names two brands appears once per run, under whichever keyword found it first, so check the text when you count head-to-head mentions."
      }
    }
  ]
}
</script>
