---
title: "Find Emails for Google Maps Leads From the Website Column"
description: "Turn a Google Maps or Outscraper CSV into leads with emails: scan the Website column, read contact pages, decode Cloudflare, merge back by domain."
---

# How to find emails for Google Maps leads from the Website column

To find emails for Google Maps leads, take the website column of your export, reduce each URL to its bare domain, fetch the home page and the contact, impressum and about pages of that domain, collect the `mailto:` links, Cloudflare-protected addresses and plain text addresses, and write the results back onto your rows using the domain as the key. A Google Maps or Outscraper export usually has a name, address, phone and website for each business but no email, so the website is where the email has to come from. Reading only the home page misses many of them, because the address is often on a contact page or hidden. This page gives a free, runnable Python script for the whole job, the reasons a simple regex fails, and a hosted option for long lists.

Disclosure: Data Gleaner, mentioned near the end as one option for large lists, is us. Every other method on this page is free and needs no account.

## Why the Maps export has no email

Google Maps listings show a business's name, category, address, phone number and website. An email is not a standard field, so scrapers of Maps data return the website and leave the email for a second step: a crawl of each business's own site, which is what this page covers.

## Step 1: clean the Website column into domains

Your column may be called `website`, `site` or `url` depending on the tool that made the file, and the values are messy: `https://www.Example.com/en?utm_source=gmaps`, `example.com`, an empty cell, or a link to a Facebook page. Reduce each to a domain first. That does three things: it gives you a key for merging back, it collapses duplicates (chains list the same site on many rows, so you scan it once), and it lets you skip rows with no site of their own, including rows whose website is a Facebook or Instagram page. The `domain_of` function in the script below does this.

## Step 2: read more than the home page

A home page often holds no email at all. Businesses tend to put it where a visitor would look for it, so the script follows links on the home page whose address contains `contact`, `kontakt`, `impressum`, `about`, `legal` or `imprint`. The impressum (legal notice) matters most for German-speaking countries, where the law requires a way to contact the business, so the email is often there and nowhere else. It reads up to four pages per domain.

## Step 3: why a homepage regex misses most of them

A single regular expression run over home page text fails for reasons that are mostly structural, not about the pattern:

- **Wrong page.** The address sits on `/contact` or `/impressum`, and the home page has only a contact form or a link.
- **`mailto:` only.** The visible text says "Email us" and the address exists only in the link, so a text regex never sees it.
- **Obfuscated text.** Sites write `name [at] domain [dot] com` or `name(at)domain.com` to hide from harvesters.
- **Cloudflare email protection.** Cloudflare can replace an address with `[email protected]` in the page and keep the real one in a `data-cfemail` attribute as a hex string. The first byte is a key; each following byte is XORed with it. The script decodes this.
- **Scripts and images.** An address injected by JavaScript, or shown as an image, is not in the HTML a plain request gets.
- **False hits.** A regex also matches things like `logo@2x.png` (an image file name). The script drops matches that end in an image extension.

The script below handles the first four and the last. It cannot read the fifth.

## The Python script

It reads your CSV, scans each unique domain, and writes a new CSV with your original columns plus `domain`, `emails`, `phones`, `socials` and `email_source` (the pages the emails came from). It uses `requests` and the standard library. We ran its parsing on sample HTML with a Cloudflare-encoded address, an `[at]`/`[dot]` address, a `mailto:` link with a query string, a `tel:` link and an image file name: it returned the three real addresses and the phone number, and ignored the image name. We also ran the full command on a small CSV with one real site listed twice in different forms, an empty row and a Facebook page: it scanned the site once, filled both of its rows and skipped the other two. We did not test it against a large list, so run it on a sample first.

