---
title: Convert a Website to Markdown for an LLM (Free and Paid)
description: "Convert a website to Markdown for an LLM: llms.txt, Jina Reader, Firecrawl, trafilatura and markdownify in Python, and a whole docs site from its sitemap."
---

# How to convert a website to Markdown for an LLM

To convert a website to Markdown for an LLM, first check whether the site already publishes an `llms.txt` or `llms-full.txt` file, which many documentation sites now do. If it does not, fetch each page and keep only its main content as Markdown: for one page, prefix the URL with `https://r.jina.ai/`; for code, use the free Python library `trafilatura` with `output_format="markdown"`; for a whole site, take the page list from its `sitemap.xml` and convert each URL in a loop. Hosted APIs such as Firecrawl do the fetching and conversion for you on a credit plan. Each option is below, with working code, prices, and what each one cannot do.

Disclosure: Data Gleaner, one of the options near the end, is us. Every other method on this page is someone else's or free.

## Why Markdown and not raw HTML

An LLM reads tokens, and raw HTML spends most of them on tags, scripts, class names, menus and footers that carry no meaning. Markdown keeps the parts that do carry meaning (headings, lists, tables, code blocks, links) in far fewer tokens, and its headings give you natural places to split a page into chunks for retrieval (RAG). The important step is not the format change itself but **main-content extraction**: a converter that turns the whole HTML document into Markdown still gives you the navigation bar and cookie banner, only in Markdown.

## Step 0: check for llms.txt

Before scraping anything, try these two URLs on the site:

```
https://example.com/llms.txt
https://example.com/llms-full.txt
```

`llms.txt` is a proposed convention: a Markdown file at the site root that summarises the site and links to LLM-friendly pages. Some sites also publish `llms-full.txt`, the whole documentation in one Markdown file. Documentation sites adopt it most (for example, `https://docs.astral.sh/uv/llms.txt` exists). Where it exists, it is the cleanest source you can get, because the site owner wrote it for this purpose. Where it does not, use one of the options below. Some docs sites also offer a Markdown version of each page (often by adding `.md` to the URL, or a "Copy as Markdown" button); check before you build a scraper.

## Option 1: Jina Reader (one page, no code)

Put `https://r.jina.ai/` in front of any public URL and you get the page back as Markdown:

```bash
curl "https://r.jina.ai/https://docs.python.org/3/tutorial/introduction.html"
```

You can also paste that URL into a browser. Jina's Reader removes navigation, headers and footers, renders JavaScript pages in a headless browser by default, and returns plain HTTP results faster if you send the header `X-Engine: direct`.

