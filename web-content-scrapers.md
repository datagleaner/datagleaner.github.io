---
title: Website to Markdown and Sitemap Scraper APIs
description: "Website to Markdown and sitemap scraper API options: free Python code, hosted APIs, and pay-per-result Actors for sitemaps, pages, Medium, Behance and Telegram."
---

# Website to Markdown and sitemap scraper APIs

To turn a whole website into Markdown, do it in two steps: first list every page URL from the site's `sitemap.xml`, then fetch each URL and convert only its main content to Markdown. You can do both for free with about 30 lines of Python, use a hosted crawl API, or call two small pay-per-result APIs on the Apify Store. This page covers all three, plus scrapers for three content sites (Medium, Behance and Telegram) whose text is easier to get from their own feeds than from raw HTML.

Disclosure: Data Gleaner is us. The Actors in the table below are ours.

## Step 1: get every URL from the sitemap

Most sites publish their page list for search engines. The usual places are:

1. The `Sitemap:` lines in `https://example.com/robots.txt`.
2. `/sitemap.xml`, `/sitemap_index.xml`, or `/wp-sitemap.xml` on WordPress.

A sitemap can be a **sitemap index** that points to more sitemaps, sometimes gzipped (`.xml.gz`). Each `<url>` entry has a `<loc>` and may have `<lastmod>`, `<changefreq>` and `<priority>`. Many sites publish only `<loc>`.

The limit to know up front: a sitemap lists only what the site chooses to list. A page left out of every sitemap is not found this way; you would need a link crawler for that.

## Step 2: convert each page to Markdown

Raw HTML-to-Markdown conversion keeps the menu, footer and cookie banner. For LLM and RAG use you want the main content only, which is what "readability" extractors do. Free Python libraries for this include `trafilatura` (main-content extraction with Markdown output) and `markdownify` (converts whatever HTML you give it, so pair it with your own content selector).

Pages that render their text with JavaScript return almost nothing over plain HTTP. For those you need a headless browser such as Playwright, which is slower and heavier.

## Free option: do it yourself in Python

This script reads robots.txt, follows sitemap indexes, and writes each page as a `.md` file. It needs `pip install requests trafilatura` (trafilatura 2.x for Markdown output).

```python
import gzip
import pathlib
import re
import time
import xml.etree.ElementTree as ET

import requests
import trafilatura

SITE = "https://docs.python.org"
HEADERS = {"User-Agent": "my-docs-export/1.0 (contact: you@example.com)"}
NS = "{http://www.sitemaps.org/schemas/sitemap/0.9}"


def fetch(url):
    r = requests.get(url, headers=HEADERS, timeout=30)
    r.raise_for_status()
    body = r.content
    if body[:2] == b"\x1f\x8b":  # gzipped sitemap
        body = gzip.decompress(body)
    return body


def sitemap_urls(site):
    robots = fetch(site + "/robots.txt").decode("utf-8", "ignore")
    queue = re.findall(r"(?im)^sitemap:\s*(\S+)", robots) or [site + "/sitemap.xml"]
    seen, pages = set(), []
    while queue:
        sm = queue.pop()
        if sm in seen:
            continue
        seen.add(sm)
        root = ET.fromstring(fetch(sm))
        if root.tag == NS + "sitemapindex":
            queue += [e.text.strip() for e in root.iter(NS + "loc")]
        else:
            pages += [e.text.strip() for e in root.iter(NS + "loc")]
    return list(dict.fromkeys(pages))


out = pathlib.Path("markdown")
out.mkdir(exist_ok=True)
for i, url in enumerate(sitemap_urls(SITE)[:50]):  # cap while testing
    html = requests.get(url, headers=HEADERS, timeout=30).text
    md = trafilatura.extract(html, output_format="markdown", include_links=True)
    if md:
        (out / f"{i:05d}.md").write_text(f"<!-- {url} -->\n\n{md}", encoding="utf-8")
    time.sleep(1)  # be polite
```

What this does not handle, and what you would add for real use: plain-text sitemaps, broken XML, JavaScript-rendered pages, retries and backoff, robots.txt `Disallow` rules for the pages themselves, and parallel fetching. For a fuller walkthrough of the first half, see [Get all URLs from a sitemap with Python](guides/get-all-urls-from-sitemap-python); for the second half, [Convert a website to Markdown for an LLM](guides/convert-website-to-markdown-for-llm).

