---
title: "Google News Scraper in Python: RSS Feeds and Real URLs"
description: "Build a Google News scraper in Python with the free RSS feed and feedparser, decode news.google.com links to publisher URLs, or use GNews or a hosted API."
---

# How to scrape Google News with Python

The simplest Google News scraper in Python does not parse HTML at all: Google News publishes a free RSS feed for any search (`https://news.google.com/rss/search?q=...`), for the front page and for each section, and `feedparser` turns it into Python objects in a few lines. Each feed returns up to about 100 articles with title, outlet and publish time. The one hard part is that every article link points to `news.google.com`, not the publisher, so you need a second step to decode it. This page shows working code for both steps, the libraries that wrap them (GNews, googlenewsdecoder), their limits, and a hosted option if you would rather not maintain it.

## Step 1: read a Google News RSS search feed

Install the two packages:

```bash
pip install feedparser requests
```

Then build the feed URL and parse it:

```python
from urllib.parse import urlencode

import feedparser
import requests

HEADERS = {"User-Agent": "Mozilla/5.0 (compatible; news-reader/1.0)"}


def google_news_search(query, hl="en-US", gl="US", ceid="US:en"):
    """Yield one dict per article in a Google News RSS search feed."""
    url = "https://news.google.com/rss/search?" + urlencode(
        {"q": query, "hl": hl, "gl": gl, "ceid": ceid}
    )
    response = requests.get(url, headers=HEADERS, timeout=20)
    response.raise_for_status()
    feed = feedparser.parse(response.content)
    for entry in feed.entries:
        yield {
            "title": entry.title,
            "source": entry.get("source", {}).get("title"),
            "published": entry.get("published"),
            "google_news_url": entry.link,
        }


for article in google_news_search('"electric vehicles" when:1d'):
    print(article["published"], "|", article["source"], "|", article["title"])
```

When we ran this on 2026-10-09, the `"electric vehicles" when:1d` query returned 100 articles. Things to know about the feed:

- **The query takes Google's search operators.** `"exact phrase"`, `OR`, `-exclude`, `site:reuters.com` and `intitle:` all work. `when:1h`, `when:1d` or `when:7d` limit results to recent coverage, and `after:2026-03-01 before:2026-03-08` limits them to a date range.
- **`hl`, `gl` and `ceid` choose the edition.** `hl` is the interface language, `gl` the country, and `ceid` is `COUNTRY:language`. Examples: `en-GB`, `GB`, `GB:en`; `ja`, `JP`, `JP:ja`; `zh-TW`, `TW`, `TW:zh-Hant`; `de`, `DE`, `DE:de`. Write the query in the edition's language to get matching results.
- **Titles end with the outlet name** (`"... - Reuters"`). Strip the suffix if you store titles; the outlet is also in `entry.source`.
- **Front page and sections have their own feeds.** The front page is `https://news.google.com/rss?hl=en-US&gl=US&ceid=US:en`, and sections are `https://news.google.com/rss/headlines/section/topic/BUSINESS?hl=en-US&gl=US&ceid=US:en` (also `WORLD`, `NATION`, `TECHNOLOGY`, `ENTERTAINMENT`, `SPORTS`, `SCIENCE`, `HEALTH`). The section URL redirects once; `requests` follows it for you. On section feeds, each item's `description` holds an HTML list of related stories from other outlets.

## Step 2: resolve news.google.com links to the publisher's URL

Every `entry.link` looks like `https://news.google.com/rss/articles/CBMi...?oc=5`. Google used to encode the publisher URL inside that id in base64, so older tutorials decode it locally. Current ids are not decodable that way: opening one in a browser runs JavaScript that asks Google for the real URL. Following redirects with `requests` therefore does not work either.

The same two requests the browser makes can be made from Python. The article page carries a signature and a timestamp in `data-n-a-sg` and `data-n-a-ts` attributes, and a POST to Google News's `batchexecute` endpoint exchanges them for the publisher URL:

```python
import json
import re


def decode_google_news_url(google_news_url):
    """Return the publisher's URL for a news.google.com/rss/articles/... link, or None."""
    article_id = google_news_url.split("/articles/")[1].split("?")[0]
    page = requests.get(f"https://news.google.com/rss/articles/{article_id}",
                        headers=HEADERS, timeout=20).text
    sg = re.search(r'data-n-a-sg="([^"]+)"', page)
    ts = re.search(r'data-n-a-ts="([^"]+)"', page)
    if not (sg and ts):
        return None
    inner = ["garturlreq",
             [["X", "X", ["X", "X"], None, None, 1, 1, "US:en", None, 1, None, None,
               None, None, None, 0, 1], "X", "X", 1, [1, 1, 1], 1, 1, None, 0, 0, None, 0],
             article_id, int(ts.group(1)), sg.group(1)]
    body = {"f.req": json.dumps([[["Fbv4je", json.dumps(inner), None, "generic"]]])}
    r = requests.post("https://news.google.com/_/DotsSplashUi/data/batchexecute",
                      data=body, headers=HEADERS, timeout=20)
    try:
        payload = json.loads(json.loads(r.text.split("\n\n", 1)[1])[0][2])
        return payload[1]
    except (IndexError, ValueError, TypeError):
        return None
```

Use it with the search function from step 1, and pause between articles:

```python
import time

for article in list(google_news_search('"electric vehicles" when:1d'))[:10]:
    article["url"] = decode_google_news_url(article["google_news_url"])
    print(article["source"], "|", article["url"])
    time.sleep(1)
```

On 2026-10-09 this returned publisher URLs such as `https://www.politico.eu/article/china-says-it-reaches-understanding-with-eu-on-hybrid-cars/`. Caveats:

- **This is an undocumented internal endpoint.** Google can change the attributes or the request format at any time, and the function then returns `None`. Keep the `google_news_url` in your data so nothing is lost when decoding fails.
- **It costs two requests per article,** and the article page is a large HTML document, so decoding 100 articles takes noticeably longer than reading the feed. Keep a delay between articles and decode only the articles you will use.
- **Getting the article text is a third step.** Once you have the publisher URL, fetch the page and extract the text with a library such as `trafilatura` or `newspaper3k`. Paywalled sites and sites that block automated requests will not return it; respect that.

## Libraries that do this for you

