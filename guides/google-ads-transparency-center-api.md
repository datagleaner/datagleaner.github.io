---
title: "Google Ads Transparency Center API: Your Options in 2026"
description: "There is no official Google Ads Transparency Center API. How to get the data anyway: the free BigQuery dataset (EEA and Turkey), the website, and scraper APIs."
---

# Google Ads Transparency Center API

Google does not offer an official API for the Google Ads Transparency Center (adstransparency.google.com). There is no endpoint in the Google Ads API for it and no developer key to request. You have three ways to get the data in bulk: Google's free public dataset in BigQuery, which covers ads shown in the European Economic Area (EEA) and Turkey; the website itself, which covers every region but has no export; and third-party scraper APIs that read the website and return JSON. Which one fits depends on whether you need ads outside Europe and whether you need impression counts.

## The options at a glance

| Route | Cost | Regions | What you get | Main limit |
|---|---|---|---|---|
| BigQuery public dataset `google_ads_transparency_center` | Free dataset; you pay normal BigQuery query costs (a free monthly query allowance applies) | Ads served in the EEA and Turkey | Advertiser, creative ID and page link, ad format, topic, first and last shown dates per region, impression ranges, audience-targeting categories, removed ads with the policy reason | No ads that ran only outside the EEA and Turkey; no creative images or ad text |
| BigQuery `google_political_ads` | Same as above | Regions where Google publishes a political ads report | Election ads with advertiser, dates, spend and impression ranges | Political ads only |
| adstransparency.google.com website | Free | All regions Google lists | Every public ad for an advertiser or domain, with the creative rendered | One page at a time, no export button |
| Scraper APIs (SerpApi, SearchApi, Apify Actors including ours) | Paid per search or per result | All regions the website covers | The website's data as JSON | Unofficial: depends on the website, which can change |

## Option 1: the free BigQuery dataset

Google publishes two tables as the public dataset `bigquery-public-data.google_ads_transparency_center`, and also as JSON files you can download (the README is at `storage.googleapis.com/ads-transparency-center/api-data/README.txt`). Both are under Google's Terms of Service and the Ads Transparency Center Additional Terms.

- **`creative_stats`**: one row per ad. Columns include `advertiser_id`, `advertiser_disclosed_name`, `advertiser_legal_name`, `advertiser_location`, `advertiser_verification_status`, `creative_id`, `creative_page_url`, `ad_format_type`, `topic`, `ad_funded_by`, `is_funded_by_google_ad_grants`, a nested `region_stats` record (region code, `first_shown`, `last_shown`, `times_shown_lower_bound` and `times_shown_upper_bound`) and `audience_selection_approach_info` (whether demographics, location, contextual signals, customer lists or topics of interest were used to target the ad).
- **`removed_creative_stats`**: ads Google removed, with a `disapproval` record holding the violated policy, the violation category, the removal location and whether the decision came from a Google investigation or a legal notice.

The catch is coverage. `region_stats` lists only the regions in the EEA and Turkey where the ad served, because the dataset exists for the EU's Digital Services Act reporting. An ad that ran only in the US or Japan is not in it. Dates are also floored: an ad first shown before 1 March 2023 reports 1 March 2023 as `first_shown`.

To query it, open the BigQuery console with any Google Cloud project (the BigQuery sandbox works without a billing card) and run something like this, which lists one advertiser's ads shown in France:

```sql
SELECT
  creative_id,
  advertiser_disclosed_name,
  ad_format_type,
  r.first_shown,
  r.last_shown,
  r.times_shown_lower_bound,
  r.times_shown_upper_bound,
  creative_page_url
FROM `bigquery-public-data.google_ads_transparency_center.creative_stats`,
  UNNEST(region_stats) AS r
WHERE advertiser_disclosed_name LIKE '%Nike%'
  AND r.region_code = 'FR'
ORDER BY r.last_shown DESC
LIMIT 100;
```

The table is large, so select only the columns you need: BigQuery bills by the bytes each query scans, and `SELECT *` on this table uses up a free allowance quickly. Check the Schema tab in the console for the full column list before you write longer queries. Users on Google's developer forum have reported that the dataset can show fewer ads for an advertiser than the website does, so spot-check a few advertisers against the site.

For political ads, use the separate dataset `bigquery-public-data.google_political_ads`, which backs Google's Political Advertising transparency report and includes spend ranges. It holds election ads only.

## Option 2: the website, by hand

adstransparency.google.com needs no login. Search an advertiser name or a domain, then filter by region, date range, platform and format. Each advertiser has a stable ID starting with `AR` in its page URL (`adstransparency.google.com/advertiser/AR...`), and each ad an ID starting with `CR`. This is the only free route to ads shown outside Europe.

It is fine for looking at a few competitors. It does not scale: there is no export, the ad list loads as you scroll, and outside the EEA and Turkey the site shows dates and formats but no impression counts or targeting details.

## Option 3: scraper APIs

Several services read the public website and return the results as JSON, which is what most people searching for an "API" end up using. SerpApi and SearchApi both offer a Google Ads Transparency Center engine billed per search on their own plans. On the Apify Store, Actors do the same and bill per result.