## Hosted options

- **Crawl APIs.** Services such as Firecrawl offer a crawl or "map" endpoint for URL discovery and a scrape endpoint that returns Markdown. They suit you if you want one vendor for both steps and are fine with a subscription or credit plan. Check each one's current pricing and whether it reads sitemaps or only follows links.
- **Reader endpoints.** Some services return Markdown when you prefix a URL with their endpoint. They are convenient for one page at a time; for a whole site you still need the URL list from step 1.
- **Apify Actors.** Each Actor is an API with pay-per-result pricing, so a run that returns nothing costs nothing. Several sellers on the Apify Store offer website-to-Markdown crawlers; ours are below.

## Data Gleaner web content Actors

Five Actors for getting text and links out of the web. All read public pages only, need no login or cookies, and are billed per result through your Apify account.

| Actor | What it returns | Price | Store |
|---|---|---|---|
| Sitemap URL Extractor (`sitemap-extractor`) | Every page URL a site's sitemaps list, with `lastmod`, `changefreq`, `priority`, image count and source sitemap. Reads robots.txt, sitemap indexes, `.xml.gz` and plain-text sitemaps. Does not crawl links. | $0.20 per 1,000 URLs | [apify.com/datagleaner/sitemap-extractor](https://apify.com/datagleaner/sitemap-extractor) |
| Web to Markdown (`web-to-markdown`) | Main content of each URL (or of the top results for a search query) as Markdown, with title, language, word count. Plain HTTP first, headless browser only when a page comes back empty. Does not follow links. | $1 per 1,000 pages; failed URLs are free | Not listed yet: [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: web-to-markdown) --> |
| Medium Scraper (`medium-scraper`) | Medium articles by URL, author, tag, publication or search, with full text and Markdown, claps, tags, author followers; optional responses (comments). Member-only stories return the public preview only. | $3 per 1,000 articles; $0.50 per 1,000 responses | Not listed yet: [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: medium-scraper) --> |
| Behance Scraper (`behance-scraper`) | Behance projects by keyword or URL (appreciations, views, fields, tools, tags, image URLs) and creator profiles (location, followers, freelance availability). | $2 per 1,000 projects; $3 per 1,000 profiles | Not listed yet: [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: behance-scraper) --> |
| Telegram Channel Scraper (`telegram-channel-scraper`) | Messages from public channels with date, text, views, reactions, media links, forwards, polls and links, plus one info row per channel (subscribers, description). Reads Telegram's public `t.me/s/` preview, so no account or phone number. | $0.50 per 1,000 messages; $1 per 1,000 channel rows | Not listed yet: [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: telegram-channel-scraper) --> |

Prices are what the Actor charges per result; Apify platform usage is included in them, and every run stops when it reaches the maximum total charge you set.

## Who uses these for what

- **RAG and AI agent builders.** List a documentation site's pages with the Sitemap URL Extractor, filter to the sections you need with `includeUrlPatterns` (for example `/docs/`), then fetch those pages as Markdown. Web to Markdown has a `maxCharsPerPage` cap so a long page does not flood an agent's context.
- **SEO audits and migrations.** Export a site's full URL inventory with `lastmod` before a redesign, or run it daily and compare `lastmod` to spot new and changed pages on a competitor's site.
- **Content and trend research.** Pull a Medium tag's top or newest articles with their claps, or the most-appreciated or newest Behance projects for a keyword, to see what is being published and what gets engagement.
- **News, crypto and OSINT monitoring.** Archive public Telegram channels with views and links, using `onlyNewerThan` for a daily or weekly feed.
- **Creative sourcing.** Search Behance for a style, then fetch the profiles of the creators you like to see who is available for freelance work.

## Python quick start: sitemap URLs through the API

Install the client with `pip install apify-client`, then set `APIFY_TOKEN` to your Apify API token. This run returns at most 10 URLs, so it costs at most $0.002.

```python
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/sitemap-extractor").call(run_input={
    "websites": ["stripe.com"],
    "maxUrlsPerSite": 10,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item["url"], item.get("lastmod"), item.get("priority"), item.get("sitemapUrl"))
```

