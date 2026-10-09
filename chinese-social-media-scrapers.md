---
title: Weibo, Bilibili and Xiaohongshu Scraper APIs Compared
description: Weibo, Bilibili and Xiaohongshu scraper API options compared, from official APIs and open-source crawlers to pay-per-result Apify Actors, with Python code.
---

# Weibo, Bilibili, Xiaohongshu and WeChat scraper APIs

None of China's big social platforms offers an open API for reading public posts at scale. Weibo's open platform needs a registered developer app and returns little public search data, Bilibili has no documented public API for video, comment or danmaku data, Xiaohongshu (RedNote) has no public data API at all, and WeChat's API covers only an Official Account you own. So in practice you have three routes: call the sites' own web endpoints yourself (with a community library or your own code), run an open-source crawler such as MediaCrawler, or pay per result for a hosted scraper that returns JSON through an API. This page compares those routes for each platform, then lists our own Actors as one option.

## What each platform gives you officially

| Platform | Official route | What it means for public data |
|---|---|---|
| Weibo (微博) | Weibo Open Platform (open.weibo.com) | You register a developer app and get access to a limited set of endpoints, mostly for your own account and users who authorize your app. Keyword search over all public posts is not part of what an ordinary developer app gets. |
| Bilibili (哔哩哔哩) | None for public data | The website calls its own JSON endpoints, some of which need a request signature (wbi). They are undocumented and change without notice. |
| Xiaohongshu (小红书, RedNote) | None for public data | The open platform serves merchants and advertisers, not reading notes. Search and comments need a logged-in session or the site's request signing. |
| WeChat (微信公众号) | Official Account API | Works only for accounts you administer. There is no public API to search or read other accounts' articles; Sogou's WeChat search is the main public discovery path. |

## Option 1: call the endpoints yourself

Free in money, costly in time. It suits a one-off research pull or a developer who wants full control.

1. **Weibo.** The mobile site (m.weibo.cn) serves search results and user timelines as JSON to anonymous visitors. Keyword search stops at roughly 25 pages (about 1,000 posts) per keyword, and a user's timeline shows guests only its first page. See [how to scrape Weibo posts with Python](guides/scrape-weibo-posts-python) for a step-by-step version.
2. **Bilibili.** Community libraries such as `bilibili-api-python` wrap the web endpoints, including the wbi signing. Danmaku (bullet comments) come from a public XML file per video part. Anonymous visitors get only about 3 to 4 top comments per video. See [how to scrape Bilibili comments and danmaku](guides/scrape-bilibili-comments-danmaku).
3. **Xiaohongshu.** Note pages embed their data in the HTML, but only when the link carries the `xsec_token` parameter Xiaohongshu adds to shared links. Keyword search needs a logged-in session. See [how to scrape Xiaohongshu (RedNote)](guides/scrape-xiaohongshu-rednote).
4. **WeChat articles.** A public `mp.weixin.qq.com` article link returns the full article HTML. To discover articles by keyword you go through Sogou's WeChat search, which returns recent articles only and shows a verification page if you query too fast.

Caveats that apply to all four: the endpoints are undocumented and break when the site changes, the sites rate-limit heavily, and you maintain the parser yourself.

## Option 2: run an open-source crawler

**MediaCrawler** (on GitHub) covers Xiaohongshu, Douyin, Kuaishou, Bilibili, Weibo, Tieba and Zhihu in one project. It drives a real browser with Playwright and keeps a logged-in session, which lets it reach search and comments that anonymous requests cannot. The costs: you run and update it yourself, you log in with your own accounts (which then carry the risk of restrictions), and its license (Non-Commercial Learning License 1.1) allows learning and research use only, with no commercial use without the author's written consent. GitHub also has many single-platform scrapers; check when each was last updated, because these sites change often.

## Option 3: hosted scrapers with an API

Hosted scrapers run on someone else's servers and return JSON, CSV or Excel through an HTTP API, usually priced per result. The Apify Store lists several for each platform from different developers. When comparing them, check:

- **What the input supports:** keyword search, user or profile pages, single post links, comments.
- **Whether it needs your login or cookies.** No-login tools are simpler to run but see only what a guest sees.
- **Price per result and whether failed pages are charged.**
- **The documented limits.** An honest listing says how many results per keyword the platform allows, which is the same for every scraper because the platform sets it.

### Our Actors for Chinese social media

Disclosure: Data Gleaner is us. We publish these Actors on the Apify Store, priced per result, with platform usage included. None of them needs a Chinese account, a login or cookies.

