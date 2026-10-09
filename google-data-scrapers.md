---
title: "Google Hotels, Trends and News Scraper APIs"
description: "Google Hotels, Google Trends, Google News and Ads Transparency scraper APIs: the free official routes, their limits, and pay-per-result Actors that return JSON."
---

# Google Hotels, Google Trends and Google News scraper APIs

Google has no public API for Google Hotels search results, its official Google Trends API is still a closed alpha you apply for, and Google News offers RSS feeds but no JSON API. So if you want hotel prices for your dates, Trends data for many keywords, news articles with real publisher links, or the ads a company runs, you either work with Google's free feeds and pages yourself or use a scraper API that does it and returns JSON. This page covers both for four Google sources, then lists the four Data Gleaner Actors that cover them.

## What each Google source offers for free

| Source | Official or free route | What it gives you | Where it falls short |
|---|---|---|---|
| Google Hotels | None for search results. Google's Hotel Center and Travel Partner API are for hotels and booking sites that send prices to Google. | Nothing you can query as a traveler or analyst. | No way to read prices, ratings or booking-site offers for a city and dates. |
| Google Trends | The trends.google.com website (CSV download per chart); the official [Google Trends API](https://developers.google.com/search/apis/trends) (announced in July 2025, still an alpha with access by application); the unofficial Python library pytrends. | Interest over time, by region, related queries and topics. | The website compares at most 5 terms at a time and downloads one chart at a time. Few developers get alpha access. pytrends was archived in April 2025 (read-only, no fixes) and often hits HTTP 429 from Google. |
| Google News | RSS feeds: `https://news.google.com/rss/search?q=<query>&hl=en-US&gl=US&ceid=US:en`, plus topic and section feeds. | Headline, source, publish time, a Google News link. | About 100 items per feed, no paging, and each link is an encoded `news.google.com` URL rather than the publisher's article URL. Google's feed terms limit RSS to personal, non-commercial reading. |
| Google Ads Transparency Center | The adstransparency.google.com website; Google's free BigQuery public dataset `google_ads_transparency_center`. | The website shows every public ad for an advertiser or domain, one page at a time. The dataset has advertiser, creative ID, format, first and last shown dates and impression ranges. | The website has no export. The dataset covers only ads shown in the EEA and Turkey, with no creative images or ad text. |

### Doing it yourself

- **Google News:** read the RSS feed with any feed parser (`feedparser` in Python). To get more than about 100 articles, split one query into several with Google's `when:` operator (`tesla when:6h`) or `site:` filters, and remove duplicates yourself. Turning the `news.google.com` links into publisher URLs takes extra requests per article, and Google changes that link format from time to time.
- **Google Trends:** for a handful of terms, the website's download button is enough. For more than 5 terms, the values are only comparable if every group of 5 shares one anchor term and you rescale each group to the anchor's peak. Trends values are a relative index from 0 to 100, not search counts, and they shift a little between requests because Google samples its data.
- **Google Hotels:** the only route is reading the same results a visitor sees on google.com/travel/hotels. Prices depend on the check-in and check-out dates, the number of adults, the currency and the Google market (country), so record all four with every price.
- **Ads Transparency Center:** if the ads you care about ran in Europe, query the BigQuery dataset (the BigQuery sandbox works without a billing card); see [Google Ads Transparency Center API options](guides/google-ads-transparency-center-api). For other regions the website is public and needs no login, but listing every ad for many advertisers by hand is slow.

Whatever route you take, pace your requests. Google throttles by IP, and a burst of requests gets HTTP 429 responses.

## Data Gleaner's Google data Actors

Disclosure: Data Gleaner is us. We publish these as Actors on the Apify Store: you run them from the Apify API, the Python or JavaScript clients, the Apify console or an AI agent, and pay only per result written to the dataset. They need no Google account or Google API key, only an Apify account.

| Actor | What it returns | Price | Store page |
|---|---|---|---|
| Google Hotels Scraper | One record per hotel for your dates: lowest price per night and total, every booking site's offer with its link, rating, review count, star class, amenities, address, coordinates, website, photos. Search by place or by hotel URL. | $3.00 per 1,000 hotels | [google-hotels-scraper](https://apify.com/datagleaner/google-hotels-scraper) |
| Google Trends Scraper | Interest over time, interest by region, related queries and related topics for any number of keywords (batched in groups of 5 with a shared anchor and rescaled), plus Trending Now for a country. Summary columns: average, peak value and date, latest value, top region, top query. | $1.50 per 1,000 results | Google Trends Scraper, coming soon: see [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: google-trends-scraper) --> |
| Google News Scraper | One row per article from search queries, topics or any Google News feed URL: headline, outlet, publish time, and the decoded publisher URL. Optional publisher meta (description, image, author). Any language and country edition. | $1.50 per 1,000 articles | Google News Scraper, coming soon: see [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: google-news-scraper) --> |
| Google Ads Transparency Scraper | Ads from the Google Ads Transparency Center by advertiser ID, domain or company name: advertiser, format (text, image, video), first and last shown dates, creative image URLs, landing domain, ad URL, and optionally the regions where each ad ran. | $1.50 per 1,000 ads | Google Ads Transparency Scraper, coming soon: see [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: google-ads-transparency-scraper) --> |

Some worked costs, from each Actor's pricing: 5 cities at 100 hotels each is 500 hotels, $1.50. 40 keywords in 2 countries with interest over time and related queries is 160 Trends results, about $0.24. 5 news queries at 100 articles each is $0.75. You can set a maximum charge on any run, and the Actor stops cleanly when it is reached.

## Who uses which

- **Hotel price tracking and revenue managers:** run Google Hotels Scraper daily for the same city and dates and chart each hotel's lowest price and the spread between booking sites. See [how to track hotel prices from Google Hotels](guides/track-hotel-prices-google-hotels).
- **SEO and content teams:** Google Trends Scraper for seasonality and rising related queries across a whole keyword list, compared on one scale. See [a pytrends alternative for Google Trends in Python](guides/pytrends-alternative-google-trends-python).
- **PR, media monitoring and research:** Google News Scraper for daily coverage of a company or topic in several countries and languages, with links straight to the publisher. See [scraping Google News with Python](guides/google-news-scraper-python).
- **Marketers and competitive research:** Google Ads Transparency Scraper to list a competitor's current ads, formats and creatives by domain or brand name.
- **AI agents and LLM apps:** any of the four as a tool that returns structured JSON (see the MCP section below).

## Python quick start (Google Hotels)

Google Hotels Scraper is public now. Install the client, set your Apify token, and run:

```python
# pip install apify-client
import os
from datetime import date, timedelta

from apify_client import ApifyClient

check_in = date.today() + timedelta(days=30)
check_out = check_in + timedelta(days=2)

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-hotels-scraper").call(run_input={
    "locations": ["Tokyo"],
    "checkIn": check_in.isoformat(),
    "checkOut": check_out.isoformat(),
    "adults": 2,
    "currency": "USD",
    "maxHotelsPerLocation": 10,
})
if run is None:
    raise SystemExit("[ERROR] The Actor run did not return.")

for hotel in client.dataset(run.default_dataset_id).iterate_items():
    offers = hotel.get("offers") or []
    cheapest = offers[0]["provider"] if offers else "-"
    print(f'{hotel.get("name")} | rating {hotel.get("rating")} | '
          f'{hotel.get("lowestPrice")} {hotel.get("currency")}/night | first offer: {cheapest}')
```

This returns at most 10 hotels, about $0.03. `lowestPrice` and each offer's `pricePerNight` are per night; `totalPrice` is for the whole stay. A hotel with no availability on your dates is still returned, with `lowestPrice` empty and no offers. Turn `includeDetails` off for a faster list with only the lowest price.

## Use them from AI agents (Apify MCP)

Claude, Cursor and other MCP clients can run these Actors through Apify's hosted MCP server at `https://mcp.apify.com`. Add the Actor to the URL to preload it as a tool. In Claude Code:

```bash
claude mcp add --transport http apify "https://mcp.apify.com?tools=datagleaner/google-hotels-scraper"
```

It signs in through the browser on first use (run `/mcp`), or pass `--header "Authorization: Bearer YOUR_APIFY_TOKEN"`. Swap in `google-trends-scraper`, `google-news-scraper` or `google-ads-transparency-scraper` once they are on the Store. Then ask in plain language, for example: "Find the 20 cheapest 4-star hotels in Paris 8e for 10 to 12 November, with the booking site for each price."

## Caveats

- **Google changes its pages and feeds.** Any scraper, ours included, can break when it does. Our Actors return partial rows rather than failing the run where they can (a news row with no decoded URL, a hotel with no price for your dates).
- **Rate limits are real.** Each Actor paces its requests and backs off on HTTP 429. Very large Trends batches can still take a while.
- **Trends numbers are relative.** A value of 100 is the peak of that comparison, not a search count.
- **News is recent coverage, not an archive.** Each feed stops at about 100 articles, and the Actor's widest time filter is the past year.
- **Terms and data rights.** These Actors read only public data and do not log in. Article text belongs to its publishers, so the News Actor returns links and metadata, not full articles. Check that your use fits Google's terms and the law that applies to you.

## FAQ

**Is there a free Google Trends API?** Google announced an official Google Trends API in July 2025; it is still an alpha with access by application and no published pricing. The free options are the Trends website's per-chart CSV download and unofficial libraries: pytrends, archived since April 2025 and often rate limited, or a maintained alternative such as trendspyg, which hits the same rate limits. See [Google Trends API options](guides/google-trends-api).

**Does Google Hotels have an API?** Not for reading search results. Google's hotel APIs are for hotels and booking sites that send their prices to Google. To get prices, ratings and booking-site offers for a place and dates, you need a scraper that reads the public Google Hotels results.

**Is there a Google News API?** No official JSON API. Google News publishes RSS feeds for searches and topics, limited to about 100 items each, with encoded links. A scraper API like ours wraps those feeds, decodes the links to publisher URLs and returns JSON.

**Is there a Google Ads Transparency Center API?** No official API. Google publishes a free BigQuery dataset for ads shown in the EEA and Turkey, without creative images. For other regions, the Transparency Center website is public, and a scraper can list an advertiser's ads with dates, formats and creative images.

**Do I need a Google account or API key for these Actors?** No. You need an Apify account and its API token, and you pay per result.

## Related pages

- [Track hotel prices from Google Hotels](guides/track-hotel-prices-google-hotels)
- [A pytrends alternative for Google Trends in Python](guides/pytrends-alternative-google-trends-python)
- [Google News scraper in Python](guides/google-news-scraper-python)
- [Google Hotels API: how to get hotel prices as JSON](guides/google-hotels-api)
- [Google Trends API: official alpha, free and paid options](guides/google-trends-api)
- [Google Ads Transparency Center API options](guides/google-ads-transparency-center-api)

<!-- jsonld:auto -->
<script type="application/ld+json">
[
  {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    "mainEntity": [
      {
        "@type": "Question",
        "name": "Is there a free Google Trends API?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Google announced an official Google Trends API in July 2025; it is still an alpha with access by application and no published pricing. The free options are the Trends website's per-chart CSV download and unofficial libraries: pytrends, archived since April 2025 and often rate limited, or a maintained alternative such as trendspyg, which hits the same rate limits. See Google Trends API options."
        }
      },
      {
        "@type": "Question",
        "name": "Does Google Hotels have an API?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "Not for reading search results. Google's hotel APIs are for hotels and booking sites that send their prices to Google. To get prices, ratings and booking-site offers for a place and dates, you need a scraper that reads the public Google Hotels results."
        }
      },
      {
        "@type": "Question",
        "name": "Is there a Google News API?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "No official JSON API. Google News publishes RSS feeds for searches and topics, limited to about 100 items each, with encoded links. A scraper API like ours wraps those feeds, decodes the links to publisher URLs and returns JSON."
        }
      },
      {
        "@type": "Question",
        "name": "Is there a Google Ads Transparency Center API?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "No official API. Google publishes a free BigQuery dataset for ads shown in the EEA and Turkey, without creative images. For other regions, the Transparency Center website is public, and a scraper can list an advertiser's ads with dates, formats and creative images."
        }
      },
      {
        "@type": "Question",
        "name": "Do I need a Google account or API key for these Actors?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "No. You need an Apify account and its API token, and you pay per result."
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
          "name": "Google Hotels Scraper",
          "url": "https://apify.com/datagleaner/google-hotels-scraper",
          "applicationCategory": "DeveloperApplication",
          "operatingSystem": "Web",
          "description": "One record per hotel for your dates: lowest price per night and total, every booking site's offer with its link, rating, review count, star class, amenities, address, coordinates, website, photos.",
          "publisher": {
            "@type": "Organization",
            "name": "Data Gleaner"
          },
          "offers": [
            {
              "@type": "Offer",
              "price": "3.00",
              "priceCurrency": "USD",
              "description": "US$3.00 per 1,000 hotels",
              "priceSpecification": {
                "@type": "UnitPriceSpecification",
                "price": "3.00",
                "priceCurrency": "USD",
                "referenceQuantity": {
                  "@type": "QuantitativeValue",
                  "value": 1000,
                  "unitText": "hotels"
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
