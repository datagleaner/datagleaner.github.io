---
title: "Clearbit Alternative: Enrich Companies by Domain (Free Script and API)"
description: "Clearbit alternatives for company enrichment by domain: a tested free Python script, what it cannot give you, paid options, and a US$0.80 per 1,000 API."
---

# Clearbit alternative: enrich a list of companies by domain

If you used Clearbit to turn a domain into a company name, logo and social profiles, you have three replacements: a free script that reads each site's own metadata, a paid data vendor, or a pay-per-result scraper over an API. Which one fits depends on what you need back. A name, logo, official social links and a contact route are public on the company's site and cost almost nothing to read. Employee counts, revenue, funding and technographics come from a vendor's database and cost real money.

Disclosure: Data Gleaner is us. The free script below works without our product.

## What happened to Clearbit

HubSpot acquired Clearbit in 2023 and folded it into Breeze Intelligence. Clearbit's free tools were shut down in 2025, and the standalone enrichment API is being replaced by HubSpot-only features. Vendor pages differ on dates and details, so check HubSpot's current notices before you plan a migration. If you only ever used Clearbit for a logo, a company name and social links, you do not need a database vendor at all.

## Decide what you need back

| Field | Public on the company's site? | Free script | Vendor database |
|---|---|---|---|
| Company name and description | Yes | Yes | Yes |
| Logo | Often (schema.org logo, icons) | Yes, with fallbacks | Yes |
| Official LinkedIn, X, Instagram, YouTube, GitHub | Often, when the site links them | Partly | Yes |
| General email and phone | Often, on contact pages | Needs more code | Yes |
| Address and founding year | Sometimes (schema.org) | Partly | Yes |
| Employee count, revenue, funding, industry codes | No | No | Yes |
| Named people and work emails | No | No | Yes, with different legal and accuracy limits |

## Option 1: a free Python script

Many sites publish a schema.org `Organization` block in JSON-LD with their name, description, logo and a `sameAs` list of their official profiles. The script below reads it, with Open Graph tags as a fallback. It uses only the standard library.

```python
import json, re, sys, urllib.request
from html.parser import HTMLParser

class P(HTMLParser):
    def __init__(self):
        super().__init__(); self.meta = {}; self.ld = []; self._ld = False; self.title = ""; self._t = False
    def handle_starttag(self, tag, a):
        a = dict(a)
        if tag == "meta":
            k = a.get("property") or a.get("name")
            if k: self.meta[k] = a.get("content", "")
        elif tag == "script" and a.get("type") == "application/ld+json":
            self._ld = True; self.ld.append("")
        elif tag == "title": self._t = True
    def handle_endtag(self, tag):
        if tag == "script": self._ld = False
        if tag == "title": self._t = False
    def handle_data(self, d):
        if self._ld: self.ld[-1] += d
        if self._t: self.title += d

def orgs(node):
    if isinstance(node, list):
        for n in node: yield from orgs(n)
    elif isinstance(node, dict):
        t = node.get("@type"); t = t if isinstance(t, list) else [t]
        if any(x in ("Organization", "Corporation", "LocalBusiness", "OnlineBusiness") for x in t): yield node
        yield from orgs(node.get("@graph", []))

def enrich(domain):
    req = urllib.request.Request(f"https://{domain}", headers={"User-Agent": "Mozilla/5.0 (compatible; enrich-script)"})
    html = urllib.request.urlopen(req, timeout=20).read().decode("utf-8", "replace")
    p = P(); p.feed(html)
    org = {}
    for block in p.ld:
        try: org = next(orgs(json.loads(block)), org) or org
        except ValueError: pass
    logo = org.get("logo")
    if isinstance(logo, dict): logo = logo.get("url")
    return {
        "domain": domain,
        "name": org.get("name") or p.meta.get("og:site_name") or p.title.strip(),
        "description": org.get("description") or p.meta.get("og:description") or p.meta.get("description"),
        "logo": logo or p.meta.get("og:image"),
        "same_as": org.get("sameAs", []),
    }

if __name__ == "__main__":
    for d in sys.argv[1:]:
        try: print(json.dumps(enrich(d), ensure_ascii=False)[:400])
        except Exception as e: print(d, "failed:", e)
```

Run `python3 enrich.py stripe.com plausible.io linear.app`. Results from a test run on 11 October 2026:

| Domain | Name | Logo | `sameAs` links |
|---|---|---|---|
| stripe.com | Stripe | An icon file from the JSON-LD | Several (X, YouTube and more) |
| plausible.io | Plausible Analytics | Falls back to the Open Graph image, a promo picture, not a logo | None published |
| linear.app | "Linear – The system for product development" (the page title) | Falls back to a generated Open Graph image | None published |

That is the honest picture of the free route. It is exact when a site publishes structured data, and it degrades quickly when it does not: names turn into page titles, logos into share images, and the social profile list is empty. You can improve it by also reading the footer's social links, the apple-touch-icon and the contact page, but at that point you are writing a scraper.

For logos alone, icon services such as Google's favicon service return a small square image for any domain with no code, at low resolution.

## Option 2: a paid data vendor

