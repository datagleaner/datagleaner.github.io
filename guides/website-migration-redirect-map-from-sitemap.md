---
title: "Website Migration Redirect Map From Old Sitemap URLs"
description: "Build a 301 redirect map for a site migration from sitemap URLs: snapshot old and new sitemaps with lastmod, diff in Google Sheets, match by slug, export CSV."
---

# How to build a website migration 301 redirect map from old sitemap URLs

To build a 301 redirect map from old sitemap URLs, save the complete list of URLs in the old site's sitemaps (with `lastmod`) before launch, save the same list from the new site after launch, put both in a spreadsheet, and match each old URL to a new one by its slug with `VLOOKUP`. Every old URL that finds no match is a page that has no redirect target yet. Export the matched pairs as a two-column CSV (old path, new URL) for your server or CDN, and decide the unmatched rows by hand. Below are a Python script for the snapshots, the Sheets formulas, a redirect test and the limits.

Disclosure: Data Gleaner, mentioned near the end as one option for the snapshots, is us. Every other method on this page is free and needs no account.

## What a sitemap snapshot gives you, and what it does not

Redirect mapping guides usually start from a crawl of the current site. One example is the [Urllo redirect mapping guide](https://www.urllo.com/resources/learn/redirect-mapping-guide-migrations), which recommends a comprehensive crawl of the current website to find indexable pages, important non-indexed pages and existing redirects. A crawl is the right main source. A sitemap snapshot is a second, cheaper source that adds three things:

- **The URLs the site itself asks search engines to index**, a good first draft of the pages that matter.
- **`lastmod` dates.** Sorted by date, they show which pages look actively maintained and which have not changed for years. That helps when you have thousands of rows and have to decide which to map by hand first.
- **A dated record** of what the old site listed, after it is gone.

It also has a hard limit: a page that is in no sitemap is not in the snapshot. Pages reachable only by links, old pages with inbound links that were dropped from the sitemap long ago, and pages that already redirect can all be missing. Use the sitemap list together with a crawl and your analytics or backlink data, not instead of them.

## Step 1: snapshot the old sitemap before launch

Large sites rarely have one sitemap file. The address in `robots.txt` is often a **sitemap index** that lists child sitemaps, and the children are often gzipped (`.xml.gz`). The [sitemaps.org protocol](https://www.sitemaps.org/protocol.html) allows a single sitemap file at most 50,000 URLs and 50 MB uncompressed, so a big site is split. A converter that reads one file returns a list of sitemap addresses instead of pages.

This script reads the `Sitemap:` lines in `robots.txt` (or tries `/sitemap.xml`, `/sitemap_index.xml`, `/sitemap-index.xml` and `/wp-sitemap.xml`), follows indexes to any depth, unpacks gzip by its first bytes rather than by file name, removes duplicate URLs and writes `url, lastmod, sitemap` to a CSV. It needs only `requests` and the standard library. I ran it against a local test site with a sitemap index, one gzipped child and one plain child that repeated a URL, and it returned the three distinct URLs with the right `lastmod` values and source files.

```python
# pip install requests
import csv
import gzip
import sys
import xml.etree.ElementTree as ET
from urllib.parse import urlsplit

import requests

FALLBACKS = ["/sitemap.xml", "/sitemap_index.xml", "/sitemap-index.xml", "/wp-sitemap.xml"]
session = requests.Session()
session.headers["User-Agent"] = "migration-audit/1.0 (contact: you@example.com)"


def fetch(url):
    try:
        r = session.get(url, timeout=30)
    except requests.RequestException:
        return None
    if r.status_code != 200:
        return None
    body = r.content
    if body[:2] == b"\x1f\x8b":  # gzip by content, not by file name
        try:
            body = gzip.decompress(body)
        except OSError:
            return None
    return body


def local(tag):  # strip the XML namespace
    return tag.rsplit("}", 1)[-1]


def child_text(node, name):
    for c in node:
        if local(c.tag) == name and c.text:
            return c.text.strip()
    return ""


def start_points(site):
    parts = urlsplit(site if "://" in site else "https://" + site)
    root = f"{parts.scheme}://{parts.netloc}"
    body = fetch(root + "/robots.txt")
    found = []
    if body:
        for line in body.decode("utf-8", "replace").splitlines():
            if line.lower().startswith("sitemap:"):
                found.append(line.split(":", 1)[1].strip())
    return found or [root + p for p in FALLBACKS]


def snapshot(site, out_path):
    queue, seen, rows = start_points(site), set(), {}
    while queue:
        sm = queue.pop(0)
        if sm in seen or len(seen) >= 3000:
            continue
        seen.add(sm)
        body = fetch(sm)
        if not body:
            print("skipped", sm, file=sys.stderr)
            continue
        try:
            root = ET.fromstring(body)
        except ET.ParseError:
            print("not valid XML", sm, file=sys.stderr)
            continue
        kind = local(root.tag)
        for node in root:
            loc = child_text(node, "loc")
            if not loc:
                continue
            if kind == "sitemapindex":
                queue.append(loc)
            elif loc not in rows:
                rows[loc] = (child_text(node, "lastmod"), sm)
    with open(out_path, "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["url", "lastmod", "sitemap"])
        for url, (lastmod, sm) in rows.items():
            w.writerow([url, lastmod, sm])
    print(f"{len(rows)} URLs from {len(seen)} sitemap files -> {out_path}")


if __name__ == "__main__":
    snapshot(sys.argv[1], sys.argv[2])
```

Run it as `python snap.py https://old.example.com old-2026-10-09.csv`. Keep the dated file. Read the "skipped" lines on stderr: each one is a sitemap file that could not be fetched, so the snapshot is missing its URLs.

Limits of this script:

- **Only what the sitemaps list.** It does not crawl links.
- **No retries and no pause.** A site that answers 403 or 429 to non-browser clients is skipped. Add a short `time.sleep` between requests for a site you do not own.
- **Password-protected sites** need credentials the script does not send.

## Step 2: snapshot the new site after launch

Run the same script on the new site once it is live and save it as `new-2026-10-20.csv` or similar. Check that its sitemap lists the final domain and not a staging address, because staging addresses would end up in your redirect targets.

## Step 3: diff the two snapshots in Google Sheets

Create one spreadsheet with two tabs named `Old` and `New`. Import the old CSV into `Old` starting at A1. Import the new CSV into `New` starting at B1, leaving column A empty, because `VLOOKUP` looks up the first column of its range and the slug has to be there.

The slug is the last part of the path: `https://old.example.com/blog/hello-world/` has the slug `hello-world`. Matching on the slug works when the migration keeps page names and changes only the folder structure or the domain. It does not work when slugs were rewritten.

In the `New` tab the URL is in B and `lastmod` in C, so in A2:

```
=REGEXEXTRACT(B2, "([^/]+)/?$")
```

In the `Old` tab, the URL is in A, `lastmod` in B and the sitemap in C. Use D for the slug and E to G for the match, the flag and the old path:

```
D2: =REGEXEXTRACT(A2, "([^/]+)/?$")
E2: =IFERROR(VLOOKUP(D2, New!A:B, 2, FALSE), "")
F2: =IF(E2="", "NO TARGET", IF(COUNTIF(New!A:A, D2) > 1, "CHECK: slug used more than once", "ok"))
G2: =REGEXREPLACE(A2, "^https?://[^/]+", "")
```

Fill each formula down to the last row. What each one does:

- `REGEXEXTRACT` returns the text matched by the group, here everything after the last slash, ignoring a trailing slash. For a bare home page URL the result is the host name, not a real slug, so handle the home page by hand.
- `VLOOKUP` with `FALSE` as the last argument requires an exact match, and column 2 of the range `New!A:B` is the new URL. `IFERROR` turns "not found" into an empty cell.
- Column F is the diff. Filter it on `NO TARGET` to see the old pages with no new equivalent. `CHECK` rows are slugs that appear several times on the new site, such as `overview` under different folders, where `VLOOKUP` returns only the first and may pick the wrong page.
- Column G removes the domain, giving the old path that a redirect rule matches.

Do the reverse check too: `=IF(COUNTIF(Old!D:D, A2)=0, "NEW PAGE", "")` in a free column of `New`. A long `NEW PAGE` list next to a long `NO TARGET` list often means slugs were renamed.

## Step 4: decide the unmatched rows

Auto-matching handles the easy part. For the rest, a simple order works:

1. Sort the `NO TARGET` rows by `lastmod`, newest first, and map the recently updated pages by hand.
2. If the old page has a real equivalent on the new site, use that URL.
3. If several old pages are merged into one, point them all at it.
4. Otherwise let the page return 404 or 410. The Urllo guide warns that search engines treat redirects of unrelated URLs to the home page as soft 404s, and its checklist asks whether a URL is better handled as a 404 when no relevant replacement exists.

## Step 5: export the redirect CSV

Make a third tab named `Redirects`. Copy only the rows with a target, with the old path (column G) in one column and the new URL (column E) in the other. Paste as values, then download that tab as a CSV. The layout your server or CDN wants differs, so check your host's redirect documentation and adjust the columns before you import.

Then test the result against the live site. This check requests each old path and prints any that does not answer with a 301 to the intended target:

```python
# pip install requests
import csv
import sys

import requests

base = sys.argv[1].rstrip("/")  # the host that serves the redirects
with open(sys.argv[2], newline="", encoding="utf-8") as f:
    for old_path, target in csv.reader(f):
        try:
            r = requests.head(base + old_path, allow_redirects=False, timeout=20)
        except requests.RequestException as e:
            print("ERROR", old_path, e)
            continue
        got = r.headers.get("Location", "")
        if r.status_code != 301 or got != target:
            print(r.status_code, old_path, "->", got or "(no Location)", "expected", target)
```

Run it as `python check.py https://old.example.com redirects.csv` after removing the header row from the CSV. It does not follow redirects, so check chains separately, and a relative `Location` header shows as a mismatch.

## One option for the snapshots: Data Gleaner Sitemap Extractor

Data Gleaner is us. If you would rather not run and maintain the script, the [Sitemap URL Extractor](https://apify.com/datagleaner/sitemap-extractor) on the Apify Store does Steps 1 and 2 as a hosted job. Run it once on the old domain before launch and once on the new domain after.

What it does, from its documentation:

- Finds sitemaps through `robots.txt`, then `/sitemap.xml`, `/sitemap_index.xml`, `/sitemap-index.xml` and `/wp-sitemap.xml`. Follows sitemap indexes, reads `.xml.gz` files by content and plain-text sitemaps, and recovers what it can from broken XML.
- Returns one row per URL with `url`, `lastmod`, `changefreq`, `priority`, `sitemapUrl`, `domain`, `imageCount`, `website` and `scrapedAt`. URLs are deduplicated per site. Fields the sitemap does not publish are `null`.
- Costs US$0.20 per 1,000 URLs, charged only for URLs saved to the dataset. Nothing is charged for sites without a sitemap or for failed requests. `maxUrlsPerSite` (default 5,000) is also your cost cap per site, and the run stops at your maximum total charge. A site with 1,200 URLs costs US$0.24.
- Skips a broken file or a down host, logs it and carries on, with a per-site summary in the `SUMMARY` record.
- Has a `checkStatus` option that sends one request per URL and adds `status`, `finalUrl` and `checkError`, so you can find sitemap URLs that already redirect or return errors. When the sitemap has no `lastmod`, it fills it from the page's `Last-Modified` header and marks the row `lastmodSource: http-header`. The input description says this is slower, about 5 pages a second.
- Has the same blind spot as the script: pages in no sitemap are not found, and sites behind bot protection may answer 403 to datacenter IPs. The `proxyConfiguration` and `requestDelaySeconds` fields are the documented workarounds.

The snippet below saves the old-site snapshot as a CSV that imports into the `Old` tab. Set `maxUrlsPerSite` above the site's size, or the export stops at the cap.

```python
# pip install apify-client
import csv
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/sitemap-extractor").call(run_input={
    "websites": ["https://old.example.com"],
    "maxUrlsPerSite": 100000,
    "excludeUrlPatterns": ["\\.pdf$"],
})
if run is None:
    raise SystemExit("The Actor run did not start.")

with open("old-2026-10-09.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.writer(f)
    w.writerow(["url", "lastmod", "sitemapUrl"])
    for item in client.dataset(run.default_dataset_id).iterate_items():
        w.writerow([item["url"], item["lastmod"] or "", item["sitemapUrl"]])
```

You can also export the dataset as CSV or Excel from the Apify Console without code. Run the same snippet with the new domain and `new-2026-10-20.csv` for the second snapshot. `excludeUrlPatterns` takes Python regular expressions, so check that a pattern does not drop pages you want to redirect.

## FAQ

**How do I make a 301 redirect map from an old sitemap?**
Export every URL from the old site's sitemaps, including the child files of a sitemap index and `.xml.gz` files, to a spreadsheet. Export the new site's sitemap the same way after launch. Extract the slug (the last path segment) from each URL, look up each old slug in the new list with `VLOOKUP`, and write the matches as old path and new URL. The unmatched rows need a manual decision. Test the finished list against the live server before you rely on it.

**Is a sitemap enough to find all the old URLs I need to redirect?**
No. A sitemap lists only the pages the site chose to publish there. Pages that were never in a sitemap, old pages that still have backlinks or traffic, and pages that already redirect can be missing. Combine the sitemap snapshot with a crawl of the old site and with your analytics and backlink exports, and map the highest-value pages first.

**Why include lastmod in the snapshot?**
`lastmod` is the date the site says a page last changed, so it is a rough way to rank the unmatched pages: recently updated pages are probably still read and deserve a hand-picked target, and untouched pages can often be handled in bulk. It is optional in the sitemap format, so many sites leave it out, and a site can fill it with the same date for every page, which makes it useless. Check that the dates vary before you rely on them.

**When does matching by slug fail?**
It fails when slugs were rewritten in the migration, when the same slug exists under several folders on the new site, and when the slug is a numeric ID. The `CHECK` flag in the formulas above catches the duplicate case, and a long `NO TARGET` list after matching usually means slugs changed. In those cases map by title or by a product or article ID, or write pattern rules for a whole section and test them.

**What should I do with old URLs that have no equivalent?**
Do not send them all to the home page. Redirect to the closest real equivalent if there is one, merge to a combined page only when the content is truly combined, and otherwise let the page return 404 or 410.

## Related guides

- [Export sitemap URLs to Excel or CSV](sitemap-to-csv-excel): the export step alone, with a lastmod filter and a Power Query method.
- [Get all URLs from a sitemap in Python](get-all-urls-from-sitemap-python): the sitemap reading code in detail.
- [How to find the sitemap of a website](find-sitemap-of-website): when the old site's sitemap address is not obvious.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I make a 301 redirect map from an old sitemap?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Export every URL from the old site's sitemaps, including the child files of a sitemap index and .xml.gz files, to a spreadsheet. Export the new site's sitemap the same way after launch. Extract the slug (the last path segment) from each URL, look up each old slug in the new list with VLOOKUP, and write the matches as old path and new URL. The unmatched rows need a manual decision. Test the finished list against the live server before you rely on it."
      }
    },
    {
      "@type": "Question",
      "name": "Is a sitemap enough to find all the old URLs I need to redirect?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. A sitemap lists only the pages the site chose to publish there. Pages that were never in a sitemap, old pages that still have backlinks or traffic, and pages that already redirect can be missing. Combine the sitemap snapshot with a crawl of the old site and with your analytics and backlink exports, and map the highest-value pages first."
      }
    },
    {
      "@type": "Question",
      "name": "Why include lastmod in the snapshot?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "lastmod is the date the site says a page last changed, so it is a rough way to rank the unmatched pages: recently updated pages are probably still read and deserve a hand-picked target, and untouched pages can often be handled in bulk. It is optional in the sitemap format, so many sites leave it out, and a site can fill it with the same date for every page, which makes it useless. Check that the dates vary before you rely on them."
      }
    },
    {
      "@type": "Question",
      "name": "When does matching by slug fail?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It fails when slugs were rewritten in the migration, when the same slug exists under several folders on the new site, and when the slug is a numeric ID. The CHECK flag in the formulas above catches the duplicate case, and a long NO TARGET list after matching usually means slugs changed. In those cases map by title or by a product or article ID, or write pattern rules for a whole section and test them."
      }
    },
    {
      "@type": "Question",
      "name": "What should I do with old URLs that have no equivalent?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Do not send them all to the home page. Redirect to the closest real equivalent if there is one, merge to a combined page only when the content is truly combined, and otherwise let the page return 404 or 410."
      }
    }
  ]
}
</script>
