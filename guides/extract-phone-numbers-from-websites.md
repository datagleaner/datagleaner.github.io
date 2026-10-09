---
title: "Extract Phone Numbers From a List of Websites"
description: "How to extract phone numbers from a list of websites: read tel: links first, avoid regex false positives, normalize to E.164 with libphonenumber, with Python."
---

# How to extract phone numbers from a list of websites

To extract phone numbers from a list of websites, fetch each site's home page and its contact page, read the `tel:` links first because they are numbers the site owner marked on purpose, then parse any remaining numbers in the page text with Google's libphonenumber instead of a regular expression. Convert every result to E.164 (for example `+886287802800`) so duplicates collapse and the numbers can be dialled or imported anywhere. The part most guides skip is local-format numbers such as `03-1234-5678`, which cannot be read without knowing the country, so you have to infer it. This page gives a working Python script for all of that, the limits of doing it yourself, and a bulk option for long lists.

Disclosure: Data Gleaner, mentioned at the end as one option for large lists, is us. Every other method on this page is free and needs no account.

## 1. Read tel: links first

A `tel:` link is the cleanest source on a page. Someone wrote it so that a phone can dial it:

```html
<a href="tel:+1-415-555-0132">Call us</a>
```

Take the part after `tel:`, decode any `%20`, and drop extensions such as `;ext=12`. These links rarely contain dates or order numbers, so they need far less filtering than text does. They are usually in the header, footer and contact page, which is why a good script reads the home page and the contact page and little else.

Some sites show a number as text and never link it, and some build the link with a script that a plain HTTP request does not run. So `tel:` links come first, but not only.

## 2. Check JSON-LD structured data

Many sites describe themselves in a `<script type="application/ld+json">` block, and an `Organization` or `LocalBusiness` entry often has a `telephone` field. It is machine-readable and usually the main switchboard number, so it is a good second source. The script below reads it.

## 3. Parse text, but not with a plain regex

The tempting approach is a regular expression for "digits, spaces and dashes". It fails because many things that are not phone numbers look like phone numbers. On a test string with a date, a timestamp, an order number and a price, the pattern `\+?\d[\d\s().-]{7,}\d` returned `2024-03-15 12` and `20240315123`. Typical false positives:

- Dates and times: `2024-03-15`, `12:30:45`.
- Order, invoice, tracking and tax IDs, which are long runs of digits.
- Prices and quantities with separators: `1,234,567.89`.
- Postal codes and years sitting next to each other.

Regexes also miss real numbers written in other countries' layouts. A better tool is `PhoneNumberMatcher` in the Python port of libphonenumber (`pip install phonenumbers`). It finds candidates in text and keeps only those that are valid for a real numbering plan, so a date or an order number is dropped. Validation is not perfect: a string of digits can still happen to be a valid number, so keep a record of which page and which source each number came from and spot-check a sample.

## 4. Normalize to E.164

E.164 is the international format: a plus sign, the country code and the national number, with no spaces or punctuation, at most 15 digits. `+886-2-8780-2800`, `+886 2 8780 2800` and `02-8780-2800` (read as Taiwan) all become `+886287802800`. Doing this once gives you a stable key for deduplication and a format that dialers, CRMs and SMS tools accept. In libphonenumber it is `phonenumbers.format_number(number, PhoneNumberFormat.E164)`.

## 5. Infer the country for local-format numbers

A number written as `03-1234-5678` has no country code, and the same digits mean different things in different countries. libphonenumber needs a region hint to parse it. With the hint `JP`, the line above is read as `+81312345678`. With `US`, the same text is rejected, because it is not a valid US number. You can infer the hint from:

- The domain's country-code TLD: `.jp` is Japan, `.de` is Germany, `.tw` is Taiwan. A `.com` says nothing.
- The page language (the `lang` attribute on `<html>`), which narrows the country but does not settle it for English, Spanish or Chinese.
- An address on the page, or a `PostalAddress` in the JSON-LD, which is the most reliable signal when present.
- A default you choose for the whole list, when you know the list comes from one market.

Numbers that already start with `+` or `00` and a country code need no hint. The script below uses the TLD and a default region; adding the other signals is the next step if your list needs it.

## The Python script

This script fetches the home page and up to three more pages whose URL contains "contact", "about", "impressum", "legal" or "kontakt". It returns each number in E.164 with how it was found and on which page. It uses `requests`, the standard-library HTML parser and `phonenumbers`. I ran its parsing on sample HTML with a `tel:` link, JSON-LD, a date, a timestamp, an order number and a price: it returned the three real numbers and ignored the rest.