**Cost and limits (as listed on jina.ai in October 2026):** without a key, 20 requests per minute. A free API key comes with 10 million tokens and raises the limit to 500 requests per minute. Reader is billed by the tokens in the output, so a long page costs more than a short one; after the free tokens, you buy token top-ups (check the current rate on Jina's pricing dashboard). Failed requests are not charged.

Good for: reading a few pages by hand or from a script without installing anything. Less good for: a whole site, because you still need the list of URLs (see the sitemap section below).

## Option 2: Firecrawl (hosted API with crawling)

Firecrawl is a hosted API whose scrape endpoint returns a page's main content as Markdown, with JavaScript rendering included:

```bash
curl -s -X POST "https://api.firecrawl.dev/v2/scrape" \
  -H "Authorization: Bearer fc-YOUR-API-KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://docs.python.org/3/tutorial/introduction.html", "formats": ["markdown"]}'
```

Or in Python (`pip install firecrawl-py`):

```python
from firecrawl import Firecrawl

firecrawl = Firecrawl(api_key="fc-YOUR-API-KEY")
doc = firecrawl.scrape("https://docs.python.org/3/tutorial/introduction.html", formats=["markdown"])
print(doc.markdown)
```

It also has crawl and map endpoints that discover a site's pages for you, so one vendor covers both steps.

**Pricing (firecrawl.dev, October 2026):** one credit per page scraped. The free plan gives 1,000 credits a month with 2 concurrent requests. Hobby is $16 a month billed yearly ($19 monthly) for 5,000 credits; Standard is $83 a month billed yearly for 100,000 credits. Extra formats such as JSON extraction cost 4 more credits per page. A page that returns an error status such as 403 or 404 still costs a credit. Firecrawl's core is also open source if you would rather host it yourself.

## Option 3: Python with trafilatura (free)

`trafilatura` is an open-source library built for main-content extraction, and it outputs Markdown directly (version 2 and later). Install it with `pip install requests trafilatura`:

```python
import requests
import trafilatura

url = "https://docs.python.org/3/tutorial/introduction.html"
html = requests.get(url, headers={"User-Agent": "docs-to-markdown/1.0 (you@example.com)"}, timeout=30).content

markdown = trafilatura.extract(html, output_format="markdown", include_tables=True)
print(markdown)
```

Two details that matter:

- Pass `.content` (bytes), not `.text`. With `.text`, `requests` sometimes guesses the wrong character encoding and you get garbled characters such as `Â¶` in the output.
- `include_links=True` keeps hyperlinks, which helps if your LLM should cite or follow them but costs tokens. Leave it off for the leanest text.

trafilatura decides on its own what the main content is. It works well on articles and documentation, less well on pages that are mostly tables, cards or navigation, where it can return too little or `None`.

## Option 4: Python with markdownify (free, full control)

`markdownify` converts whatever HTML you hand it, so you choose the content yourself with a CSS selector. That is more work than trafilatura, but nothing gets dropped by a guess. Install it with `pip install requests beautifulsoup4 markdownify`:

```python
import requests
from bs4 import BeautifulSoup
from markdownify import markdownify

url = "https://docs.python.org/3/tutorial/introduction.html"
html = requests.get(url, headers={"User-Agent": "docs-to-markdown/1.0 (you@example.com)"}, timeout=30).content

soup = BeautifulSoup(html, "html.parser")
main = soup.select_one("main, article, [role=main]") or soup.body
for tag in main(["script", "style", "nav", "footer", "aside"]):
    tag.decompose()

markdown = markdownify(str(main), heading_style="ATX")
print(markdown)
```

The selector matters: the Python docs, like many Sphinx sites, have no `<main>` tag and mark the content with `role="main"` instead. Without a selector, `markdownify` converts the whole page and the output starts with the navigation menu. `heading_style="ATX"` gives `#`-style headings, which are easier to split on than underlined ones.

## Convert a whole docs site from its sitemap

To convert a whole site, get the list of page URLs first, then run one of the converters above on each URL. The fastest way to get the list is the site's sitemap: look for a `Sitemap:` line in `https://example.com/robots.txt`, or try `/sitemap.xml` (see [how to find the sitemap of a website](find-sitemap-of-website)).

This script reads a sitemap (following sitemap index files), keeps only one section of the docs, and writes each page as a `.md` file with its source URL at the top. It uses `requests` and `trafilatura`:

```python
import pathlib
import time
import xml.etree.ElementTree as ET
from urllib.parse import urlparse

import requests
import trafilatura

SITEMAP = "https://docs.astral.sh/uv/sitemap.xml"
PREFIX = "https://docs.astral.sh/uv/concepts/"   # only this section
OUT = pathlib.Path("docs_md")
HEADERS = {"User-Agent": "docs-to-markdown/1.0 (you@example.com)"}
NS = "{http://www.sitemaps.org/schemas/sitemap/0.9}"


def sitemap_urls(url):
    root = ET.fromstring(requests.get(url, headers=HEADERS, timeout=30).content)
    if root.tag == NS + "sitemapindex":  # an index: follow each child sitemap
        for loc in root.iter(NS + "loc"):
            yield from sitemap_urls(loc.text.strip())
    else:
        for loc in root.iter(NS + "loc"):
            yield loc.text.strip()


OUT.mkdir(exist_ok=True)
urls = [u for u in sitemap_urls(SITEMAP) if u.startswith(PREFIX)]
print(len(urls), "pages")
for url in urls:
    html = requests.get(url, headers=HEADERS, timeout=30).content
    md = trafilatura.extract(html, output_format="markdown", include_tables=True)
    if not md:
        print("[WARN] no content:", url)
        continue
    name = urlparse(url).path.strip("/").replace("/", "_") or "index"
    (OUT / f"{name}.md").write_text(f"Source: {url}\n\n{md}\n", encoding="utf-8")
    time.sleep(1)  # be polite: about one page per second
```

What it leaves out, which you may need on other sites: gzipped (`.xml.gz`) and plain-text sitemaps, retries, and pages that only render with JavaScript. A fuller sitemap reader is in [Get all URLs from a sitemap with Python](get-all-urls-from-sitemap-python). If a site has no sitemap, or the sitemap leaves pages out, you need a link-following crawler instead (Firecrawl's crawl endpoint, or an open-source crawler such as Crawl4AI or Scrapy).

## Pages that need JavaScript

Some sites send an almost empty HTML page and build the text in the browser. `requests` then gets nothing useful, and trafilatura returns `None`. You have three choices: use a hosted reader that renders JavaScript (Jina Reader and Firecrawl both do), render the page yourself with a headless browser such as Playwright and pass the rendered HTML to trafilatura or markdownify, or look for the site's data source (many docs sites are built from Markdown files in a public GitHub repository, which you can clone directly).

## Making the Markdown work well in an LLM

- **Keep the source URL** at the top of each file or chunk, so answers can cite where they came from.
- **Split on headings**, not on a fixed character count, so each chunk is one topic.
- **Drop repeated boilerplate** (version banners, "Edit this page" lines) that appears on every page and would otherwise match every query.
- **Cap page length** if the Markdown goes straight into a prompt rather than a vector store, so one very long page does not crowd out the rest.
- **Check a sample by eye.** Open five converted files before converting five thousand; extraction mistakes are easy to see and expensive to find later.

## Option 5: Data Gleaner Actors on Apify (pay per result)

Disclosure: Data Gleaner is us. We publish two small Actors on the Apify Store for the two steps above. You run them from the Apify console, the API, or as tools in an MCP client, and pay per result through your Apify account.

- **Sitemap URL Extractor** ([apify.com/datagleaner/sitemap-extractor](https://apify.com/datagleaner/sitemap-extractor)) does step 1: give it domains, and it finds the sitemaps through robots.txt and common paths, follows sitemap indexes, reads `.xml.gz` and plain-text sitemaps, and returns one row per page URL with `lastmod`. `includeUrlPatterns` keeps one section, such as `/docs/`. It costs $0.20 per 1,000 URLs. It does not follow links, so it finds only pages the sitemap lists.
- **Web to Markdown** (`web-to-markdown`) does step 2: give it URLs (or a search query), and it returns each page's main content as Markdown, with title, language and word count. It fetches over plain HTTP and starts a headless browser only for pages that come back empty. A `maxCharsPerPage` cap protects an LLM's context window. It costs $1 per 1,000 pages, and failed URLs are free. It is not listed on the Store yet; see [apify.com/datagleaner](https://apify.com/datagleaner) for when it is. <!-- TODO(store-link: web-to-markdown) -->

Listing a docs section with the Sitemap URL Extractor from Python (`pip install apify-client`, then set `APIFY_TOKEN`). This run returns at most 200 URLs, so it costs at most $0.04:

```python
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/sitemap-extractor").call(run_input={
    "websites": ["https://docs.astral.sh/uv/sitemap.xml"],
    "maxUrlsPerSite": 200,
    "includeUrlPatterns": ["/uv/concepts/"],
})
if run is None:
    raise SystemExit("The Actor run did not start.")

urls = [item["url"] for item in client.dataset(run.default_dataset_id).iterate_items()]
print(len(urls), "URLs")
```

Feed `urls` to the free trafilatura loop above, or to any converter on this page.

Both Actors read public pages only and see what a logged-out visitor sees. Sites that refuse automated readers return an error item from Web to Markdown, free of charge; it does not solve CAPTCHAs.

## Which option to choose

| Option | Cost | JavaScript pages | Finds a site's pages | Best for |
|---|---|---|---|---|
| `llms.txt` / `llms-full.txt` | Free | Not needed | Already one file | Docs sites that publish it |
| Jina Reader | Free key with 10M tokens, then token top-ups | Yes | No | A few pages, no code |
| Firecrawl | Free 1,000 pages a month; Hobby $16-19 a month for 5,000 | Yes | Yes (crawl, map) | One vendor for crawl and convert |
| trafilatura | Free | No (add Playwright) | No (add a sitemap reader) | Articles and docs in your own code |
| markdownify | Free | No (add Playwright) | No | Full control over what is kept |
| Data Gleaner Actors (us) | $0.20 per 1,000 URLs; $1 per 1,000 pages | Browser fallback | Sitemaps only | Pay-per-result runs from the API or an agent |

## Caveats

- **Terms and robots.txt.** Check the site's terms of use and robots.txt before converting it in bulk, keep your request rate low, and identify yourself in the User-Agent.
- **Copyright.** Page text is usually copyrighted. Using it for your own retrieval and analysis is different from republishing it; know which one you are doing.
- **Logins and paywalls.** Every option here sees only the public page. Content behind a login is out of scope.
- **Prices change.** The prices above were read from each vendor's site in October 2026; check before you commit to a large job.

## FAQ

**Is there a free way to convert a website to Markdown?**
Yes. `llms.txt` files, Jina Reader's free tier, Firecrawl's free 1,000 pages a month, and the Python libraries trafilatura and markdownify all cost nothing. For a large site, the Python route is the one with no usage limit, at the cost of writing and running the script yourself.

**Is there a website to Markdown API?**
Yes, several: Jina Reader (`r.jina.ai`), Firecrawl's `/v2/scrape` endpoint, and Apify Actors such as our Web to Markdown, which return one item per URL with a `markdown` field.

**How do I convert HTML to Markdown in Python?**
Use `markdownify` to convert a piece of HTML you have selected, or `trafilatura.extract(html, output_format="markdown")` to extract the main content and convert it in one step. Both are shown above.

**Can I convert a website to Markdown through MCP?**
Yes. An MCP client such as Claude or Cursor can call a converter as a tool. Firecrawl publishes an MCP server, and any Apify Actor, including ours, can be added through Apify's hosted MCP server at `https://mcp.apify.com`.

**Should I give an LLM Markdown or plain text?**
Markdown, in most cases. Headings, lists, tables and code blocks tell the model how the content is structured, at a small token cost over plain text. Plain text is fine for short prose with no structure.

## Related pages

- [Website to Markdown and sitemap scraper APIs](../web-content-scrapers)
- [Get all URLs from a sitemap with Python](get-all-urls-from-sitemap-python)
