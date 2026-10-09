---
title: How to Find the Sitemap of a Website (7 Ways)
description: "How to find the sitemap of a website: check robots.txt, try sitemap.xml and CMS default paths, search Google, and follow sitemap index files to every URL."
---

# How to find the sitemap of a website

To find a website's sitemap, open `https://example.com/robots.txt` and look for a line that starts with `Sitemap:`. That line gives the exact sitemap URL. If there is no such line, add `/sitemap.xml` or `/sitemap_index.xml` to the domain, then try the default path for the site's CMS (WordPress uses `/wp-sitemap.xml`). If those fail, a Google search for `site:example.com filetype:xml` can show sitemap files Google has indexed. The sections below cover each method, what a sitemap index is, and how to do this for many sites at once.

Disclosure: Data Gleaner, mentioned at the end as one option for bulk extraction, is us. Every other method on this page is free and needs no account.

## 1. Check robots.txt

Most sites that publish a sitemap announce it in `robots.txt`, a plain text file that always sits at the root of the domain:

```
https://www.shopify.com/robots.txt
```

Look for lines like this:

```
Sitemap: https://www.shopify.com/sitemaps_list.xml
```

A site can list several `Sitemap:` lines (for example one per language or per section), and the file name does not have to contain "sitemap.xml", as the Shopify example shows. This is why robots.txt is the best first check: it tells you the real location instead of making you guess.

From a terminal:

```bash
curl -s https://example.com/robots.txt | grep -i '^sitemap:'
```

## 2. Try the common sitemap paths

If robots.txt has no `Sitemap:` line, try these paths on the domain, in this order:

- `/sitemap.xml`
- `/sitemap_index.xml`
- `/sitemap-index.xml`
- `/sitemap.xml.gz`
- `/sitemap.txt`

A valid sitemap opens as XML (or plain text for `.txt`) starting with `<urlset>` or `<sitemapindex>`. A styled page in your browser can still be a sitemap: some plugins attach an XSL stylesheet so the XML displays as a table. View the page source to confirm.

Check both `www` and non-`www`, and `https`, because the sitemap usually only exists on the site's main host.

## 3. Use the CMS default location

If you know (or can see from the page source) which platform runs the site, its default path usually works:

| Platform | Default sitemap URL | Notes |
|---|---|---|
| WordPress (core, 5.5 and later) | `/wp-sitemap.xml` | Built in. Replaced when an SEO plugin takes over sitemaps. |
| WordPress with Yoast SEO or Rank Math | `/sitemap_index.xml` | An index that links to post, page and category sitemaps. |
| Shopify | `/sitemap.xml` | An index linking separate sitemaps for products, collections, pages and blogs. |
| Wix | `/sitemap.xml` | Generated automatically, split by page type. |
| Squarespace | `/sitemap.xml` | Generated automatically. |
| Webflow | `/sitemap.xml` | Only when the auto-generated sitemap is switched on in site settings. |
| Blogger | `/sitemap.xml` | Generated automatically. |
| Ghost | `/sitemap.xml` | An index for posts, pages, authors and tags. |

These are defaults. A site owner or a plugin can move or rename the file, so robots.txt still wins when the two disagree.

## 4. Search Google for indexed sitemap files

Google sometimes indexes sitemap files themselves. Try these searches, replacing the domain:

```
site:example.com filetype:xml
site:example.com inurl:sitemap
```

This finds sitemaps at unusual paths, but it often returns nothing: many sitemaps are not indexed, and some plugins mark them `noindex` on purpose. Treat an empty result as "unknown", not "no sitemap".

## 5. Look for an HTML sitemap in the footer

Some sites have a human-readable "Sitemap" page linked in the footer (often `/sitemap` or `/site-map`). That is a page of links for visitors, not the XML file search engines read. It is useful for browsing a site's structure, but its links may differ from the XML sitemap.

## 6. Check Search Console or Bing Webmaster Tools (your own site)

