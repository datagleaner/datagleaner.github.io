---
title: "Google Hotels API: How to Get Hotel Prices as JSON"
description: Google has no public Google Hotels API for prices. Here are the options that do work, from the Places API to SerpApi and scrapers, with Python and sample JSON.
---

# Google Hotels API: how to get Google Hotels prices as JSON

Google does not offer a public Google Hotels API for reading hotel prices. The Hotel APIs and Hotel Center that Google documents are for hotels and booking sites that send their own prices *to* Google, not for developers who want to read prices *out of* Google Hotels. If you need Google Hotels data in code, you have four practical routes: the Google Places API (hotel names, ratings and locations, no prices), a SERP API that returns Google Hotels results (SerpApi, DataForSEO, HasData and others), a booking-supplier API such as Booking.com's for affiliate partners, or a hosted scraper you call through an API. This page explains what each one gives you, what it costs and where it falls short.

## What Google officially offers

| Google product | Who it is for | Does it give you hotel prices? |
|---|---|---|
| Hotel Center and the Hotel APIs (Travel Partner API, price feeds) | Hotels, chains and online travel agencies that advertise on Google | No. You push your own rates and inventory to Google and read reports on your own listings. Access needs an approved partner account. |
| Google Places API (Text Search, Place Details) | Any developer with a Google Cloud API key | No. It returns hotels as places: name, address, coordinates, rating, review count, photos, website. There is no nightly price and no list of booking sites. |
| Google Hotels website (google.com/travel/hotels) | Travellers | Yes, for your dates, but only in the web interface. There is no documented endpoint. |

So a search for a "Google Hotels API key" usually ends at the Places API. That is the right tool if you only need a list of hotels in an area with ratings. It is the wrong tool for prices.

### Free option: hotels without prices from the Places API

If ratings and locations are enough, the Places API (New) Text Search can return hotels for a query. You need a Google Cloud project with billing enabled and an API key; Google bills Places requests per call after a monthly free allowance, so check the current Places pricing page before running it at volume.

```bash
curl -X POST "https://places.googleapis.com/v1/places:searchText" \
  -H "Content-Type: application/json" \
  -H "X-Goog-Api-Key: YOUR_GOOGLE_API_KEY" \
  -H "X-Goog-FieldMask: places.displayName,places.formattedAddress,places.rating,places.userRatingCount,places.location" \
  -d '{"textQuery": "hotels in Shinjuku, Tokyo", "includedType": "lodging"}'
```

You get names, addresses, ratings and coordinates. You do not get availability, prices or the booking sites Google Hotels compares.

## Options for Google Hotels prices

### 1. SERP APIs with a Google Hotels endpoint

Several search-result API companies return Google Hotels results as JSON for a query and dates:

- **SerpApi Google Hotels API** (`engine=google_hotels`). You pass a query, `check_in_date`, `check_out_date`, adults and currency, and get properties with rates, ratings, amenities and, on a property request, prices by booking site. It is billed per search from a shared monthly plan: at the time of writing SerpApi lists a free plan with 250 searches a month and paid plans from $25 a month for 1,000 searches. Each page of results and each property detail lookup is its own search.
- **DataForSEO Google Hotels API**. A Hotel Search endpoint billed per block of 20 results and a Hotel Info endpoint billed per hotel. It is pay-as-you-go against a prepaid balance.
- **HasData, ScrapingBee and similar**. Credit-based APIs that return Google Hotels listings with prices; check each one's docs for which fields and how many credits a request uses.

These suit you if you already use one of these services for Google search results, or you want an immediate synchronous response per request.

### 2. Booking and supplier APIs

If what you actually need is bookable inventory rather than Google's comparison view, go to a supplier:

- **Booking.com Demand API** is for approved Booking.com affiliate and connectivity partners. It returns Booking.com's own availability and prices, not other sites' prices.
- **Amadeus hotel APIs**. Amadeus paused new registrations for its free Self-Service developer portal, which included Hotel Search, and shut the portal down on July 17, 2026, disabling its API keys. Hotel content is now available only through the Amadeus Enterprise portal under a commercial agreement.
- **Hotel bed banks and channel managers** (Hotelbeds, Expedia Rapid and others) require a commercial agreement.

These give you real bookable rates from one supplier. They do not show you what Google Hotels shows, which is the price across many booking sites side by side.

### 3. Do it yourself

You can load Google Hotels in a headless browser and parse the page yourself. It costs nothing in fees, but the page is built by JavaScript from undocumented responses that change without notice, prices depend on dates, currency, market and language, and you maintain the parser. Google's terms of service also limit automated access; read them and decide for your use case. Keep the request rate low.

### 4. A hosted Google Hotels scraper on Apify

Disclosure: Google Hotels Scraper is ours (Data Gleaner).