Vendors in this space include Apollo, People Data Labs, Hunter, FullContact, Abstract API and others, plus HubSpot's own Breeze Intelligence. They return firmographics your own script cannot: headcount, revenue ranges, funding, industry codes and, from some, contacts. Pricing is a mix of monthly plans and pay-per-record credits, and free tiers are small and change. Check each vendor's pricing page and ask how often a record is refreshed and where the contact data comes from. A vendor list in a vendor's own blog post is marketing, not a comparison.

## Option 3: a hosted scraper over an API

[Company Enrichment by Domain](https://apify.com/datagleaner/company-enrichment-by-domain) on the Apify Store does Option 1 at scale, with the fallbacks written. For each domain it reads the home page and up to two linked contact and about pages over plain HTTP and returns one row: company name, description, logo, the official social profiles the site itself claims (LinkedIn company page, X, Facebook, Instagram, YouTube, TikTok, GitHub, Pinterest), a general email and phone, address, country, founding date and light tech hints such as Shopify, WordPress or HubSpot.

- **Price:** US$0.0008 per enriched company, US$0.80 per 1,000. A domain counts only when it returns a company name plus at least one social profile or contact. Unreachable, parked and empty domains are free, so a messy list costs less.
- **Profiles are official only.** A social link counts when the site's own header, footer or schema.org data claims it; links in the page body (testimonials, blog authors) are left out, and LinkedIn returns company pages, never a person's profile.
- **What it will not give you:** employee counts, revenue, funding, or a named contact. It returns a general inbox such as info@ or hello@. For people on a company's own site use [Company Leadership & Team Page Scraper](https://apify.com/datagleaner/team-page-contacts).

Call it with one request:

```bash
curl -X POST "https://api.apify.com/v2/acts/datagleaner~company-enrichment-by-domain/run-sync-get-dataset-items?token=<APIFY_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"domains": ["stripe.com", "linear.app"]}'
```

Or start from a public example task, which has the input filled in:

- [Enrich a list of company domains (Clearbit alternative)](https://apify.com/datagleaner/company-enrichment-by-domain/enrich-a-list-of-domains-clearbit-alternative)
- [Company socials and tech stack from a domain](https://apify.com/datagleaner/company-enrichment-by-domain/company-tech-stack-and-socials-from-domain)
- [Add logo, description and contacts to CRM leads by domain](https://apify.com/datagleaner/company-enrichment-by-domain/crm-lead-enrichment-logo-description-contacts)

It also runs from n8n, Make, Zapier, LangChain and any MCP client; the Actor's README has the steps.

## Which one to pick

| You need | Pick |
|---|---|
| A name, logo and social links for a few hundred leads, at the lowest cost | The Actor or the free script |
| The same for sites without structured data | The Actor |
| Headcount, revenue, funding | A paid vendor |
| A work email for a named person | A paid vendor or a team-page scraper, and check the legal basis |
| A step in an AI agent's tool chain | The Actor through MCP, paying only for hits |

## Using the results responsibly

Everything the Actor returns is published by the company on its own site. Role addresses such as info@ are lower risk than personal ones, but cold email rules (CAN-SPAM, GDPR, PECR) still apply to how you use them.

## FAQ

**Is there a free Clearbit alternative?**
For a company name, description, logo and the social links a site publishes, yes: read the site's schema.org data with a short script. For firmographics such as employee count and revenue, there is no reliable free source.

**What replaced the Clearbit API?**
HubSpot folded Clearbit into Breeze Intelligence, and the standalone API is being phased out. Developers who only need public company profile data can use a scraper or a vendor API instead.

**How do I get a company logo from a domain?**
Read the schema.org `logo` or the site's icons, or use an icon service for a low-resolution image. The Open Graph image is usually a share picture, not the logo.

**How much does the Actor cost for 10,000 domains?**
At US$0.0008 per enriched company, 10,000 domains cost at most US$8.00, less for domains that fail or are parked.

## Related

- [SEC Form D: find companies that just raised money](sec-form-d-newly-funded-companies)
- [Find social media profiles from a list of company websites](find-company-social-media-profiles-from-website)
- [Enrich HubSpot companies with email and phone from the domain](enrich-hubspot-companies-contact-details)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {"@type": "Question", "name": "Is there a free Clearbit alternative?", "acceptedAnswer": {"@type": "Answer", "text": "For a company name, description, logo and the social links a site publishes, yes: read the site's schema.org data with a short script. For firmographics such as employee count and revenue there is no reliable free source."}},
    {"@type": "Question", "name": "What replaced the Clearbit API?", "acceptedAnswer": {"@type": "Answer", "text": "HubSpot folded Clearbit into Breeze Intelligence and the standalone API is being phased out. Developers who only need public company profile data can use a scraper or a vendor API instead."}},
    {"@type": "Question", "name": "How do I get a company logo from a domain?", "acceptedAnswer": {"@type": "Answer", "text": "Read the schema.org logo or the site's icons, or use an icon service for a low-resolution image. The Open Graph image is usually a share picture, not the logo."}},
    {"@type": "Question", "name": "How much does Company Enrichment by Domain cost for 10,000 domains?", "acceptedAnswer": {"@type": "Answer", "text": "At US$0.0008 per enriched company, 10,000 domains cost at most US$8.00, less for domains that fail or are parked."}}
  ]
}
</script>
