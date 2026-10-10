---
title: Find Emails From a List of Websites (API and Python)
description: How to find emails from a website list with an API or Python. Free scripts, finder APIs and pay-per-contact scrapers, with code, costs and caveats.
---

# Find emails from a list of websites with an API

To find emails for a list of websites, send each domain to a tool that fetches the site's home, contact, about and legal-notice pages and reads the addresses published there. You can do that with a short Python script for free, with an email-finder API that also guesses or verifies addresses, or with a hosted scraper you call over an API and pay per site that returns a contact. This page covers all three, then the Data Gleaner Actors built for this job.

Disclosure: Data Gleaner is us. The free options below work without our product.

## Three ways to get emails from a website list

| Approach | What you get | Cost | Good for |
|---|---|---|---|
| Your own script (Python `requests` + regex) | Addresses written in the HTML of the pages you fetch | Free, plus your time | A few hundred sites, one-off lists, full control |
| Email-finder API (Hunter, Prospeo and similar) | Addresses from their own database, plus pattern guesses such as `first.last@domain` and verification | Free tiers with low limits, then a subscription or credits | Finding a named person's address, not just the company inbox |
| Hosted contact scraper (Apify Actors) | Every public email, phone and social link on the site, with the page each came from | Pay per result | Thousands of domains, pipelines and AI agents that need one call |

The difference that matters: a scraper only returns what the site publishes, so you mostly get role addresses such as `info@`, `sales@` or `press@`. A finder API can return a person's address, but some of those are inferred from a naming pattern rather than seen on a page, so verify before you send.

## Option 1: a free Python script

This fetches the home page and a few common contact paths for each site and pulls out `mailto:` links and plain-text addresses.

```python
# pip install requests
import re
import requests

SITES = ["stripe.com", "appier.com"]
PATHS = ["", "/contact", "/contact-us", "/about", "/impressum"]
EMAIL = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")
SKIP = (".png", ".jpg", ".jpeg", ".gif", ".svg", ".webp")

for site in SITES:
    found = set()
    for path in PATHS:
        url = f"https://{site}{path}"
        try:
            r = requests.get(url, timeout=10, headers={"User-Agent": "contact-finder/0.1"})
        except requests.RequestException:
            continue
        if r.ok:
            found |= {e for e in EMAIL.findall(r.text) if not e.lower().endswith(SKIP)}
    print(site, sorted(found))
```

What it will miss, and what to add if you need it:

- **Contact pages at other paths.** Real sites use `/kontakt`, `/company`, `/お問い合わせ` and footer links. Parse the home page links and follow the ones whose text looks like contact, about, team or legal notice.
- **Obfuscated addresses** such as `name [at] domain [dot] com`, and Cloudflare's email protection, which hides the address in an encoded attribute.
- **JavaScript-rendered pages.** Plain HTTP sees only the HTML the server sends. A headless browser (Playwright) is slower but sees script-inserted content.
- **Emails shown as images.** No text extractor reads these.
- **False positives.** Image names such as `logo@2x.png`, tracking IDs like `abc123@sentry.io` and placeholders such as `you@example.com` match the regex. Filter them out before you use the list.
- **Redirects.** `example.com` may redirect to `www.example.com/en/`. Compare hosts after the redirect, or a link-following script skips every internal link.
- **robots.txt and rate limits.** Check `robots.txt`, keep one or two requests per site at a time, and treat a 403 or 429 as a "no" rather than retrying hard.

## Option 2: an email-finder API

Services such as Hunter's Domain Search or Prospeo take a domain and return addresses they have collected or inferred, often with a confidence score and a verification status. They are the right tool when you need a specific person's address. Check how each one labels guessed versus found addresses, and what its free tier allows, before you build a pipeline on it.

## Option 3: a hosted contact scraper over an API

If you have thousands of domains, or an agent that needs "the contact details for these companies" in one call, a hosted scraper saves you from writing and running the crawler. On the Apify Store you pass a list of sites, the Actor runs in Apify's cloud, and you read the results from a dataset over the API or export CSV or Excel.

## Data Gleaner contact and lead scrapers

These Actors charge only when they find something: a website with no email, a YouTube channel with no public email or an app with no developer email costs nothing. The output has the same fields on every row and records the page each email was found on, so lead pipelines and AI agents can use it without cleanup.

