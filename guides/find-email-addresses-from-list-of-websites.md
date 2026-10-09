---
title: How to Find Email Addresses from a List of Websites
description: Find email addresses from a list of websites by hand, with a free Python script, or with bulk tools like Hunter. Steps, pitfalls, prices and GDPR basics.
---

# How to find email addresses from a list of websites

To find email addresses from a list of websites, visit each site's home page and its contact, about, legal-notice and team pages, and collect every address shown in `mailto:` links, in the page text and in the site's structured data. Under about 20 sites, doing it by hand is fine. For a few hundred or more, use a script or a bulk tool. A short Python script works for free but misses hidden (obfuscated) addresses and contact pages it does not know to visit. Paid tools like Hunter go further: they return addresses from their own database and guessed name patterns, not only what the site shows.

This guide covers all three routes, what each one misses, and the legal basics you need before you email anyone on the list.

## Before you start: published versus guessed emails

Two kinds of tools answer this question, and they give different results:

- **Website scrapers** read the pages of each site and return only the addresses the site itself shows, such as `info@`, `sales@` or `press@`, along with the page each one came from. Every result can be checked by opening that page.
- **Email finders** (Hunter, Snov.io, Apollo and similar) look a domain up in their own database, which they build from many sources across the web, and can guess a named person's address from the company's email pattern (for example `first.last@`). They are better for reaching a specific person and worse at showing where an address came from.

If you want a company's general contact address, a scraper is usually enough. If you need a named decision-maker, you need an email finder or a lookup on LinkedIn.

## Option 1: by hand (free, under about 20 sites)

For each website:

1. Open the home page and scroll to the footer. Many companies put an email there.
2. Open the Contact, About, Team or Careers pages. In Europe, check the **Impressum** or **Legal notice** page. German and Austrian law requires one, and it usually lists an email. Japanese shops publish one on the 特定商取引法 page.
3. Search the page for `@` with Ctrl+F (Cmd+F on a Mac). If nothing shows, view the page source (Ctrl+U, or Cmd+Option+U on a Mac) and search for `mailto:` or `@`.
4. Check the privacy policy. It often names a data-protection contact.
5. Paste the address and the page URL into a spreadsheet so you can check it later.

If a site has only a contact form and no address, note that. Some companies never publish an email.

## Option 2: a free Python script

This script reads a list of sites from `sites.txt` (one per line), fetches each home page plus any link that looks like a contact, about or legal page, and writes the emails it finds to `emails.csv`. It needs `pip install requests beautifulsoup4`.

```python
import csv
import re
import time
from urllib.parse import urljoin, urlparse

import requests
from bs4 import BeautifulSoup

EMAIL_RE = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")
# "name [at] example [dot] com", "name(at)example.com"
OBFUSCATED_RE = re.compile(
    r"([A-Za-z0-9._%+-]+)\s*[\[\(\{]\s*at\s*[\]\)\}]\s*([A-Za-z0-9-]+(?:\s*(?:\.|[\[\(\{]\s*dot\s*[\]\)\}])\s*[A-Za-z0-9-]+)+)",
    re.I,
)
CONTACT_WORDS = ("contact", "about", "impressum", "legal", "team", "kontakt", "imprint")
SKIP_ENDINGS = (".png", ".jpg", ".jpeg", ".gif", ".svg", ".webp")
HEADERS = {"User-Agent": "Mozilla/5.0 (compatible; email-list-check)"}


def decode_cfemail(hex_string):
    """Decode Cloudflare's email protection (data-cfemail attribute)."""
    key = int(hex_string[:2], 16)
    return "".join(chr(int(hex_string[i:i + 2], 16) ^ key) for i in range(2, len(hex_string), 2))


def emails_in(html):
    soup = BeautifulSoup(html, "html.parser")
    found = set()
    for a in soup.select('a[href^="mailto:"]'):
        found.add(a["href"][7:].split("?")[0])
    for tag in soup.select("[data-cfemail]"):
        found.add(decode_cfemail(tag["data-cfemail"]))
    text = soup.get_text(" ")
    found.update(EMAIL_RE.findall(text))
    for user, domain in OBFUSCATED_RE.findall(text):
        domain = re.sub(r"\s*[\[\(\{]\s*dot\s*[\]\)\}]\s*|\s*\.\s*", ".", domain, flags=re.I)
        found.add(f"{user}@{domain}")
    return {e.strip().lower() for e in found if not e.lower().endswith(SKIP_ENDINGS)}, soup


def scan(site, max_pages=6):
    if not site.startswith("http"):
        site = "https://" + site
    host = urlparse(site).netloc
    queue, seen, results = [site], set(), {}
    while queue and len(seen) < max_pages:
        url = queue.pop(0)
        if url in seen:
            continue
        seen.add(url)
        try:
            resp = requests.get(url, headers=HEADERS, timeout=15)
            resp.raise_for_status()
        except requests.RequestException as exc:
            print(f"[WARN] {url}: {exc}")
            continue
        emails, soup = emails_in(resp.text)
        for e in emails:
            results.setdefault(e, url)
        if url == site:  # queue likely contact pages linked from the home page
            host = urlparse(resp.url).netloc  # follow redirects such as example.com -> www.example.com
            for a in soup.select("a[href]"):
                link = urljoin(url, a["href"]).split("#")[0]
                if urlparse(link).netloc == host and any(w in link.lower() for w in CONTACT_WORDS):
                    queue.append(link)
        time.sleep(1)  # be polite: one request per second per site
    return results


with open("sites.txt") as f, open("emails.csv", "w", newline="") as out:
    writer = csv.writer(out)
    writer.writerow(["website", "email", "found_on"])
    for line in f:
        site = line.strip()
        if site:
            for email, page in scan(site).items():
                writer.writerow([site, email, page])
```

