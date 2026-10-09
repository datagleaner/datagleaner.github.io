---
title: Get All URLs From a Sitemap in Python (Index and .gz)
description: "Get all URLs from a sitemap in Python with requests and xml.etree: find sitemaps in robots.txt, follow sitemap indexes, unpack .xml.gz files. Full code."
---

# How to get all URLs from a sitemap in Python

To get all URLs from a sitemap in Python, download the sitemap with `requests`, parse it with `xml.etree.ElementTree`, and collect the text of every `<loc>` inside a `<url>` element. Real sites need three more steps: find the sitemap through the `Sitemap:` lines in `robots.txt`, follow sitemap index files down to the child sitemaps, and unpack gzip (`.xml.gz`) files. The script below does all of that with only `requests` and the standard library.

## What a sitemap looks like

A sitemap is either a list of pages (`<urlset>`) or a list of other sitemaps (`<sitemapindex>`). Both use the namespace `http://www.sitemaps.org/schemas/sitemap/0.9`, which is why a plain `root.findall("url")` returns nothing: the tags are really `{http://www.sitemaps.org/schemas/sitemap/0.9}url`.

```xml
<!-- A page sitemap -->
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url><loc>https://example.com/</loc><lastmod>2026-09-30</lastmod></url>
  <url><loc>https://example.com/about</loc></url>
</urlset>

<!-- A sitemap index: points to more sitemaps, not to pages -->
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap><loc>https://example.com/sitemap-posts.xml.gz</loc></sitemap>
  <sitemap><loc>https://example.com/sitemap-pages.xml</loc></sitemap>
</sitemapindex>
```

Large sites almost always use an index, because one sitemap file is limited to 50,000 URLs and 50 MB uncompressed.

## The quick version: one sitemap file

If you already have the sitemap URL and it lists pages directly, this is enough:

```python
import requests
import xml.etree.ElementTree as ET

NS = {"sm": "http://www.sitemaps.org/schemas/sitemap/0.9"}

resp = requests.get("https://example.com/sitemap.xml", timeout=30)
resp.raise_for_status()
root = ET.fromstring(resp.content)
urls = [loc.text.strip() for loc in root.findall("sm:url/sm:loc", NS)]
print(len(urls), urls[:5])
```

Pass `resp.content` (bytes), not `resp.text`, so the parser reads the encoding from the XML declaration itself. If this returns an empty list, the file is probably a sitemap index; use the full script.

## Full script: robots.txt, sitemap indexes and .xml.gz

Install the one dependency with `pip install requests`, save this as `sitemap_urls.py` and run `python sitemap_urls.py example.com > urls.txt`.

```python
import gzip
import sys
import xml.etree.ElementTree as ET
from urllib.parse import urljoin

import requests

NS = "{http://www.sitemaps.org/schemas/sitemap/0.9}"
HEADERS = {"User-Agent": "sitemap-urls/1.0 (you@example.com)"}


def fetch(url):
    """Download a sitemap and return its bytes, unpacking gzip if needed."""
    resp = requests.get(url, headers=HEADERS, timeout=30)
    resp.raise_for_status()
    data = resp.content
    if data[:2] == b"\x1f\x8b":  # gzip magic bytes, whatever the file is named
        data = gzip.decompress(data)
    return data


def find_sitemaps(site):
    """Read the Sitemap: lines in robots.txt, falling back to /sitemap.xml."""
    root = urljoin(site if "://" in site else "https://" + site, "/")
    found = []
    try:
        resp = requests.get(urljoin(root, "robots.txt"), headers=HEADERS, timeout=30)
        if resp.ok:
            for line in resp.text.splitlines():
                if line.lower().startswith("sitemap:"):
                    found.append(line.split(":", 1)[1].strip())
    except requests.RequestException:
        pass
    return found or [urljoin(root, "sitemap.xml")]


def sitemap_urls(sitemap_url, seen=None, depth=0):
    """Yield every page URL in a sitemap, following sitemap indexes."""
    seen = set() if seen is None else seen
    if sitemap_url in seen or depth > 5:
        return
    seen.add(sitemap_url)
    try:
        data = fetch(sitemap_url)
    except requests.RequestException as err:
        print(f"[WARN] skipped {sitemap_url}: {err}", file=sys.stderr)
        return
    try:
        root = ET.fromstring(data)
    except ET.ParseError:
        # Plain-text sitemap: one URL per line
        for line in data.decode("utf-8", "replace").splitlines():
            if line.strip().startswith("http"):
                yield line.strip()
        return
    if root.tag == NS + "sitemapindex":
        for loc in root.iter(NS + "loc"):
            yield from sitemap_urls(loc.text.strip(), seen, depth + 1)
    else:
        for url in root.iter(NS + "url"):
            loc = url.find(NS + "loc")
            if loc is not None and loc.text:
                yield loc.text.strip()


if __name__ == "__main__":
    site = sys.argv[1] if len(sys.argv) > 1 else "https://stripe.com"
    urls = set()
    for sitemap in find_sitemaps(site):
        urls.update(sitemap_urls(sitemap))
    for url in sorted(urls):
        print(url)
    print(f"{len(urls)} URLs", file=sys.stderr)
```

