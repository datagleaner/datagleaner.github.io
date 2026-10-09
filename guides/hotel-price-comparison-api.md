---
title: "Hotel Price Comparison API: Booking vs Agoda vs Expedia as JSON"
description: "Booking.com, Expedia and Agoda APIs are partner-only. How to get hotel prices from many booking sites as JSON via Google Hotels, with Python and an agent example."
---

# Hotel price comparison API: Booking.com vs Agoda vs Expedia prices as JSON

There is no open hotel price comparison API that returns Booking.com, Expedia and Agoda prices side by side. Each of those companies offers its own API only through an approved partner or affiliate program, and that API returns that company's own rates, not its competitors'. The practical way to get several booking sites' prices for the same hotel and dates as JSON is Google Hotels, which shows a list of providers with a price each. Google has no public API for reading that list, so you read it with a SERP API or a hosted scraper. Below: why the OTA APIs do not fit comparison, free Python that turns provider offers into a cheapest-provider and rate-parity table, and a pay-per-result scraper you can call from Python or an AI agent.

Disclosure: Data Gleaner, mentioned further down as one option, is us. The analysis code in the first half works on data from any source and needs no account.

## Why the OTA APIs do not work for comparison

Online travel agencies (OTAs) such as Booking.com, Expedia and Agoda each run a developer or partner program. The programs differ, but the pattern is the same:

- **Access is approved, not self-serve.** You apply as an affiliate or connectivity partner and are accepted or refused. Booking.com's Demand API is for approved affiliate and connectivity partners; Expedia's Rapid API requires a commercial agreement.
- **One company's data only.** Each API returns that company's own availability and prices. To compare three OTAs you would need three separate partner agreements, and you would still be comparing whatever each one lets you display.
- **The terms are built for sending bookings, not for comparing.** Affiliate programs exist to send travellers to book with that company. Whether a price comparison product is allowed is set by each program's terms, which you must read and agree to. We do not quote them here, because they change and differ by partner type.
- **Third-party "hotel price comparison APIs" are aggregators.** Some, such as MakCorps (the API in a popular GeeksforGeeks tutorial), return prices from several booking sites for a hotel. They sell subscription plans with a trial; AlternativeTo lists MakCorps at $1,200 to $2,600 a month while G2 shows no pricing at all, so ask the vendor for a current quote. A fixed subscription is a lot to carry for a small project or a one-off analysis.

If you are building a booking product, the partner programs are the right route and you should apply. If you only need to compare what several booking sites charge for the same dates, you want a feed that already aggregates them.

## Google Hotels as a comparison feed

Google Hotels (google.com/travel/hotels) shows, for a hotel and your dates, a list of booking sites with a price each. Which sites appear depends on the market and the hotel: one hotel may show Booking.com and Agoda, another may show Trip.com and the hotel's own site, and a given OTA may not appear at all. That variability is a limit of the method, and the cheapest or highest price you find is only about the providers Google lists.

Google does not publish an endpoint for reading these provider offers. Its documented Hotel APIs and Hotel Center are for hotels and OTAs that send their own prices to Google. For the longer comparison of SERP APIs, the Places API and scrapers, see [Google Hotels API: hotel prices as JSON](google-hotels-api).

What you can read from a Google Hotels feed, per hotel and per set of dates, is:

- the lowest price,
- each provider's name, price per night, total for the stay and a booking link,
- rating, review count, star class and location.

That is enough to build the two tables people usually want: who is cheapest, and how far apart the providers are.

## Free: turn provider offers into a cheapest-provider and rate-parity table

Rate parity means a hotel charging the same price on every channel. In a comparison feed you can measure how close to parity a hotel is by looking at the spread between its cheapest and most expensive provider. A feed shows the price Google lists for a stay, not a guarantee that the room type, cancellation terms or taxes are identical across providers, so treat a spread as a signal to check, not as proof of a parity breach.

