---
title: How to Extract Emails From a Website for Free (5 Ways)
description: Extract emails from a website for free with view source, a browser console snippet, an extension, Hunter or Snov free credits, or a short Python script.
---

# How to extract emails from a website for free

To extract emails from a website for free, open the site's contact, about and legal-notice pages, press Ctrl+U (Cmd+Option+U on a Mac) to view the source, and search for `@` and `mailto:`. That finds every address written in the page's HTML. For more than a few pages, paste a one-line snippet into the browser console, use a free Chrome extension, spend the free monthly credits of an email finder such as Hunter or Snov.io, or run the 20-line Python script below. All five cost nothing; which one fits depends on how many pages you need to read and whether you need a named person's address or just the company inbox.

Disclosure: Data Gleaner, mentioned near the end, is us. Every method before that works without our product.

## Which free method to use

| Method | What it finds | Free limit | Best for |
|---|---|---|---|
| View source and search | Addresses in the HTML of the page you open | None | One site, a handful of pages |
| Browser console snippet | Addresses in the page as rendered, including script-inserted text | None | The page you are on, copied as a clean list |
| Chrome extension | Addresses on the pages you visit | Varies; some free tiers hide results or block export | Collecting while you browse |
| Hunter or Snov.io free plan | Addresses from their database, plus pattern guesses, with verification | 50 credits a month each (October 2026) | A specific person's address at a company |
| Python script | Addresses on the home page and common contact paths | None | Repeating the job, or reading several pages at once |

A page scrape only returns what the site publishes, which is mostly role inboxes such as `info@`, `sales@` or `press@`. Finder services can return a person's address, but some of those are inferred from a naming pattern (`first.last@domain`) rather than seen on a page.

## 1. View the source and search for "@"

1. Open the page most likely to hold an email: Contact, About, Impressum or Legal Notice, Team, Press, or the footer of the home page.
2. Press Ctrl+U (Windows, Linux) or Cmd+Option+U (Mac) to open the page source.
3. Press Ctrl+F or Cmd+F and search for `mailto:`, then for `@`.
4. Copy each address you find. Ignore hits like `image@2x.png` or `@media` in CSS.

Also look for addresses written to dodge scrapers, such as `name [at] example [dot] com` or `name(at)example.com`. Search for `[at]` and `(at)` too. If you see `/cdn-cgi/l/email-protection` instead of an address, the site uses Cloudflare's email protection; the address shows normally on the rendered page, so read it there or use the console method.

## 2. Copy every email on a page with the browser console

View source shows the HTML the server sent. The console reads the page as it appears after scripts have run, so it also catches addresses a script inserted. Open the page, press F12 (Cmd+Option+J on a Mac in Chrome), open the Console tab and paste:

```js
copy([...new Set(
  (document.body.innerText + " " +
   [...document.querySelectorAll('a[href^="mailto:"]')].map(a => a.href.slice(7).split("?")[0]).join(" "))
  .match(/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g) || []
)].join("\n"))
```

The unique addresses are now on your clipboard, one per line. Chrome may ask you to type "allow pasting" the first time you paste into the console; that warning exists because pasted code can act on the page, so read any snippet before you run it. This one only reads the page and writes to your clipboard.

## 3. Use a free Chrome extension

Search the Chrome Web Store for "email extractor" and you will find several that collect addresses from each page you visit. Before you install one, check three things on its listing:

- **What the free tier really allows.** Some show only the last few addresses found, or charge for export.
- **Which permissions it asks for.** An extension that can "read and change all your data on all websites" sees everything you browse, not only the sites you scrape. Prefer one with a clear privacy policy, and remove it when you are done.
- **When it was last updated.** Old extensions break on modern sites.

An extension still only reads pages you open, so it saves copying, not clicking.

## 4. Use the free plans of Hunter and Snov.io

Email finders keep a database of addresses seen across the web and add guesses based on each company's naming pattern. Their free plans, as listed on their pricing pages in October 2026:

- **[Hunter](https://hunter.io/pricing):** 50 credits a month. Domain Search lists addresses Hunter knows for a domain, with the sources where it saw them; verifying an address costs half a credit. Credits reset monthly and do not roll over.
- **[Snov.io](https://snov.io/pricing):** a Trial plan with 50 credits a month shared between finding and verifying, renewing every 30 days. It excludes bulk search, data export and API access.

Use these when you need a person rather than an inbox, for example the marketing lead at one company. Treat any address marked as guessed or "accept-all" as unconfirmed until it is verified. Prices and limits change, so check the pricing pages before you plan around them.

## 5. Run a short Python script

If you want the same job done on several pages without clicking through them, this script fetches a site's home page and common contact paths and prints the addresses it finds, including `[at]`/`[dot]` forms.

```python
# pip install requests
import re
import sys

import requests

site = sys.argv[1] if len(sys.argv) > 1 else "example.com"
PATHS = ["", "/contact", "/contact-us", "/about", "/about-us", "/impressum", "/legal", "/team"]
EMAIL = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")
SKIP = (".png", ".jpg", ".jpeg", ".gif", ".svg", ".webp", ".css", ".js")


def deobfuscate(text):
    text = re.sub(r"\s*[\[\(]\s*at\s*[\]\)]\s*", "@", text, flags=re.I)
    return re.sub(r"\s*[\[\(]\s*dot\s*[\]\)]\s*", ".", text, flags=re.I)


found = {}
for path in PATHS:
    url = f"https://{site}{path}"
    try:
        r = requests.get(url, timeout=10, headers={"User-Agent": "email-check/0.1"})
    except requests.RequestException:
        continue
    if not r.ok:
        continue
    for email in EMAIL.findall(deobfuscate(r.text)):
        if not email.lower().endswith(SKIP):
            found.setdefault(email.lower(), url)

for email, page in sorted(found.items()):
    print(email, "found on", page)
```

Save it as `emails.py` and run `python emails.py example.com`. It is deliberately small, so know what it misses:

- **Contact pages at other paths**, such as `/kontakt` or `/company/contact`. Follow the home page's links whose text says contact, about or team if you need them.
- **Addresses inserted by JavaScript.** `requests` sees only the server's HTML. Use the console method above, or a headless browser such as Playwright.
- **Cloudflare-protected addresses and emails shown as images.**
- **Sites that refuse automated requests.** If a site answers 403 or 429, stop. Check `robots.txt`, keep requests slow, and read the page by hand instead.

## When a paid, per-result tool costs less than "free"

Free methods cost your time. For one site, view source wins. Once you have dozens or thousands of sites, or you need the same fields every time for a spreadsheet or a pipeline, a pay-per-result scraper can cost less than the hour you would spend.

**Website Contact Details Scraper** ([apify.com/datagleaner/website-contact-details-scraper](https://apify.com/datagleaner/website-contact-details-scraper)) takes website URLs or bare domains, reads up to 8 pages per site by default, 30 at most (the home page plus its contact, about, legal-notice and team pages), and returns emails, phone numbers in E.164, social profiles, contact forms and schema.org addresses, each with the page it came from. It decodes `[at]`/`[dot]` forms and Cloudflare-protected addresses and respects `robots.txt` by default. It costs US$0.004 per website that returns at least one contact (US$4.00 per 1,000); sites with nothing found are not charged. Apify's free plan includes a monthly platform credit, which covers small test runs.

Its limits are the same as the script's on two points: it does not render JavaScript, and a site that refuses automated requests is reported as `unreachable` rather than worked around.

```python
# pip install apify-client
# Run: APIFY_TOKEN=... python website-contact-details-scraper.py
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/website-contact-details-scraper").call(
    run_input={
        "websites": ["https://www.appier.com", "stripe.com", "https://www.sakura.ad.jp"],
        "maxPagesPerSite": 5,
    }
)
if run is None:
    raise SystemExit("The run did not return.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item["website"], "->", item["status"])
    print("  emails:", [e["value"] for e in item.get("emails") or []])
```

Three sites cap this run at three billable results, at most about US$0.012. For whole lists of domains, see [contact and lead scrapers](../contact-and-lead-scrapers).

## Caveats

- **Expect role inboxes.** Most published addresses are `info@` or `sales@`. Some sites publish no email at all, only a contact form.
- **Verify before you send.** Addresses on old pages go stale, and pattern guesses can be wrong. Bounces hurt your sender reputation.
- **The law applies to what you do with the addresses.** An email can be personal data. GDPR, CAN-SPAM and local anti-spam laws govern storing and emailing people, whichever method found the address. Some sites' terms also forbid automated collection.

## FAQ

**How do I extract all email addresses from a website?**
No single page lists them all. Check the home page footer, Contact, About, Team, Press and Legal Notice or Impressum pages, using view source or the console snippet on each. To cover a whole site, start from its sitemap ([how to find a website's sitemap](find-sitemap-of-website)) and run a script over those URLs, keeping the request rate low.

**Is there a free online tool to extract emails from a website?**
Yes. Several web tools take a URL and return the addresses they find without sign-up, and Hunter and Snov.io give 50 free credits a month. Free online tools usually cap pages or exports, so for one site the view-source method is just as fast.

**Can I extract emails from a website with a Chrome extension?**
Yes. Email extractor extensions collect addresses from pages as you browse. Check the free tier's limits and the permissions it requests before you install one.

**How do I get an email from a website link without opening every page?**
Something still has to fetch the pages, but only a few matter: home, contact, about, legal notice and team. The Python script above fetches those for you from one domain.

**Why does view source show "email protected" instead of an address?**
The site uses Cloudflare's email protection, which encodes the address and decodes it in your browser. Read it on the rendered page or with the console snippet.

## Related

- [How to find the sitemap of a website](find-sitemap-of-website)
- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp)