To narrow a large site, add `"includeUrlPatterns": ["/blog/"]` or `"excludeUrlPatterns": ["\\.pdf$"]` (Python regular expressions). Feed the resulting URLs to the free trafilatura loop above, or to any Markdown converter you prefer.

The same run over plain HTTP, returning the rows in the response:

```bash
curl -X POST "https://api.apify.com/v2/acts/datagleaner~sitemap-extractor/run-sync-get-dataset-items?token=<YOUR_APIFY_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"websites": ["stripe.com"], "maxUrlsPerSite": 1000, "includeUrlPatterns": ["/docs/"]}'
```

## Use them from AI agents (Apify MCP)

Claude, Cursor and other MCP clients can call any of these Actors as a tool through Apify's hosted MCP server at `https://mcp.apify.com`. The `tools` parameter preloads an Actor, so the agent sees it as a ready tool:

```
https://mcp.apify.com?tools=datagleaner/sitemap-extractor
```

In Claude Code:

```bash
claude mcp add --transport http apify "https://mcp.apify.com?tools=datagleaner/sitemap-extractor"
```

On first use your client opens a browser to sign in to Apify; clients that take a header can send `Authorization: Bearer <YOUR_APIFY_TOKEN>` instead. Several Actors can be preloaded at once, separated by commas. Then ask in plain words, for example:

> Use datagleaner/sitemap-extractor to list every /blog/ URL on stripe.com, capped at 500, and tell me which ones were updated in the last 30 days.

## Caveats

- **Sitemaps are incomplete on some sites.** No sitemap means no URLs from the extractor; the run summary says so. A link crawler is the fallback.
- **Bot protection.** Some sites refuse automated readers from datacenter IPs. Web to Markdown returns a free error item for those pages and does not solve CAPTCHAs.
- **Logins and paywalls.** Everything here sees only what a logged-out visitor sees. Medium member-only stories come back as the public preview, flagged `contentIsPartial`.
- **Copyright and personal data.** Page text is usually copyrighted, so use it for analysis and retrieval rather than republishing. Telegram messages, Behance profiles and Medium authors can include personal data, and you are responsible for a lawful basis under GDPR and similar laws. Respect each site's terms and robots.txt.

## FAQ

**How do I convert a whole website to Markdown?**
List the site's page URLs from its sitemap, then convert each page's main content to Markdown. The Python script above does both for small sites. For larger ones, the Sitemap URL Extractor returns the URL list and any page-to-Markdown converter handles the second step.

**Is there an API that converts a URL to Markdown?**
Yes, several: hosted crawl services, reader endpoints, and Apify Actors such as Web to Markdown, which returns one item per URL with a `markdown` field. You can also self-host with trafilatura.

**How do I extract all URLs from a sitemap.xml?**
Read the `Sitemap:` lines in robots.txt (or try `/sitemap.xml`), follow any `<sitemapindex>` to its child sitemaps, and collect every `<loc>`. Unzip `.xml.gz` files first. The code above shows it, and [this guide](guides/get-all-urls-from-sitemap-python) goes step by step.

**Is a sitemap scraper the same as a crawler?**
No. A sitemap scraper reads the page list the site publishes, which is fast and cheap but finds only listed pages. A crawler follows links from page to page, which finds unlisted pages but takes longer and makes many more requests.

**Can I scrape a Telegram channel without an account?**
For public channels, yes: Telegram serves a web preview at `t.me/s/<channel>` that anyone can open. Private channels and channels with the preview turned off are not reachable that way. Telegram's official API (for example through the Telethon library) needs an account and an API ID.

## Related pages

- [Convert a website to Markdown for an LLM](guides/convert-website-to-markdown-for-llm)
- [Get all URLs from a sitemap with Python](guides/get-all-urls-from-sitemap-python)
- [Contact and lead scrapers](contact-and-lead-scrapers)
- [How to find the sitemap of a website](guides/find-sitemap-of-website)
- [Scrape Medium articles](guides/scrape-medium-articles)
- [Export Telegram channel messages](guides/export-telegram-channel-messages)
- [Web scraping for AI agents with MCP](guides/web-scraping-for-ai-agents-mcp)
