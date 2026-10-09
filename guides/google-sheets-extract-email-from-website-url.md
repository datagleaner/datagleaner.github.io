---
title: Google Sheets - Extract Email From a Website URL
description: "Google Sheets extract email from website URL: an IMPORTXML + REGEXEXTRACT formula for a column of URLs, where it fails, and a Python or Apify route back to Sheets."
---

# Google Sheets: extract email from a website URL

To extract an email address from a website URL in Google Sheets, use `IMPORTXML` to fetch the page's `mailto:` links and `REGEXEXTRACT` to clean the address out of the result. With the URLs in column A, this formula in B2 works for many small business sites:

```
=IFERROR(REGEXEXTRACT(JOIN(" ", IMPORTXML(A2, "//a[starts-with(@href,'mailto:')]/@href")), "mailto:([^?\s]+)"), "not found")
```

Drag it down the column and each row shows the first email the page links to. It is free and needs no script, but it only reads the one page you give it, it does not run JavaScript, and it misses obfuscated emails. The sections below extend the formula to plain-text emails and contact pages, then cover what to do when it runs out: a short Python script, or a bulk run that lands back in your sheet.

Disclosure: Data Gleaner, mentioned near the end as one option for bulk runs, is us. Every other method on this page is free and needs no account beyond Google.

## Why most Google Sheets email guides do not answer this

Many guides for this question show how to pull an address out of text that is already in a cell, using `REGEXEXTRACT` on a pasted paragraph. That is a different job. Here the cell holds only a URL, so the sheet has to fetch the page first. In Google Sheets the function that fetches a web page is `IMPORTXML` (or `IMPORTHTML` and `IMPORTDATA` for tables and CSV files), and everything below builds on it.

## 1. The IMPORTXML + REGEXEXTRACT formula

`IMPORTXML(url, xpath_query)` downloads the page at `url` and returns whatever the XPath query selects. For emails, the most reliable target is the `mailto:` link, because it is the form most sites use for a clickable address:

```
=IMPORTXML(A2, "//a[starts-with(@href,'mailto:')]/@href")
```

If the page has three mailto links, this spills three rows downward, which collides with the next URL row. To keep one result per row, join the matches into a single cell and pull out one address:

```
=IFERROR(REGEXEXTRACT(JOIN(" ", IMPORTXML(A2, "//a[starts-with(@href,'mailto:')]/@href")), "mailto:([^?\s]+)"), "not found")
```

How it works:

- `IMPORTXML(...)` returns the `href` values such as `mailto:hello@example.com?subject=Hi`.
- `JOIN(" ", ...)` turns the possibly many rows into one text string, so nothing spills.
- `REGEXEXTRACT(..., "mailto:([^?\s]+)")` keeps what follows `mailto:` up to a `?` or a space, which drops any `?subject=` part. The parentheses make Sheets return only the captured address.
- `IFERROR(..., "not found")` prints a label when the page has no mailto link, instead of an error.

A site with several addresses returns only the first one. To see all of them, drop `REGEXEXTRACT` and keep the `JOIN`, so the cell shows every `mailto:` link separated by spaces.

### Emails written as plain text

Some pages show the address as text without a mailto link. Fetch the page text and run a general email pattern over it:

```
=IFERROR(REGEXEXTRACT(TEXTJOIN(" ", TRUE, IMPORTXML(A2, "//body")), "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"), "not found")
```

`TEXTJOIN` is used here because the body text can come back spread over several cells in more than one direction, which `JOIN` does not accept. This returns the first email-shaped string in the page text. It can pick up an address from a footer or a cookie notice that is not the one you want, so check the output before using it.

### Try the contact page too

Companies rarely put their email on the home page. If the home page has no address, the contact page often does. When column A holds a site root with no trailing slash, such as `https://example.com`, build the contact URL in the formula:

```
=IFERROR(REGEXEXTRACT(JOIN(" ", IMPORTXML(A2 & "/contact", "//a[starts-with(@href,'mailto:')]/@href")), "mailto:([^?\s]+)"), "not found")
```

To try the home page first and the contact page second, nest the two with `IFERROR`:

```
=IFERROR(
  REGEXEXTRACT(JOIN(" ", IMPORTXML(A2, "//a[starts-with(@href,'mailto:')]/@href")), "mailto:([^?\s]+)"),
  IFERROR(
    REGEXEXTRACT(JOIN(" ", IMPORTXML(A2 & "/contact", "//a[starts-with(@href,'mailto:')]/@href")), "mailto:([^?\s]+)"),
    "not found"))
```

The path is a guess: `/contact`, `/contact-us`, `/about` and `/impressum` are all common, and a fixed path will miss any site that uses another one.

## 2. Where the formula fails

Check these before you run it on a long column:

- **JavaScript-rendered sites.** `IMPORTXML` fetches the HTML the server sends and does not run scripts. If the site builds its page in the browser (many single-page apps do), the email is not in that HTML and the formula returns "not found" even though you can see the address on screen. View the page source in your browser: if the address is not in it, `IMPORTXML` cannot see it either.
- **Contact pages that are not at a guessable path.** The address may live on `/pages/get-in-touch`, a team page, or a legal-notice page. Fixed paths in a formula only cover the usual names.
- **Obfuscated emails.** Sites write `name [at] example [dot] com`, use Cloudflare's email protection (the address is encoded in the HTML and decoded by a script), or show the address as an image. The mailto and plain-text patterns above match none of these.
- **Contact forms only.** Many sites publish no address at all, just a form. There is nothing to extract.
- **Blocked or slow requests.** Some sites refuse requests that come from Google's servers. The formula then shows an error, and `IFERROR` hides it behind "not found", so a blocked site looks the same as one with no email.
- **Limits and slowness.** Every `IMPORTXML` call is a live fetch, and each cell recalculates on its own. Sheets shows "Loading..." while it waits, and a long column of them is slow and can start returning errors. Google does not publish a firm quota for the import functions, so test with a small batch first.
- **Formulas re-run.** Because the fetch is a live formula, the results can change or disappear later if the site is slow or down when the sheet reloads. Copy the finished column and use Paste special, Values only, to freeze it.

A practical rule: the formula is good for a quick list of up to a few dozen well-built sites. For anything larger, move the fetching out of Sheets.

## 3. Free Python script that fills a CSV

A script can do what the formula does, with more control: several pages per site, mailto links and plain text together, and a pause between requests. Export your sheet's URL column as `urls.csv` (File, Download, CSV), run this, then import `emails.csv` back (File, Import).

```python
# pip install requests
import csv
import re
import time
from urllib.parse import urljoin

import requests

PATHS = ["", "/contact", "/contact-us", "/about", "/impressum"]
MAILTO = re.compile(r"mailto:([^\"'?\s>]+)", re.I)
PLAIN = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")
JUNK_SUFFIXES = (".png", ".jpg", ".jpeg", ".gif", ".svg", ".webp")

session = requests.Session()
session.headers["User-Agent"] = "email-check/1.0 (contact: you@example.com)"


def emails_on(url):
    try:
        r = session.get(url, timeout=20)
    except requests.RequestException:
        return []
    if r.status_code != 200:
        return []
    found = MAILTO.findall(r.text) + PLAIN.findall(r.text)
    # drop image file names that look like emails, such as logo@2x.png
    return [e.lower() for e in found if not e.lower().endswith(JUNK_SUFFIXES)]


def find_email(site):
    if not site.startswith("http"):
        site = "https://" + site
    for path in PATHS:
        hits = emails_on(urljoin(site, path))
        time.sleep(1)  # pause between requests
        if hits:
            return hits[0], urljoin(site, path)
    return "", ""


with open("urls.csv", newline="", encoding="utf-8") as f:
    sites = [row[0].strip() for row in csv.reader(f) if row and row[0].strip()]

with open("emails.csv", "w", newline="", encoding="utf-8") as f:
    out = csv.writer(f)
    out.writerow(["url", "email", "found_on"])
    for site in sites:
        email, page = find_email(site)
        out.writerow([site, email, page])
        print(site, "->", email or "none")
```

Limits of this script: it does not run JavaScript either, it does not decode Cloudflare-protected or `[at]`-style addresses, it fetches one page at a time with a one-second pause, and it does not check `robots.txt`. It also takes the first match, which can be a generic or unrelated address. Respect each site's terms.

## 4. A bulk run that lands back in Sheets

When the formula and the script are not enough, usually because of volume, obfuscated emails or the need for phone numbers and social links too, a hosted scraper is the next step. The [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper) on the Apify Store does the crawl-and-extract part for a list of websites.

What it does, from its documentation:

- For each website it fetches the home page, then the pages most likely to hold contact details: contact, about, impressum or legal notice, team, company profile and footer links, with equivalents in Japanese, Chinese, German, French, Spanish and Italian.
- Finds emails in `mailto:` links, plain text, obfuscated forms such as `name [at] domain [dot] com` and `name(at)domain.com`, Cloudflare email protection and JSON-LD.
- Also returns phone numbers normalized to E.164, social profiles (LinkedIn, X, Facebook, Instagram, YouTube, TikTok, GitHub), contact forms and schema.org addresses.
- Returns one row per website, with the page each value was found on. Emails on other domains are kept apart in `otherEmails`, so `emails` holds the site's own addresses.
- Costs US$4.00 per 1,000 websites with contacts (US$0.004 each). You are charged only for sites that return at least one contact; unreachable sites and sites with no contacts are free.

Its limits: it uses plain HTTP with no browser, so details injected only by scripts are not found, and sites that refuse or rate-limit the request (HTTP 403 or 429) are reported as `unreachable`. Emails shown as images are not read. It crawls up to 30 pages per site (8 by default) and stays on the same domain.

Run it from Python with the exact input field names from its input schema:

```python
# pip install apify-client
import csv
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/website-contact-details-scraper").call(run_input={
    "websites": ["appier.com", "stripe.com", "https://www.sakura.ad.jp"],
    "maxPagesPerSite": 8,
    "respectRobotsTxt": True,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

with open("emails.csv", "w", newline="", encoding="utf-8") as f:
    out = csv.writer(f)
    out.writerow(["website", "status", "email", "found_on"])
    for item in client.dataset(run.default_dataset_id).iterate_items():
        emails = item.get("emails") or []
        first = emails[0] if emails else {}
        out.writerow([item["website"], item["status"], first.get("value", ""), first.get("foundOn", "")])
```

Paste the contents of your URL column into `websites` (full URLs or bare domains both work).

### Getting the results into your sheet

There are two routes:

1. **CSV.** In the Apify Console, open the run's Output tab and export the dataset as CSV. In Google Sheets choose File, Import, Upload, and pick "Insert new sheet". Then match rows to your original list with `VLOOKUP` or `XLOOKUP` on the website column. The `website` value is the input as you typed it, so keep your sheet's URL column in the same form you gave the Actor.
2. **An Apify integration.** In the Integrations tab of the Actor or a saved task, you can attach another Actor that runs when this one finishes, such as a Google Sheets export Actor from the Apify Store, so the dataset is written to a spreadsheet automatically. Follow that Actor's own setup steps; its options are not part of this scraper.

If you only need to try it, a run of a few sites costs a fraction of a cent, which Apify's free plan credit covers.

## FAQ

**How do I extract an email address from a URL in Google Sheets?**
Use `IMPORTXML` to fetch the page's `mailto:` links and `REGEXEXTRACT` to keep the address: with the URL in A2, `=IFERROR(REGEXEXTRACT(JOIN(" ", IMPORTXML(A2, "//a[starts-with(@href,'mailto:')]/@href")), "mailto:([^?\s]+)"), "not found")`. It only reads the single page in the cell, so for contact pages on other paths or for many sites, use a script or a bulk tool.

**Why does my IMPORTXML formula return an empty result or "not found"?**
The usual causes are that the address is added by JavaScript after the page loads, that it is written in an obfuscated form such as `name [at] example [dot] com`, that it is on a different page than the one you gave, or that the site blocked the request. View the page source in your browser: if you cannot find the address there, neither can `IMPORTXML`.

**Can Google Sheets extract emails from a list of websites automatically?**
Partly. A formula dragged down a column does it for a modest list, but each cell is a live fetch and large columns get slow or fail. For hundreds or thousands of sites, run a script or a bulk Actor outside Sheets and import the CSV.

**Does REGEXEXTRACT work on a URL by itself?**
No. `REGEXEXTRACT` only reads text that is already in the cell. A URL is just the address of a page, so you need `IMPORTXML` (or a script) to download the page first, then `REGEXEXTRACT` to pick the email out.

**Is it legal to collect emails from websites?**
Reading publicly published pages is technically simple, but what you do with the addresses is governed by the law that applies to you, including data protection rules such as GDPR and CCPA and anti-spam law. Business inboxes such as `info@` carry different risks from named personal addresses. Check the rules for your country and your recipients before you store or contact anyone.

## Related guides

- [How to extract emails from a website for free](extract-emails-from-website-free): five free methods, from view source to a browser console snippet.
- [Find emails from a list of websites](find-email-addresses-from-list-of-websites): the bulk version of this task, with no spreadsheet formulas.
- [Get all URLs from a sitemap in Python](get-all-urls-from-sitemap-python): build the list of pages to check before you extract.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I extract an email address from a URL in Google Sheets?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Use IMPORTXML to fetch the page's mailto: links and REGEXEXTRACT to keep the address: with the URL in A2, =IFERROR(REGEXEXTRACT(JOIN(\" \", IMPORTXML(A2, \"//a[starts-with(@href,'mailto:')]/@href\")), \"mailto:([^?\\s]+)\"), \"not found\"). It only reads the single page in the cell, so for contact pages on other paths or for many sites, use a script or a bulk tool."
      }
    },
    {
      "@type": "Question",
      "name": "Why does my IMPORTXML formula return an empty result or \"not found\"?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The usual causes are that the address is added by JavaScript after the page loads, that it is written in an obfuscated form such as name [at] example [dot] com, that it is on a different page than the one you gave, or that the site blocked the request. View the page source in your browser: if you cannot find the address there, neither can IMPORTXML."
      }
    },
    {
      "@type": "Question",
      "name": "Can Google Sheets extract emails from a list of websites automatically?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Partly. A formula dragged down a column does it for a modest list, but each cell is a live fetch and large columns get slow or fail. For hundreds or thousands of sites, run a script or a bulk Actor outside Sheets and import the CSV."
      }
    },
    {
      "@type": "Question",
      "name": "Does REGEXEXTRACT work on a URL by itself?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. REGEXEXTRACT only reads text that is already in the cell. A URL is just the address of a page, so you need IMPORTXML (or a script) to download the page first, then REGEXEXTRACT to pick the email out."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to collect emails from websites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Reading publicly published pages is technically simple, but what you do with the addresses is governed by the law that applies to you, including data protection rules such as GDPR and CCPA and anti-spam law. Business inboxes such as info@ carry different risks from named personal addresses. Check the rules for your country and your recipients before you store or contact anyone."
      }
    }
  ]
}
</script>