If you own or manage the site, Google Search Console (Indexing, then Sitemaps) and Bing Webmaster Tools list every sitemap that was submitted, with its status and the number of URLs discovered. This only works for sites you have verified; you cannot see another site's submitted sitemaps there.

## 7. Script it, or use a tool for many sites

Checking one site by hand takes a minute. For a list of domains, a script is quicker. This free Python script checks robots.txt, falls back to the common paths, and follows sitemap index files down to the page URLs:

```python
# pip install requests
import gzip
import xml.etree.ElementTree as ET

import requests

FALLBACKS = ["/sitemap.xml", "/sitemap_index.xml", "/sitemap-index.xml", "/wp-sitemap.xml"]
NS = "{http://www.sitemaps.org/schemas/sitemap/0.9}"
session = requests.Session()
session.headers["User-Agent"] = "sitemap-check/1.0 (contact: you@example.com)"


def fetch(url):
    try:
        r = session.get(url, timeout=20)
    except requests.RequestException:
        return None  # host down, timeout, TLS error: skip it
    if r.status_code != 200:
        return None
    body = r.content
    if body[:2] == b"\x1f\x8b":  # gzip, whatever the file is called
        body = gzip.decompress(body)
    return body


def find_sitemaps(domain):
    root = f"https://{domain}"
    robots = fetch(root + "/robots.txt") or b""
    listed = [line.split(":", 1)[1].strip()
              for line in robots.decode("utf-8", "replace").splitlines()
              if line.lower().startswith("sitemap:")]
    if listed:
        return listed
    return [root + p for p in FALLBACKS if fetch(root + p)]


def page_urls(sitemap_url, depth=0, seen=None):
    seen = seen if seen is not None else set()
    if depth > 5 or sitemap_url in seen:
        return
    seen.add(sitemap_url)
    body = fetch(sitemap_url)
    if not body:
        return
    try:
        tree = ET.fromstring(body)
    except ET.ParseError:
        return  # broken XML; a stricter parser or recovery is needed here
    locs = [loc.text.strip() for loc in tree.iter(NS + "loc") if loc.text]
    if tree.tag == NS + "sitemapindex":
        for loc in locs:
            yield from page_urls(loc, depth + 1, seen)
    else:
        yield from locs


for domain in ["example.com", "wordpress.org"]:
    sitemaps = find_sitemaps(domain)
    print(domain, "sitemaps:", sitemaps or "none found")
    for i, url in enumerate(u for s in sitemaps for u in page_urls(s)):
        if i >= 20:
            break
        print("  ", url)
```

Limits of this script: it skips plain-text sitemaps and broken XML, it does not retry, and it fetches sequentially. Keep a pause between requests if you run it on many domains, and respect each site's terms.

If you would rather not maintain a script, there are free online "sitemap finder" tools that check the standard file names for one site at a time, and SEO crawlers such as Screaming Frog that can read a sitemap you give them.

## What is a sitemap index?

