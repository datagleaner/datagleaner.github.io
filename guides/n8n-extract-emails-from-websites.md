---
title: "n8n Workflow to Scrape Emails From a List of Websites"
description: "An n8n workflow that reads websites from a Google Sheet and writes emails, E.164 phones, socials and contact forms back to the same row. JSON included."
---

# n8n workflow to scrape emails from a list of websites

To scrape emails from a list of websites in n8n, read the sites from a Google Sheet, send each one to a scraper that crawls its contact, impressum and team pages, and write the emails, phones, social links and contact-form URLs back to the same row with a Google Sheets "Append or Update Row" step. The simplest scraper step is an HTTP Request node that calls Apify's `run-sync-get-dataset-items` endpoint, because it needs no community node, no self-hosting and no regex Code node, so it works on n8n Cloud. This page gives the importable workflow JSON for that, a free no-code version that only reads the home page, a free Python script that reads several pages, and the limits of each.

Disclosure: Data Gleaner, the scraper used in the main workflow, is us. The two free methods on this page need no account beyond n8n and Google.

## Option 1: free, no code, home page only

n8n can do the simplest version with built-in nodes. Chain these:

1. **Google Sheets**, "Get Row(s)", to read the list of websites.
2. **HTTP Request**, a GET to each site's URL.
3. **HTML**, operation "Extract HTML content", with the CSS selector `a[href^="mailto:"]`, "Return Value" set to Attribute, the attribute name `href`, and "Return Array" on.
4. **Google Sheets**, "Append or Update Row", to save the result.

Field names are as the n8n docs list them for the HTML node (Source Data, Extraction Values, Key, CSS Selector, Return Value, Return Array). The `href` values look like `mailto:hello@example.com?subject=Hi`, so you strip `mailto:` and anything after `?` in the next step.

What this misses, and it is a lot:

- **Only the page you request.** Most emails live on a contact, impressum or team page, not on the home page.
- **Only `mailto:` links.** An address written as plain text, as `name [at] domain [dot] com`, or protected by Cloudflare email protection is not found.
- **No JavaScript.** Content injected by scripts is not in the HTML that the HTTP Request node receives.
- **One failing site can stop the run** unless you set the node to continue on error.

It is a good fit for a list of a few dozen small-business sites where you only need the visible address. For anything else, use Option 2 or 3.

## Option 2: free, a Python script that reads several pages

If you can run Python outside n8n, this script fetches the home page and up to three more pages whose URL contains "contact", "about", "impressum", "kontakt", "team" or "legal". On each page it collects `mailto:` links, Cloudflare-protected addresses (decoded from the `data-cfemail` attribute) and plain or `[at]`/`[dot]` text addresses, and records which page each came from. It takes a full URL or a bare domain. Run on sample HTML holding a `mailto:` link with a subject, a Cloudflare-encoded address, an `info [at] foo [dot] com` string and an image called `a@2x.png`, its parser returned the three real addresses and ignored the image.

```python
# pip install requests
import re
from html import unescape
from html.parser import HTMLParser
from urllib.parse import unquote, urljoin, urlparse

import requests

CONTACT_WORDS = ("contact", "about", "impressum", "kontakt", "team", "legal")
EMAIL = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)*\.[A-Za-z]{2,}")
AT_DOT = re.compile(r"\s*[\[(]\s*(at|dot)\s*[\])]\s*", re.I)


class Links(HTMLParser):
    def __init__(self):
        super().__init__()
        self.mailto, self.links, self.cf = [], [], []

    def handle_starttag(self, tag, attrs):
        a = dict(attrs)
        href = a.get("href") or ""
        if tag == "a" and href.lower().startswith("mailto:"):
            self.mailto.append(unquote(href[7:]).split("?")[0].strip())
        elif tag == "a" and href:
            self.links.append(href)
        if a.get("data-cfemail"):
            self.cf.append(a["data-cfemail"])


def decode_cf(hexstr):  # Cloudflare email protection: first byte is the XOR key
    key = int(hexstr[:2], 16)
    return "".join(chr(int(hexstr[i:i + 2], 16) ^ key) for i in range(2, len(hexstr), 2))


def emails_from_html(html):
    p = Links()
    p.feed(html)
    found = {e.lower() for e in p.mailto if EMAIL.fullmatch(e)}
    found |= {decode_cf(h).lower() for h in p.cf}
    text = AT_DOT.sub(lambda m: "@" if m.group(1).lower() == "at" else ".", unescape(html))
    found |= {m.lower() for m in EMAIL.findall(text)}
    found = {e for e in found if not e.endswith((".png", ".jpg", ".gif", ".webp", ".svg"))}
    return found, p.links


def emails_for_site(site, max_pages=4):
    domain = urlparse(site if "//" in site else "//" + site).netloc.lower()  # URL or bare domain
    root = f"https://{domain}/"
    queue, seen, result = [root], set(), {}
    s = requests.Session()
    s.headers["User-Agent"] = "contact-check/1.0 (contact: you@example.com)"
    while queue and len(seen) < max_pages:
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
        found, links = emails_from_html(r.text)
        for e in found:
            result.setdefault(e, url)
        if url == root:
            for href in links:
                full = urljoin(url, href).split("#")[0]
                host = urlparse(full).netloc.lower()
                if (host == domain or host.endswith("." + domain)) and any(w in full.lower() for w in CONTACT_WORDS):
                    queue.append(full)
    return result


if __name__ == "__main__":
    for d in ["example.com", "example.org"]:
        print(d, emails_for_site(d) or "no email found")
```

