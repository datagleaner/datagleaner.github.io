---
title: "Weibo API in Python: Official API, m.weibo.cn and Options"
description: "Weibo API Python guide: what the official Open Platform still allows, the m.weibo.cn JSON endpoints, open-source libraries, and a hosted option."
---

# Weibo API in Python: what works in 2026

If you want Weibo data in Python, you have three routes. The official Weibo Open Platform API (open.weibo.com) works through OAuth and mainly covers your own account and the users who authorize your app; searching all public posts is not something an ordinary developer app gets. The JSON endpoints that the mobile site m.weibo.cn calls are free and need no key, but they are undocumented and limited for anonymous visitors. A hosted scraper returns the same public posts through a stable API for a per-result fee. This page explains each route with working Python, so you can pick the one that fits your project.

## Route 1: the official Weibo Open Platform API

Sina runs an official REST API (version 2, at `api.weibo.com/2/...`), and several community Python SDKs wrap it: `weibo` and `pyweibo` on PyPI, among others. They all follow the same flow:

1. Register a developer account at [open.weibo.com](https://open.weibo.com) and create an app. The site and its documentation are in Chinese, and the account goes through Weibo's identity verification.
2. Copy the App Key and App Secret from the app's settings and set an OAuth 2.0 redirect URI.
3. Send a user through the OAuth authorize page, exchange the returned code for an access token, and call endpoints with that token.

A typical call with the `requests` library looks like this once you hold a token:

```python
import requests

ACCESS_TOKEN = "token from the OAuth flow"
r = requests.get(
    "https://api.weibo.com/2/users/show.json",
    params={"access_token": ACCESS_TOKEN, "screen_name": "some_user"},
)
print(r.json())
```

What to expect before you invest time in it:

- **Scope is narrow for ordinary apps.** Reading and posting for the authorized user works. Broader reads, such as keyword search over all public posts or arbitrary users' full timelines, sit behind higher access levels that Weibo grants on application, mainly to partners and businesses. Most third-party guides and community threads report that search is not available to a new app.
- **Rate limits apply per app and per user**, with tighter limits on unreviewed apps. Weibo does not publish one current table, so check the limits shown in your app console.
- **Non-Chinese developers often stall at registration.** Verification and the review of an app for wider access are designed around mainland identities and businesses. Plan for that before building on this route.
- **The SDKs are old.** Several of the Python wrappers have not been updated in years. Because the API is plain REST, calling it with `requests` as above is usually simpler than adopting a stale SDK.

The official API is the right choice when your app acts on behalf of Weibo users who log in to it, for example to post or read their own timeline. For research, monitoring or analytics over public posts, people usually end up on route 2 or 3.

## Route 2: the m.weibo.cn JSON endpoints

The mobile website m.weibo.cn loads its content from JSON endpoints. The most used one is `https://m.weibo.cn/api/container/getIndex`, which returns a user's timeline when you pass `containerid=107603<uid>`. It is the endpoint behind many open-source Weibo crawlers.

A plain request without cookies returns `{"ok": -100}` and a login URL. Weibo first expects the anonymous visitor cookie (`SUB`) that every new browser receives. This script gets that cookie and then reads a user's latest posts. We ran it on 2026-10-09 and it returned ten posts:

```python
import requests

UA = ("Mozilla/5.0 (iPhone; CPU iPhone OS 16_6 like Mac OS X) AppleWebKit/605.1.15 "
      "(KHTML, like Gecko) Version/16.6 Mobile/15E148 Safari/604.1")
s = requests.Session()
s.headers["User-Agent"] = UA

# 1. Get the anonymous visitor cookie (SUB) that m.weibo.cn expects.
s.post("https://visitor.passport.weibo.cn/visitor/genvisitor2",
       data={"cb": "visitor_gray_callback", "tid": "", "from": "weibo"},
       headers={"Referer": "https://visitor.passport.weibo.cn/visitor/visitor"})
if "SUB" not in s.cookies:
    raise SystemExit("Weibo did not issue a visitor cookie")

# 2. Read the first page of a user's timeline (UID from weibo.com/u/<uid>).
uid = "1669879400"
r = s.get("https://m.weibo.cn/api/container/getIndex",
          params={"type": "uid", "value": uid, "containerid": f"107603{uid}"},
          headers={"Referer": "https://m.weibo.cn/", "X-Requested-With": "XMLHttpRequest"})
data = r.json()
if data.get("ok") != 1:
    raise SystemExit(f"Weibo refused the request: {data}")

for card in data["data"]["cards"]:
    post = card.get("mblog")
    if post:
        print(post["created_at"], post["attitudes_count"], post["text"][:60])
```

Fields worth knowing in each `mblog` object: `id`, `created_at` (Beijing time, `+0800`), `text` (HTML, with links and emoji images), `reposts_count`, `comments_count`, `attitudes_count` (likes), `pics`, `isLongText` and `user`.

Limits of this route:

- **Guests see one page of a timeline.** Asking for `page=2` with only a visitor cookie returns `ok: -100` again (login required). Going further needs a logged-in cookie from a real account, which ties your script to that account.
- **Long posts are truncated.** When `isLongText` is true, fetch the full text from `https://m.weibo.cn/statuses/extend?id=<post id>`.
- **Keyword search is capped.** Weibo serves roughly 25 pages of results per keyword (about 1,000 posts) however big the topic. In our test the m.weibo.cn search container returned an empty list to a visitor session, while the desktop site's `weibo.com/ajax/statuses/search` endpoint did return posts.
- **The text needs cleaning.** Strip the HTML from `text` and convert dates to ISO 8601 yourself.
- **It can change without notice.** These endpoints are undocumented. Keep the request rate low, cache what you fetch, and expect to fix the parser now and then.

### Open-source libraries built on these endpoints

If you would rather not write the requests yourself, these GitHub projects are the most widely used:

| Project | What it does | Needs a login cookie |
|---|---|---|
| [dataabc/weibo-crawler](https://github.com/dataabc/weibo-crawler) | Downloads users' posts, images and videos from m.weibo.cn into CSV, JSON or a database | Optional; without one you get what guests see |
| [dataabc/weiboSpider](https://github.com/dataabc/weiboSpider) | Crawls user timelines from the older weibo.cn site | Yes |
| [nghuyong/WeiboSpider](https://github.com/nghuyong/WeiboSpider) | Scrapy project for users, posts, comments, reposts and keyword search | Yes |
| [NanmiCoder/MediaCrawler](https://github.com/NanmiCoder/MediaCrawler) | Browser-based crawler for several Chinese platforms, Weibo included | Yes (you log in by QR code) |

Check each project's license and recent commits before relying on it. A crawler that uses your own account cookie puts that account at risk if Weibo flags the traffic.

## Route 3: a hosted Weibo API (pay per result)

Disclosure: Data Gleaner is us. Our [Weibo Scraper](https://apify.com/datagleaner/weibo-scraper) on the Apify Store wraps route 2's guest session in a maintained API. You send keywords or user IDs and get clean JSON back: full text with the HTML removed (long posts expanded), ISO 8601 timestamps, like, repost and comment counts, images, video, the poster's region and the author. It needs no Weibo account, cookie or developer app, only an Apify account. It costs **$3 per 1,000 posts** ($0.003 per post), and you pay only for posts saved.

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

That run fetches at most 10 posts, so about $0.03. Other inputs: `userIds` (numeric IDs or profile URLs), `sinceDate` to skip older posts, `includeUserProfile` to add follower counts and bios, and `requestDelaySeconds` to slow the pace.

It does not remove Weibo's own limits: keyword search still stops at about 1,000 posts per keyword, user timelines return only the latest page (about 10 posts), and comments, follower lists and the hot-search board are not included.

## Which route to choose

| | Official Open Platform | m.weibo.cn endpoints | Hosted scraper (ours) |
|---|---|---|---|
| Cost | Free | Free | $3 per 1,000 posts |
| Setup | Developer account, identity verification, OAuth | None beyond a visitor cookie | Apify account and token |
| Keyword search over public posts | Only with granted higher access | About 1,000 posts per keyword | About 1,000 posts per keyword |
| Other users' timelines | Limited | Latest page only as a guest | Latest page only |
| Posting or acting for a user | Yes | No | No |
| Maintenance | Stable, documented | You fix it when Weibo changes | We fix it |

Use the official API if your app posts or reads on behalf of logged-in Weibo users. Use the m.weibo.cn endpoints or an open-source crawler for a one-off pull or when you want full control and have time to maintain it. Use a hosted scraper when you need public posts on a schedule and would rather pay a few dollars than maintain a parser.

## FAQ

**Is there a free Weibo API?** Yes, in two senses. The official Open Platform API costs nothing but needs a verified developer account and covers mainly your own and your authorized users' data. The m.weibo.cn JSON endpoints are free and keyless but undocumented and limited for anonymous visitors.

**How do I get a Weibo API key?** Register at open.weibo.com, create an app, and copy the App Key and App Secret from its settings. You then still need an OAuth access token from a user who authorizes the app before most endpoints answer.

**Can I search Weibo posts by keyword with the API?** Not with an ordinary developer app; search belongs to the higher access levels Weibo grants on application. In practice people search through the website's own endpoints, either directly or through a crawler or hosted scraper, all of which hit Weibo's cap of about 1,000 posts per keyword.

**Is there a Weibo API on GitHub?** There is no official one. GitHub hosts community SDKs for the Open Platform API and crawlers for the website, such as dataabc/weibo-crawler and nghuyong/WeiboSpider, listed in the table above.

**Is it legal to collect Weibo data?** Public posts are still personal data. Follow Weibo's terms, keep your request rate low, and make sure your use meets the privacy laws that apply to you, such as GDPR or China's PIPL. This is not legal advice.

## Related guides

- [How to scrape Weibo posts with Python](scrape-weibo-posts-python)
- [Weibo scraper guide in Chinese (微博爬虫)](weibo-scraper-zh)
- [Bilibili API in Python](bilibili-api-python)
- [Web scraping for AI agents over MCP](web-scraping-for-ai-agents-mcp)
