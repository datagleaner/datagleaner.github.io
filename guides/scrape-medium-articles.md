---
title: Scrape Medium Articles with Python (RSS, API, Tools)
description: How to scrape Medium articles with Python - RSS feeds for authors, tags and publications, the unofficial API, paywall limits and a hosted option.
---

# How to scrape Medium articles with Python

The simplest way to scrape Medium articles with Python is the public RSS feed: `https://medium.com/feed/@handle` for an author, `https://medium.com/feed/<publication>` for a publication and `https://medium.com/feed/tag/<tag>` for a tag. Parse it with `xml.etree` or `feedparser` and you get title, link, author, date, tags and, for free stories from authors and publications, the full article HTML. The catch is that every feed holds only the 10 newest stories, tag feeds carry only a short snippet, and member-only stories give you a preview, not the text. For more than that you need Medium's undocumented internal API, a browser, or a hosted scraper. This page covers each option, with code you can run, and what each one cannot do.

Disclosure: Data Gleaner, mentioned near the end as one hosted option, is us. Everything before that section is free and needs no account.

## Which method fits your job

| You need | Best method | Cost | Main limit |
|---|---|---|---|
| The latest posts of a few authors or publications | RSS feed | Free | 10 newest stories per feed |
| New posts under a tag, as they appear | RSS tag feed, polled on a schedule | Free | Title, link and snippet only, no full text |
| Full text of free articles you already have links to | RSS (if recent) or a browser | Free | Plain HTTP requests to article pages are often refused with a 403 |
| An author's or publication's whole archive, claps, follower counts | Medium's internal GraphQL API, or a hosted scraper | Free (your time) or per article | Undocumented and can change without notice |
| Member-only article text | Not possible without a paid Medium account, and not covered here | - | Paywall |

## Method 1: Medium RSS feeds (free, no key)

Medium publishes an RSS feed for every author, publication and tag. These are the URL patterns:

| Feed | URL |
|---|---|
| Author | `https://medium.com/feed/@netflixtechblog` |
| Publication on medium.com | `https://medium.com/feed/better-programming` |
| Publication on its own domain | `https://netflixtechblog.com/feed` |
| Tag | `https://medium.com/feed/tag/python` |

What we saw when we fetched these feeds in October 2026:

- **Each feed returns exactly 10 items**, the newest ones. There is no page parameter, so older stories are not reachable this way. If you need more, poll the feed on a schedule and keep what you collect.
- **Author and publication feeds** include the full article HTML in `content:encoded` for stories that are free to read. Other items in the same feed carry only a short `description` snippet, so check which field is present.
- **Tag feeds** carry only a snippet and a link for every item, never the full text.
- Feeds do not include claps, response counts or reading time.

This script reads any of the three feed types and turns each item into plain text:

```python
# pip install requests beautifulsoup4
import time
import xml.etree.ElementTree as ET

import requests
from bs4 import BeautifulSoup

CONTENT = "{http://purl.org/rss/1.0/modules/content/}encoded"
CREATOR = "{http://purl.org/dc/elements/1.1/}creator"
session = requests.Session()
session.headers["User-Agent"] = "medium-rss-reader/1.0 (contact: you@example.com)"


def medium_feed(path):
    """path: '@handle', 'tag/python', or a publication slug such as 'better-programming'."""
    r = session.get(f"https://medium.com/feed/{path}", timeout=20)
    r.raise_for_status()
    for item in ET.fromstring(r.content).iter("item"):
        full_html = item.findtext(CONTENT)
        html = full_html or item.findtext("description") or ""
        yield {
            "title": item.findtext("title"),
            "url": item.findtext("link").split("?")[0],
            "author": item.findtext(CREATOR),
            "published": item.findtext("pubDate"),
            "tags": [c.text for c in item.findall("category")],
            "has_full_text": full_html is not None,
            "text": BeautifulSoup(html, "html.parser").get_text("\n", strip=True),
        }


for path in ["@netflixtechblog", "tag/python", "better-programming"]:
    for post in medium_feed(path):
        print(path, "|", post["title"], "|", post["has_full_text"], "|", len(post["text"]), "chars")
    time.sleep(2)
```

`has_full_text` tells you whether the item came with the whole article or only a snippet. If you want Markdown rather than plain text, pass the HTML to a converter such as `markdownify` or `html2text` instead of `get_text`.

For a publication with its own domain, request `https://<domain>/feed` directly instead of going through `medium_feed`.

## Method 2: Medium's API (official and unofficial)

**The official Medium API is not a reading API.** It was built for publishing posts to your own account, and Medium's help center says it no longer issues new integration tokens or allows new integrations. It does not return other people's articles.

**The unofficial API is what Medium's own web app uses.** The site loads data from a GraphQL endpoint at `https://medium.com/_/graphql`. It returns far more than RSS: paged author and publication archives, claps, response counts, reading time, follower counts and the article body as structured paragraphs. You can find the operations and their variables by opening your browser's developer tools on a Medium page and watching the network requests to `_/graphql`.

Before building on it, know what you are taking on:

- It is undocumented. Query names, fields and required headers change when Medium changes its front end, and nothing tells you when.
- Medium throttles it. Keep a pause of a second or more between requests, retry slowly on errors, and stop if you are refused.
- Old tutorials that append `?format=json` to a Medium URL no longer work reliably: in our test that request returned a 403.