| Actor | What it returns | Price | Store page |
|---|---|---|---|
| Weibo Scraper (`weibo-scraper`) | Public posts by keyword or user: full text, timestamp, like / repost / comment counts, images, video, poster's region, author. Optional author profiles (followers, bio, verification). | $0.003 per post ($3 per 1,000) | [apify.com/datagleaner/weibo-scraper](https://apify.com/datagleaner/weibo-scraper) |
| Bilibili Scraper (`bilibili-scraper`) | Videos by keyword or link with views, likes, coins, favorites, tags and uploader; comments and replies with IP location; danmaku with time in video. | $0.008 per video, $0.002 per comment, $0.0005 per danmaku | [apify.com/datagleaner/bilibili-scraper](https://apify.com/datagleaner/bilibili-scraper) |
| Xiaohongshu (RedNote) Scraper (`xiaohongshu-rednote-scraper`) | Full notes from note links, the recommended feed and ten trending category feeds (such as food, fashion and beauty), and profiles with followers, likes and up to about 30 newest notes. | $0.003 per note or profile | Coming soon: [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: xiaohongshu-rednote-scraper) --> |
| WeChat Articles Scraper (`wechat-articles`) | Public Official Account articles from links or keyword search: title, account, publish time, full text, Markdown, images. | $0.005 per article ($5 per 1,000) | Coming soon: [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: wechat-articles) --> |
| China Brand Report (`china-brand-report`) | One English social-listening report on a brand from Weibo and Bilibili: volume over time, sentiment, translated top posts and videos, hashtags, top accounts, plus a CSV of every post. | $4.99 per report | Coming soon: [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: china-brand-report) --> |

What they do not do, so you can rule them out quickly:

- **Weibo:** about 1,000 posts per keyword (Weibo's own search cap), only the latest ~10 posts of a user's timeline, no comments, follower lists or hot-search board.
- **Bilibili:** about 3 to 4 comments per video without a login (an optional `SESSDATA` cookie of your own account gives full depth), no uploader video lists, no video downloads.
- **Xiaohongshu:** no keyword search and no comments; note links must include `xsec_token`.
- **WeChat:** keyword discovery returns recent articles only, up to about 100 per keyword; read and like counts are usually not public.
- **China Brand Report:** sentiment is keyword-based, so read it as direction rather than a measurement; it does not cover WeChat, Xiaohongshu or Douyin.

## Who uses this data, and for what

- **Brand and marketing teams outside China** track mentions of a product, campaign or competitor on Weibo and Bilibili, with engagement and the poster's province.
- **Investors and analysts** read public sentiment on a Chinese consumer, auto or tech name before an earnings date or launch. The China Brand Report packages this in English for readers who do not read Chinese.
- **Creator and influencer research:** find Bilibili uploaders and Xiaohongshu creators in a niche and compare their numbers.
- **Researchers and journalists** collect public discourse on an event or hashtag.
- **LLM and RAG pipelines:** Weibo post text and WeChat articles in Markdown are clean inputs for embedding, summarising or translation.

## Python quick start: Weibo posts by keyword

Install the Apify client and set your Apify API token (free account; the token is under Settings, API & Integrations):

```bash
pip install apify-client
export APIFY_TOKEN=your_token_here
```

```python
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
    print(f'    {item.get("url")}')
```

This fetches at most 10 posts, about $0.03. Chinese keywords return far more posts than English ones. To keep a daily feed, add `"sinceDate": "2026-10-01"` and run it on a schedule, deduplicating on `id`.

The same pattern works for Bilibili; only the input changes:

```python
run = client.actor("datagleaner/bilibili-scraper").call(
    run_input={"searchKeywords": ["露营装备"], "searchOrder": "click", "maxItems": 5,
               "includeComments": True, "maxCommentsPerVideo": 10}
)
for item in client.dataset(run.default_dataset_id).iterate_items():
    if item["type"] == "video":
        print(item["title"], item["stats"]["views"], item["url"])
    elif item["type"] == "comment":
        print("  ", item.get("ipLocation"), (item.get("text") or "")[:60])
```

## Calling them from an AI agent (Apify MCP)

Claude, Cursor and other MCP clients can run these Actors through Apify's hosted MCP server at `https://mcp.apify.com`. Add an Actor to the URL to preload it as a tool, for example in Claude Code:

```bash
claude mcp add --transport http apify "https://mcp.apify.com?tools=datagleaner/weibo-scraper,datagleaner/bilibili-scraper"
```

Run `/mcp` to sign in to Apify in the browser, or pass `--header "Authorization: Bearer YOUR_APIFY_TOKEN"`. Then ask in plain language:

> Get the 50 newest Weibo posts mentioning 瑞幸 since 2026-10-01 and the 10 most-viewed Bilibili videos about it, and summarise in English what people complain about most.

Without preloading, the agent can find any of these Actors with Apify's `search-actors` tool and run it with `call-actor`; running needs your Apify account either way.

## FAQ

**Is there a Weibo API for scraping posts?** Weibo's open platform requires a registered developer app and does not give ordinary apps keyword search over all public posts. Most people use the mobile site's JSON endpoints or a hosted scraper instead. Either way, Weibo caps keyword search at about 1,000 posts per keyword.

**Is there a Xiaohongshu (RedNote) API?** Not for reading public notes. Scrapers read the public note pages (which need the `xsec_token` from a shared link) or use a logged-in session for search and comments.

**Can I scrape Bilibili comments without logging in?** Yes, but Bilibili shows anonymous visitors only about 3 to 4 top comments per video. Full comment lists need a logged-in session cookie from an account you own.

**Is there a Weibo scraper on GitHub?** Yes, several, plus MediaCrawler for multiple platforms at once. Check the last commit date and the license, since Weibo changes its endpoints and some projects, MediaCrawler among them, forbid commercial use.

**Is it legal to scrape Chinese social media?** It depends on what you collect, where you are and how you use it. Stick to public data, keep the request rate low, respect each platform's terms, and treat posts and profiles as personal data under China's PIPL, the GDPR and any other law that applies. This is not legal advice.

## Related pages

- [How to scrape Weibo posts with Python](guides/scrape-weibo-posts-python)
- [How to scrape Bilibili comments and danmaku](guides/scrape-bilibili-comments-danmaku)
- [How to scrape Xiaohongshu (RedNote)](guides/scrape-xiaohongshu-rednote)
- [Weibo API in Python: official API, m.weibo.cn and options](guides/weibo-api-python)
- [Bilibili API in Python: bilibili-api-python and wbi signing](guides/bilibili-api-python)
- [WeChat Official Account articles scraper](guides/wechat-official-account-articles-scraper)
- [Web scraping MCP server for AI agents](guides/web-scraping-for-ai-agents-mcp)

<!-- jsonld:auto -->
<script type="application/ld+json">
[
  {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    "mainEntity": [
      {
        "@type": "Question",
        "name": "Is there a Weibo API for scraping posts?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Weibo's open platform requires a registered developer app and does not give ordinary apps keyword search over all public posts. Most people use the mobile site's JSON endpoints or a hosted scraper instead. Either way, Weibo caps keyword search at about 1,000 posts per keyword."
        }
      },
      {
        "@type": "Question",
        "name": "Is there a Xiaohongshu (RedNote) API?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Not for reading public notes. Scrapers read the public note pages (which need the xsec_token from a shared link) or use a logged-in session for search and comments."
        }
      },
      {
        "@type": "Question",
        "name": "Can I scrape Bilibili comments without logging in?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Yes, but Bilibili shows anonymous visitors only about 3 to 4 top comments per video. Full comment lists need a logged-in session cookie from an account you own."
        }
      },
      {
        "@type": "Question",
        "name": "Is there a Weibo scraper on GitHub?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Yes, several, plus MediaCrawler for multiple platforms at once. Check the last commit date and the license, since Weibo changes its endpoints and some projects, MediaCrawler among them, forbid commercial use."
        }
      },
      {
        "@type": "Question",
        "name": "Is it legal to scrape Chinese social media?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "It depends on what you collect, where you are and how you use it. Stick to public data, keep the request rate low, respect each platform's terms, and treat posts and profiles as personal data under China's PIPL, the GDPR and any other law that applies. This is not legal advice."
        }
      }
    ]
  },
  {
    "@context": "https://schema.org",
    "@type": "ItemList",
    "itemListElement": [
      {
        "@type": "ListItem",
        "position": 1,
        "item": {
          "@type": "SoftwareApplication",
          "name": "Weibo Scraper",
          "url": "https://apify.com/datagleaner/weibo-scraper",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Public posts by keyword or user: full text, timestamp, like / repost / comment counts, images, video, poster's region, author.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.003",
              "priceCurrency": "USD",
              "description": "US$0.003 per post",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.003",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "post"
                }
              }
            }
          ]
        }
      },
      {
        "@type": "ListItem",
        "position": 2,
        "item": {
          "@type": "SoftwareApplication",
          "name": "Bilibili Scraper",
          "url": "https://apify.com/datagleaner/bilibili-scraper",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Videos by keyword or link with views, likes, coins, favorites, tags and uploader; comments and replies with IP location; danmaku with time in video.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.008",
              "priceCurrency": "USD",
              "description": "US$0.008 per video",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.008",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "video"
                }
              }
            },
            {
              "@type": "Offer",
              "price": "0.002",
              "priceCurrency": "USD",
              "description": "US$0.002 per comment",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.002",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "comment"
                }
              }
            },
            {
              "@type": "Offer",
              "price": "0.0005",
              "priceCurrency": "USD",
              "description": "US$0.0005 per danmaku",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.0005",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "danmaku"
                }
              }
            }
          ]
        }
      }
    ]
  }
]
</script>