### Where a regex script goes wrong

- **Obfuscated addresses.** Sites write `name [at] domain [dot] com`, `name(at)domain.com`, or use Cloudflare's email protection, which replaces the address with an encoded string that a script decodes in the browser. A plain `@` regex misses all of these; the script above handles the common ones, but there are many variants.
- **Emails in images or added by JavaScript.** `requests` sees only the HTML the server sends. If the address is drawn as an image or inserted by a script after the page loads, you need a headless browser (Playwright or Selenium) or you will not see it.
- **The contact page is not linked from the home page,** or its link text is in another language (Kontakt, お問い合わせ, 聯絡我們). Add the words for the countries on your list, or check the sitemap. Our guide to [getting all URLs from a sitemap](get-all-urls-from-sitemap-python) shows how.
- **False positives.** Image names such as `logo@2x.png`, tracking IDs like `abc123@sentry.io`, and placeholder addresses (`you@example.com`) match the regex. Filter them out before you use the list.
- **Redirects and `www`.** `example.com` may redirect to `www.example.com/en/`. Compare hosts after the redirect or you will skip every internal link.
- **Rate limits and refusals.** Some sites answer scripts with HTTP 403 or 429. Respect that, slow down, and check `robots.txt`; do not hammer a site to get past it.

## Option 3: bulk tools

| Tool | What it returns | Price (October 2026, check before buying) | Best for |
|---|---|---|---|
| Hunter (Domain Search, Bulk) | Emails from its index plus pattern-based guesses, with a confidence score and sources | Free: 50 credits a month. Starter: $49 a month ($34 billed yearly) for 2,000 credits a month | Finding named people at a company |
| Snov.io | Domain search and email finder, credits shared with verification and outreach | Starter: $39 a month (about $30 billed yearly) for 1,000 credits a month | Finding plus sending from one tool |
| Desktop extractors and Chrome extensions | Addresses published on the pages they visit | Free to one-off licence | Small lists on one computer |
| Data Gleaner Website Contact Details Scraper (Apify) | Published emails, phones, social profiles, contact forms and addresses, each with its source page | $4 per 1,000 websites with contacts; sites with nothing found are free | Large lists of company domains |

Email finders charge per lookup whether or not the result is the address you wanted, and their guessed addresses need verification before you send. Scrapers charge per site or per run and never return an address the site does not show.

## Using the Website Contact Details Scraper