Limits of the script:

- **No JavaScript, no retries, no block handling.** A site that answers 403 or 429 to non-browser clients is skipped, and sites are fetched one at a time.
- **Plain-text matching is loose.** A regex for the shape of an email also returns addresses that are not contacts, such as a privacy vendor's, so sample the output.
- **Politeness.** Pause between requests on a long list, respect each site's `robots.txt` and terms, and remember that an email that identifies a person can be personal data under rules such as GDPR.

To use it from n8n you would run it as a separate job and import its output, which is the part that is easier to get from a hosted node.

## Option 3: a Google Sheet through n8n and the Data Gleaner Actor

Here the Sheet is both the input and the output.

### The sheet

Make a sheet with a header row and these columns: `website`, `emails`, `phones`, `socials`, `contact_form`, `status`. Fill only `website`, one site per row, as a full URL or a bare domain, with no blank rows in between. The workflow writes the other columns. A blank `website` cell sends an empty list, and the Actor then runs its built-in example site instead, so delete empty rows first.

### The workflow

The Actor is [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper) on the Apify Store. From its documentation, for each website it fetches the home page and the pages most likely to hold contact details (contact, about, impressum / legal notice, team, company profile and footer links, including Japanese, Chinese, German, French, Spanish and Italian names), on the same site only (plus a subdomain when a link clearly names a contact page), up to `maxPagesPerSite` pages (8 by default, 30 at most). It respects `robots.txt` by default. It returns emails (including obfuscated forms and Cloudflare email protection), phone numbers validated with libphonenumber and normalized to E.164, social profiles, contact forms, schema.org addresses and the company name, as one item per website, each value with the page it was found on.

The JSON below has four nodes: Manual Trigger, Google Sheets "Get Row(s)", HTTP Request, and Google Sheets "Append or Update Row" matching on the `website` column. The HTTP Request node sends `{"websites": ["<the row's site>"], "maxPagesPerSite": 8}` and receives that site's result. The Google Sheets step maps the result onto your columns with plain n8n expressions (`.map(...).join(', ')`), not a Code node.