| Option | What it does | Cost | Watch out for |
|---|---|---|---|
| `feedparser` + your own code (above) | Search, section and front-page feeds; decoding with the function above | Free | You maintain the decoder when Google changes it |
| [GNews](https://pypi.org/project/gnews/) (`pip install gnews`) | Wraps the RSS feeds: `get_news(query)`, top news, topics, language, country, period, date range, max results | Free | Returns the `news.google.com` link, not the publisher URL |
| [googlenewsdecoder](https://pypi.org/project/googlenewsdecoder/) | Decodes `news.google.com` links: `gnewsdecoder(url, interval=1)` returns `{"success": ..., "decoded_url": ...}` | Free | Version 0.2.1 failed to import with selectolax 1.0 in our test; `pip install "selectolax<1"` fixed it |
| pygooglenews | An older RSS wrapper that many tutorials still use | Free | No release in years and old dependency pins; prefer GNews |
| SERP APIs (SerpApi, ScrapingBee, Scrapingdog and others) | Google News results as JSON from their own infrastructure | Paid per search, with small free tiers | Billed per request; check which fields and which Google page (News tab or news.google.com) they return |
| Google News Scraper on Apify (ours, below) | Search, sections and any news.google.com URL, with publisher URLs decoded | $1.50 per 1,000 articles | Not free; needs an Apify account |

GNews and googlenewsdecoder together give you the same result as the code above:

```python
# pip install gnews googlenewsdecoder
from gnews import GNews
from googlenewsdecoder import gnewsdecoder

google_news = GNews(language="en", country="US", period="1d", max_results=20)
for item in google_news.get_news("electric vehicles"):
    decoded = gnewsdecoder(item["url"], interval=1)
    url = decoded["decoded_url"] if decoded.get("success") else item["url"]
    print(item["published date"], "|", item["publisher"]["title"], "|", url)
```

In GNews, `item["publisher"]` is a dict with `href` and `title`, not a string.

## A hosted option: Google News Scraper on Apify

Disclosure: Google News Scraper is ours (Data Gleaner).

**Google News Scraper is coming to the Apify Store.** Until it is listed, see [Data Gleaner on Apify](https://apify.com/datagleaner).
<!-- TODO(store-link: google-news-scraper) -->

It runs the same RSS approach on Apify's servers: you send search queries, section names (`TOP`, `WORLD`, `BUSINESS` and the rest) or any `news.google.com` URL, and get one JSON row per article with `title`, `source`, `publishedAt`, the decoded publisher URL in `articleUrl` and the original `googleNewsUrl`. Section feeds also carry `relatedArticles`. Set `language` and `country` (for example `ja` and `JP`) and it derives the `ceid` for you; `timeframe` adds a `when:` filter to every query. An optional `fetchArticleMeta` setting opens each publisher page for its image, description and author. Articles are deduplicated across the run. If one link cannot be decoded, the row still comes back with `articleUrl` empty.

It costs **$1.50 per 1,000 articles** ($0.0015 per article), billed per article returned. The 10-article run below costs about $0.015. You need an Apify account and its API token.

```bash
pip install apify-client
export APIFY_TOKEN=your_apify_token
```

```python
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-news-scraper").call(
    run_input={"queries": ["openai"], "timeframe": "1d", "maxArticlesPerQuery": 10}
)
if run is None:
    raise SystemExit("The run did not return; check it in the Apify Console.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(f'{item.get("publishedAt")}  {item.get("source")}  {item.get("title")}')
    print(f'    {item.get("articleUrl") or item.get("googleNewsUrl")}')
```

The other inputs are `topics`, `feedUrls`, `language` (default `en-US`), `country` (default `US`), `decodeUrls` (default on), `fetchArticleMeta` (default off), `requestDelaySeconds` and `maxConcurrency`. It has the same per-feed ceiling as the free route: about 100 articles per query.

## Limits that apply to every route

- **About 100 articles per feed, with no paging.** Section feeds return fewer (66 for `BUSINESS` when we checked). To collect more, split the work: narrower `when:` windows, `after:`/`before:` date ranges, `site:` operators or different wording, then remove duplicates by URL.
- **It is a recency feed, not an archive.** Date-range operators reach back further, but coverage thins out, and older items often carry only a date (the time shows as a fixed hour).
- **The feed has no article text.** For search results the RSS `description` mostly repeats the headline. Full text comes from the publisher's page.
- **Terms of use.** Google's terms for its feeds allow personal, non-commercial feed reading; check that your use is permitted. Article text and images belong to their publishers.
- **Be polite.** Send requests at a modest rate and back off on HTTP 429 or 503 rather than retrying in a tight loop.

## FAQ

**Is there an official Google News API?**
No. Google retired its Google News Search API years ago. The RSS feeds are the public, documented-by-use way to read Google News; SERP APIs and hosted scrapers sell structured access on top.

**How do I get the Google News RSS feed for a search in Python?**
Request `https://news.google.com/rss/search?q=YOUR+QUERY&hl=en-US&gl=US&ceid=US:en` and parse the response with `feedparser`, as in step 1. Change `hl`, `gl` and `ceid` together for another country or language.

**Why do Google News RSS links not go to the article?**
They point to `news.google.com/rss/articles/...`, which a browser resolves with JavaScript. Use the decode function in step 2, the googlenewsdecoder package, or a tool that decodes for you.

**How do I get Google News articles from a specific date range?**
Add `after:YYYY-MM-DD before:YYYY-MM-DD` to the query, or use `when:7d` for the last week. GNews also accepts `start_date` and `end_date`.

**How do I scrape more than 100 Google News results?**
You cannot page a single feed. Run several narrower queries (by date window, site or wording) and merge them, removing duplicates.

## Related pages

- [Google data scrapers](../google-data-scrapers)
- [Convert a website to Markdown for an LLM](convert-website-to-markdown-for-llm)
- [Web content scrapers](../web-content-scrapers)