This script is plain Python with the standard library only. It reads a JSON file of hotel records, picks the cheapest provider per hotel and computes the spread. Without a file it runs on one real record (shortened to two offers), so you can run it as is:

```python
import json
import sys

# One real record from a Paris search, shortened to two offers.
SAMPLE = [{
    "name": "Le Tsuba Hotel",
    "currency": "USD",
    "nights": 2,
    "offers": [
        {"provider": "Bluepillow.tw", "pricePerNight": 185.28, "totalPrice": 370.56},
        {"provider": "Amimir.com", "pricePerNight": 193.7, "totalPrice": 387.4},
    ],
}]


def parity_row(hotel):
    offers = [o for o in hotel.get("offers") or [] if o.get("pricePerNight") is not None]
    if not offers:
        return None  # no availability for these dates
    offers.sort(key=lambda o: o["pricePerNight"])
    cheapest, dearest = offers[0], offers[-1]
    spread = dearest["pricePerNight"] - cheapest["pricePerNight"]
    return {
        "hotel": hotel["name"],
        "providers": len(offers),
        "cheapest": cheapest["provider"],
        "cheapest_night": cheapest["pricePerNight"],
        "dearest": dearest["provider"],
        "dearest_night": dearest["pricePerNight"],
        "spread": round(spread, 2),
        "spread_pct": round(100 * spread / cheapest["pricePerNight"], 1),
        "currency": hotel.get("currency", ""),
    }


if len(sys.argv) > 1:
    with open(sys.argv[1], encoding="utf-8") as f:
        hotels = json.load(f)
else:
    hotels = SAMPLE
rows = [r for r in map(parity_row, hotels) if r]

print("hotel | providers | cheapest | per night | dearest | per night | spread | %")
for r in sorted(rows, key=lambda r: -r["spread_pct"]):
    print(f"{r['hotel']} | {r['providers']} | {r['cheapest']} | {r['cheapest_night']} | "
          f"{r['dearest']} | {r['dearest_night']} | {r['spread']} {r['currency']} | {r['spread_pct']}%")
```

On the sample record it prints one row: Bluepillow.tw is cheapest at 185.28 USD a night, Amimir.com is the dearest at 193.7, a spread of 8.42 USD or 4.5 percent. Run it with a path to a JSON file of records (for example a dataset exported from the scraper below) and you get one row per hotel, biggest spread first.

The resulting table looks like this, with your own hotels in the rows:

| Hotel | Providers | Cheapest | Per night | Dearest | Per night | Spread |
|---|---|---|---|---|---|---|
| Le Tsuba Hotel | 2 | Bluepillow.tw | 185.28 USD | Amimir.com | 193.7 USD | 8.42 USD (4.5%) |

Limits of this approach: it compares per-night prices as Google lists them, a hotel with only one provider has no spread to measure, and providers whose names differ slightly across markets (for example a regional domain) count as separate rows unless you normalise the names yourself.

## The data source: options for getting provider offers

| Route | What it gives you | Cost model | Catch |
|---|---|---|---|
| OTA partner APIs (Booking.com, Expedia, Agoda) | That OTA's own rates | Set by each program | Approval needed, one company per API, terms built for booking |
| Aggregator APIs such as MakCorps | Prices from several booking sites | Subscription plans, quoted on request | Fixed recurring cost |
| SERP APIs with a Google Hotels endpoint (SerpApi, DataForSEO and others) | Google Hotels results, with per-provider prices on a property request | Per search or per block of results | Each property lookup counts as its own request |
| Your own headless-browser scraper | Anything on the page | No fees | Undocumented responses change, you maintain the parser, and Google's terms limit automated access |
| A hosted Google Hotels scraper | One JSON record per hotel with provider offers | Per result | Providers listed depend on the market |

The [track hotel prices guide](track-hotel-prices-google-hotels) covers the do-it-yourself and SERP routes in more detail.

## Data Gleaner Google Hotels Scraper as one option

Disclosure: Data Gleaner is us.

