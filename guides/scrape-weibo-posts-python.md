---
title: How to Scrape Weibo Posts with Python (With and Without Login)
description: How to scrape Weibo posts with Python - the official API's limits, a working no-login script, weibo-crawler and weibo-search cookie needs, and a hosted option.
---

# How to scrape Weibo posts with Python

To scrape Weibo posts with Python you have three practical routes, because Weibo's official API does not give an ordinary developer keyword search over public posts. You can call the JSON endpoints of the mobile site (m.weibo.cn) yourself with `requests`, which works without a login for the first page of a user's timeline; you can run an open-source crawler such as `weibo-crawler` or `weibo-search`, which reach further but need the cookie of a logged-in Weibo account for keyword search; or you can call a hosted scraper through an API and get JSON back. This guide shows each one with code and says where each stops.

## What Weibo gives you officially

The Weibo Open Platform (open.weibo.com) is built for apps that act on behalf of Weibo users. You register a developer app, pass review, and users authorize your app with OAuth. The endpoints that come with that cover mostly your own account and the users who authorize you. Searching everyone's public posts by keyword is not part of what an ordinary developer app gets, and the platform's documentation and registration process are in Chinese.

So when people search for a "Weibo API key" to pull posts on a topic, what they end up using in practice is the site's own web endpoints, either directly or through a tool that wraps them.

## What a visitor without an account can see

Knowing the login walls up front saves time, because they apply to every tool, free or paid:

- **A user's timeline:** a visitor without a login sees the first page, about 10 of the newest posts. Asking for page 2 returns a redirect to the sign-in page.
- **Keyword search:** visitors can search, but Weibo serves a limited number of result pages per keyword, roughly 1,000 posts. Logged-in search also stops at a fixed number of pages (the `weibo-search` README puts it at about 50), which is why that tool splits a search into hour-by-hour windows.
- **Comments, reposts and follower lists:** most of this needs a logged-in session.
- **Large counts are rounded:** on popular posts the like, repost and comment counts can come back capped (for example `1000000` for "100万+") rather than exact.

## Option 1: call the mobile endpoints yourself

This is free and good for a one-off pull or for learning how the data is shaped. The mobile site hands any first-time visitor an anonymous visitor cookie (`SUB`); with it, the timeline endpoint returns JSON.

```python
# pip install requests
import re

import requests

UA = ("Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 "
      "(KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1")

session = requests.Session()
session.headers["User-Agent"] = UA

# 1. Get the anonymous visitor cookie that any first-time visitor receives.
session.post(
    "https://visitor.passport.weibo.cn/visitor/genvisitor2",
    data={"cb": "visitor_gray_callback", "tid": "", "from": "weibo"},
    headers={"Referer": "https://visitor.passport.weibo.cn/visitor/visitor"},
    timeout=20,
)

# 2. Read the first page of a user's timeline as JSON.
uid = "1669879400"  # the number in https://weibo.com/u/1669879400
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
    text = re.sub(r"<[^>]+>", "", post["text"])  # the text field is HTML
    print(post["id"], post["created_at"], post["attitudes_count"],
          post["reposts_count"], post["comments_count"], text[:60])
```

What to know before you build on it:

- `created_at` comes as `Thu Oct 01 01:07:17 +0800 2026` (Beijing time), and `attitudes_count` is the like count.
- Long posts are truncated in the timeline (`isLongText` is true); the full text is at `https://m.weibo.cn/statuses/extend?id=<post id>`.
- Page 2 of a timeline returns `{"ok": -100, ...}` with a sign-in URL. That is the guest limit, not a bug in your code.
- The endpoints are undocumented and change without notice. Keep the request rate low (a pause of a second or more between calls), and check `ok` on every response rather than assuming JSON with posts.

Keyword search through the desktop site's `weibo.com/ajax/statuses/search` endpoint takes a second, longer visitor-cookie handshake. If you need search at volume, the open-source tools below or a hosted scraper save you writing it.

## Option 2: open-source Weibo crawlers on GitHub

The two most used projects are by the same author, dataabc:

| Tool | What it does | Login cookie needed? | Output |
|---|---|---|---|
| [weibo-crawler](https://github.com/dataabc/weibo-crawler) | Posts and profiles of the users you list in `config.json`, with optional images, videos, comments and reposts | Optional for user posts; required for keyword search and for crawling reposts. Without one, its README says you get most of a user's posts, but for accounts with more than about 2,000 posts mainly the latest 2,000 | CSV, JSON, MySQL, MongoDB, SQLite |
| [weibo-search](https://github.com/dataabc/weibo-search) | Keyword search with date, region and type filters, built on Scrapy | Required: you paste the cookie of a logged-in weibo.com session into `settings.py` | CSV, MySQL, MongoDB, SQLite, plus media |

Getting the cookie is a manual step: log in to weibo.com in Chrome, open DevTools, and copy the `Cookie` request header from any request to weibo.com. Things to weigh:

- The crawl runs as your account, so a heavy crawl puts that account at risk of restrictions. Many people use a secondary account.
- Cookies expire, so scheduled runs need someone to refresh them.
- Registering a Weibo account needs a phone number that can receive Weibo's SMS.
- Check each repository's license and last commit date before you rely on it; Weibo changes its pages often.

Older PyPI packages named `weibo-scraper` and `weibo-trending` also exist. Test them on a small pull first, since several have not been updated in years.

## Option 3: a hosted scraper with an API

A hosted scraper runs on someone else's servers and returns JSON, CSV or Excel through an HTTP API, usually priced per post. This suits recurring jobs (brand monitoring, daily keyword pulls) where you do not want to maintain a crawler or a logged-in account. The Apify Store lists several Weibo scrapers from different developers. Compare what each accepts (keywords, users, post links, comments), whether it asks for your cookie, the price per result, and the limits it documents. The per-keyword cap is the same for all of them, because Weibo sets it.

### Data Gleaner Weibo Scraper

Disclosure: Data Gleaner is us. Our [Weibo Scraper](https://apify.com/datagleaner/weibo-scraper) takes keywords or user IDs and returns public posts without a Weibo login, cookie or browser, using the same anonymous visitor session as the script above. Each post has the full text with HTML removed, an ISO 8601 timestamp, like, repost and comment counts, images, video, the poster's region, the client it was sent from, the author, and the original post when it is a repost. It costs $3 per 1,000 posts ($0.003 per post), with platform usage included.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/weibo-scraper").call(
    run_input={"searchQueries": ["瑞幸"], "maxItemsPerQuery": 10}
)
if run is None:
    raise SystemExit("The run did not return; check it in the Apify Console.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    author = (item.get("author") or {}).get("screenName")
    text = (item.get("text") or "").replace("\n", " ")[:80]
    print(f'{item.get("createdAt")}  likes={item.get("likesCount")}  @{author}  {text}')
```

That run fetches at most 10 posts, so it costs about $0.03. Other inputs: `userIds` (numeric IDs or profile URLs), `sinceDate` (for example `2026-10-01`) to skip older posts, and `includeUserProfile` to add each author's follower count, bio and verification.

What it does not do, so you can rule it out quickly: it stops at Weibo's cap of about 1,000 posts per keyword, returns only the latest ~10 posts of a user's timeline, and does not return comments, follower lists or the hot-search board. For those, the logged-in open-source crawlers above are the route.

## Which route to pick

| You need | Best fit |
|---|---|
| A few users' latest posts, once | The `requests` script above |
| A user's full post history, comments or reposts | `weibo-crawler` with a cookie |
| Deep keyword search over a long date range | `weibo-search` with a cookie |
| Recurring keyword or user pulls without managing an account | A hosted scraper such as ours |

## FAQ

**Does Weibo have a public API for posts?** Weibo has an Open Platform, but it is built for apps that users authorize through OAuth, and it does not give an ordinary developer app keyword search over all public posts. For public posts on a topic, people use the site's web endpoints through their own code, an open-source crawler or a hosted scraper.

**Do I need a Weibo API key or account to scrape Weibo?** Not for the first page of a user's timeline or for limited keyword search: an anonymous visitor cookie is enough, as the script above shows. You need a logged-in account's cookie for full user histories, deep keyword search, comments and reposts.

**Does weibo-crawler still need a cookie?** For user posts the cookie is optional, and its README says most posts come back without one. Keyword search and repost crawling require it, and `weibo-search` requires it for everything.

**Why does Weibo return `ok: -100`?** That response carries a sign-in URL and means the page you asked for is closed to visitors without a login, such as page 2 of a user's timeline. A fresh visitor cookie does not change it; only a logged-in session does.

**Is it legal to scrape Weibo?** It depends on your jurisdiction and use. Read Weibo's terms, keep your request rate low, collect only public posts, and remember that posts and profiles are personal data under laws such as GDPR and China's PIPL. This is not legal advice.

## Related

- [Weibo, Bilibili and Xiaohongshu scraper APIs compared](../chinese-social-media-scrapers)
- [How to scrape Bilibili comments and danmaku](scrape-bilibili-comments-danmaku)
- [How to scrape Xiaohongshu (RedNote)](scrape-xiaohongshu-rednote)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Does Weibo have a public API for posts?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Weibo has an Open Platform, but it is built for apps that users authorize through OAuth, and it does not give an ordinary developer app keyword search over all public posts. For public posts on a topic, people use the site's web endpoints through their own code, an open-source crawler or a hosted scraper."
      }
    },
    {
      "@type": "Question",
      "name": "Do I need a Weibo API key or account to scrape Weibo?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not for the first page of a user's timeline or for limited keyword search: an anonymous visitor cookie is enough, as the script above shows. You need a logged-in account's cookie for full user histories, deep keyword search, comments and reposts."
      }
    },
    {
      "@type": "Question",
      "name": "Does weibo-crawler still need a cookie?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "For user posts the cookie is optional, and its README says most posts come back without one. Keyword search and repost crawling require it, and weibo-search requires it for everything."
      }
    },
    {
      "@type": "Question",
      "name": "Why does Weibo return ok: -100?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "That response carries a sign-in URL and means the page you asked for is closed to visitors without a login, such as page 2 of a user's timeline. A fresh visitor cookie does not change it; only a logged-in session does."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to scrape Weibo?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It depends on your jurisdiction and use. Read Weibo's terms, keep your request rate low, collect only public posts, and remember that posts and profiles are personal data under laws such as GDPR and China's PIPL. This is not legal advice."
      }
    }
  ]
}
</script>
