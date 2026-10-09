---
title: Export Sitemap URLs to Excel or CSV (Index, .gz, Many Sites)
description: "Export sitemap URLs to Excel or CSV: a free Python script for sitemap indexes, .xml.gz and many domains, an Excel Power Query method and a lastmod filter."
---

# How to export sitemap URLs to Excel or CSV

To export sitemap URLs to Excel, the quickest free route is a short script that downloads the sitemap, reads every `<loc>` (and `<lastmod>`) value and writes a CSV, which Excel opens directly. Excel can also read a sitemap itself through Power Query (Data, Get Data, From Other Sources, From Web). Both work for a plain sitemap. The harder cases are what one-file online converters usually skip: sitemap index files that point to other sitemaps, compressed `.xml.gz` files, sites that only announce their sitemap in `robots.txt`, and lists of many domains. This guide covers those, plus a `lastmod` filter that turns the export into a content audit list.

Disclosure: Data Gleaner, mentioned near the end as one option for bulk extraction, is us. Every other method on this page is free and needs no account.

## What a sitemap file contains

A sitemap is an XML file. A normal one has a `<urlset>` root with one `<url>` entry per page:

```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/blog/first-post</loc>
    <lastmod>2026-09-30</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.6</priority>
  </url>
</urlset>
```

Only `<loc>` is required. `lastmod`, `changefreq` and `priority` are optional, and many sites publish only `<loc>`. If your export has an empty date column, the site did not publish dates; nothing went wrong in the export.