Large sites split their sitemap into several files, because one sitemap file may hold at most 50,000 URLs and 50 MB uncompressed (the limits in the [sitemaps.org protocol](https://www.sitemaps.org/protocol.html)). The file you find first is then often a **sitemap index**: an XML file whose root is `<sitemapindex>` and which lists other sitemap files instead of pages.

```xml
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap><loc>https://example.com/sitemap-posts.xml</loc></sitemap>
  <sitemap><loc>https://example.com/sitemap-products.xml</loc></sitemap>
</sitemapindex>
```

To get every page URL, open each `<loc>` in the index. Those child files contain `<urlset>` with one `<url>` per page, sometimes with `<lastmod>`, `<changefreq>` and `<priority>`. Indexes can be nested, and child files are often gzipped (`.xml.gz`).

## Extracting all sitemap URLs from many sites

When you need the page URLs from dozens or thousands of sites (for an SEO audit, a migration inventory, competitor research or a crawler's seed list), the [Sitemap URL Extractor](https://apify.com/datagleaner/sitemap-extractor) on the Apify Store does the steps above as a hosted job. Disclosure: Data Gleaner is us.

What it does, from its documentation:

- Finds sitemaps through robots.txt `Sitemap:` lines, then falls back to `/sitemap.xml`, `/sitemap_index.xml`, `/sitemap-index.xml` and `/wp-sitemap.xml`.
- Follows sitemap indexes (up to 6 levels), reads `.xml.gz` and plain-text sitemaps, and recovers what it can from broken XML.
- Returns one row per page URL with `lastmod`, `changefreq`, `priority`, image count and the sitemap file it came from, exportable as JSON, CSV or Excel.
- Skips sites with no sitemap and keeps going, with a per-site summary at the end.
- Costs $0.20 per 1,000 URLs. Sites with no sitemap cost nothing, and `maxUrlsPerSite` caps the cost per site.

It only reads sitemaps, it does not crawl links, so a page missing from every sitemap is not found. Some sites with bot protection refuse requests from datacenter IPs; for those, the Actor has an optional proxy setting.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/sitemap-extractor").call(run_input={
    "websites": ["stripe.com", "wordpress.org"],
    "maxUrlsPerSite": 10,
    "includeUrlPatterns": ["/blog/"],
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item["url"], item.get("lastmod"), item.get("sitemapUrl"))
```

`includeUrlPatterns` and `excludeUrlPatterns` take Python regular expressions, so you can keep only `/blog/` or drop PDF links.

## FAQ

**How do I find the sitemap.xml of a website?**
Open `/robots.txt` on the domain and read the `Sitemap:` line. If there is none, try `/sitemap.xml`, then `/sitemap_index.xml`, then the CMS default from the table above.

**How do I find the sitemap of any website online?**
Free online sitemap finders check the common file names for you. They work the same way as steps 1 and 2 here, so they miss sitemaps at non-standard paths that robots.txt does not list.

**Does every website have a sitemap?**
No. A sitemap is optional. Small sites with good internal links often have none, and search engines still find their pages by following links. If robots.txt, the common paths and a Google search all come up empty, the site most likely has no public sitemap.

**How do I find the sitemap of a Squarespace or Shopify site?**
Both generate one automatically at `/sitemap.xml`. On Shopify that file is an index that links to separate product, collection, page and blog sitemaps.

**How do I get all the URLs of a website from its sitemap?**
Open the sitemap, and if it is an index, open every child sitemap it lists, then collect each `<loc>` value. The Python script above does this for a few sites; for many sites a hosted extractor saves the maintenance.

## Related guides

- [Find emails from a list of websites](extract-emails-from-website-free): once you have a site's page list, pull the contact details published on it.
- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp): let Claude or Cursor run sitemap extraction as a tool.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I find the sitemap.xml of a website?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Open /robots.txt on the domain and read the Sitemap: line. If there is none, try /sitemap.xml, then /sitemap_index.xml, then the CMS default from the table above."
      }
    },
    {
      "@type": "Question",
      "name": "How do I find the sitemap of any website online?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Free online sitemap finders check the common file names for you. They work the same way as steps 1 and 2 here, so they miss sitemaps at non-standard paths that robots.txt does not list."
      }
    },
    {
      "@type": "Question",
      "name": "Does every website have a sitemap?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. A sitemap is optional. Small sites with good internal links often have none, and search engines still find their pages by following links. If robots.txt, the common paths and a Google search all come up empty, the site most likely has no public sitemap."
      }
    },
    {
      "@type": "Question",
      "name": "How do I find the sitemap of a Squarespace or Shopify site?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Both generate one automatically at /sitemap.xml. On Shopify that file is an index that links to separate product, collection, page and blog sitemaps."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get all the URLs of a website from its sitemap?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Open the sitemap, and if it is an index, open every child sitemap it lists, then collect each <loc> value. The Python script above does this for a few sites; for many sites a hosted extractor saves the maintenance."
      }
    }
  ]
}
</script>