Disclosure: Data Gleaner is us. Our **Google Ads Transparency Scraper** takes advertiser IDs, domains or company names and returns one row per ad: advertiser ID and name, creative ID, format (text, image or video), first and last shown dates, creative image URLs or a preview URL, the landing domain for domain searches, and the ad's Transparency Center link. With `includeDetails` on, it also fetches every creative variant and the regions where each ad ran, at one extra request per ad. Filters cover region (two-letter country code or `anywhere`), start and end date, and format. It needs no Google account. It costs $1.50 per 1,000 ads, so a test run capped at 10 ads is about $0.015, and you can set a maximum charge on any run.

It is not on the Apify Store yet. Until it is, see [apify.com/datagleaner](https://apify.com/datagleaner) for our listed Actors. <!-- TODO(store-link: google-ads-transparency-scraper) -->

Once it is listed, a run from Python looks like this:

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-ads-transparency-scraper").call(
    run_input={"domains": ["nike.com"], "region": "US", "maxAdsPerAdvertiser": 10}
)
if run is None:
    raise SystemExit("The run did not start.")
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item.get("advertiserName"), "|", item.get("format"), "|", item.get("lastShown"), "|", item.get("adUrl"))
```

Other input fields: `advertiserIds` (the `AR...` IDs), `searchTerms` (company names, resolved to the best-matching advertisers, `maxAdvertisersPerTerm` up to 10), `startDate` and `endDate` (YYYY-MM-DD), `format` (`any`, `text`, `image`, `video`) and `includeDetails`. It can also run from an AI agent through Apify's MCP server; see [web scraping for AI agents over MCP](web-scraping-for-ai-agents-mcp).

What it does not return: impression ranges and audience-targeting categories (use the BigQuery dataset for those in the EEA and Turkey), video files, and ad copy as text.

## Caveats that apply to every route

- **Text ads are archived as images.** The Transparency Center stores most text and image ads as rendered images on `tpc.googlesyndication.com`, so no route gives you the headline and description as plain text. If you need the words, run OCR on the images. Video and some other formats are available only as Google's preview script for that creative.
- **No spend for commercial ads.** Spend ranges exist only in the political ads data. Impressions exist only as lower and upper bounds, and only for the EEA and Turkey.
- **Region and date matter.** An ad's first and last shown dates differ by region, and the website and scrapers return ads newest first, so filter by region and date before comparing counts.
- **Unofficial means it can change.** Any route that reads the website depends on how Google serves it. Pace requests; Google throttles a single IP that sends many quickly.

## FAQ

**Is there an official Google Ads Transparency Center API?**
No. The Google Ads API does not cover it. The closest official source is the free BigQuery dataset `bigquery-public-data.google_ads_transparency_center`, which covers ads shown in the EEA and Turkey.

**What is in the Google Ads Transparency Center BigQuery dataset?**
Two tables: `creative_stats` (one row per ad, with advertiser, format, topic, first and last shown dates and impression ranges per region, and targeting categories) and `removed_creative_stats` (removed ads with the policy reason). It does not include creative images or ad text, and only covers the EEA and Turkey.

**How do I find a Google advertiser ID?**
Search the advertiser on adstransparency.google.com and copy the part of the page URL that starts with `AR`. A scraper that accepts company names, such as ours via `searchTerms`, resolves the ID for you.

**Can I download all of a competitor's Google ads?**
For ads shown in Europe, query the BigQuery dataset by `advertiser_disclosed_name` or `advertiser_id` and export the result. For other regions the website has no download, so use a scraper API with the competitor's domain or advertiser ID.

## Related guides

- [Google Trends API: free routes and their limits](google-trends-api)
- [Google Hotels API: getting hotel prices as JSON](google-hotels-api)
- [Web scraping for AI agents over MCP](web-scraping-for-ai-agents-mcp)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there an official Google Ads Transparency Center API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. The Google Ads API does not cover it. The closest official source is the free BigQuery dataset bigquery-public-data.google_ads_transparency_center, which covers ads shown in the EEA and Turkey."
      }
    },
    {
      "@type": "Question",
      "name": "What is in the Google Ads Transparency Center BigQuery dataset?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Two tables: creative_stats (one row per ad, with advertiser, format, topic, first and last shown dates and impression ranges per region, and targeting categories) and removed_creative_stats (removed ads with the policy reason). It does not include creative images or ad text, and only covers the EEA and Turkey."
      }
    },
    {
      "@type": "Question",
      "name": "How do I find a Google advertiser ID?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Search the advertiser on adstransparency.google.com and copy the part of the page URL that starts with AR. A scraper that accepts company names, such as ours via searchTerms, resolves the ID for you."
      }
    },
    {
      "@type": "Question",
      "name": "Can I download all of a competitor's Google ads?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "For ads shown in Europe, query the BigQuery dataset by advertiser_disclosed_name or advertiser_id and export the result. For other regions the website has no download, so use a scraper API with the competitor's domain or advertiser ID."
      }
    }
  ]
}
</script>
