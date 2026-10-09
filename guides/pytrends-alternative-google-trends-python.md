---
title: "Pytrends Alternative: Google Trends Data in Python (2026)"
description: "A pytrends alternative for Google Trends in Python: why pytrends returns 429 errors, Google's official Trends API alpha, free libraries and paid APIs."
---

# Pytrends alternatives for getting Google Trends data in Python

pytrends, the unofficial Python library most people use for Google Trends, is archived: its GitHub repository is read-only and the last release on PyPI (4.9.2) dates from April 2023. It still installs, but it often fails with HTTP 429 (too many requests) errors. The alternatives are: Google's official Trends API (still a closed alpha you have to apply for), a maintained open-source library such as trendspyg, a hosted API that bills per search such as SerpApi, or a pay-per-result scraper on Apify. Which one fits depends on how many keywords you track and whether you can wait for alpha access. The sections below compare them and show code for each.

## What happened to pytrends

- **Status.** The [GeneralMills/pytrends](https://github.com/GeneralMills/pytrends) repository is archived, so no fixes will be merged. When Google changes the Trends front end, pytrends will not be updated to match.
- **429 errors.** pytrends calls the same internal endpoints the trends.google.com website uses. Google limits those by IP address, and a loop over many keywords soon gets `TooManyRequestsError` (HTTP 429). Cloud servers and shared IPs hit the limit faster than a home connection.
- **Five terms per comparison.** pytrends compares at most 5 terms in one request (the limit of the classic Explore page; Google's redesigned Explore page now allows up to 8), and Trends values are scaled 0 to 100 within each request. Values from two separate requests are not on the same scale.

If you only need a few terms now and then, pytrends with pauses and retries can still work:

```python
# pip install pytrends
import time

from pytrends.exceptions import TooManyRequestsError
from pytrends.request import TrendReq

pytrends = TrendReq(hl="en-US", tz=0)

def interest_over_time(terms, timeframe="today 12-m", geo="US", tries=5):
    for attempt in range(tries):
        try:
            pytrends.build_payload(terms, timeframe=timeframe, geo=geo)
            return pytrends.interest_over_time()
        except TooManyRequestsError:
            time.sleep(60 * (attempt + 1))  # wait longer after each 429
    raise RuntimeError("Google kept returning 429")

df = interest_over_time(["python", "rust", "go"])
print(df.tail())
```

Because the library is archived, expect this to break without warning at some point.

## The options compared

| Option | Cost | Access | Terms per comparison | Good for |
|---|---|---|---|---|
| pytrends | Free | `pip install` | 5 | Occasional small pulls; archived, frequent 429s |
| trends.google.com website | Free | Browser, CSV download per chart | Up to 8 (5 on classic Explore) | A few terms by hand |
| Google Trends API (official) | Not published | Closed alpha, by application | Dozens, per Google | Long-term monitoring, if you are accepted |
| trendspyg (open source) | Free | `pip install trendspyg` | 2 to 5 | A maintained drop-in for pytrends code; still rate-limited by Google |
| SerpApi Google Trends API | Free plan with 250 searches a month; paid plans from $25 a month for 1,000 searches | API key | Up to 5 (interest over time) | Teams already on SerpApi for other Google data |
| Google Trends Scraper on Apify (ours) | $1.50 per 1,000 results | Apify account and API token | Any number, rescaled to one scale | Large keyword lists, scheduled runs |

### Google's official Trends API (alpha)

Google announced an official [Google Trends API](https://developers.google.com/search/apis/trends) in 2025, and it is still an alpha with access by application. Google says it is giving access to a limited group of developers with a clear use case who can start soon and give feedback. According to Google's page, the API covers a rolling window of the last 5 years, aggregates by day, week, month or year, breaks data down by country and sub-region, uses consistently scaled values, so you can add new periods without pulling the whole history again, and compares dozens of terms at once. If you have a long-term project, apply; if you need data this week, plan for one of the other routes in the meantime.

### trendspyg and other open-source libraries

[trendspyg](https://github.com/flack0x/trendspyg) is a maintained open-source library on PyPI (version 1.9.0, October 2026) that describes itself as a pytrends alternative. It covers Trending Now, interest over time, related queries, interest by region and 2 to 5 keyword comparisons, and ships a pytrends-compatible `TrendReq` (`from trendspyg.compat.request import TrendReq`, installed with `pip install "trendspyg[analysis]"`), so existing code may need few changes; its documentation says `related_topics()` and `top_charts()` are not available that way. For Explore data it drives a real Chrome browser against the Trends website, so it needs Chrome installed and Google's rate limits still apply: its docs advise waiting after a rate-limit error.

### SerpApi

SerpApi's Google Trends API (`engine=google_trends`) returns interest over time and compared breakdown by region for up to 5 queries per request, and interest by region, related topics and related queries for one query per request. It bills by searches per month across all its engines: a free plan with 250 searches, then $25 for 1,000, $75 for 5,000 and $150 for 15,000 (prices from [serpapi.com/pricing](https://serpapi.com/pricing) in October 2026). Cached searches are not counted.

```python
# pip install requests
import os

import requests

params = {
    "engine": "google_trends",
    "q": "python,rust,go",
    "data_type": "TIMESERIES",
    "geo": "US",
    "date": "today 12-m",
    "api_key": os.environ["SERPAPI_KEY"],
}
data = requests.get("https://serpapi.com/search", params=params, timeout=60).json()
for point in data["interest_over_time"]["timeline_data"][-3:]:
    print(point["date"], [(v["query"], v["extracted_value"]) for v in point["values"]])
```

## Comparing more than 5 keywords on one scale

pytrends, trendspyg and SerpApi compare at most 5 terms per request, and the website at most 8. To compare 20 or 50 terms on one scale, put one anchor term in every group of up to 5, then rescale each group so the anchor matches its value in the first group:

1. Pick an anchor with steady, non-zero interest (for example the most popular term in your list).
2. Split the other terms into groups of 4 and add the anchor to each group.
3. For each group, multiply every value by `anchor_peak_in_group_1 / anchor_peak_in_this_group`.

```python
def rescale(groups, anchor):
    """groups: list of {term: [values...]} dicts, each containing the anchor."""
    base = max(groups[0][anchor])
    combined = {}
    for group in groups:
        peak = max(group[anchor])
        factor = base / peak if peak else None
        for term, values in group.items():
            if term == anchor and combined.get(anchor):
                continue
            combined[term] = [v * factor for v in values] if factor else values
    return combined
```

If the anchor has zero interest in a group, that group cannot be rescaled. Also keep in mind that Trends values are a sampled, relative index, not search counts, so the same request can return slightly different numbers on different days.

## Our option: Google Trends Scraper on Apify

Disclosure: Data Gleaner is us. Our Google Trends Scraper is an Actor on the Apify Store that does the batching, rescaling and 429 handling above for you. It is not public on the Store yet; when it is listed it will be at [apify.com/datagleaner](https://apify.com/datagleaner). <!-- TODO(store-link: google-trends-scraper) -->

What it does, from its README:

- Takes any number of `searchTerms`, splits them into groups of up to 5 that share the first term as anchor, and rescales every group to the first group's scale. The unscaled number is kept in `rawValue`.
- Returns interest over time, interest by region (country, region, city or DMA), related queries and related topics (top and rising), for web, YouTube, News, Images or Shopping search, plus a Trending Now mode for a country.
- Paces requests (`requestDelaySeconds`, default 1.5 seconds) and backs off and retries on HTTP 429 (`maxRetries`, default 6). One failing term does not fail the run.
- Adds summary columns next to each timeline: `average`, `peakValue`, `peakDate`, `latestValue`, `topRegion`, `topQuery`.
- Needs no Google account or Google API key, only an Apify account.

Pricing is $1.50 per 1,000 results, where a result is one dataset item per term, location and output type. For example, 40 terms in 2 countries with interest over time and related queries is 160 results, about $0.24. Speed is still bounded by Google's rate limits: in testing, a 40-term, 2-country, 3-output batch (264 requests) took about 14 minutes.

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

This input returns 5 results (one timeline per term), about $0.0075.

## Which one to choose

- **A few terms, occasionally:** the website's CSV download, or pytrends or trendspyg with pauses between requests.
- **Long-term monitoring with time to wait:** apply for Google's official Trends API alpha.
- **Already paying for SerpApi:** use its Google Trends engine; remember each request counts toward your monthly searches.
- **Dozens of keywords compared on one scale, on a schedule:** a hosted scraper that handles batching and retries, such as ours, or your own code following the anchor method above.

## FAQ

**Is pytrends still working?**
It installs and can still return data, but the project is archived and its last release is from April 2023. It often fails with 429 errors, and when Google changes its Trends endpoints nobody will fix it.

**How do I fix the pytrends 429 error?**
Slow down. Wait between requests (a minute or more after a 429), retry with growing pauses, and send fewer requests per run. Running from a cloud server makes 429s more likely than from a home connection. There is no setting that removes Google's rate limit.

**Is there an official Google Trends API?**
Yes, but only as an alpha. Google is accepting applications from developers and has not announced pricing or a date for general availability.

**Is there a free pytrends alternative?**
Yes. The trends.google.com website (with CSV downloads), open-source libraries such as trendspyg, and SerpApi's free plan of 250 searches a month are all free. The free libraries are still rate-limited by Google.

**Can I get more than 5 keywords in Google Trends?**
The redesigned trends.google.com Explore page compares up to 8, but pytrends and similar tools are limited to 5 per request. For more, compare them in groups of up to 5 that share one anchor term, then rescale each group to the anchor, as shown above.

## Related pages

- [Google Hotels, Trends and News scraper APIs](../google-data-scrapers)
- [How to scrape Google News with Python](google-news-scraper-python)
