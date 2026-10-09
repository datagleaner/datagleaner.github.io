---
title: "Google Trends API: Official Alpha, Free and Paid Options"
description: "Is there a Google Trends API? The official alpha (who gets access, what it returns), free unofficial routes, SerpApi and a pay-per-result scraper API, compared."
---

# Google Trends API: what exists in 2026 and how to get access

Google does have an official Google Trends API, but it is an alpha that you apply for, and most people who apply will not have access yet. Until you do, the working routes are the free Trends website and RSS feed, the unofficial JSON endpoints behind the website (which the archived pytrends library used), and paid APIs such as SerpApi or a pay-per-result scraper on Apify. This page explains what the official API returns, how it differs from the website's numbers, and which option fits which job.

## The official Google Trends API (alpha)

Google announced the Trends API on 24 July 2025 and opened applications the same day through Google Search Central. The page to apply is [developers.google.com/search/apis/trends](https://developers.google.com/search/apis/trends). As of October 2026 it is still an alpha with no general availability date and no published pricing or quotas.

**Who can get access.** Google says it gives priority to applicants who know what they want to build, can start soon and will give direct feedback. Google's announcement names researchers, journalists and developers as the audience. There is no self-serve key: you apply and wait.

**What it returns**, according to Google's API page and announcement:

- Search interest for your terms over a rolling window of about the last five years (1,800 days), up to about two days before the request.
- Daily, weekly, monthly or yearly aggregation.
- Breakdowns by country and sub-region, using ISO 3166-2 codes.
- Values that are scaled consistently across requests, not rescaled to 0 to 100 on every request as the website does. That means you can fetch only the newest days for a term you monitor and join them to what you already have, and compare dozens of terms, where the website's Explore page stops at eight.

**What it does not cover**, as far as Google has published: absolute search counts (it is still an interest index), Trending Now (the live list of trending searches), and related queries or related topics, which Google's API page does not mention. Data older than the five-year window, which the website shows back to 2004, is also outside it.

If your use case fits and you can wait, apply. Approved alpha access is the only route Google itself supports.

## Free routes while you wait

### 1. The Google Trends website

[trends.google.com](https://trends.google.com/trends/explore) shows interest over time, interest by region, related topics and related queries for up to 8 terms on the Explore page Google redesigned in 2026 (5 on the older layout), and each chart has a download button for a CSV. This is enough for a one-off comparison. It gets slow when you need many terms, many countries or a weekly refresh, because every chart is a separate download and each comparison is scaled so its own peak is 100.

To compare more terms than one chart holds, keep one term (an anchor) in every group, then rescale each group so the anchor's peak matches its peak in the first group. Without the anchor, numbers from different groups are on different scales and cannot be compared.

### 2. The Trending Now RSS feed

Google publishes the current trending searches for a country as RSS, with no key:

```
https://trends.google.com/trending/rss?geo=US
```

Each item has the search, an approximate traffic bucket (such as `200+`), the publish time and related news links. In Python:

```python
# pip install feedparser
import feedparser

feed = feedparser.parse("https://trends.google.com/trending/rss?geo=US")
for entry in feed.entries:
    print(entry.title, entry.get("ht_approx_traffic"), entry.published)
```

The feed covers only what is trending now. It has no history and no interest-over-time data.

### 3. The unofficial endpoints behind the website (and pytrends)

The Trends website loads its charts from undocumented JSON endpoints on trends.google.com: an `explore` call returns a token per chart, and `widgetdata` calls (`multiline`, `comparedgeo`, `relatedsearches`) return the timeline, regions and related queries for that token. The responses start with a short junk prefix (`)]}'`) that you strip before parsing. pytrends, the Python library most tutorials use, wraps these endpoints.

Caveats before you build on them:

- They are not an API Google supports. They are undocumented, have no terms of their own, and change without notice.
- pytrends has been archived on GitHub and is no longer maintained, so fixes for those changes no longer land there.
- Google rate-limits these endpoints by IP. A loop of requests soon gets HTTP 429 responses, so pace requests (a second or more apart) and back off when you get one.
- You still get the website's per-request 0 to 100 scaling and the 5-term limit per comparison, so the anchor method above applies.

For a small script run now and then, this is free and works. For anything a business depends on, plan for it to break.

## Paid Google Trends APIs

### SerpApi

[SerpApi's Google Trends API](https://serpapi.com/google-trends-api) returns the website's data as JSON through a documented, keyed API. Its `data_type` parameter selects interest over time (`TIMESERIES`), interest by region (`GEO_MAP`, `GEO_MAP_0`), related topics and related queries, and it has a separate Trending Now engine. Up to 5 terms per request for timelines and region maps; related queries and topics take one term per request.

It is billed per search from a monthly plan: a free tier of 250 searches a month, then plans from $25 a month for 1,000 searches ([pricing](https://serpapi.com/pricing), checked October 2026). The search quota is shared with SerpApi's other Google engines. Comparing more than 5 terms is up to you: you run several requests and rescale them yourself.

### Google Trends Scraper on Apify (ours)

Disclosure: Data Gleaner is us. Our Google Trends Scraper is an Actor on the Apify Store that you call over the Apify API, from Python or Node.js, or from an AI agent. It needs an Apify account and token, not a Google account or Google API key. It is not public on the Store yet; when it is, it will be listed at [apify.com/datagleaner](https://apify.com/datagleaner). <!-- TODO(store-link: google-trends-scraper) -->

What it returns, one dataset item per term, location and output:

- Interest over time, interest by region (country, region, city or DMA), related queries and related topics (top and rising), and Trending Now for a country (4, 24, 48 or 168-hour window).
- Any number of terms in one run: more than 5 are split into groups of up to 5 that share the first term as anchor, and each group is rescaled to the first group's scale. The unscaled value is kept in `rawValue`.
- Summary columns next to the arrays: `average`, `peakValue`, `peakDate`, `latestValue`, `topRegion`, `topQuery`.
- Web, YouTube, News, Image or Shopping search, any category ID, preset or custom date ranges back to 2004.

It reads the same public Trends data as the website, so the numbers are the website's relative index, not the official API's consistently scaled values. It paces requests and backs off on HTTP 429; a large batch can take several minutes.

Pricing is $1.50 per 1,000 results, charged per dataset item. For example, 40 terms in 2 countries with interest over time and related queries is 160 results, about $0.24. You can cap the charge for any run.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-trends-scraper").call(run_input={
    "mode": "search",
    "searchTerms": ["python", "javascript", "rust", "go", "kotlin"],
    "geo": ["US"],
    "timeRange": "today 12-m",
    "outputs": ["interestOverTime"],
})
if run is None:
    raise SystemExit("[ERROR] The Actor run did not return.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item.get("searchTerm"), item.get("average"), item.get("peakValue"), item.get("peakDate"))
```

This returns 5 results, under $0.01. Add `"interestByRegion"`, `"relatedQueries"` or `"relatedTopics"` to `outputs` for the other data, or set `"mode": "trendingNow"` for the trending list.

## Which Google Trends API to use

| Option | Access | Cost | Terms per comparison | Related queries | Trending Now | History |
|---|---|---|---|---|---|---|
| Official Trends API (alpha) | Apply and wait for approval | Not published | Dozens, consistently scaled | Not mentioned by Google | No | About 5 years |
| Trends website + CSV | Anyone | Free | Up to 8 | Yes | Yes (on the site) | Back to 2004 |
| Trending Now RSS | Anyone | Free | n/a | Related news only | Yes | None |
| Unofficial endpoints / pytrends | Anyone, unsupported | Free | 5 per request | Yes | Yes | Back to 2004 |
| SerpApi | API key | Free 250 searches/month, then from $25/month | 5 per request | Yes | Yes | Back to 2004 |
| Google Trends Scraper (Apify, ours) | Apify token | $1.50 per 1,000 results | Any number, anchor-rescaled | Yes | Yes | Back to 2004 |

A rough rule:

- **Long-running research or a product built on Trends, and you can wait:** apply for the official alpha. Its consistent scaling removes the anchor workaround.
- **A few terms, now and then:** the website and its CSV download.
- **Just what is trending today:** the RSS feed.
- **A script you control, and you accept it may break:** the unofficial endpoints, paced.
- **Many keywords or countries on one scale, on a schedule, as JSON:** a paid API. SerpApi suits per-request access with a monthly plan; our Actor suits large keyword batches billed per result.

## Caveats for every option

- **Trends is an index, not search volume.** Even the official API returns search interest, not counts. Trending Now traffic figures are rounded buckets.
- **Website-style values move between requests.** Google samples its data, so the same query can return slightly different numbers on different days, and a term's values depend on what it is compared with.
- **Low-volume terms and regions return zeros or nothing.** Google withholds data where there are too few searches.
- **Terms of use.** Only the official API comes with Google's own terms for programmatic use. Check that your use of any other route fits Google's terms and the law that applies to you.

## FAQ

**Is there an official Google Trends API?** Yes, since July 2025, but only as an alpha. You apply on Google's Trends API page and Google picks testers; there is no self-serve key, published price or general availability date yet.

**Is the Google Trends API free?** Google has not published pricing for the alpha. The free routes today are the Trends website's CSV downloads, the Trending Now RSS feed and the unofficial endpoints that pytrends used. Paid options charge per search or per result.

**How do I get access to the Google Trends API?** Apply through the form on developers.google.com/search/apis/trends with a concrete use case. Google prioritizes applicants who can start soon and give feedback. While you wait, use one of the other routes on this page.

**What are the Google Trends API limits?** Google has not published quotas for the alpha. Its data covers about the last five years at daily, weekly, monthly or yearly granularity. The unofficial endpoints have no published limits either, but Google returns HTTP 429 when requests come too fast from one IP.

**Is there a Google Trends API for Python?** The official alpha does not have a public Python library yet. pytrends is archived. You can call SerpApi or our Actor from Python with their clients, as in the example above, or parse the RSS feed with feedparser.

## Related guides

- [Google Hotels API: options for hotel prices](google-hotels-api)
- [Google Ads Transparency Center API](google-ads-transparency-center-api)
- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp)
- [Pytrends alternatives for Google Trends in Python](pytrends-alternative-google-trends-python)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there an official Google Trends API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, since July 2025, but only as an alpha. You apply on Google's Trends API page and Google picks testers; there is no self-serve key, published price or general availability date yet."
      }
    },
    {
      "@type": "Question",
      "name": "Is the Google Trends API free?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Google has not published pricing for the alpha. The free routes today are the Trends website's CSV downloads, the Trending Now RSS feed and the unofficial endpoints that pytrends used. Paid options charge per search or per result."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get access to the Google Trends API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Apply through the form on developers.google.com/search/apis/trends with a concrete use case. Google prioritizes applicants who can start soon and give feedback. While you wait, use one of the other routes on this page."
      }
    },
    {
      "@type": "Question",
      "name": "What are the Google Trends API limits?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Google has not published quotas for the alpha. Its data covers about the last five years at daily, weekly, monthly or yearly granularity. The unofficial endpoints have no published limits either, but Google returns HTTP 429 when requests come too fast from one IP."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a Google Trends API for Python?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The official alpha does not have a public Python library yet. pytrends is archived. You can call SerpApi or our Actor from Python with their clients, as in the example above, or parse the RSS feed with feedparser."
      }
    }
  ]
}
</script>