Large sites split their URLs over several files, because one sitemap may hold at most 50,000 URLs and 50 MB uncompressed (the limits in the [sitemaps.org protocol](https://www.sitemaps.org/protocol.html)). The address you find first is then a **sitemap index**, whose root is `<sitemapindex>` and which lists child sitemaps instead of pages. If you paste an index into a one-file converter, you get a list of sitemap addresses, not page URLs. Child files are often gzipped (`.xml.gz`) as well.

## Method 1: a Python script (index, .gz and robots.txt)

This script takes a domain or a direct sitemap address, reads `robots.txt` for `Sitemap:` lines, falls back to the common paths, follows sitemap indexes, unpacks gzip files and writes one CSV. It uses `requests` plus the standard library.

```python
# pip install requests
import csv
import gzip
import sys
import xml.etree.ElementTree as ET

import requests

FALLBACKS = ["/sitemap.xml", "/sitemap_index.xml", "/sitemap-index.xml", "/wp-sitemap.xml"]
session = requests.Session()
session.headers["User-Agent"] = "sitemap-export/1.0 (contact: you@example.com)"


def fetch(url):
    try:
        r = session.get(url, timeout=30)
    except requests.RequestException:
        return None  # host down, timeout, TLS error: skip it
    if r.status_code != 200:
        return None
    body = r.content
    if body[:2] == b"\x1f\x8b":  # gzip magic bytes, whatever the file is called
        body = gzip.decompress(body)
    return body


def local(tag):
    return tag.rsplit("}", 1)[-1]  # drop the XML namespace


def find_sitemaps(target):
    if target.startswith("http") and target.split("?")[0].endswith((".xml", ".gz", ".txt")):
        return [target]  # a direct sitemap address
    root = target if target.startswith("http") else f"https://{target}"
    root = root.rstrip("/")
    robots = fetch(root + "/robots.txt") or b""
    listed = [line.split(":", 1)[1].strip()
              for line in robots.decode("utf-8", "replace").splitlines()
              if line.lower().startswith("sitemap:")]
    if listed:
        return listed
    return [root + p for p in FALLBACKS if fetch(root + p)]


def rows(sitemap_url, depth=0, seen=None):
    seen = seen if seen is not None else set()
    if depth > 6 or sitemap_url in seen:
        return
    seen.add(sitemap_url)
    body = fetch(sitemap_url)
    if not body:
        return
    try:
        tree = ET.fromstring(body)
    except ET.ParseError:
        # not XML: treat it as a plain-text sitemap (one URL per line)
        for line in body.decode("utf-8", "replace").splitlines():
            if line.startswith("http"):
                yield {"url": line.strip(), "lastmod": "", "sitemap": sitemap_url}
        return
    if local(tree.tag) == "sitemapindex":
        for sm in tree:
            loc = next((c.text for c in sm if local(c.tag) == "loc" and c.text), None)
            if loc:
                yield from rows(loc.strip(), depth + 1, seen)
        return
    for entry in tree:
        fields = {local(c.tag): (c.text or "").strip() for c in entry}
        if fields.get("loc"):
            yield {"url": fields["loc"], "lastmod": fields.get("lastmod", ""),
                   "sitemap": sitemap_url}


if __name__ == "__main__":
    targets = sys.argv[1:] or ["wordpress.org"]
    with open("sitemap_urls.csv", "w", newline="", encoding="utf-8-sig") as f:
        out = csv.DictWriter(f, fieldnames=["domain", "url", "lastmod", "sitemap"])
        out.writeheader()
        for target in targets:
            sitemaps = find_sitemaps(target)
            print(target, "->", len(sitemaps), "sitemap(s)")
            seen_files, seen_urls = set(), set()
            for sm in sitemaps:
                for row in rows(sm, seen=seen_files):
                    if row["url"] not in seen_urls:  # image/video sitemaps repeat pages
                        seen_urls.add(row["url"])
                        out.writerow({"domain": target, **row})
```

Save it as `sitemap_export.py` and run it with one or several domains, or with a direct sitemap address:

```bash
python sitemap_export.py example.com another-site.com
python sitemap_export.py https://example.com/sitemap-posts.xml.gz
```

The file is written with a byte-order mark (`utf-8-sig`) so Excel shows non-English characters correctly when you double-click it. A page listed in several sitemaps of the same site (a post sitemap and a video sitemap, say) is written once. Many domains work because the script loops over its arguments; for hundreds of domains, put them in a text file and read them in a loop.

Limits of this script:

- It fetches one file at a time and does not retry, so a big site takes a while.
- Broken or truncated XML is not recovered. The text-sitemap fallback reads nothing useful from it, so that file yields no rows.
- It cannot see pages that are missing from the sitemap. It reads the sitemap and does not crawl links.
- Some sites with bot protection refuse scripted requests. Keep a pause between requests on many domains and respect each site's terms.

## Method 2: filter by lastmod for a content audit

Once you have the CSV, the `lastmod` column answers audit questions: which pages have not been touched in two years, and which were updated this month. Do it in Excel (sort or filter the column; if the dates arrive as text, use `DATEVALUE(LEFT(C2,10))` in a helper column) or add a filter to the script:

```python
from datetime import date, timedelta

CUTOFF = (date.today() - timedelta(days=365)).isoformat()  # pages not updated in a year


def stale(rows_iter):
    for row in rows_iter:
        # lastmod starts with YYYY-MM-DD in both date and datetime forms
        if row["lastmod"] and row["lastmod"][:10] < CUTOFF:
            yield row
```

Swap `rows(sm, seen=seen_files)` for `stale(rows(sm, seen=seen_files))` in the main loop to export only pages whose sitemap date is older than the cutoff. Use `>=` instead to see recent changes. Two cautions: pages with no `lastmod` are dropped by this filter (they are unknown, not old), and some CMSs bump `lastmod` on every site-wide rebuild, so a date is only as honest as the plugin that writes it. Treat it as a first pass, then check the pages you plan to delete or rewrite.

## Method 3: Excel Power Query (no code)

Excel can import a sitemap without any script. In Excel for Microsoft 365 or Excel 2016 and later on Windows:

1. Go to Data, Get Data, From Other Sources, From Web, and paste the sitemap address. (For a file you saved, use From File, From XML.)
2. In the Navigator, pick the table named `url` and choose Transform Data.
3. The columns appear as `loc`, `lastmod` and so on. Remove the ones you do not need, set `lastmod` to the Date type, then Close and Load.

The same thing in the Advanced Editor, with a gzip step for `.xml.gz` files:

```
let
    Source = Web.Contents("https://example.com/sitemap.xml.gz"),
    Unzipped = Binary.Decompress(Source, Compression.GZip),
    Xml = Xml.Tables(Unzipped),
    Urls = Xml{[Name = "url"]}[Table]
in
    Urls
```

For a plain `.xml` address, drop the `Unzipped` step and pass `Source` to `Xml.Tables`. The table layout depends on what the sitemap contains, so if the `url` table has a different name or the step errors, open the Navigator and pick the table by hand. A sitemap index gives a `sitemap` table of child addresses instead; you then need one query per child file, or a function applied to that column, which is where Power Query gets tedious. Its real advantage is that a recurring report refreshes with one click. It does not read `robots.txt` for you, and it handles one sitemap address per query.

## Free online converters

Web tools that convert a sitemap to CSV, such as [xmlfiles.com](https://www.xmlfiles.com/tools/sitemap-to-csv/) and [365i.co.uk](https://www.365i.co.uk/tools/post-sitemap-to-csv/), let you paste one sitemap address and download a CSV. They are the fastest answer for a single, small, plain sitemap. Before relying on one, check whether it follows a sitemap index, whether it opens `.xml.gz`, and whether it caps the number of URLs. They take one address at a time, so many domains means many pastes.

## Many domains at once, hosted

When you have dozens or thousands of domains, or want a scheduled run without maintaining a script, the [Sitemap URL Extractor](https://apify.com/datagleaner/sitemap-extractor) on the Apify Store does the steps above as a hosted job. Disclosure: Data Gleaner is us.

What it does, from its documentation:

- Finds sitemaps through `robots.txt` `Sitemap:` lines, then falls back to `/sitemap.xml`, `/sitemap_index.xml`, `/sitemap-index.xml` and `/wp-sitemap.xml`.
- Follows sitemap indexes, reads `.xml.gz` (detected by content) and plain-text sitemaps, and recovers what it can from broken XML.
- Returns one row per page URL, deduplicated per site, with `lastmod`, `changefreq`, `priority`, image count, the sitemap file it came from and the domain, exportable as CSV, Excel or JSON from the run's dataset.
- Has an optional `checkStatus` setting that requests each page and adds its HTTP status and final URL after redirects, which finds 404s and redirects listed in the sitemap. It fills a missing `lastmod` from the page's `Last-Modified` header when the server sends one. It is slower, about 5 pages a second.
- Accepts website addresses, bare domains or a direct sitemap address, and skips sites with no sitemap and keeps going.
- Has `includeUrlPatterns` and `excludeUrlPatterns` (regular expressions) to keep only, say, `/blog/` URLs.
- Costs $0.20 per 1,000 URLs, paid only for URLs you receive. Sites with no sitemap cost nothing, and `maxUrlsPerSite` caps the cost per site (default 5,000). Example from its README: 20 sites at the default cap is up to $20.

It only reads sitemaps and does not crawl links, so a page missing from every sitemap is not found. Some sites with bot protection refuse datacenter IPs; the Actor has an optional proxy setting for those. It has no built-in `lastmod` filter, so filter the export in Excel or in Python as below.

```python
# pip install apify-client
import os
from datetime import date, timedelta

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/sitemap-extractor").call(run_input={
    "websites": ["stripe.com", "wordpress.org"],
    "maxUrlsPerSite": 1000,
    "includeUrlPatterns": ["/blog/"],
})
if run is None:
    raise SystemExit("The Actor run did not start.")

cutoff = (date.today() - timedelta(days=365)).isoformat()
for item in client.dataset(run.default_dataset_id).iterate_items():
    lastmod = item.get("lastmod")
    if lastmod and lastmod[:10] < cutoff:  # not updated in a year
        print(item["domain"], item["url"], lastmod)
```

To skip code, run it in the Apify Console and use Export in the dataset view to download CSV or Excel.

## FAQ

**How do I export sitemap URLs to Excel?**
Either download a CSV with a script like Method 1 and open it in Excel, or import the sitemap through Data, Get Data, From Web and choose the `url` table in Power Query. Both give you one row per URL, with its `lastmod` date when the site publishes one.

**Why does my converter return sitemap addresses instead of page URLs?**
You gave it a sitemap index. An index lists child sitemaps, and the page URLs are inside those children. Use a tool that follows indexes, such as the Python script in Method 1 of this guide, or run each child address through your converter.

**Can I open an .xml.gz sitemap in Excel?**
Not directly. Either unzip it first and import the XML, or use the `Binary.Decompress(Source, Compression.GZip)` step in a Power Query (Method 3). The Python script in Method 1 detects gzip by its first two bytes, so it works whatever the file is called.

**How do I get the sitemap of a site if I do not know its address?**
Open `https://example.com/robots.txt` and look for `Sitemap:` lines, then try `/sitemap.xml`. Our guide to [finding the sitemap of a website](find-sitemap-of-website) lists the CMS default paths.

**Does the export include pages that are not in the sitemap?**
No. A sitemap lists only what the site owner or its plugin chose to include. For a full inventory, compare the export against a crawl, or against Google Search Console's indexed pages if you own the site.

**Can I filter the export to pages updated in the last year or two?**
Yes, if the site publishes `lastmod`. Filter the date column in Excel or use the filter in Method 2. Pages without a `lastmod` have no date to filter on, so list them separately.

## Related guides

- [How to find the sitemap of a website](find-sitemap-of-website): robots.txt, default paths and CMS locations when you do not know the sitemap address.
- [Get all URLs from a sitemap in Python](get-all-urls-from-sitemap-python): a code-focused walk-through of reading sitemap files.
- [Convert a website to Markdown for an LLM](convert-website-to-markdown-for-llm): turn the exported page list into text for an AI pipeline.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I export sitemap URLs to Excel?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Either download a CSV with a script like Method 1 and open it in Excel, or import the sitemap through Data, Get Data, From Web and choose the url table in Power Query. Both give you one row per URL, with its lastmod date when the site publishes one."
      }
    },
    {
      "@type": "Question",
      "name": "Why does my converter return sitemap addresses instead of page URLs?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "You gave it a sitemap index. An index lists child sitemaps, and the page URLs are inside those children. Use a tool that follows indexes, such as the Python script in Method 1 of this guide, or run each child address through your converter."
      }
    },
    {
      "@type": "Question",
      "name": "Can I open an .xml.gz sitemap in Excel?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not directly. Either unzip it first and import the XML, or use the Binary.Decompress(Source, Compression.GZip) step in a Power Query (Method 3). The Python script in Method 1 detects gzip by its first two bytes, so it works whatever the file is called."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get the sitemap of a site if I do not know its address?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Open https://example.com/robots.txt and look for Sitemap: lines, then try /sitemap.xml. Our guide to finding the sitemap of a website lists the CMS default paths."
      }
    },
    {
      "@type": "Question",
      "name": "Does the export include pages that are not in the sitemap?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. A sitemap lists only what the site owner or its plugin chose to include. For a full inventory, compare the export against a crawl, or against Google Search Console's indexed pages if you own the site."
      }
    },
    {
      "@type": "Question",
      "name": "Can I filter the export to pages updated in the last year or two?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, if the site publishes lastmod. Filter the date column in Excel or use the filter in Method 2. Pages without a lastmod have no date to filter on, so list them separately."
      }
    }
  ]
}
</script>