```python
# pip install requests
import csv
import re
import sys
from html.parser import HTMLParser
from urllib.parse import unquote, urljoin, urlparse

import requests

EMAIL = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)*\.[A-Za-z]{2,}")
AT_DOT = re.compile(r"\s*[\[(]\s*(at|dot)\s*[\])]\s*", re.I)
CONTACT_WORDS = ("contact", "kontakt", "impressum", "about", "legal", "imprint")
SOCIAL_HOSTS = ("linkedin.com", "facebook.com", "instagram.com", "x.com",
                "twitter.com", "youtube.com", "tiktok.com")
IMAGE_SUFFIX = (".png", ".jpg", ".jpeg", ".gif", ".webp", ".svg")


def domain_of(value):
    """'https://www.Example.com/path' and 'example.com' both become 'example.com'; social pages become ''."""
    value = (value or "").strip()
    if not value:
        return ""
    if "//" not in value:
        value = "//" + value
    host = (urlparse(value).hostname or "").lower()
    host = host[4:] if host.startswith("www.") else host
    # A Facebook or Instagram page in the Website column is not the business's own site.
    if any(host == s or host.endswith("." + s) for s in SOCIAL_HOSTS):
        return ""
    return host


def cf_decode(hexstr):
    """Cloudflare email protection: the first byte is an XOR key for the rest."""
    try:
        key = int(hexstr[:2], 16)
        return "".join(chr(int(hexstr[i:i + 2], 16) ^ key) for i in range(2, len(hexstr), 2))
    except ValueError:
        return ""


class Page(HTMLParser):
    def __init__(self):
        super().__init__()
        self.mailto, self.tel, self.cf, self.links, self.text = [], [], [], [], []
        self._skip = False

    def handle_starttag(self, tag, attrs):
        a = dict(attrs)
        href = a.get("href") or ""
        if tag == "a" and href.lower().startswith("mailto:"):
            self.mailto.append(unquote(href[7:]).split("?")[0])
        elif tag == "a" and href.lower().startswith("tel:"):
            self.tel.append(unquote(href[4:]).split(";")[0].strip())
        elif tag == "a" and href:
            self.links.append(href)
        if a.get("data-cfemail"):
            self.cf.append(cf_decode(a["data-cfemail"]))
        if tag == "a" and "/cdn-cgi/l/email-protection#" in href:
            self.cf.append(cf_decode(href.split("#", 1)[1]))
        self._skip = tag in ("script", "style")

    def handle_endtag(self, tag):
        self._skip = False

    def handle_data(self, data):
        if not self._skip:
            self.text.append(data)


def scan(html):
    p = Page()
    p.feed(html)
    emails = {e.lower() for e in p.mailto + p.cf if "@" in e}
    text = AT_DOT.sub(lambda m: "@" if m.group(1).lower() == "at" else ".", " ".join(p.text))
    emails |= {e.lower() for e in EMAIL.findall(text)}
    emails = {e for e in emails if not e.endswith(IMAGE_SUFFIX)}
    return emails, set(p.tel), p.links


def enrich(domain, session, max_pages=4):
    found = {"emails": {}, "phones": {}, "socials": set()}
    queue, seen = [f"https://{domain}/"], set()
    while queue and len(seen) < max_pages:
        url = queue.pop(0)
        if url in seen:
            continue
        seen.add(url)
        try:
            r = session.get(url, timeout=20)
        except requests.RequestException:
            continue
        if r.status_code != 200 or "html" not in r.headers.get("content-type", ""):
            continue
        emails, phones, links = scan(r.text)
        for e in emails:
            found["emails"].setdefault(e, r.url)
        for t in phones:
            found["phones"].setdefault(t, r.url)
        for href in links:
            full = urljoin(r.url, href).split("#")[0]
            host = (urlparse(full).hostname or "").lower()
            if any(host == s or host.endswith("." + s) for s in SOCIAL_HOSTS):
                if urlparse(full).path.strip("/"):
                    found["socials"].add(full)
            elif host in (domain, "www." + domain) and any(w in full.lower() for w in CONTACT_WORDS):
                queue.append(full)
    return found


def main(path_in, column, path_out):
    session = requests.Session()
    session.headers["User-Agent"] = "lead-enrich/1.0 (contact: you@example.com)"
    with open(path_in, newline="", encoding="utf-8-sig") as f:
        rows = list(csv.DictReader(f))
    results = {}
    for row in rows:
        d = domain_of(row.get(column))
        if d and d not in results:
            results[d] = enrich(d, session)
            print(d, len(results[d]["emails"]), "emails", flush=True)
    fields = list(rows[0].keys()) + ["domain", "emails", "phones", "socials", "email_source"]
    with open(path_out, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=fields)
        w.writeheader()
        for row in rows:
            d = domain_of(row.get(column))
            r = results.get(d)
            row.update(domain=d, emails="", phones="", socials="", email_source="")
            if r:
                row["emails"] = "; ".join(r["emails"])
                row["phones"] = "; ".join(r["phones"])
                row["socials"] = "; ".join(sorted(r["socials"]))
                row["email_source"] = "; ".join(sorted(set(r["emails"].values())))
            w.writerow(row)


if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2], sys.argv[3])
```

Run it with the path of your CSV, the name of the website column and an output path:

```
python enrich.py leads.csv website leads_enriched.csv
```

Limits of this script:

- **No JavaScript.** It reads the HTML the server sends, so addresses built by scripts are missed.
- **One site at a time, no retries.** A site that answers 403 or 429 to non-browser clients is skipped. Add a pause between sites on a long list, and respect each site's `robots.txt` and terms; this script does not check `robots.txt`.
- **Only the pages it can find.** If the contact page is not linked with one of the keywords, it is not opened.
- **Whose address is it?** A page can show an address that belongs to a web agency, a franchisor or a booking platform. Check a sample.

## Realistic hit rates by business type

