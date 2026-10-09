---
title: "Scrape Google Hotels with Python (No Browser, No Proxy)"
description: "How to scrape Google Hotels with Python over plain HTTP: per-date prices and every booking site's offer, runnable code, datacenter-IP limits and a hosted option."
---

# How to scrape Google Hotels with Python

You can scrape Google Hotels with Python using plain HTTP requests, without Selenium, a headless browser or a residential proxy. Google Hotels loads its results from an internal endpoint (`batchexecute` on `google.com/_/TravelFrontendUi`) that answers ordinary POST requests with JSON-like data, including the price for your exact check-in and check-out dates and the offer from every booking site Google lists for the hotel (Booking.com, Agoda, Trip.com and others, depending on the market). The code below uses only `requests` and the standard library. The catch is that the endpoint is undocumented, so it can change without notice, and Google may refuse requests from datacenter IPs at higher volume. Both limits are covered below.

Disclosure: Data Gleaner, mentioned at the end as one option for running this without maintaining code, is us. Everything before that section is free and needs no account.

## Why the usual tutorials use a browser and proxies

Most "scrape Google Hotels" articles you will find first come from proxy and SERP-API vendors, for example the guides from [Oxylabs](https://oxylabs.io/blog/how-to-scrape-google-hotels), [ScrapeHero](https://www.scrapehero.com/google-hotels-web-scraping/) and [Crawlbase](https://crawlbase.com/blog/how-to-scrape-google-hotels-with-python/). They load the hotels page in a real or headless browser and parse the rendered HTML, and each one points to its own product (a proxy network, a scraper API) for the blocking problems that approach brings. That route works, but it is slow and heavy: a full Google Hotels page is large, and the HTML structure changes often.

The page itself gets its data from a small number of internal calls. If you call those yourself, you skip the browser, the rendered HTML and most of the weight. In the Actor we maintain, a 100-hotel search takes about 7 requests, and full details add one request per hotel (the address, check-in times and every provider's offer).

## What you can get over plain HTTP

For each hotel, for the dates and currency you set:

- Name, an `entityId` that identifies the hotel, coordinates, star class, rating and review count.
- The lowest price for your dates.
- With the details call: every booking site's price per night and total for the stay.

Prices are specific to the dates, number of adults, currency and market country (`gl`) in the request. Change any of them and the numbers change, so fix them when you compare over time.

## Step 1: search a city and read each provider's offers

This script makes the two calls: `AtySUc` returns a page of hotels for a query like "hotels in Lisbon", and `lvkpBc` returns one hotel with its provider offers for your dates. The response is a nested array without field names, so the code reads values by position. That is the fragile part, and the reason the script is defensive about missing values.

```python
# pip install requests
import base64
import json
from datetime import date, timedelta

import requests

RPC_URL = "https://www.google.com/_/TravelFrontendUi/data/batchexecute"
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
                  "(KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36",
    "Accept-Language": "en",
    # Pre-accepted consent cookie so EU IPs are not bounced to consent.google.com
    "Cookie": "SOCS=CAESEwgDEgk0ODE3Nzk3MjQaAmVuIAEaBgiAo_CmBg",
}


def cursor(offset):
    """Protobuf field 1 (varint) holding the result offset, base64 encoded."""
    n, out = offset, b""
    while True:
        b, n = n & 127, n >> 7
        out += bytes([b | 128 if n else b])
        if not n:
            break
    return base64.b64encode(b"\x08" + out).decode()


def rpc(rpcid, args, source_path):
    inner = json.dumps(args, separators=(",", ":"), ensure_ascii=False)
    body = {"f.req": json.dumps([[[rpcid, inner, None, "1"]]], separators=(",", ":"), ensure_ascii=False)}
    params = {"rpcids": rpcid, "source-path": source_path, "hl": "en", "gl": "us",
              "soc-app": "162", "soc-platform": "1", "soc-device": "1", "rt": "c"}
    r = requests.post(RPC_URL, params=params, data=body, headers=HEADERS, timeout=30)
    r.raise_for_status()
    start = r.text.find('[["wrb.fr"')
    if start < 0:
        raise RuntimeError("no data envelope (blocked or consent page)")
    row = json.JSONDecoder().raw_decode(r.text[start:])[0][0]
    if not row[2]:
        raise RuntimeError(f"Google rejected the request: {row[5:] or 'unknown'}")
    return json.loads(row[2])


def ymd(d):
    return [d.year, d.month, d.day]


def at(x, *path):
    """Safe positional lookup: None when any step is missing."""
    for p in path:
        try:
            x = x[p]
        except (IndexError, KeyError, TypeError):
            return None
    return x


def search(query, check_in, check_out, adults=2, currency="USD"):
    dates = [None, [None, [ymd(check_in), ymd(check_out), adults], None, None, None, [None, 0]]]
    args = [query,
            [1, [[[3], [3]]], dates, None, [[None, None, None, None, None, None, currency]]],
            [None, cursor(18), None, None, None, None, 13, None, 0], None, 1]
    payload = rpc("AtySUc", args, "/travel/search")
    hotels = []
    for slot in at(payload, 0, 0, 0, 1) or []:
        if isinstance(slot, list) and len(slot) > 1 and isinstance(slot[1], dict):
            for v in slot[1].values():
                rec = v[0] if v and isinstance(v[0], list) else None
                if rec and len(rec) > 20 and isinstance(rec[1], str) and isinstance(rec[20], str):
                    hotels.append(rec)
    return hotels


def offers(entity_id, check_in, check_out, adults=2, currency="USD"):
    dates = [ymd(check_in), ymd(check_out), adults, None, 0]
    args = [None, [None, None, None, currency, dates, None, None, None, None, None, None, None, None, [adults]],
            [], entity_id]
    rec = rpc("lvkpBc", args, "/travel/hotels/entity")[0]
    out = []
    for group in at(rec, 6, 2) or []:
        if isinstance(group, list) and group and all(
                isinstance(at(o, 0, 0), str) and isinstance(at(o, 12), list) for o in group):
            for o in group:
                nightly, total = at(o, 12, 4, 2), at(o, 12, 5, 2)
                if isinstance(nightly, (int, float)):
                    out.append({"provider": at(o, 0, 0), "pricePerNight": round(nightly, 2),
                                "totalPrice": round(total, 2) if isinstance(total, (int, float)) else None})
            break
    return sorted(out, key=lambda x: x["pricePerNight"])


if __name__ == "__main__":
    check_in = date.today() + timedelta(days=30)
    check_out = check_in + timedelta(days=2)
    for rec in search("hotels in Lisbon", check_in, check_out)[:3]:
        print(json.dumps({"name": rec[1], "entityId": rec[20],
                          "offers": offers(rec[20], check_in, check_out)[:4]}, indent=2))
```

Save it as `google_hotels_plain.py` and run it. This is real output from a run on 2026-10-09 (dates 30 days out, USD, market `us`), trimmed to one hotel. Your hotels and prices will differ by run date:

```json
{
  "name": "Inspira Liberdade Boutique Hotel",
  "entityId": "ChcIkoiYtZukqJkEGgsvZy8xd2N4ZmMxbBAB",
  "offers": [
    {"provider": "Priceline", "pricePerNight": 157.41, "totalPrice": 314.82},
    {"provider": "Expedia.com.tw", "pricePerNight": 218.43, "totalPrice": 436.86},
    {"provider": "Booking.com", "pricePerNight": 250.3, "totalPrice": 500.61},
    {"provider": "Agoda", "pricePerNight": 250.31, "totalPrice": 500.62}
  ]
}
```

The spread between providers is the point: for this hotel and these dates the cheapest and the Booking.com offer were about $93 a night apart. Which providers appear depends on the hotel and the market, so a hotel may list Trip.com, Hostelworld or only its own website.

## Step 2: price the same hotel across many dates

Because the dates are part of the request, per-date prices are a loop. This reuses `offers()` from the script above to price one hotel for several check-in dates, with a pause between requests:

```python
import time
from datetime import date, timedelta

from google_hotels_plain import offers

ENTITY_ID = "ChcIkoiYtZukqJkEGgsvZy8xd2N4ZmMxbBAB"  # entityId from step 1
start = date.today() + timedelta(days=30)

for i in range(5):
    check_in = start + timedelta(days=i)
    check_out = check_in + timedelta(days=2)
    found = offers(ENTITY_ID, check_in, check_out)
    if found:
        print(check_in, found[0]["provider"], found[0]["pricePerNight"])
    else:
        print(check_in, "no offers")
    time.sleep(2)  # keep the rate low
```

Store each row with the dates, adults, currency, market and the time you ran it, and you have a price history you can chart. A date with no availability comes back with no offers.

## Limits of the plain-HTTP route

- **It is undocumented.** There is no contract. The call names, the argument layout and the positions in the response are what the web page uses today, and Google can change them. Expect to fix the parser from time to time.
- **Datacenter IPs.** The script works from a home connection. In our testing Google answered datacenter IPs for the hosted Actor below without a proxy, but at very high volume Google may start refusing requests. If you run this from a cloud server, add retries with backoff and a long delay between requests, and be ready to route through a proxy if you see blocks. Nothing here guarantees that a given IP range will always be accepted.
- **Consent pages.** From some regions Google redirects to a consent page. The `SOCS` cookie in the script is there to avoid that. If it stops working you will see the "no data envelope" error.
- **Coverage per query is capped.** Google lists around 500 distinct hotels per query at most. For a large city, search neighbourhoods separately (`hotels in Shinjuku`, `hotels in Asakusa`). The script above reads only the first page of 20 (`cursor(18)` is the offset the web page sends for page 1); for later pages, raise it in steps of 20 (`cursor(38)`, `cursor(58)` and so on).
- **Highlights, not everything.** The search list carries only the highlight amenities Google shows on each result card, and neither the script nor these two calls return reviews or room types.
- **Terms of use.** Google's terms limit automated access and each booking site has its own terms. Read them for your use case, keep the request rate low and do not republish the data blindly.

## A hosted option with no proxy and no code to maintain

Disclosure: Google Hotels Scraper is ours (Data Gleaner).

[Google Hotels Scraper](https://apify.com/datagleaner/google-hotels-scraper) is an Apify Actor that does what the script above does as a hosted job. You give it places or hotel URLs and dates, and it returns one flat JSON record per hotel through the Apify API, with the lowest price for your dates, every booking site's per-night and total price with its link, rating, review count, star class, highlight amenities and coordinates, plus the address, website and check-in times when details are on. It calls Google Travel over plain HTTP, with no browser, no login and no Google account, and its proxy setting is off by default because Google answered datacenter IPs in our testing. If blocks start at high volume, you can switch the proxy setting on.

It costs **$3.00 per 1,000 hotels** ($0.003 per hotel), billed only for hotels pushed to the dataset, with no monthly plan. A 10-hotel test run costs $0.03, and 5 cities at 100 hotels each is $1.50. You need a free Apify account and its API token.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-hotels-scraper").call(run_input={
    "locations": ["Lisbon"],
    "checkIn": "2026-11-10",
    "checkOut": "2026-11-12",
    "adults": 2,
    "currency": "USD",
    "maxHotelsPerLocation": 10,
    "includeDetails": True,
    "countryCode": "us",
})
if run is None:
    raise SystemExit("[ERROR] The Actor run did not return.")

for hotel in client.dataset(run.default_dataset_id).iterate_items():
    print(hotel["name"], hotel.get("rating"), hotel.get("lowestPrice"), hotel.get("currency"))
    for offer in hotel.get("offers", []):
        print("   ", offer["provider"], offer["pricePerNight"], offer["totalPrice"])
```

The input fields are `locations` (free text such as `Paris 8e` or `hostels in Berlin`), `hotelUrls` (Google Hotels hotel URLs or an `entityId` from an earlier run), `checkIn` and `checkOut` (`YYYY-MM-DD`), `adults` (1 to 8), `currency`, `maxHotelsPerLocation` (default 20, up to 1,000), `includeDetails` (default on; off gives a faster list with only the lowest price), `minRating`, `hotelClass`, `minPrice`, `maxPrice`, `language`, `countryCode`, `requestDelaySecs` and `proxyConfiguration`. Filters are applied before the details request, and hotels skipped by a filter are not charged.

This is one record, shortened, from a Paris search (two offers shown):

```json
{
  "name": "Le Tsuba Hotel",
  "entityId": "ChkI8bPIkrOozLonGg0vZy8xMWNseWdzeDVmEAE",
  "address": "45 Rue des Acacias, 75017 Paris, France",
  "starClass": 4,
  "rating": 4.6,
  "reviewCount": 1590,
  "lowestPrice": 185.28,
  "currency": "USD",
  "pricesForDates": true,
  "offers": [
    {"provider": "Bluepillow.tw", "pricePerNight": 185.28, "priceDisplay": "$185", "totalPrice": 370.56, "totalPriceDisplay": "$371", "link": "https://www.google.com/travel/lodging/clk?..."},
    {"provider": "Amimir.com", "pricePerNight": 193.7, "priceDisplay": "$194", "totalPrice": 387.4, "totalPriceDisplay": "$387", "link": "https://www.google.com/travel/lodging/clk?..."}
  ],
  "checkIn": "2026-11-10",
  "checkOut": "2026-11-12",
  "nights": 2,
  "adults": 2,
  "scrapedAt": "2026-10-07T15:59:05+00:00"
}
```

`lowestPrice` and each `pricePerNight` are per night; `totalPrice` covers the whole stay. `pricesForDates: true` confirms Google priced the exact dates you asked for. A hotel with no availability on those dates still comes back, with `lowestPrice: null` and empty `offers`.

The Actor has the same limits as the script, since it reads the same source: about 500 hotels per query at most, highlight amenities only, providers that differ by market, and no reviews or room types in this version. The reason to use it is not that it sees more. It is that someone else keeps the parser working, retries and backs off on blocks, and bills you only for hotels returned.

## Which route to choose

| Situation | Route |
|---|---|
| One-off look at a few hotels, you like owning the code | The plain-HTTP script above |
| Daily prices for many hotels or dates, no maintenance | A hosted scraper such as Google Hotels Scraper (ours) |
| You already pay for a SERP API | Its Google Hotels endpoint; see [Google Hotels API options](google-hotels-api) |
| You need pages beyond the hotel data, or JavaScript-only content | A headless browser, with proxies if you hit blocks |

## FAQ

**Can you scrape Google Hotels without Selenium?**
Yes. The Google Hotels page fills itself from internal POST calls, and the script above makes those calls with `requests`. You get hotels, date-specific prices and provider offers without rendering the page. The trade-off is that the calls are undocumented and may change.

**Do you need proxies to scrape Google Hotels?**
Not for small, polite runs: the script works from a normal connection, and in our testing the hosted Actor ran from datacenter IPs without a proxy. At very high volume Google may start refusing requests, and then a proxy and slower pacing can help. Treat that as a limit to plan for, not a guarantee either way.

**Is it legal to scrape Google Hotels?**
This guide cannot give legal advice. Google's terms of service limit automated access, and the booking sites have their own terms. Hotel records are business data, but if you combine them with anything about individuals, data-protection duties are yours. Keep the request rate low and read the terms for your use case.

**How do I get prices for specific dates?**
Pass the check-in and check-out dates in the request. The price you get is for those dates, adults, currency and market. Loop over dates, as in step 2, to build a per-date price series.

**How many hotels can I get per search?**
Google lists around 500 distinct hotels per query at most, in pages of 20. For more coverage in a large city, search each neighbourhood as its own query.

**Which booking sites show up?**
Whichever providers Google lists for that hotel in your market, such as Booking.com, Agoda, Trip.com or Hostelworld, each with its own per-night and total price. Set the market country (`gl` in the script, `countryCode` in the Actor) to the market you care about.

## Related guides

- [Google Hotels API: the options for hotel prices as JSON](google-hotels-api)
- [How to track hotel prices on Google Hotels](track-hotel-prices-google-hotels)
- [Google News scraper in Python](google-news-scraper-python)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Can you scrape Google Hotels without Selenium?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. The Google Hotels page fills itself from internal POST calls, and the script above makes those calls with requests. You get hotels, date-specific prices and provider offers without rendering the page. The trade-off is that the calls are undocumented and may change."
      }
    },
    {
      "@type": "Question",
      "name": "Do you need proxies to scrape Google Hotels?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not for small, polite runs: the script works from a normal connection, and in our testing the hosted Actor ran from datacenter IPs without a proxy. At very high volume Google may start refusing requests, and then a proxy and slower pacing can help. Treat that as a limit to plan for, not a guarantee either way."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to scrape Google Hotels?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "This guide cannot give legal advice. Google's terms of service limit automated access, and the booking sites have their own terms. Hotel records are business data, but if you combine them with anything about individuals, data-protection duties are yours. Keep the request rate low and read the terms for your use case."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get prices for specific dates?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Pass the check-in and check-out dates in the request. The price you get is for those dates, adults, currency and market. Loop over dates, as in step 2, to build a per-date price series."
      }
    },
    {
      "@type": "Question",
      "name": "How many hotels can I get per search?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Google lists around 500 distinct hotels per query at most, in pages of 20. For more coverage in a large city, search each neighbourhood as its own query."
      }
    },
    {
      "@type": "Question",
      "name": "Which booking sites show up?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Whichever providers Google lists for that hotel in your market, such as Booking.com, Agoda, Trip.com or Hostelworld, each with its own per-night and total price. Set the market country (gl in the script, countryCode in the Actor) to the market you care about."
      }
    }
  ]
}
</script>