```python
# pip install requests phonenumbers
import json
import re
from html.parser import HTMLParser
from urllib.parse import unquote, urljoin, urlparse

import phonenumbers
import requests

TLD_REGION = {"jp": "JP", "tw": "TW", "de": "DE", "fr": "FR", "uk": "GB", "au": "AU",
              "ca": "CA", "es": "ES", "it": "IT", "nl": "NL", "sg": "SG", "kr": "KR"}
CONTACT_WORDS = ("contact", "about", "impressum", "legal", "kontakt")
E164 = phonenumbers.PhoneNumberFormat.E164


class Page(HTMLParser):
    def __init__(self):
        super().__init__()
        self.tel, self.links, self.text, self.ldjson = [], [], [], []
        self._skip = self._ld = False

    def handle_starttag(self, tag, attrs):
        a = dict(attrs)
        href = a.get("href") or ""
        if tag == "a" and href.lower().startswith("tel:"):
            self.tel.append(unquote(href[4:]).split(";")[0])
        elif tag == "a" and href:
            self.links.append(href)
        self._skip = tag in ("script", "style")
        self._ld = tag == "script" and a.get("type") == "application/ld+json"

    def handle_endtag(self, tag):
        self._skip = self._ld = False

    def handle_data(self, data):
        if self._ld:
            self.ldjson.append(data)
        elif not self._skip:
            self.text.append(data)


def walk(node):  # yield every "telephone" value in a JSON-LD document
    if isinstance(node, dict):
        for k, v in node.items():
            if k == "telephone" and isinstance(v, str):
                yield v
            else:
                yield from walk(v)
    elif isinstance(node, list):
        for v in node:
            yield from walk(v)


def to_e164(raw, region):
    try:
        n = phonenumbers.parse(raw, region)  # region matters only for local formats
    except phonenumbers.NumberParseException:
        return None
    return phonenumbers.format_number(n, E164) if phonenumbers.is_valid_number(n) else None


def phones_from_page(html, region):
    p = Page()
    p.feed(html)
    found = {}  # E.164 -> how it was found, best source first
    for raw in p.tel:  # 1. tel: links are deliberate, so trust them most
        if num := to_e164(raw, region):
            found.setdefault(num, "tel link")
    for block in p.ldjson:  # 2. structured data
        try:
            raws = list(walk(json.loads(block)))
        except ValueError:
            continue
        for raw in raws:
            if num := to_e164(raw, region):
                found.setdefault(num, "json-ld")
    text = re.sub(r"\s+", " ", " ".join(p.text))  # 3. visible text, last resort
    for m in phonenumbers.PhoneNumberMatcher(text, region):  # VALID leniency by default
        found.setdefault(phonenumbers.format_number(m.number, E164), "text")
    return found, p.links


def phones_for_site(domain, default_region="US"):
    session = requests.Session()
    session.headers["User-Agent"] = "phone-check/1.0 (contact: you@example.com)"
    domain = domain.lower()
    tld = domain.rsplit(".", 1)[-1].lower()
    region = TLD_REGION.get(tld, default_region)  # .com says nothing: use the default
    root = f"https://{domain}/"
    queue, seen, result = [root], set(), {}
    while queue and len(seen) < 4:
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
        found, links = phones_from_page(r.text, region)
        for num, how in found.items():
            result.setdefault(num, (how, url))
        if url == root:
            for href in links:
                full = urljoin(url, href).split("#")[0]
                host = urlparse(full).netloc.lower()
                same_site = host == domain or host.endswith("." + domain)
                if same_site and any(w in full.lower() for w in CONTACT_WORDS):
                    queue.append(full)
    return result


if __name__ == "__main__":
    for d in ["example.com", "example.co.jp"]:
        print(d, phones_for_site(d) or "no phone found")
```

Limits of this script:

- **No JavaScript.** It reads the HTML the server sends. Numbers injected by scripts, or shown only after a click, are missed.
- **No retries or block handling.** A site that answers 403 or 429 to non-browser clients is skipped, and the script fetches one site at a time.
- **Small TLD table.** Anything not in `TLD_REGION` falls back to `default_region`. Extend the table for your markets.
- **Numbers shown as images** are not read.
- **Whose number is it?** A number on a page may belong to a partner, a distributor or a support vendor. The script cannot tell.
- **Politeness.** Keep a pause between requests on a long list, respect each site's `robots.txt` and terms, and remember that business phone numbers can still be personal data under rules such as GDPR if they identify a person.

For a handful of sites, a browser extension that lists the `tel:` links on the current page is faster than writing code. For a long list, the script above or a hosted job is the practical route.

## Bulk route: Data Gleaner Website Contact Details Scraper

When the list has thousands of sites, or you do not want to maintain the script, the [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper) on the Apify Store does the same steps as a hosted job.

What it does, from its documentation:

- Reads `tel:` links, text and JSON-LD, validates numbers with libphonenumber and returns them in E.164, with the original text, the country and the page each number came from.
- Reads local numbers using the site's country, inferred from the domain, page language and structured data, or your `defaultCountry` when you set one.
- Fetches the home page and the pages most likely to hold contact details (contact, about, impressum, team and footer links, including Japanese, Chinese, German, French, Spanish and Italian names), on the same domain only, up to `maxPagesPerSite` pages (8 by default, 30 at most). It respects `robots.txt` by default.
- Also returns emails, social profiles, contact forms and schema.org addresses for each site, one row per website.
- Costs US$4.00 per 1,000 websites that return at least one contact. Unreachable sites and sites with no contacts are free.

It uses plain HTTP with no browser, so it has the same blind spot as the script above for details that only scripts inject and for sites that block non-browser clients. Numbers shown as images are not read.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/website-contact-details-scraper").call(run_input={
    "websites": ["https://www.appier.com", "stripe.com", "sakura.ad.jp"],
    "maxPagesPerSite": 8,
    "defaultCountry": "",
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    for phone in item["phones"]:
        print(item["website"], phone["value"], phone["country"], phone["foundOn"])
```

Each item has a `status` (`ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl` or `error`) and a `phones` list, so sites with no number are easy to filter out. You can also export the dataset as CSV, Excel or JSON from the Apify Console instead of writing code.

## FAQ

**How do I extract phone numbers from a website?**
Fetch the home page and the contact page, collect the `tel:` links and any `telephone` field in the JSON-LD, then parse the visible text with libphonenumber (`PhoneNumberMatcher`) rather than a regex, and convert every result to E.164 so duplicates collapse. For a list of websites, run the same steps in a loop over the domains.

**Why does my regex return dates and order numbers?**
Because digits separated by dashes, dots or spaces look the same whether they are a date, an ID, a price or a phone number. A regex only checks the shape. libphonenumber also checks that the number is valid for a real numbering plan, which removes most of these.

**What is E.164 and why use it?**
It is the international phone format: `+`, the country code, then the national number, with no spaces or dashes. Using one format means the same number written three ways becomes one value, and most dialers, CRMs and SMS tools accept it.

**How do I get the country for a number without a country code?**
Guess the country from the domain's country-code TLD, the address or the page language, or use a default for the whole list, and pass it to libphonenumber as the region. A wrong region either rejects the number or reads it as a different country's number, so check a sample.

**Can I extract phone numbers from many websites at once?**
Yes. A script can loop over a list, slowly and politely, or you can run a hosted scraper that handles the concurrency. Either way, check the results by sampling.

**Is it legal to scrape phone numbers from websites?**
It depends on where you and the people you contact are, and on what you do with the numbers. Reading public pages is one thing; storing numbers that identify a person, or calling and texting them, is regulated by data protection and anti-spam laws. This page is not legal advice, so check the rules that apply to you.

## Related guides

- [Find email addresses from a list of websites](../contact-and-lead-scrapers): the same approach for emails, including obfuscated forms.
- [Extract emails from a website for free](extract-emails-from-website-free): free methods for a single site, with a script.
- [Get all URLs from a sitemap in Python](get-all-urls-from-sitemap-python): build the list of pages to scan.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I extract phone numbers from a website?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Fetch the home page and the contact page, collect the tel: links and any telephone field in the JSON-LD, then parse the visible text with libphonenumber (PhoneNumberMatcher) rather than a regex, and convert every result to E.164 so duplicates collapse. For a list of websites, run the same steps in a loop over the domains."
      }
    },
    {
      "@type": "Question",
      "name": "Why does my regex return dates and order numbers?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Because digits separated by dashes, dots or spaces look the same whether they are a date, an ID, a price or a phone number. A regex only checks the shape. libphonenumber also checks that the number is valid for a real numbering plan, which removes most of these."
      }
    },
    {
      "@type": "Question",
      "name": "What is E.164 and why use it?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It is the international phone format: +, the country code, then the national number, with no spaces or dashes. Using one format means the same number written three ways becomes one value, and most dialers, CRMs and SMS tools accept it."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get the country for a number without a country code?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Guess the country from the domain's country-code TLD, the address or the page language, or use a default for the whole list, and pass it to libphonenumber as the region. A wrong region either rejects the number or reads it as a different country's number, so check a sample."
      }
    },
    {
      "@type": "Question",
      "name": "Can I extract phone numbers from many websites at once?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. A script can loop over a list, slowly and politely, or you can run a hosted scraper that handles the concurrency. Either way, check the results by sampling."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to scrape phone numbers from websites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It depends on where you and the people you contact are, and on what you do with the numbers. Reading public pages is one thing; storing numbers that identify a person, or calling and texting them, is regulated by data protection and anti-spam laws. This page is not legal advice, so check the rules that apply to you."
      }
    }
  ]
}
</script>