How it works:

- **Finding the sitemap.** `find_sitemaps` reads `robots.txt`, where most sites list their sitemaps on `Sitemap:` lines. If there are none, it tries `/sitemap.xml`. WordPress sites without an SEO plugin use `/wp-sitemap.xml`; add it to the fallback list if you work with many WordPress sites.
- **Sitemap indexes.** When the root element is `<sitemapindex>`, the function calls itself on each child sitemap. The `seen` set stops loops, and `depth` stops runaway nesting.
- **Gzip.** The check looks at the first two bytes (`1f 8b`) rather than the file name. `requests` already unpacks responses sent with `Content-Encoding: gzip`, so a `.xml.gz` URL can arrive either compressed or not; checking the bytes handles both.
- **Plain-text sitemaps.** The sitemap protocol also allows a `.txt` file with one URL per line. If the XML parser fails, the script reads it that way.
- **Duplicates.** The same page can be listed in several sitemaps, so the URLs go into a `set`.
- **Errors.** One broken or missing child sitemap is logged and skipped instead of stopping the whole run.

### Keep lastmod and other fields

To get `lastmod` (useful for finding new or changed pages), read it next to `loc`. This helper does that for one parsed `<urlset>`; call it in place of the loop in the `else` branch with `yield from url_entries(root)`:

```python
def url_entries(root):
    """Yield (url, lastmod) pairs from a parsed <urlset> element."""
    for url in root.iter(NS + "url"):
        loc = url.findtext(NS + "loc")
        lastmod = url.findtext(NS + "lastmod")  # None if the site omits it
        if loc:
            yield loc.strip(), lastmod
```

The script then yields `(url, lastmod)` pairs instead of plain strings, so keep them in a `dict` keyed by URL rather than a `set` if you still want to drop duplicates.

Many sites publish only `<loc>`, so expect `lastmod`, `changefreq` and `priority` to be missing often.

### Save to CSV

```python
import csv

with open("urls.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["url"])
    for sitemap in find_sitemaps("example.com"):
        for url in sitemap_urls(sitemap):
            writer.writerow([url])
```

## Other ways to do it