| Actor | Input | What it returns | Price | Store |
|---|---|---|---|---|
| Company Contact Details & Website Email Finder | Website URLs or bare domains | Emails, phones (normalized to E.164), LinkedIn, X, Facebook, Instagram, YouTube, TikTok and GitHub links, contact forms, schema.org addresses, company name and description | US$0.002 per website with an email (US$2.00 per 1,000); sites with only phones or socials are free | [website-contact-details-scraper](https://apify.com/datagleaner/website-contact-details-scraper) |
| YouTube Channel Contacts | Channel URLs, @handles, IDs, or a niche keyword | Public emails from the description, About links, link-in-bio page and the creator's website, plus socials, subscribers, videos and country | US$0.015 per channel with at least one public email | [youtube-channel-contacts](https://apify.com/datagleaner/youtube-channel-contacts) |
| App Developer Email Finder | App URLs or IDs, or a keyword, for Google Play, the Apple App Store, Steam and the Chrome Web Store | Developer email (tagged by where it was found), website, address, phone, privacy policy, installs or users, rating and developer socials | US$0.0015 per app with a developer email; apps without one are free | [app-developer-email-finder](https://apify.com/datagleaner/app-developer-email-finder) |
| Company Leadership & Team Page Scraper | Company domains | Every named person on the company's own team, about, leadership and Impressum pages, with job title, seniority, LinkedIn link and published email; it never guesses an email | US$0.002 per person | [team-page-contacts](https://apify.com/datagleaner/team-page-contacts) |
| Shopify Partner & Agency Contacts | Countries, services, languages, tiers or partner URLs | Agencies and experts from the Shopify Partner Directory with public email, phone, website, location, services, rating, reviews, price range and tier | US$0.005 per partner with an email or phone; others are free | [shopify-partner-contacts](https://apify.com/datagleaner/shopify-partner-contacts) |
| Researcher Email Finder | Research topics, author names, PMIDs or DOIs | Researchers with the email they published in PubMed papers, plus affiliation, country, ORCID and recent papers | US$0.005 per researcher with an email | [researcher-email-finder](https://apify.com/datagleaner/researcher-email-finder) |

**Company Contact Details & Website Email Finder** reads the home page and up to `maxPagesPerSite` pages per site (default 8, up to 30): contact, about, impressum or legal notice, team and company profile pages, including Japanese, Chinese, German, French, Spanish and Italian equivalents. It decodes obfuscated forms such as `name [at] domain [dot] com` and Cloudflare-protected addresses, stays on the same domain and respects `robots.txt` by default. Worked example from its README: 1,000 websites, of which about 60% have an email, costs 600 x US$0.002 = US$1.20.

**YouTube Channel Contacts** returns only what a creator published openly. It does not open YouTube's "View email address" button, which sits behind a sign-in and a CAPTCHA; it reports whether the button exists (`hasBusinessEmailButton`) so you know which creators you would have to contact by hand. On the Actor's own test runs, about 45% of a mixed sample of 20 channels had a public email, and about 20 to 30% of a 30-channel "fitness coach" search. See [how to find a YouTube channel's email](guides/find-youtube-channel-email).

**App Developer Email Finder** reads the store's own listing for each app and, with `followLinks` on, up to three pages of the developer's website, support page and privacy policy. Every email is tagged with where it came from, and hosting or agency addresses are filtered out. Worked example from its README: 1,000 Google Play apps, of which about 95% list a developer email, cost 950 x US$0.0015 = US$1.43.

**Company Leadership & Team Page Scraper** finds each domain's team, about, leadership and Impressum pages and reads schema.org Person data, team cards and German-language legal-notice titles such as Geschäftsführer. It returns people most senior first and lets you filter by `seniority`; people you filter out are not billed. Worked example from its README: 500 domains averaging 6 people each cost 3,000 x US$0.002 = US$6.

**Shopify Partner & Agency Contacts** reads the official Shopify Partner Directory, which is open to the public. Worked example from its README: the directory lists 221 partners in Germany, so a run with `countries: ["de"]` returns all of them for about 221 x US$0.005 = US$1.11 at its measured contact rate.

**Researcher Email Finder** searches PubMed (or Europe PMC) for a topic and returns each researcher whose email is printed in a paper, with affiliation, ORCID and the papers that carried the address. Worked example from its README: three topics with `maxResearchers: 150` cost 150 x US$0.005 = US$0.75.

### Who uses them for what

- **Sales and lead generation:** a list of company domains from a CRM export or a directory becomes rows of emails, phones and LinkedIn pages.
- **CRM enrichment:** filling in missing contact fields and checking that published details are still current.
- **Influencer and sponsorship outreach:** search YouTube by niche, filter by subscriber range and country, keep only channels with a public email.
- **Agency, partner and developer outreach:** Shopify agencies by country, app developers by niche, and the leadership of a target account list.
- **Life-science sales and recruiting:** researchers on a topic, with the email they published and their affiliation.
- **AI agents:** an agent asked to "find the contact email for these companies" calls one tool and gets structured rows with sources it can cite.

### Python quick start

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
    print("  company: ", item.get("companyName"))
    print("  emails:  ", [e["value"] for e in item.get("emails") or []])
    print("  phones:  ", [p["value"] for p in item.get("phones") or []])
    print("  linkedin:", [s["url"] for s in (item.get("socials") or {}).get("linkedin", [])])
```

Three sites cap this run at three billable items, at most about US$0.006. Each row has a `status` of `ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl` or `error`; only rows where at least one email was found are charged, and rows with only phones, socials or a contact form are free.

### Use from an AI agent through Apify MCP

Claude, Cursor and other MCP clients can call these Actors through Apify's hosted MCP server. Add this URL as a remote MCP server, which preloads the Actor as a tool (add more Actors to the list, separated by commas, such as `datagleaner/youtube-channel-contacts` or `datagleaner/team-page-contacts`):

```text
https://mcp.apify.com?tools=datagleaner/website-contact-details-scraper
```

Sign in with OAuth when the client opens a browser, or send the header `Authorization: Bearer YOUR_APIFY_TOKEN`. For a client that runs local MCP servers, Apify's package works too:

```json
{
  "mcpServers": {
    "actors-mcp-server": {
      "command": "npx",
      "args": ["-y", "@apify/actors-mcp-server"],
      "env": { "APIFY_TOKEN": "YOUR_APIFY_TOKEN" }
    }
  }
}
```

Then ask in plain words, for example: "Find the contact email, phone number and LinkedIn page for appier.com, stripe.com and sakura.ad.jp, in a table with the page each was found on."

## Caveats

- **Published addresses are mostly role inboxes.** Expect `info@` and `sales@` more often than named people. Some sites publish no email at all, only a contact form; Website Contact Details Scraper reports contact forms and social profiles, and YouTube Channel Contacts reports a channel's links and socials, so those rows are still useful. Team-page and researcher results are people, so the legal basics below apply with more force.
- **Some sites refuse automated requests.** A site that answers 403 or 429 is reported as `unreachable` and not charged. Neither the script above nor our Actors are built to get around a site that refuses.
- **No JavaScript rendering in our Actors.** Contact details inserted only by scripts are not found; a headless browser is the fix if you need them.
- **The law applies to what you do with the list.** Emails can be personal data, whichever tool found the address. See the legal basics below.

## Legal basics: GDPR and CAN-SPAM

This is general information, not legal advice.

- **GDPR (EU and UK).** A work email that names a person (`jane.doe@company.com`) is personal data even when it is published. You need a lawful basis to store and use it, usually legitimate interest for business-to-business outreach, and you must tell the person where you got their data and let them object. Some countries, such as Germany, require prior consent before marketing email even to businesses.
- **CAN-SPAM (US).** Cold commercial email is allowed, but each message needs an honest sender and subject line, your postal address and a working opt-out, and you must honour opt-outs within 10 business days.
- **Other rules.** Canada (CASL) and Australia (Spam Act) generally require consent first, and some sites forbid collecting their contact details in their terms of use.

In practice: collect only what you need, keep the source page for each address (it answers "where did you get my email?"), verify addresses before sending, and keep an unsubscribe list.

## FAQ

**How do I find email addresses from a list of websites for free?**
Use a short script like the one above: fetch each site's home and contact pages and match `mailto:` links and email patterns. It is free and works for small lists; add link-following and de-obfuscation as your list grows. Under about 20 sites, checking by hand is fine: look at the footer, the contact, about and team pages, the Impressum or legal notice (German and Austrian law requires one, and it usually lists an email), and the privacy policy, then search the page source for `mailto:`.

**Is there an API to find emails from a website?**
Yes, two kinds. Email-finder APIs such as Hunter return addresses from their database and pattern guesses. Scraper APIs such as the Apify Actors above fetch the site live and return only what it publishes, with the page it came from.

**How do I extract emails from multiple websites at once?**
Pass the whole list in one call. Website Contact Details Scraper takes thousands of domains per run, works through them in parallel and returns one row per site, exportable as CSV, Excel or JSON.

**Can I find an email address from a website link without visiting every page?**
A tool still has to fetch the pages, but it only needs a few: the home page, contact, about, legal notice and team pages hold most published addresses. That is why the default here is 8 pages per site.

**Can I find all email addresses on a domain?**
Only the ones that are published or that a finder tool has seen elsewhere. A scraper returns what the site shows; Hunter-style tools add addresses from their database and guesses from the company's email pattern. No tool can list every mailbox on a domain.

**Do I need a proxy or a login?**
No login. For a normal list, no proxy either; the Actors run without one by default.

## Related

- [Find a YouTube channel's email](guides/find-youtube-channel-email)
- [How to extract emails from a website for free](guides/extract-emails-from-website-free)
- [Get all URLs from a sitemap in Python](guides/get-all-urls-from-sitemap-python), useful for building the site list in the first place

<!-- jsonld:auto -->
<script type="application/ld+json">
[
  {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    "mainEntity": [
      {
        "@type": "Question",
        "name": "How do I find email addresses from a list of websites for free?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Use a short script like the one above: fetch each site's home and contact pages and match mailto: links and email patterns. It is free and works for small lists; add link-following and de-obfuscation as your list grows. Under about 20 sites, checking by hand is fine: look at the footer, the contact, about and team pages, the Impressum or legal notice (German and Austrian law requires one, and it usually lists an email), and the privacy policy, then search the page source for mailto:."
        }
      },
      {
        "@type": "Question",
        "name": "Is there an API to find emails from a website?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Yes, two kinds. Email-finder APIs such as Hunter return addresses from their database and pattern guesses. Scraper APIs such as the Apify Actors above fetch the site live and return only what it publishes, with the page it came from."
        }
      },
      {
        "@type": "Question",
        "name": "How do I extract emails from multiple websites at once?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Pass the whole list in one call. Website Contact Details Scraper takes thousands of domains per run, works through them in parallel and returns one row per site, exportable as CSV, Excel or JSON."
        }
      },
      {
        "@type": "Question",
        "name": "Can I find an email address from a website link without visiting every page?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "A tool still has to fetch the pages, but it only needs a few: the home page, contact, about, legal notice and team pages hold most published addresses. That is why the default here is 8 pages per site."
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
        "name": "Do I need a proxy or a login?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "No login. For a normal list, no proxy either; the Actors run without one by default."
        }
      }
    ]
  },
  {
    "@context": "https://schema.org",
    "@type": "ItemList",
    "itemListElement": [
      {
        "@type": "ListItem",
        "position": 1,
        "item": {
          "@type": "SoftwareApplication",
          "name": "Company Contact Details & Website Email Finder",
          "url": "https://apify.com/datagleaner/website-contact-details-scraper",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Emails, phones (normalized to E.164), LinkedIn, X, Facebook, Instagram, YouTube, TikTok and GitHub links, contact forms, schema.org addresses, company name and description.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.002",
              "priceCurrency": "USD",
              "description": "US$0.002 per website with an email",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.002",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "website"
                }
              }
            }
          ]
        }
      },
      {
        "@type": "ListItem",
        "position": 2,
        "item": {
          "@type": "SoftwareApplication",
          "name": "YouTube Channel Contacts",
          "url": "https://apify.com/datagleaner/youtube-channel-contacts",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Public emails from the description, About links, link-in-bio page and the creator's website, plus socials, subscribers, videos and country.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.015",
              "priceCurrency": "USD",
              "description": "US$0.015 per channel with at least one public email",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.015",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "channel"
                }
              }
            }
          ]
        }
      },
      {
        "@type": "ListItem",
        "position": 3,
        "item": {
          "@type": "SoftwareApplication",
          "name": "App Developer Email Finder",
          "url": "https://apify.com/datagleaner/app-developer-email-finder",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Developer email (tagged by where it was found), website, address, phone, privacy policy, installs or users, rating and developer socials.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.0015",
              "priceCurrency": "USD",
              "description": "US$0.0015 per app with a developer email",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.0015",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "app"
                }
              }
            }
          ]
        }
      },
      {
        "@type": "ListItem",
        "position": 4,
        "item": {
          "@type": "SoftwareApplication",
          "name": "Company Leadership & Team Page Scraper",
          "url": "https://apify.com/datagleaner/team-page-contacts",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Every named person on the company's own team, about, leadership and Impressum pages, with job title, seniority, LinkedIn link and published email; it never guesses an email.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.002",
              "priceCurrency": "USD",
              "description": "US$0.002 per person",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.002",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "person"
                }
              }
            }
          ]
        }
      },
      {
        "@type": "ListItem",
        "position": 5,
        "item": {
          "@type": "SoftwareApplication",
          "name": "Shopify Partner & Agency Contacts",
          "url": "https://apify.com/datagleaner/shopify-partner-contacts",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Agencies and experts from the Shopify Partner Directory with public email, phone, website, location, services, rating, reviews, price range and tier.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.005",
              "priceCurrency": "USD",
              "description": "US$0.005 per partner with an email or phone",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.005",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "partner"
                }
              }
            }
          ]
        }
      },
      {
        "@type": "ListItem",
        "position": 6,
        "item": {
          "@type": "SoftwareApplication",
          "name": "Researcher Email Finder",
          "url": "https://apify.com/datagleaner/researcher-email-finder",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "Researchers with the email they published in PubMed papers, plus affiliation, country, ORCID and recent papers.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "0.005",
              "priceCurrency": "USD",
              "description": "US$0.005 per researcher with an email",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "0.005",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1,
                  "unitText": "researcher"
                }
              }
            }
          ]
        }
      }
    ]
  }
]
</script>