{% raw %}
```json
{
  "name": "Sheet of websites to contacts (Data Gleaner)",
  "nodes": [
    {
      "parameters": {},
      "name": "Manual Trigger",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [0, 0]
    },
    {
      "parameters": {
        "documentId": { "__rl": true, "mode": "list", "value": "" },
        "sheetName": { "__rl": true, "mode": "list", "value": "" },
        "options": {}
      },
      "name": "Get websites",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.5,
      "position": [220, 0]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://api.apify.com/v2/actors/datagleaner~website-contact-details-scraper/run-sync-get-dataset-items",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ { websites: [$json.website], maxPagesPerSite: 8 } }}",
        "options": {
          "batching": { "batch": { "batchSize": 5, "batchInterval": 0 } },
          "timeout": 310000
        }
      },
      "name": "Scrape contacts",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [440, 0]
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": { "__rl": true, "mode": "list", "value": "" },
        "sheetName": { "__rl": true, "mode": "list", "value": "" },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "website": "={{ $json.website }}",
            "emails": "={{ $json.emails.map(e => e.value).join(', ') }}",
            "phones": "={{ $json.phones.map(p => p.value).join(', ') }}",
            "socials": "={{ Object.values($json.socials).flat().map(s => s.url).join(', ') }}",
            "contact_form": "={{ ($json.contactForms[0] || {}).url || '' }}",
            "status": "={{ $json.status }}"
          },
          "matchingColumns": ["website"],
          "schema": []
        },
        "options": {}
      },
      "name": "Write back to the same row",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.5,
      "position": [660, 0]
    }
  ],
  "connections": {
    "Manual Trigger": { "main": [[{ "node": "Get websites", "type": "main", "index": 0 }]] },
    "Get websites": { "main": [[{ "node": "Scrape contacts", "type": "main", "index": 0 }]] },
    "Scrape contacts": { "main": [[{ "node": "Write back to the same row", "type": "main", "index": 0 }]] }
  },
  "settings": { "executionOrder": "v1" }
}
```
{% endraw %}

To use it:

1. In n8n, create a new workflow, choose the menu in the top right, and import it from a file or paste the JSON onto the canvas.
2. Open both Google Sheets nodes, select your Google credential, and choose your document and sheet. Check in the second node that the `website` column is the matching column and that the columns on the right match your headers.
3. Copy your API token from Settings > API & Integrations in the Apify Console. Open the HTTP Request node and create a Header Auth credential with the header name `Authorization` and the value `Bearer YOUR_APIFY_TOKEN`. Apify's documentation recommends the header over the `token` URL parameter as the more secure method.
4. Run it. Start with five rows to check the output before you run the list. If one site times out (HTTP 408), the node stops the run; set the HTTP Request node's On Error setting to continue if you want the rest of the list to finish.

I validated that the JSON parses, and the endpoint, the `websites` and `maxPagesPerSite` fields and the output fields come from the Apify and Actor documentation. I have not imported this file into a live n8n instance, so the field defaults of the Google Sheets nodes may differ slightly by n8n version: check the mapping in step 2 on your first run. The `website` value the Actor returns is the one you sent, so the update step finds the row; confirm that on your first five rows.

### Why this HTTP Request route

Apify's `run-sync-get-dataset-items` endpoint runs the Actor and returns its dataset items in one response, so there is no polling loop. The documented limit is 300 seconds: past that the response is HTTP 408, so the workflow sends one website per request and sets the node's timeout option to 310,000 ms. Items per batch is set to 5 through the node's Batching option, which controls how many input items go in each batch. If you would rather send a whole list in one run, put all the sites in the `websites` array yourself and start the run with the asynchronous Run Actor endpoint, then read the dataset when it finishes.

### The Apify node instead

n8n also has an Apify node. According to Apify's integration documentation, on n8n Cloud you find it by searching "Apify" in the nodes panel and clicking Install node (instance owners may first need to enable verified community node visibility in the Cloud Admin Panel), while self-hosted n8n installs `@apify/n8n-nodes-apify` under Settings > Community Nodes. Its Actors operations include "Run an Actor" and "Run an Actor and get dataset", and the Storage operation "Get dataset items" reads a dataset by "Dataset ID". Use it if you prefer a node over an HTTP call: choose the Actor `datagleaner/website-contact-details-scraper`, paste the same JSON input, and connect the output to the same Google Sheets step. The HTTP route above avoids the install step.

### Python equivalent

The same call from Python, with apify-client 3.x:

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
    print(item["website"], item["status"], [e["value"] for e in item["emails"]])