| Option | Best for | Cost | Notes |
|---|---|---|---|
| Script above (`requests` + `xml.etree`) | Full control, one or a few sites | Free | No extra libraries; you maintain it |
| `ultimate-sitemap-parser` (`pip install ultimate-sitemap-parser`) | Python users who want a library | Free | `from usp.tree import sitemap_tree_for_homepage`, then `tree.all_pages()`; handles indexes, gzip, robots.txt |
| `advertools` (`pip install advertools`) | SEO work in pandas | Free | `adv.sitemap_to_df("https://example.com/robots.txt")` returns a DataFrame with `loc`, `lastmod` and the source sitemap |
| `pandas.read_xml` | A single, simple sitemap file | Free | Default parser needs `lxml`; pass the sitemap namespace; does not follow sitemap indexes |
| Open the sitemap in a browser | A quick look at a small site | Free | Visit `/robots.txt` or `/sitemap.xml` and read the `<loc>` lines; impractical past a few hundred URLs |
| Online sitemap URL extractors | No code, one site | Usually free | Paste a sitemap URL, copy the list; many cap the number of URLs or do not follow indexes |
| [Sitemap URL Extractor](https://apify.com/datagleaner/sitemap-extractor) on Apify | Many sites, scheduled runs, CSV or API output | $0.20 per 1,000 URLs | Hosted; see below |

### Sitemap URL Extractor for many sites

Disclosure: Data Gleaner is us, and we built this Actor.

A script is the right tool for one site. When you need the URLs of dozens or hundreds of sites, on a schedule, or from a no-code tool, [Sitemap URL Extractor](https://apify.com/datagleaner/sitemap-extractor) runs the same steps on Apify's servers. Give it a list of domains and it reads `robots.txt`, tries the common sitemap paths (`/sitemap.xml`, `/sitemap_index.xml`, `/sitemap-index.xml`, `/wp-sitemap.xml`), follows indexes, unpacks `.xml.gz`, reads plain-text sitemaps and recovers what it can from broken XML. It returns one row per URL with `lastmod`, `changefreq`, `priority`, image count and the sitemap the URL came from, exportable as JSON, CSV or Excel. A site without a sitemap or a failed request is logged and skipped, and the run carries on.

It costs $0.20 per 1,000 URLs, and you pay only for URLs returned. `maxUrlsPerSite` (default 5,000) caps the cost per site; `includeUrlPatterns` and `excludeUrlPatterns` take regular expressions such as `/blog/` or `\.pdf$`.

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

Install the client with `pip install apify-client` and set `APIFY_TOKEN` to your Apify API token. This example returns at most 10 URLs, so it costs at most $0.002.

## Caveats

- **A sitemap is not the whole site.** It lists only the pages the site owner chose to include. Pages left out of every sitemap need a link crawler instead.
- **Some sites have no sitemap at all.** Then both the script and the hosted Actor return nothing for that site.
- **Very large sites.** News and e-commerce sites can list millions of URLs across thousands of child sitemaps. Filter by sitemap name (for example only `post-sitemap*.xml`) or cap the count before you start.
- **Be polite.** Add a short `time.sleep()` between requests when you fetch many child sitemaps from one host, and set a `User-Agent` that says who you are. Sitemaps are published for crawlers, but respect the site's terms and `robots.txt` rules when you fetch the pages afterwards.
- **Some hosts refuse requests from scripts or cloud servers** and answer 403. Check the site's terms, and ask the owner for an export if you need its data regularly.
- **Untrusted XML.** `xml.etree` is fine for normal sitemaps. If you parse XML from arbitrary sources at scale, the Python docs recommend `defusedxml` to guard against malicious files.

## FAQ

**How do I extract all URLs from a sitemap XML file?**
Parse it with `xml.etree.ElementTree` and collect every `<loc>` inside `<url>`, using the sitemap namespace `http://www.sitemaps.org/schemas/sitemap/0.9`. If the root is `<sitemapindex>`, fetch each listed sitemap and repeat, as the full script above does.

**How do I find a website's sitemap URL?**
Open `https://example.com/robots.txt` and look for `Sitemap:` lines. If there are none, try `/sitemap.xml`, `/sitemap_index.xml` and, on WordPress, `/wp-sitemap.xml`.

**Why does `findall("url")` return an empty list?**
Sitemap tags are in a namespace. Use `root.findall("sm:url/sm:loc", {"sm": "http://www.sitemaps.org/schemas/sitemap/0.9"})` or the `{namespace}tag` form shown above.

**How do I read a sitemap.xml.gz file in Python?**
Download it with `requests` and call `gzip.decompress()` on the bytes if they start with `1f 8b`, then parse as normal. Checking the bytes is safer than checking the extension, because some servers already decompress the file in transit.

**Can I get all URLs from a sitemap without coding?**
Yes. Open the sitemap in your browser for a small site, use an online sitemap URL extractor for one site, or run a hosted tool such as Sitemap URL Extractor for many sites and download the result as CSV.

## Related guides

- [Convert a website to Markdown for an LLM](convert-website-to-markdown-for-llm): fetch the pages once you have their URLs.
- [How to find the sitemap of a website](find-sitemap-of-website): when robots.txt and /sitemap.xml come up empty.
- [Find email addresses from a list of websites](find-email-addresses-from-list-of-websites)
- [Web content scrapers](../web-content-scrapers)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I extract all URLs from a sitemap XML file?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Parse it with xml.etree.ElementTree and collect every <loc> inside <url>, using the sitemap namespace http://www.sitemaps.org/schemas/sitemap/0.9. If the root is <sitemapindex>, fetch each listed sitemap and repeat, as the full script above does."
      }
    },
    {
      "@type": "Question",
      "name": "How do I find a website's sitemap URL?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Open https://example.com/robots.txt and look for Sitemap: lines. If there are none, try /sitemap.xml, /sitemap_index.xml and, on WordPress, /wp-sitemap.xml."
      }
    },
    {
      "@type": "Question",
      "name": "Why does findall(\"url\") return an empty list?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Sitemap tags are in a namespace. Use root.findall(\"sm:url/sm:loc\", {\"sm\": \"http://www.sitemaps.org/schemas/sitemap/0.9\"}) or the {namespace}tag form shown above."
      }
    },
    {
      "@type": "Question",
      "name": "How do I read a sitemap.xml.gz file in Python?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Download it with requests and call gzip.decompress() on the bytes if they start with 1f 8b, then parse as normal. Checking the bytes is safer than checking the extension, because some servers already decompress the file in transit."
      }
    },
    {
      "@type": "Question",
      "name": "Can I get all URLs from a sitemap without coding?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. Open the sitemap in your browser for a small site, use an online sitemap URL extractor for one site, or run a hosted tool such as Sitemap URL Extractor for many sites and download the result as CSV."
      }
    }
  ]
}
</script>