There are also third-party "Medium API" services sold on API marketplaces such as RapidAPI. They wrap the same public data behind a key and a monthly plan. Check their current price and rate limits before you rely on one.

## Method 3: a browser for single articles

If you only need the text of a few free articles you already have links to, use a real browser. In our tests, plain `requests` calls to article pages were answered with a 403 page instead of the article. Playwright or Selenium loads the page the way a normal visitor's browser does, and you read the text from the `<article>` element:

```python
# pip install playwright && playwright install chromium
from playwright.sync_api import sync_playwright

url = "https://netflixtechblog.com/trading-a-cloud-identity-for-your-own-workload-attestation-on-managed-compute-516d5a29b252"
with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto(url, wait_until="domcontentloaded")
    print(page.locator("article").inner_text()[:2000])
    browser.close()
```

This is slow (a full browser per page) and fine for tens of articles, not thousands. If a page shows a challenge or block, treat that as the site's answer and use RSS or another source; do not try to get around it.

## The paywall: what you can and cannot get

Member-only stories show a logged-out visitor the first part of the article and a prompt to subscribe. Every free method above returns only that preview. Getting the rest would mean scraping through a paid account, which Medium's terms do not allow and which takes the authors' paid work. Plan your dataset around previews for those stories, and keep the `has_full_text` flag (or its equivalent) so you know which rows are partial.

Two more rules worth following whichever method you use:

- **Articles are the authors' copyrighted work.** Scraping them for analysis, search or research is a different thing from republishing them. Link back instead of copying full text onto another site.
- **Author names and handles are personal data** in many jurisdictions. If you store them, you need a lawful basis under GDPR, CCPA or whatever law applies to you.

## A hosted option for bulk scraping

If you need more than 10 stories per feed, full archives, claps and follower counts, or results on a schedule without maintaining a GraphQL client, a hosted scraper does the paging, retries and parsing for you. Several are listed on the Apify Store. One of them is Data Gleaner's **Medium Scraper** (`medium-scraper`). Disclosure: Data Gleaner is us.

**Medium Scraper is not on the Apify Store yet.** Until it is listed, see [Data Gleaner on Apify](https://apify.com/datagleaner). <!-- TODO(store-link: medium-scraper) -->

What it does, from its documentation:

- Takes any mix of article URLs, author handles, tags (newest or top order), publications (slug, URL or custom domain) and a search query, with a cap per source (up to 1,000 each).
- Returns one JSON item per article: title, subtitle, author with follower count, publication, dates, reading time, claps, response count, tags, an `isPaywalled` flag, and the text as both plain text and Markdown.
- Optionally returns the responses (comments) under each article as separate items.
- Uses Medium's public GraphQL endpoint, article pages and RSS, with no login, no cookies and no browser. If Medium refuses the main endpoint, it falls back to RSS and says so in the log, which means 10 stories per feed without claps or follower counts.
- Member-only stories come back as the public preview, marked `contentIsPartial: true`. It does not log in and cannot read paywalled text.
- Costs $0.003 per article ($3 per 1,000) and $0.0005 per response. You pay only for items returned.

The same tag query as above, run through it:

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/medium-scraper").call(
    run_input={
        "tags": ["machine-learning"],
        "tagSort": "top",
        "maxArticlesPerTag": 10,
        "includeContent": False,
    }
)
if run is None:
    raise SystemExit("The run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    author = (item.get("author") or {}).get("name")
    print(f'{item.get("title")} | {author} | {item.get("claps")} claps | {item.get("readingTimeMinutes")} min | {item.get("url")}')
```

That run returns up to 10 articles, so it costs at most $0.03. Set `includeContent` to `True` to add `contentText` and `contentMarkdown`; the price per article is the same.

## FAQ

**How do I scrape Medium articles with Python for free?**
Read the RSS feed for the author, publication or tag (`https://medium.com/feed/...`) with `requests` and an XML parser, as in Method 1. It is free and needs no key, but returns only the 10 newest stories per feed, and tag feeds give snippets rather than full text.

**What is the Medium RSS feed limit?**
Every Medium feed returns the 10 most recent stories, and there is no parameter to page further back. To build a longer history, poll the feed regularly and store new items, or use a source that pages through the archive.

**What is the Medium RSS feed URL for a user or publication?**
For a user, `https://medium.com/feed/@username`. For a publication on medium.com, `https://medium.com/feed/publication-slug`. For a publication on its own domain, `https://domain.com/feed`. For a tag, `https://medium.com/feed/tag/tagname`.

**Is there a Medium API for Python?**
Not an official one for reading. Medium's official API only published posts and no longer issues tokens. You can call the undocumented GraphQL endpoint the website uses, pay for a third-party wrapper, or run a hosted scraper through the `apify-client` Python package.

**Can I scrape member-only Medium articles?**
Only the preview Medium shows to logged-out visitors. The full text of member-only stories is behind the paywall; respect it.

## Related guides

- [How to find the sitemap of a website](find-sitemap-of-website), another way to list every article URL on a blog that publishes one.
- [WeChat Official Account articles scraper](wechat-official-account-articles-scraper), for collecting articles from China's main publishing platform.
- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp), to let Claude or Cursor run scrapers like the ones above.
