---
title: Website Contact & Email Scrapers Compared: Price per 1,000 Sites
description: What 1,000 and 10,000 websites cost on eight Apify contact scrapers and Hunter, Snov.io and Skrapp, with current prices, limits and who each suits. Fetched 11 Oct 2026.
---

# Website contact and email scrapers compared: price per 1,000 sites

Reading the emails off 1,000 company websites costs between about US$2 and US$12 on the Apify contact scrapers we compared, and between US$39 and US$49 on the cheapest monthly plan of an email-finder service such as Hunter, Snov.io or Skrapp. The gap comes from how each tool counts: some charge per page crawled, some per website, one per email, and the finder services per credit on a monthly plan. This page puts them on one scale, 1,000 and 10,000 websites, using list prices we fetched on 11 October 2026.

Disclosure: Data Gleaner, the publisher of this site, makes one of the scrapers compared (row 8 in the first table). We say where the others are better. Where a number is our assumption, it is marked as one.

## Apify Store Actors: cost of 1,000 and 10,000 websites

All prices are the BRONZE tier (the Apify Starter plan), read from the Apify Store API on 11 October 2026. Higher plans get discounts on most of these. Platform usage is included in pay-per-event prices, so these are the whole bill apart from any per-run start fee.

| # | Actor | Unit price (BRONZE) | 1,000 sites | 10,000 sites | Billed per |
|---|---|---|---|---|---|
| 1 | [betterdevsscrape/contact-details-extractor](https://apify.com/betterdevsscrape/contact-details-extractor) | $0.001 | $3.00 | $30 | Page crawled (HTTP) |
| 2 | [poidata/contact-details-scraper](https://apify.com/poidata/contact-details-scraper) | $0.002 | $2.00 | $20 | Result (one per website) |
| 3 | [rp_openpro.ai/contact-details-scraper](https://apify.com/rp_openpro.ai/contact-details-scraper) | $0.002 | $2.00 (+$0.05 per run) | $20 (+$0.05 per run) | Domain |
| 4 | [vdrmota/contact-info-scraper](https://apify.com/vdrmota/contact-info-scraper) | $0.002 | $6.00 | $60 | Page crawled |
| 5 | [thenetaji/website-email-scraper](https://apify.com/thenetaji/website-email-scraper) | $0.004 | $12.00 | $120 | Item (one per page by default) |
| 6 | [maximedupre/website-emails-scraper](https://apify.com/maximedupre/website-emails-scraper) | $0.0058 | $5.80 | $58 | Unique email saved |
| 7 | [delicious_zebu/contact-info-scraper](https://apify.com/delicious_zebu/contact-info-scraper) | $0.008 | $8.00 | $80 | Dataset item |
| 8 | [datagleaner/website-contact-details-scraper](https://apify.com/datagleaner/website-contact-details-scraper) | $0.002 | up to $2.00 | up to $20 | Website with an email found |

How the 1,000-site figures were worked out (our assumptions):

- **Per-page Actors (rows 1, 4, 5):** 3 pages per site, which is the home page plus a contact page plus an about page. Row 4 allows up to 20 pages per start URL by default, so its range is $2 to $40 per 1,000 sites. Row 1 has a 20-page default cap as well ($1 to $20). Row 5 outputs one item per crawled page and defaults to crawl depth 2.
- **Per-email Actor (row 6):** one email per site on average. A site with three addresses is billed for three.
- **Per-item Actor (row 7):** `summary_only` mode, one row per website. In its default mode it also writes one row per page, which costs more.
- **Per-website Actors (rows 2, 3):** one result per website. Row 2's README does not say whether a website with no contact found is billed.
- **Row 8:** the $2.00 is the ceiling, reached only if every site yields an email. Websites with no email, phones or socials only, or an error, cost nothing, so the real bill is $2 multiplied by your hit rate. It crawls up to 8 pages per site by default (1 to 30) at no extra charge.

Per-run start fees are small: row 3 charges $0.05 per run, row 1 $0.001 per run, row 4 $0.001 per GB of memory.

## What each one returns, and who it suits

| # | Actor | Returns | Users / rating (11 Oct 2026) | Best for |
|---|---|---|---|---|
| 1 | betterdevsscrape | Emails, phones, 25+ social profiles, addresses; optional SEO, form and business-info add-ons at +$0.001 per page each | 671 users, 5.0 (1 review) | Lowest per-page price when you want extra page data too |
| 2 | poidata | Emails, phones, socials, company details, contact names; handles JavaScript pages | 379 users, no reviews | Simple per-website pricing from a newer Actor |
| 3 | rp_openpro.ai | Emails, 14 socials, MX check, a 0 to 100 lead score, optional WHOIS and LinkedIn enrichment; 5 pages per domain by default | 320 users, 5.0 (1 review) | Lead scoring and email checks included in one price |
| 4 | vdrmota (Apify's most-used contact scraper) | Emails, phones, social links; optional browser mode, social-profile scraping and paid lead enrichment with verification | 62,605 users, 4.7 (92 reviews) | Hard JavaScript sites, and anything you want proven at scale |
| 5 | thenetaji | Emails, phones, socials per page; stealth and Cloudflare solving | 1,803 users, 4.4 (4 reviews) | Sites that block simple crawlers |
| 6 | maximedupre | Unique emails only, with MX check | 1,246 users, 5.0 (2 reviews) | A clean email-only list where you pay for results, not pages |
| 7 | delicious_zebu | Emails with deliverability checks (MX, role mailbox), phones, 14 socials; HTTP first, browser when needed; one row per company | 886 users, 5.0 (29 reviews) | Checked emails and a tidy one-row-per-company file |
| 8 | Data Gleaner | Emails (own-domain only; other-domain addresses kept apart), phones in E.164, own social accounts, contact-form pages, schema.org addresses, company name; the page each value came from | 3 users, no reviews | Big lists where many sites have no email, and multilingual contact pages (Japanese, Chinese, German, French, Spanish, Italian) |

Where the others are better than ours, plainly:

- **No JavaScript rendering.** Ours uses plain HTTP. Details drawn only by scripts, or on sites that return 403 or 429 to non-browser clients, are missed. Rows 4, 5 and 7 can use a browser.
- **No track record.** Three users and no reviews, against 62,605 users and 92 reviews for vdrmota. If a scraper failing on one site costs you more than a few dollars, start with the proven one.
- **No verification or scoring.** Rows 3, 4 and 7 can check whether an address is deliverable. Ours returns what the site publishes.
- **Per-page prices can win.** If your sites hold their email on the home page, rows 1 and 4 cost $1 to $2 per 1,000 sites, which beats $2 per site with an email.

## Email-finder services: what 1,000 and 10,000 lookups cost

Hunter, Snov.io and Skrapp do a different job. They search their own databases and verify addresses, so they can return a named person's address that no website publishes. They bill in monthly credits, so the cost of a batch is the plan you must buy. Apollo is left out of the maths because its own pricing page lists no prices or credit amounts (checked 11 October 2026); third-party sites report US$49 to US$119 per user per month on annual billing with conflicting credit counts.

| Service | Plan that covers 1,000 lookups | Plan that covers 10,000 lookups | Cheapest effective rate | Free plan |
|---|---|---|---|---|
| [Hunter](https://hunter.io/pricing) | Starter, $49 a month ($34 billed yearly), 2,000 credits | Growth, $149 a month ($104 billed yearly), 10,000 credits | $8.36 per 1,000 credits (Scale, yearly) | 50 credits a month |
| [Snov.io](https://snov.io/pricing) | Starter, $39 a month, 1,000 credits | Pro M, $189 a month, 20,000 credits | $7.38 per 1,000 (Pro L and Ultra) | 50 credits a month |
| [Skrapp](https://www.skrapp.io/pricing) | Professional, $39 a month ($29 billed yearly), 2,000 credits | Enterprise, $349 a month ($262 billed yearly), 50,000 credits | $5.24 per 1,000 (Enterprise, yearly) | 50 credits a month |

Assumptions: one credit per website, which is Hunter's rule for a Domain Search that finds one email (it charges one credit per email found) and what Snov.io's table implies. A domain with many emails uses more credits on Hunter. Unused monthly credits generally expire. These are best cases for the finders, and they still cost several times the Apify scrapers for plain contact pages, because a finder is selling a database, verification and outreach tools, not page reading.

Use a finder when you need a person (the head of marketing at a company), a verified address, or sequencing and CRM features. Use a website scraper when you need what the company publishes: its info@, sales@ and press@ inboxes, phones and social links, for thousands of sites, with no subscription.

## Method

- **Prices:** Apify prices come from `https://api.apify.com/v2/store?search=contact%20details` (also `email scraper` and `website email`), field `currentPricingInfo`, BRONZE tier event prices, on 11 October 2026. Each Actor's README was read for what it bills and its defaults. Hunter, Snov.io and Skrapp prices come from their pricing pages on the same date. Prices change; each link above goes to the live page.
- **Selection:** the Actors with the most users among those whose main job is contact details or emails from a list of websites. We left out Google Maps, YouTube, Instagram and Facebook-specific Actors, and Actors under 300 users, apart from our own.
- **Not measured:** we did not run the competitors on a common list, so hit rates, accuracy and speed are not compared. Only list prices and documented behavior are.
- **Corrections:** if a price here is wrong or has changed, the live page wins. Tell us and we will fix it.

## Try ours on three real examples

[Company Contact Details & Website Email Finder](https://apify.com/datagleaner/website-contact-details-scraper) costs $0.002 per site with an email and nothing otherwise. A new Apify account includes free monthly credit, which is about 2,500 sites at this rate (the Free plan's US$5). Three example tasks run with the input already filled in:

- [Find emails from company websites](https://apify.com/datagleaner/website-contact-details-scraper/examples/find-emails-company-websites)
- [SaaS company contact details](https://apify.com/datagleaner/website-contact-details-scraper/examples/saas-company-contact-details)
- [Taiwan company phones and emails](https://apify.com/datagleaner/website-contact-details-scraper/examples/taiwan-company-phones-emails)

Test it and one of the Actors above on the same 50 sites. The one that finds more of the addresses you need is the cheaper one.

## FAQ

**What is the cheapest website email scraper?**
On list prices, betterdevsscrape/contact-details-extractor at $0.001 per page and poidata, rp_openpro.ai and Data Gleaner at $0.002 per website or per site with an email come to $2 to $3 per 1,000 sites. The real cost depends on pages crawled and how many sites yield nothing, so test on your own list.

**How much does it cost to scrape emails from 10,000 websites?**
About $20 to $120 on the Apify Actors compared here (our assumptions: 3 pages per site for per-page Actors), and $149 to $349 for one month of Hunter, Snov.io or Skrapp credits, as of 11 October 2026.

**Is an email finder like Hunter better than a website scraper?**
It is better for a named person's address and for verification, because it draws on its own database. A website scraper is cheaper for company inboxes and contact pages across a large list, but it only returns what each site publishes.

**Which Apify contact scraper is the most proven?**
vdrmota/contact-info-scraper, with 62,605 users and 4.7 stars from 92 reviews on 11 October 2026. It also offers a browser mode for JavaScript sites.

**Why do per-page and per-website prices differ so much?**
A per-page Actor bills each page it opens, and a thorough crawl of a site can open 20. A per-website Actor bills once per site however many pages it reads. Compare the cost per site, not the unit price.

**Do I pay when no email is found?**
It depends on the Actor. Data Gleaner's Actor charges only for sites where an email was found. Per-page Actors charge for every page opened, including empty ones.

## Related

- [Find emails from a list of websites (API and Python)](../contact-and-lead-scrapers)
- [How to extract emails from a website for free](extract-emails-from-website-free)
- [Find emails for Google Maps leads from the website column](enrich-google-maps-leads-with-emails)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What is the cheapest website email scraper?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "On list prices, betterdevsscrape/contact-details-extractor at $0.001 per page and poidata, rp_openpro.ai and Data Gleaner at $0.002 per website or per site with an email come to $2 to $3 per 1,000 sites. The real cost depends on pages crawled and how many sites yield nothing, so test on your own list."
      }
    },
    {
      "@type": "Question",
      "name": "How much does it cost to scrape emails from 10,000 websites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "About $20 to $120 on the Apify Actors compared here (our assumptions: 3 pages per site for per-page Actors), and $149 to $349 for one month of Hunter, Snov.io or Skrapp credits, as of 11 October 2026."
      }
    },
    {
      "@type": "Question",
      "name": "Is an email finder like Hunter better than a website scraper?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It is better for a named person's address and for verification, because it draws on its own database. A website scraper is cheaper for company inboxes and contact pages across a large list, but it only returns what each site publishes."
      }
    },
    {
      "@type": "Question",
      "name": "Which Apify contact scraper is the most proven?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "vdrmota/contact-info-scraper, with 62,605 users and 4.7 stars from 92 reviews on 11 October 2026. It also offers a browser mode for JavaScript sites."
      }
    },
    {
      "@type": "Question",
      "name": "Why do per-page and per-website prices differ so much?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "A per-page Actor bills each page it opens, and a thorough crawl of a site can open 20. A per-website Actor bills once per site however many pages it reads. Compare the cost per site, not the unit price."
      }
    },
    {
      "@type": "Question",
      "name": "Do I pay when no email is found?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It depends on the Actor. Data Gleaner's Actor charges only for sites where an email was found. Per-page Actors charge for every page opened, including empty ones."
      }
    }
  ]
}
</script>
