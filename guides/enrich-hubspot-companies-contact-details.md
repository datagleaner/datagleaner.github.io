---
title: "Enrich HubSpot Companies With Email and Phone From Their Domain"
description: "Export HubSpot company domains, scrape each site for a generic inbox, phone, LinkedIn page and address, and import them back as company properties."
---

# How to enrich HubSpot companies with email and phone from the website domain

To enrich HubSpot companies with an email and phone number, export the companies with their Record ID and Company domain name, read each domain's home page and contact page for the details the company publishes, then import the results as a CSV that updates the existing records by Record ID. You get a generic inbox such as `info@`, a main phone number, a LinkedIn company page and a postal address. You do not get a named person's email: that is the job of a finder such as Apollo or Hunter, and running this cheap step first means you only pay a finder for companies worth approaching. This page has the field mapping, a free Python script for the round trip, a Zapier and Make variant for new companies, and the limits.

Disclosure: Data Gleaner, mentioned near the end as one option for long lists and automation, is us. The export, script and import need only your HubSpot account.

## Check HubSpot's own enrichment first

If your portal has a paid hub (Marketing, Sales, Service, Data or Content Hub at Starter or above, or Smart CRM Professional or Enterprise), HubSpot's [data enrichment](https://knowledge.hubspot.com/records/enrich-your-contact-and-company-data) can fill company properties from the Company domain name, including Phone number, LinkedIn company page, Street address, City, Postal code and Country/Region. It needs Super Admin or the Data enrichment permission, and companies without a domain are not enriched. Run it first if you have it.

What it does not give you is a company email: HubSpot has no default company property for one. And its values come from HubSpot's data provider, not from what the company publishes today. The company's own website fills those gaps: the inbox it publishes, the phone on its contact page, and the fields enrichment left blank, including on the free CRM.