```

### Price and limits

The price is US$2.00 per 1,000 websites where at least one email is found (on the site's own domain; addresses on other domains do not count). Sites with only phones, contact forms, addresses or social profiles, unreachable sites and sites with no contacts are free. The Actor's own example: 1,000 websites, of which about 60% have an email, costs 600 x $0.002 = $1.20.

Its limits, from its documentation: it uses plain HTTP with no browser, so details injected only by scripts, and sites that block non-browser clients (HTTP 403 or 429), are not found. Emails and phones shown as images are not read. Text addresses are not parsed, only schema.org addresses. Each item has a `status` of `ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl` or `error`, which the workflow writes to the `status` column so you can see why a row is empty.

## FAQ

**How do I scrape emails from a list of websites in n8n?**
Read the list from a Google Sheet, call a scraper for each site, and write the result back to the same row with "Append or Update Row" matched on the website column. A scraper that crawls the contact, impressum and team pages finds more than a home-page-only request, and an HTTP Request node calling Apify's `run-sync-get-dataset-items` endpoint does this without a community node.

**Does this work on n8n Cloud?**
The HTTP Request version does, because it uses only built-in nodes: Google Sheets, HTTP Request and a Header Auth credential. The Apify node is also available on Cloud as an installable node, according to Apify's documentation, but the HTTP route avoids that step.

**Do I need a Code node or a regex?**
Not for the Actor workflow. The scraper returns emails, E.164 phones, socials and contact forms as structured fields, and the Google Sheets node maps them with short n8n expressions. The free options on this page use an HTML node selector or a Python regex plus parsing instead.

**Why did a row come back empty?**
Check the `status` column. `noContacts` means the pages it was allowed to read had none, `unreachable` means a timeout or an HTTP error such as 403 or 429, and `blockedByRobots` means the site's robots.txt disallows the pages. Sites that only inject contact details with JavaScript will show `noContacts`, because the Actor does not run a browser.

**What does it cost to run on 1,000 sites?**
With Data Gleaner's Actor it is US$2.00 per 1,000 websites where at least one email is found, and nothing for unreachable sites or sites without an email, so 1,000 sites of which 60% have an email (an assumed share) would cost about $1.20. n8n bills separately under its own plans.

**Is it legal to scrape emails from websites?**
It depends on where you and the people you contact are and on what you do with the addresses. Reading public pages is one thing; storing addresses that identify a person and sending them marketing is regulated by data protection and anti-spam laws such as GDPR. This page is not legal advice, so check the rules that apply to you before you contact anyone.

## Related guides

- [Google Sheets: extract email from a website URL](google-sheets-extract-email-from-website-url): the formula-only route inside a sheet, and where it stops.
- [Find email addresses from a list of websites](../contact-and-lead-scrapers): the same job outside n8n, including obfuscated forms.
- [Extract phone numbers from a list of websites](extract-phone-numbers-from-websites): normalizing numbers to E.164 with libphonenumber.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I scrape emails from a list of websites in n8n?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Read the list from a Google Sheet, call a scraper for each site, and write the result back to the same row with \"Append or Update Row\" matched on the website column. A scraper that crawls the contact, impressum and team pages finds more than a home-page-only request, and an HTTP Request node calling Apify's run-sync-get-dataset-items endpoint does this without a community node."
      }
    },
    {
      "@type": "Question",
      "name": "Does this work on n8n Cloud?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The HTTP Request version does, because it uses only built-in nodes: Google Sheets, HTTP Request and a Header Auth credential. The Apify node is also available on Cloud as an installable node, according to Apify's documentation, but the HTTP route avoids that step."
      }
    },
    {
      "@type": "Question",
      "name": "Do I need a Code node or a regex?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not for the Actor workflow. The scraper returns emails, E.164 phones, socials and contact forms as structured fields, and the Google Sheets node maps them with short n8n expressions. The free options on this page use an HTML node selector or a Python regex plus parsing instead."
      }
    },
    {
      "@type": "Question",
      "name": "Why did a row come back empty?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Check the status column. noContacts means the pages it was allowed to read had none, unreachable means a timeout or an HTTP error such as 403 or 429, and blockedByRobots means the site's robots.txt disallows the pages. Sites that only inject contact details with JavaScript will show noContacts, because the Actor does not run a browser."
      }
    },
    {
      "@type": "Question",
      "name": "What does it cost to run on 1,000 sites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "With Data Gleaner's Actor it is US$2.00 per 1,000 websites where at least one email is found, and nothing for unreachable sites or sites without an email, so 1,000 sites of which 60% have an email (an assumed share) would cost about $1.20. n8n bills separately under its own plans."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to scrape emails from websites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It depends on where you and the people you contact are and on what you do with the addresses. Reading public pages is one thing; storing addresses that identify a person and sending them marketing is regulated by data protection and anti-spam laws such as GDPR. This page is not legal advice, so check the rules that apply to you before you contact anyone."
      }
    }
  ]
}
</script>