The [Google Hotels Scraper](https://apify.com/datagleaner/google-hotels-scraper) is an Apify Actor (Apify's name for a hosted scraper) that reads Google Hotels results for the places or hotels and dates you give it. It returns one JSON record per hotel with the lowest price for your dates and every booking site's per-night price, total price and link, such as Booking.com, Agoda, Trip.com and Hostelworld where Google lists them. It works over plain HTTP, with no browser and no Google account.

Pricing: **$3.00 per 1,000 hotels** ($0.003 per hotel), billed only for hotels written to the dataset, with no monthly plan. A 10-hotel test run costs $0.03, and 5 cities at 100 hotels each costs $1.50. You need an Apify account and your API token.

Input fields used for comparison:

- `locations`: free text such as `Tokyo`, `Paris 8e` or `hostels in Berlin`.
- `hotelUrls`: Google Hotels hotel URLs or an `entityId` from an earlier run, to compare one hotel repeatedly.
- `checkIn` and `checkOut`: `YYYY-MM-DD`; prices are for exactly these dates.
- `adults`: 1 to 8.
- `currency`: an ISO code such as `USD`, `EUR` or `JPY`.
- `maxHotelsPerLocation`: default 20.
- `includeDetails`: must stay on (the default) to get the provider offers; off gives a faster list with only the lowest price.
- `countryCode`: the Google market (`us`, `gb`, `de`); providers and deals can differ by market.

This script runs the Actor, then applies the same cheapest-provider and spread logic to the results:

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-hotels-scraper").call(run_input={
    "locations": ["Shinjuku, Tokyo"],
    "checkIn": "2026-11-10",
    "checkOut": "2026-11-12",
    "adults": 2,
    "currency": "USD",
    "countryCode": "us",
    "maxHotelsPerLocation": 10,
    "includeDetails": True,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for hotel in client.dataset(run.default_dataset_id).iterate_items():
    offers = [o for o in hotel.get("offers") or [] if o.get("pricePerNight") is not None]
    if not offers:
        print(hotel["name"], "- no prices for these dates")
        continue
    offers.sort(key=lambda o: o["pricePerNight"])
    low, high = offers[0], offers[-1]
    print(f"{hotel['name']}: cheapest {low['provider']} {low['pricePerNight']} {hotel['currency']}, "
          f"{len(offers)} providers, spread {round(high['pricePerNight'] - low['pricePerNight'], 2)}")
```

Move the dates to a future stay: check-in must be today or later, with check-out after check-in. A hotel with no availability on your dates still comes back, with `lowestPrice: null` and empty `offers`, and the script above reports it as having no prices. To track parity over time, schedule the same run daily; every record carries `scrapedAt`.

Limits worth knowing before you rely on it:

- Providers depend on the market and the hotel, so Expedia, Booking.com or Agoda will not always appear. Check the `provider` values rather than assuming a particular OTA is present.
- Google lists around 500 distinct hotels per query at most. For a large city, search neighbourhoods separately.
- Prices are what Google Hotels shows for your dates, currency and market. Booking links go through a Google redirect, so use them to send a user to book, not as stable identifiers.
- At very high volume Google may start refusing requests. The Actor backs off and finishes what it has, and has an optional proxy setting.
- Read Google's terms and each provider's terms for how you may use and republish the data.

## Call it from an AI agent or MCP client

Claude, Cursor and other MCP clients can call the Actor through Apify's hosted MCP server at `https://mcp.apify.com`. Adding `?tools=datagleaner/google-hotels-scraper` preloads it as a ready tool. In Claude Code:

```bash
claude mcp add --transport http apify "https://mcp.apify.com?tools=datagleaner/google-hotels-scraper"
```

Run `/mcp` inside Claude Code to finish the browser sign-in. For clients that read an `mcpServers` config:

```json
{
  "mcpServers": {
    "apify": { "url": "https://mcp.apify.com?tools=datagleaner/google-hotels-scraper" }
  }
}
```

Then ask in plain language:

> Find 10 hotels in Shinjuku, Tokyo for 2 adults from November 10 to 12, prices in USD. For each hotel list the cheapest booking site and the most expensive one, and flag hotels where the gap is more than 5 percent.

The agent runs the Actor, reads the hotels from the dataset and does the cheapest-provider and spread comparison itself from the `offers` array. For a repeatable pipeline, prefer the Python scripts above and give the agent the finished table, so the arithmetic is not left to the model. The general setup for other clients and ways to cap an agent's spend are in [Web scraping MCP server for AI agents](web-scraping-for-ai-agents-mcp).

## FAQ

**Is there a Booking.com API for price comparison?**
Booking.com offers its Demand API to approved affiliate and connectivity partners, and it returns Booking.com's own availability and prices. It does not return Expedia or Agoda prices, and whether you may use it for a comparison product depends on the terms you agree to as a partner.

**Can I get Expedia and Agoda prices through an API?**
Expedia's Rapid API needs a commercial agreement with Expedia. For Agoda, check its affiliate and partner program for current access. Both return that company's own data. To see their prices next to each other you need an aggregator or Google Hotels data, where they appear only if Google lists them for that hotel and market.

**Is there a free hotel price comparison API?**
Not a lasting one: aggregator APIs that return several booking sites' prices are paid subscriptions, sometimes with a trial. Affiliate programs can be free to join if approved, but they return one company's data. SerpApi has a free plan of 250 searches a month, and Apify gives new accounts a small monthly free credit that covers small test runs.

**How do I check rate parity across booking sites?**
Collect each provider's price for the same hotel, dates, guests, currency and market, then compare the cheapest with the most expensive. The script in this guide prints that spread per hotel. Confirm that room type, cancellation terms and taxes match before calling a difference a parity breach.

**Why is the same hotel priced differently on different sites?**
Providers can show different rates because of room type, cancellation rules, taxes and fees, member discounts, and the market the search is made from. Set the same dates, guests, currency and `countryCode` when you compare, and treat a gap as something to check.

## Related guides

- [Google Hotels API: hotel prices as JSON](google-hotels-api): the full comparison of SERP APIs, the Places API and scrapers.
- [How to track hotel prices on Google Hotels](track-hotel-prices-google-hotels): run the same search on a schedule and watch prices move.
- [Web scraping MCP server for AI agents](web-scraping-for-ai-agents-mcp): set up Claude or Cursor to run scrapers as tools.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there a Booking.com API for price comparison?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Booking.com offers its Demand API to approved affiliate and connectivity partners, and it returns Booking.com's own availability and prices. It does not return Expedia or Agoda prices, and whether you may use it for a comparison product depends on the terms you agree to as a partner."
      }
    },
    {
      "@type": "Question",
      "name": "Can I get Expedia and Agoda prices through an API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Expedia's Rapid API needs a commercial agreement with Expedia. For Agoda, check its affiliate and partner program for current access. Both return that company's own data. To see their prices next to each other you need an aggregator or Google Hotels data, where they appear only if Google lists them for that hotel and market."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a free hotel price comparison API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not a lasting one: aggregator APIs that return several booking sites' prices are paid subscriptions, sometimes with a trial. Affiliate programs can be free to join if approved, but they return one company's data. SerpApi has a free plan of 250 searches a month, and Apify gives new accounts a small monthly free credit that covers small test runs."
      }
    },
    {
      "@type": "Question",
      "name": "How do I check rate parity across booking sites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Collect each provider's price for the same hotel, dates, guests, currency and market, then compare the cheapest with the most expensive. The script in this guide prints that spread per hotel. Confirm that room type, cancellation terms and taxes match before calling a difference a parity breach."
      }
    },
    {
      "@type": "Question",
      "name": "Why is the same hotel priced differently on different sites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Providers can show different rates because of room type, cancellation rules, taxes and fees, member discounts, and the market the search is made from. Set the same dates, guests, currency and countryCode when you compare, and treat a gap as something to check."
      }
    }
  ]
}
</script>