Apify also has a [HubSpot integration](https://docs.apify.com/integrations/hubspot) that runs Apify's own Contact Details Scraper from a company record. It creates or updates HubSpot **contacts** associated with the company and does not change the company record itself, so use it if you want contact records rather than company properties.

## The field mapping

A company website usually publishes four things that map onto HubSpot company properties: a phone number, a LinkedIn page, a postal address and a generic inbox. The labels below are from HubSpot's list of default company properties. Check labels and internal names in your portal under Settings, Properties before importing, because accounts can rename or add properties.

| What the site publishes | HubSpot company property | Internal name | Note |
|---|---|---|---|
| Main phone number | Phone number | `phone` | Use E.164, for example `+886287802800`. |
| LinkedIn company page | LinkedIn company page | `linkedin_company_page` | A `linkedin.com/company/...` link, not a personal profile. |
| Street address | Street address | `address` | From schema.org markup when present. |
| City | City | `city` | |
| State or region | State/Region | `state` | Often empty outside the US. |
| Postal code | Postal code | `zip` | |
| Country | Country/Region | `country` | Sites give a code such as `TW`; test five rows to see how HubSpot stores it. |
| Generic inbox | A custom property you create | your choice | HubSpot has no default company property for this. Create a single-line text property, for example "Generic email". |

## Step 1: Export your company domains

In HubSpot, go to CRM, Companies, open a view (or "All companies"), click Export, pick CSV, and click Customize to choose the properties. HubSpot emails you a download link, and you need Export permissions. The file needs two columns: **Record ID** and **Company domain name**. Export only the companies missing the field you want, so you do not overwrite what you already have.

## Step 2: Read each site with a free script

The script reads your export, fetches each domain's home page and up to three pages whose URL contains "contact", "about", "impressum" or "kontakt", and writes an import file. It takes:

- the email from `mailto:` links, keeping only generic inboxes (`info`, `contact`, `hello`, `sales`, `support`, `office`, `enquiries`, `team`), so it never records a named person;
- the phone from `tel:` links, normalized to E.164 with `phonenumbers`;
- the LinkedIn company page from links to `linkedin.com/company/`;
- the address from `PostalAddress` in JSON-LD structured data.

```python
# pip install requests phonenumbers
import csv
import json
import re
from html.parser import HTMLParser
from urllib.parse import unquote, urljoin, urlparse

import phonenumbers
import requests

GENERIC = {"info", "contact", "hello", "sales", "support", "office", "enquiries", "team"}
CONTACT_WORDS = ("contact", "about", "impressum", "kontakt")
DEFAULT_REGION = "US"  # used only for numbers written without a country code


class Page(HTMLParser):
    def __init__(self):
        super().__init__()
        self.mail, self.tel, self.links, self.ld = [], [], [], []
        self._ld = False

    def handle_starttag(self, tag, attrs):
        a = dict(attrs)
        href = a.get("href") or ""
        low = href.lower()
        if tag == "a" and low.startswith("mailto:"):
            self.mail.append(unquote(href[7:]).split("?")[0].strip())
        elif tag == "a" and low.startswith("tel:"):
            self.tel.append(unquote(href[4:]).split(";")[0])
        elif tag == "a" and href:
            self.links.append(href)
        self._ld = tag == "script" and a.get("type") == "application/ld+json"

    def handle_endtag(self, tag):
        self._ld = False

    def handle_data(self, data):
        if self._ld:
            self.ld.append(data)


def postal_addresses(node):  # yield every PostalAddress dict in a JSON-LD document
    if isinstance(node, dict):
        if node.get("@type") == "PostalAddress":
            yield node
        for v in node.values():
            yield from postal_addresses(v)
    elif isinstance(node, list):
        for v in node:
            yield from postal_addresses(v)


def scan(domain):
    domain = domain.lower().removeprefix("www.")
    tld = domain.rsplit(".", 1)[-1].upper()
    region = tld if tld in phonenumbers.SUPPORTED_REGIONS else DEFAULT_REGION
    out = dict.fromkeys(["email", "phone", "linkedin", "address", "city", "state", "zip",
                         "country"], "")
    s = requests.Session()
    s.headers["User-Agent"] = "company-enrich/1.0 (contact: you@example.com)"
    root = f"https://{domain}/"
    queue, seen = [root], set()
    while queue and len(seen) < 4:
        url = queue.pop(0)
        if url in seen:
            continue
        seen.add(url)
        try:
            r = s.get(url, timeout=20)
        except requests.RequestException:
            continue
        if r.status_code != 200 or "html" not in r.headers.get("content-type", ""):
            continue
        p = Page()
        p.feed(r.text)
        for m in p.mail:
            if not out["email"] and m.split("@")[0].lower() in GENERIC:
                out["email"] = m
        for raw in p.tel:
            try:
                n = phonenumbers.parse(raw, region)
            except phonenumbers.NumberParseException:
                continue
            if not out["phone"] and phonenumbers.is_valid_number(n):
                out["phone"] = phonenumbers.format_number(n, phonenumbers.PhoneNumberFormat.E164)
        for href in p.links:
            if not out["linkedin"] and re.search(r"linkedin\.com/company/", href, re.I):
                out["linkedin"] = href.split("?")[0]
        for block in p.ld:
            try:
                found = list(postal_addresses(json.loads(block)))
            except ValueError:
                continue
            if found and not out["address"]:
                a = found[0]
                country = a.get("addressCountry")
                if isinstance(country, dict):
                    country = country.get("name")
                out.update(address=a.get("streetAddress") or "", city=a.get("addressLocality") or "",
                           state=a.get("addressRegion") or "", zip=str(a.get("postalCode") or ""),
                           country=country or "")
        if url == root:
            for href in p.links:
                full = urljoin(r.url, href).split("#")[0]
                host = urlparse(full).netloc.lower().removeprefix("www.")
                if (host == domain or host.endswith("." + domain)) and any(
                        w in full.lower() for w in CONTACT_WORDS):
                    queue.append(full)
    return out


# Import headers must match HubSpot property labels. Edit the right-hand side if your
# portal renamed one, and set "Generic email" to your custom property's label.
COLUMNS = {"email": "Generic email", "phone": "Phone number", "linkedin": "LinkedIn company page",
           "address": "Street address", "city": "City", "state": "State/Region",
           "zip": "Postal code", "country": "Country/Region"}

with open("hubspot-companies-export.csv", newline="", encoding="utf-8-sig") as f, \
        open("hubspot-companies-import.csv", "w", newline="", encoding="utf-8") as g:
    w = csv.DictWriter(g, fieldnames=["Record ID"] + list(COLUMNS.values()))
    w.writeheader()
    for row in csv.DictReader(f):
        domain = row["Company domain name"].strip()
        if domain:
            data = scan(domain)
            w.writerow({"Record ID": row["Record ID"], **{COLUMNS[k]: v for k, v in data.items()}})
            print(domain, {k: v for k, v in data.items() if v})
```

## Step 3: Import the results back

In HubSpot, start an import and choose the file. For companies, HubSpot matches rows to existing records with a unique identifier: Record ID, Company domain name, or a custom property that requires unique values. With Record ID in the file, no Name column is needed to update. Each column header must correspond to a HubSpot property, in any order. The file must be .csv, .xlsx or .xls with one sheet and a header row, and UTF-8 encoded if it contains non-English characters. HubSpot's import ignores blank cells, so a company where nothing was found keeps its current values; but a non-blank cell does overwrite what is there. Try 5 rows first and read the records afterwards.

## Limits of the do-it-yourself route

- **No JavaScript.** It reads the HTML the server sends. Details inserted by scripts or shown after a click are missed, and sites that answer 403 or 429 to non-browser clients are skipped.
- **Generic inboxes only.** `info@` is the company's front door, not a person.
- **Partial addresses.** Only `PostalAddress` JSON-LD is read, so a company that writes its address as plain text gets none, and one with several offices gets the first.
- **Wrong phone.** A site can list a distributor first. Sample the output.
- **Politeness and law.** It fetches one site at a time. Pause between requests, respect each site's `robots.txt` and terms, and remember that contact details of individuals can be personal data under rules such as GDPR.

## Before Apollo or Hunter

A published inbox answers "can I reach this company at all", not "who runs finance, and what is their email". That is person-level data, which finders such as Apollo or Hunter are built for. The cheap order is to run the website step on every company, drop those with no site or no contact, and spend finder credits only on companies you plan to approach.

## Zapier and Make variant for new companies

To enrich each company as it is created instead of in batches, use the steps below. Module names are as Zapier, Make and Apify list them; follow each tool's own setup screens for field mapping.

**Zapier**
1. Trigger: HubSpot, **New Company**.
2. Action: Apify, **Run Actor**. Sign in to Apify through the OAuth window, choose the Data Gleaner Actor below, and put the new company's domain in its `websites` input. You can run it synchronously or asynchronously; Apify's docs say a synchronous run has a 30-second hard timeout, after which the run is terminated, so test with your own domains.
3. Search: Apify, **Fetch dataset items**, to read the run's result.
4. Action: HubSpot, **Update Company**, mapping the fields from the table above.

**Make**
1. Trigger: HubSpot CRM, **Watch Companies**.
2. Apify module **Run an Actor**, with "Run synchronously" set to "Yes".
3. Apify module **Get Dataset Items**, with the default dataset ID from the previous module in the "Dataset ID" field.
4. HubSpot CRM, **Update a Company**.

Make applies a hard timeout to synchronous runs that varies by plan; for longer runs Apify's docs describe the **Watch Actor Runs** trigger.

## Option: Data Gleaner Website Contact Details Scraper

Disclosure: Data Gleaner is us. The [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper) on the Apify Store does the reading step as a hosted run. For each domain it fetches the home page and the pages most likely to hold contact details (contact, about, impressum and team links, including Japanese, Chinese, German, French, Spanish and Italian names), on the same domain only, up to `maxPagesPerSite` pages (8 by default, 30 at most), and it respects `robots.txt` by default. It returns one row per website with `emails`, `phones` (E.164, validated with libphonenumber), `socials` (LinkedIn, X, Facebook, Instagram, YouTube, TikTok, GitHub), `contactForms`, `addresses` (schema.org `PostalAddress` only), `companyName` and a `status` of `ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl` or `error`.

It costs US$2.00 per 1,000 websites where at least one email is found (US$0.002 each); unreachable sites, sites with only phones or social profiles, and sites with no contacts are free. Its limits match the script's: plain HTTP with no browser, so no JavaScript-only details, sites that block non-browser clients come back `unreachable`, emails and phones shown as images are not read, and `emails` holds every address on the site's own domain (others go to `otherEmails`), so the snippet below picks the generic one itself.

This replaces `scan()` above and writes the same import file:

```python
# pip install apify-client
import csv
import os

from apify_client import ApifyClient

GENERIC = ("info", "contact", "hello", "sales", "support", "office", "enquiries", "team")

client = ApifyClient(os.environ["APIFY_TOKEN"])
with open("hubspot-companies-export.csv", newline="", encoding="utf-8-sig") as f:
    rows = [r for r in csv.DictReader(f) if r["Company domain name"].strip()]

run = client.actor("datagleaner/website-contact-details-scraper").call(run_input={
    "websites": [r["Company domain name"].strip() for r in rows],
    "maxPagesPerSite": 8,
    "respectRobotsTxt": True,
    "defaultCountry": "",
})
if run is None:
    raise SystemExit("The Actor run did not start.")

by_site = {item["website"]: item
           for item in client.dataset(run.default_dataset_id).iterate_items()}

with open("hubspot-companies-import.csv", "w", newline="", encoding="utf-8") as g:
    w = csv.writer(g)
    w.writerow(["Record ID", "Generic email", "Phone number", "LinkedIn company page",
                "Street address", "City", "State/Region", "Postal code", "Country/Region"])
    for r in rows:
        item = by_site.get(r["Company domain name"].strip())
        if not item or item["status"] != "ok":
            continue
        email = next((e["value"] for e in item["emails"]
                      if e["value"].split("@")[0].lower() in GENERIC), "")
        phone = item["phones"][0]["value"] if item["phones"] else ""
        linkedin = item["socials"]["linkedin"][0]["url"] if item["socials"]["linkedin"] else ""
        a = item["addresses"][0] if item["addresses"] else {}
        w.writerow([r["Record ID"], email, phone, linkedin, a.get("streetAddress") or "",
                    a.get("addressLocality") or "", a.get("addressRegion") or "",
                    a.get("postalCode") or "", a.get("addressCountry") or ""])
```

For no code, paste the domains into **Websites**, click **Start**, export the dataset as CSV or Excel, and reshape the columns to the mapping table.

## FAQ

**Can HubSpot find a company's email and phone number automatically?**
Partly. On a paid hub, HubSpot's data enrichment can fill a company's Phone number, LinkedIn company page and address from its domain, using HubSpot's data provider. It does not fill a company email, because there is no default company property for one. For the inbox a company publishes, and for any field enrichment left blank or on the free CRM, the source is the company's own website, which a script or scraper can read and you can import back.

**What is the difference between this and Apollo or Hunter?**
This step reads what a company publishes about itself: a generic inbox, a main phone, a LinkedIn page and an address. Apollo and Hunter are for person-level emails, such as the right contact at the company. Running the website step first is cheaper, because you only buy person-level data for companies that are worth approaching.

**Which HubSpot properties should I import the results into?**
Use the default company properties Phone number (`phone`), LinkedIn company page (`linkedin_company_page`), Street address (`address`), City (`city`), State/Region (`state`), Postal code (`zip`) and Country/Region (`country`). HubSpot has no default company property for a generic inbox, so create a custom text property. Check your portal's property settings first, because accounts can differ.

**Will the import overwrite data I already have?**
It overwrites a property when your file has a non-blank value for it, and leaves the property alone when the cell is blank, because HubSpot's import ignores blank cells. To protect good data, export only the companies where the field is empty and test the import on a few rows.

**How do I enrich each new HubSpot company automatically?**
Use Zapier with the HubSpot New Company trigger, the Apify Run Actor action and the HubSpot Update Company action, or the equivalent modules in Make. Both tools apply timeouts to synchronous runs, so use an asynchronous run if the scrape takes longer than the limit.

**Is it legal to collect these contact details?**
It depends on where you and the recipients are and on what you do with the data. A company's general inbox and switchboard are lower risk than details that identify a person, which can be personal data under GDPR and similar laws, and cold email and calling are regulated separately. This page is not legal advice.

## Related guides

- [Find email addresses from a list of websites](../contact-and-lead-scrapers): the email half of this job in more depth, including obfuscated forms.
- [Extract phone numbers from a list of websites](extract-phone-numbers-from-websites): how the phone parsing and E.164 step work.
- [Find company social media profiles from a website](find-company-social-media-profiles-from-website): the LinkedIn column and the other profiles.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Can HubSpot find a company's email and phone number automatically?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Partly. On a paid hub, HubSpot's data enrichment can fill a company's Phone number, LinkedIn company page and address from its domain, using HubSpot's data provider. It does not fill a company email, because there is no default company property for one. For the inbox a company publishes, and for any field enrichment left blank or on the free CRM, the source is the company's own website, which a script or scraper can read and you can import back."
      }
    },
    {
      "@type": "Question",
      "name": "What is the difference between this and Apollo or Hunter?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "This step reads what a company publishes about itself: a generic inbox, a main phone, a LinkedIn page and an address. Apollo and Hunter are for person-level emails, such as the right contact at the company. Running the website step first is cheaper, because you only buy person-level data for companies that are worth approaching."
      }
    },
    {
      "@type": "Question",
      "name": "Which HubSpot properties should I import the results into?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Use the default company properties Phone number (phone), LinkedIn company page (linkedin_company_page), Street address (address), City (city), State/Region (state), Postal code (zip) and Country/Region (country). HubSpot has no default company property for a generic inbox, so create a custom text property. Check your portal's property settings first, because accounts can differ."
      }
    },
    {
      "@type": "Question",
      "name": "Will the import overwrite data I already have?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It overwrites a property when your file has a non-blank value for it, and leaves the property alone when the cell is blank, because HubSpot's import ignores blank cells. To protect good data, export only the companies where the field is empty and test the import on a few rows."
      }
    },
    {
      "@type": "Question",
      "name": "How do I enrich each new HubSpot company automatically?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Use Zapier with the HubSpot New Company trigger, the Apify Run Actor action and the HubSpot Update Company action, or the equivalent modules in Make. Both tools apply timeouts to synchronous runs, so use an asynchronous run if the scrape takes longer than the limit."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to collect these contact details?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It depends on where you and the recipients are and on what you do with the data. A company's general inbox and switchboard are lower risk than details that identify a person, which can be personal data under GDPR and similar laws, and cold email and calling are regulated separately. This page is not legal advice."
      }
    }
  ]
}
</script>