Disclosure: Data Gleaner is us. The [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper) runs on the Apify Store. You give it a list of domains, and for each one it reads the home page and up to `maxPagesPerSite` likely contact pages (8 by default, 30 at most). It looks for contact, about, impressum, team and company-profile pages, including Japanese, Chinese, German, French, Spanish and Italian names for them. It handles the cases a simple script misses: `mailto:` links, `[at]`/`(at)` forms, Cloudflare email protection and JSON-LD structured data. It also returns phone numbers normalized to international (E.164) format, social profiles and contact-form pages. Every value comes with the page it was found on, and it respects `robots.txt` by default.

It returns one row per website. You pay $0.004 per website where at least one contact was found; unreachable sites and sites with no contacts are free. 1,000 sites that all return a contact cost $4.

To run it without code, open the Actor, paste your domains into **Websites**, click **Start**, then export the results as CSV or Excel. From Python (`pip install apify-client`):

```python
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
    print("  emails:", [(e["value"], e["foundOn"]) for e in item.get("emails") or []])
```

`status` tells you what happened on each site: `ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl` or `error`.

Its limits are the same as any plain-HTTP scraper's: it does not run JavaScript, so addresses inserted by scripts or shown only as images are not found, and sites that refuse non-browser requests come back as `unreachable`. It finds published addresses, not named people's personal emails.

## Legal basics: GDPR and CAN-SPAM

This is general information, not legal advice.

- **GDPR (EU and UK).** A work email that names a person (`jane.doe@company.com`) is personal data, even if it is published. You need a lawful basis to store and use it, usually "legitimate interest" for business-to-business outreach, and you must tell the person where you got their data and let them object. Generic addresses such as `info@` are less sensitive but still fall under national e-marketing rules. Some countries, such as Germany, require prior consent before you send marketing email even to businesses.
- **CAN-SPAM (US).** Cold commercial email is allowed, but each message must have an honest sender and subject line, your physical postal address, and a working way to opt out, and you must honour opt-outs within 10 business days.
- **Other rules.** Canada (CASL) and Australia (Spam Act) generally require consent first. Some sites also forbid collecting their contact details in their terms of use.

In practice: collect only what you need, keep the source page for each address (it answers "where did you get my email?"), verify addresses before sending, and keep an unsubscribe list.

## FAQ

**How do I extract emails from websites for free?**
Check the contact and legal pages by hand, or run the Python script above. Hunter's free plan gives 50 credits a month, and on our scraper a small run costs a fraction of a cent, which Apify's free plan credit covers.

**Can I find all email addresses on a domain?**
Only the ones that are published or that a finder tool has seen elsewhere. A scraper returns what the site shows; Hunter-style tools add addresses from their database and guesses from the company's email pattern. No tool can list every mailbox on a domain.

**Why does my scraper find no emails on a site that clearly shows one?**
The address is probably hidden with `[at]` text, Cloudflare email protection, an image, or JavaScript, or it sits on a contact page your script never visited. See the pitfalls list above.

**Is it legal to scrape email addresses from websites?**
Reading public pages is generally allowed, but storing and emailing the addresses is regulated by GDPR, CAN-SPAM and similar laws. See the legal section above and check the rules where you and your recipients are.

## Related

- [Contact and lead scrapers](../contact-and-lead-scrapers)
- [How to extract emails from a website for free](extract-emails-from-website-free)
- [How to find a YouTube channel's email](find-youtube-channel-email)
- [Get all URLs from a sitemap with Python](get-all-urls-from-sitemap-python)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I extract emails from websites for free?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Check the contact and legal pages by hand, or run the Python script above. Hunter's free plan gives 50 credits a month, and on our scraper a small run costs a fraction of a cent, which Apify's free plan credit covers."
      }
    },
    {
      "@type": "Question",
      "name": "Can I find all email addresses on a domain?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Only the ones that are published or that a finder tool has seen elsewhere. A scraper returns what the site shows; Hunter-style tools add addresses from their database and guesses from the company's email pattern. No tool can list every mailbox on a domain."
      }
    },
    {
      "@type": "Question",
      "name": "Why does my scraper find no emails on a site that clearly shows one?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The address is probably hidden with [at] text, Cloudflare email protection, an image, or JavaScript, or it sits on a contact page your script never visited. See the pitfalls list above."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to scrape email addresses from websites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Reading public pages is generally allowed, but storing and emailing the addresses is regulated by GDPR, CAN-SPAM and similar laws. See the legal section above and check the rules where you and your recipients are."
      }
    }
  ]
}
</script>