The best published measurement we know of is from [HasData](https://hasdata.com/blog/extract-emails-from-google-maps), a scraping vendor, on 1,994 Google Maps businesses across 20 categories in five US cities (updated September 2026). Per 100 businesses a search returned, 90 listed a website, 88 of those sites answered, and 51 had an email somewhere on the site: 30 a person's address and 21 a generic `info@` or `contact@`. That was with plain HTTP and no JavaScript, like the script above; rendering the pages in a browser raised it to about 59 per 100. By category, gyms returned 66 contacts per 100 businesses and dry cleaners 23, largely because only 61 in 100 dry cleaners had a site at all. We have not repeated the measurement, so treat it as one data point. The pattern by business type:

- **Companies with a website of their own** (agencies, clinics, law firms, contractors) usually publish a contact address on a contact page or in the footer. These are your best rows.
- **German-speaking businesses** are legally expected to publish contact details in the impressum, so read that page.
- **Restaurants, salons and small shops** often have only a contact form, a phone number or a social page, so many rows will return a phone number and no email.
- **Sites on a booking or ordering platform, or a Facebook page used as a website**, usually have no email on the page at all.
- **Chains** list one corporate site on many rows. You get one result, which is probably a head-office address and not the local branch.

To get a real number for your niche, take 100 random rows, run them, count how many return an email and check ten of the misses by hand. A miss on a site that shows no email is a limit of the data; a miss on a site that does show one tells you what your script needs to handle next.

## Hosted option: Data Gleaner Website Contact Details Scraper

If the list has thousands of rows or you do not want to maintain the script, the [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper) on the Apify Store does the same steps as a hosted job.

What it does, from its documentation:

- Fetches the home page and the pages most likely to hold contact details (contact, about, impressum, team and footer links, including Japanese, Chinese, German, French, Spanish and Italian names), on the same domain only, up to `maxPagesPerSite` pages (8 by default, 30 at most). It respects `robots.txt` by default.
- Reads emails from `mailto:` links, plain text, obfuscated forms such as `name [at] domain [dot] com`, Cloudflare email protection and JSON-LD.
- Returns phone numbers validated and normalized to E.164, plus social profiles (LinkedIn, X, Facebook, Instagram, YouTube, TikTok, GitHub), contact forms and schema.org addresses, one row per website, each value with the page it came from (`foundOn`).
- Costs US$4.00 per 1,000 websites that return at least one contact. Unreachable sites and sites with no contacts are free.

It uses plain HTTP with no browser, so it has the same blind spot as the script above for details that only scripts inject, and for sites that block non-browser clients. Addresses shown as images are not read.

This snippet reads a `leads.csv` with a `website` column, runs the Actor on the unique sites, and merges the results back by domain. It reuses `domain_of` from the script above (save that script as `enrich.py` next to it):

```python
# pip install apify-client
import csv
import os

from apify_client import ApifyClient

from enrich import domain_of  # the helper from the script above

client = ApifyClient(os.environ["APIFY_TOKEN"])

with open("leads.csv", newline="", encoding="utf-8-sig") as f:
    rows = list(csv.DictReader(f))

# One URL per domain, so a site listed on many rows is scanned and charged once.
sites = {}
for row in rows:
    d = domain_of(row.get("website"))
    if d:
        sites.setdefault(d, row["website"].strip())

run = client.actor("datagleaner/website-contact-details-scraper").call(run_input={
    "websites": list(sites.values()),
    "maxPagesPerSite": 8,
    "defaultCountry": "US",  # used only for local-format phone numbers
})
if run is None:
    raise SystemExit("The Actor run did not start.")

by_domain = {}
for item in client.dataset(run.default_dataset_id).iterate_items():
    by_domain[domain_of(item["website"])] = item

fields = list(rows[0].keys()) + ["emails", "phones", "linkedin", "email_source", "status"]
with open("leads_enriched.csv", "w", newline="", encoding="utf-8") as f:
    out = csv.DictWriter(f, fieldnames=fields)
    out.writeheader()
    for row in rows:
        item = by_domain.get(domain_of(row.get("website")))
        if item:
            row["emails"] = "; ".join(e["value"] for e in item["emails"])
            row["phones"] = "; ".join(p["value"] for p in item["phones"])
            row["linkedin"] = "; ".join(s["url"] for s in item["socials"]["linkedin"])
            row["email_source"] = "; ".join(sorted({e["foundOn"] for e in item["emails"]}))
            row["status"] = item["status"]
        out.writerow(row)
```

Each item has a `status` (`ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl` or `error`), so rows that failed are easy to separate from rows that are simply empty. You can also export the dataset as CSV, Excel or JSON from the Apify Console and merge it yourself.

## FAQ

**How do I find emails for Google Maps leads?**
Take the website column from your Google Maps or Outscraper export, reduce each URL to its domain, fetch the home page and the contact, impressum and about pages of each domain, and collect `mailto:` links, Cloudflare-protected addresses and plain text addresses. Then write the results back onto the original rows by domain. Google Maps itself rarely lists an email, so the website is the usual source. The script above does this for free; a hosted scraper does the same at larger volume.

**Why does my script find no email on a site that shows one?**
The address is usually on a page other than the home page, or it is hidden. Common causes are a contact or impressum page that the script never opened, an address written as `name [at] domain [dot] com`, Cloudflare email protection, which replaces the address with an encoded string, an address injected by JavaScript, or an address shown as an image. The first three can be fixed in a plain HTTP script; the last two need a browser or are not readable.

**What share of Google Maps leads will return an email?**
Roughly half, in the one large published measurement: HasData found an email on the sites of 51 of every 100 US Maps businesses with a plain HTTP fetch, and about 59 when pages were rendered in a browser, ranging from gyms (high) to dry cleaners (low). It depends on the business type, the country and how many pages you read, so measure it on your own list: run 100 random rows, count how many return an email and check ten misses by hand. That tells you the hit rate for your niche and shows whether the misses are hidden addresses or sites that simply publish no email.

**What if a lead has no website in Google Maps?**
Then there is nothing to scan, and a website-based method cannot help. Those rows keep their phone number and address from the Maps export. Some businesses list only a Facebook or Instagram page as their website; public contact details on such a page are often not served without a login, so expect few results there.

**Can I merge the results back into my CSV?**
Yes. Use the domain as the key: strip the scheme, the path and a leading `www.`, lower-case it, and join on that. Two Maps rows for the same chain or the same site then share one result. Keep the page each email was found on in its own column so you can check a sample before you use the list.

**Is it legal to email addresses collected from business websites?**
It depends on where you and the recipients are and on who the address belongs to. Anti-spam and data protection laws such as CAN-SPAM and GDPR treat business-to-business email differently from country to country, and an address like `firstname@company.com` can identify a person. This page is not legal advice, so check the rules that apply to you before sending.

## Related guides

- [Find email addresses from a list of websites](../contact-and-lead-scrapers): the general method for any list of domains, including obfuscated forms.
- [Extract phone numbers from a list of websites](extract-phone-numbers-from-websites): the same approach for phones, normalized to E.164.
- [Find company social media profiles from a website](find-company-social-media-profiles-from-website): get LinkedIn, Facebook and Instagram pages for the same leads.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I find emails for Google Maps leads?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Take the website column from your Google Maps or Outscraper export, reduce each URL to its domain, fetch the home page and the contact, impressum and about pages of each domain, and collect mailto: links, Cloudflare-protected addresses and plain text addresses. Then write the results back onto the original rows by domain. Google Maps itself rarely lists an email, so the website is the usual source. The script above does this for free; a hosted scraper does the same at larger volume."
      }
    },
    {
      "@type": "Question",
      "name": "Why does my script find no email on a site that shows one?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The address is usually on a page other than the home page, or it is hidden. Common causes are a contact or impressum page that the script never opened, an address written as name [at] domain [dot] com, Cloudflare email protection, which replaces the address with an encoded string, an address injected by JavaScript, or an address shown as an image. The first three can be fixed in a plain HTTP script; the last two need a browser or are not readable."
      }
    },
    {
      "@type": "Question",
      "name": "What share of Google Maps leads will return an email?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Roughly half, in the one large published measurement: HasData found an email on the sites of 51 of every 100 US Maps businesses with a plain HTTP fetch, and about 59 when pages were rendered in a browser, ranging from gyms (high) to dry cleaners (low). It depends on the business type, the country and how many pages you read, so measure it on your own list: run 100 random rows, count how many return an email and check ten misses by hand. That tells you the hit rate for your niche and shows whether the misses are hidden addresses or sites that simply publish no email."
      }
    },
    {
      "@type": "Question",
      "name": "What if a lead has no website in Google Maps?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Then there is nothing to scan, and a website-based method cannot help. Those rows keep their phone number and address from the Maps export. Some businesses list only a Facebook or Instagram page as their website; public contact details on such a page are often not served without a login, so expect few results there."
      }
    },
    {
      "@type": "Question",
      "name": "Can I merge the results back into my CSV?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. Use the domain as the key: strip the scheme, the path and a leading www., lower-case it, and join on that. Two Maps rows for the same chain or the same site then share one result. Keep the page each email was found on in its own column so you can check a sample before you use the list."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to email addresses collected from business websites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It depends on where you and the recipients are and on who the address belongs to. Anti-spam and data protection laws such as CAN-SPAM and GDPR treat business-to-business email differently from country to country, and an address like firstname@company.com can identify a person. This page is not legal advice, so check the rules that apply to you before sending."
      }
    }
  ]
}
</script>