[Google Hotels Scraper](https://apify.com/datagleaner/google-hotels-scraper) is an Apify Actor that you call through the Apify API with places or hotel URLs and dates. It returns one JSON record per hotel with the lowest price for your dates, every booking site's per-night and total price with its link, rating, review count, star class, highlight amenities, coordinates and, with details on, the address, website and check-in times. It runs over plain HTTP without a browser or a Google account.

It costs **$3.00 per 1,000 hotels** ($0.003 per hotel), billed per hotel returned, with no monthly plan. A 10-hotel test run costs $0.03. You need a free Apify account and its API token.

Install the client and set your token:

```bash
pip install apify-client
export APIFY_TOKEN=your_apify_token
```

Then run a search for dates 30 days out:

```python
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

The input fields are `locations` (free text such as `Paris 8e` or `hostels in Berlin`), `hotelUrls` (Google Hotels entity URLs or the `entityId` from an earlier run), `checkIn` and `checkOut` (`YYYY-MM-DD`), `adults` (1 to 8), `currency`, `maxHotelsPerLocation` (default 20, up to 1,000), `includeDetails` (default on; off gives a faster list with only the lowest price), `language` and `countryCode`.

## The JSON you get back

One record per hotel. This is a real record from a Paris search, shortened (two of its offers shown):

```json
{
  "name": "Le Tsuba Hotel",
  "entityId": "ChkI8bPIkrOozLonGg0vZy8xMWNseWdzeDVmEAE",
  "url": "https://www.google.com/travel/hotels/entity/ChkI8bPIkrOozLonGg0vZy8xMWNseWdzeDVmEAE?hl=en&gl=us&ts=...",
  "address": "45 Rue des Acacias, 75017 Paris, France",
  "latitude": 48.877756,
  "longitude": 2.2930822,
  "countryCode": "FR",
  "starClass": 4,
  "propertyType": "4-star tourist hotel",
  "rating": 4.6,
  "reviewCount": 1590,
  "amenities": ["Free Wi-Fi", "Air conditioning", "Pet-friendly", "Fitness center", "Parking"],
  "website": "http://www.tsubahotel.com/",
  "checkInTime": "3:00 PM",
  "checkOutTime": "12:00 PM",
  "lowestPrice": 185.28,
  "currency": "USD",
  "lowestPriceDisplay": "$185",
  "totalPriceDisplay": "$371",
  "pricesForDates": true,
  "offers": [
    {"provider": "Bluepillow.tw", "pricePerNight": 185.28, "priceDisplay": "$185", "totalPrice": 370.56, "totalPriceDisplay": "$371", "link": "https://www.google.com/travel/lodging/clk?..."},
    {"provider": "Amimir.com", "pricePerNight": 193.7, "priceDisplay": "$194", "totalPrice": 387.4, "totalPriceDisplay": "$387", "link": "https://www.google.com/travel/lodging/clk?..."}
  ],
  "searchLocation": "Paris 8e",
  "checkIn": "2026-11-10",
  "checkOut": "2026-11-12",
  "nights": 2,
  "adults": 2,
  "source": "search",
  "scrapedAt": "2026-10-07T15:59:05+00:00"
}
```

`lowestPrice` and each `pricePerNight` are per night; `totalPrice` covers the whole stay. `pricesForDates: true` confirms Google priced the exact dates you asked for. A hotel with no availability on those dates still comes back, with `lowestPrice: null` and empty `offers`.

## Which option to choose

| Need | Best fit |
|---|---|
| Hotel names, ratings and locations, no prices | Google Places API |
| Bookable rates from one supplier, with booking | Booking.com Demand API or another supplier, under a partner agreement |
| Google Hotels prices, occasional lookups, free tier | SerpApi free plan (250 searches a month) |
| Google Hotels prices billed per hotel, no monthly plan | Google Hotels Scraper on Apify (ours) or DataForSEO |
| Full control and no fees, and you will maintain it | Your own headless-browser scraper |

## Caveats for any route

- **Prices depend on the request.** Dates, number of guests, currency, market country and language all change what Google shows. Fix them when you compare runs over time.
- **Coverage per query is capped.** Google lists a few hundred hotels per search at most (around 500 in our testing). For a large city, search neighbourhoods separately.
- **Amenities from the result list are highlights**, not the full amenity list.
- **Booking-site links go through Google's redirect**, so they are for sending a user to book, not stable identifiers.
- **Terms of use.** Read Google's terms and each provider's terms for how you may use and republish the data.

## FAQ

**Is there a Google Hotels API?**
Not for reading prices. Google's Hotel APIs and Hotel Center are for partners that send their own prices to Google. To read Google Hotels prices you use a third-party API or scraper.

**Is there a free Google Hotels API?**
The Google Places API has a monthly free allowance, but it returns no prices. For prices, SerpApi's free plan gives 250 searches a month, and Apify gives new accounts a small monthly free credit that covers small test runs of an Actor.

**How do I get a Google Hotels API key?**
There is no key for a Google Hotels price API. A Google Cloud API key works with the Places API (no prices). For prices, you get a key from the third-party service you use, such as SerpApi or Apify.

**How do I track Google Hotels prices over time?**
Run the same search on a schedule for the same dates, guests, currency and market, and compare the lowest price or each booking site's price between runs. Apify can schedule an Actor run daily; each record carries `scrapedAt`.

**How much does a Google Hotels API cost?**
It depends on the route: per search on SerpApi (from $25 a month for 1,000 searches), per block of results or per hotel on DataForSEO, and $3.00 per 1,000 hotels on our Apify Actor.

## Related guides

- [How to track hotel prices on Google Hotels](track-hotel-prices-google-hotels)
- [Google Trends API: the options and how to use them](google-trends-api)
- [Google Ads Transparency Center API](google-ads-transparency-center-api)
- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp)
