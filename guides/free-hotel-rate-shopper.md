---
title: "Free Hotel Rate Shopper for Small Hotels: Daily Compset Rates"
description: "A free or near-free hotel rate shopper for independent hotels: shop 5-10 competitors daily for fixed stay dates, get every OTA price in a sheet, with a pivot."
---

# A free hotel rate shopper for small hotels

A small hotel can rate shop for free on Google Hotels, which shows each competitor's price on every booking site for the dates you set. Check five to ten competitors for fixed stay dates, save one row per hotel, per site, per day in a sheet, and a pivot of your rate against the compset median answers the daily question: am I priced above or below my market for this night? By hand it costs nothing; automated with the script below it costs a few dollars a month. This page covers the manual route, your own free script, a hosted daily shop with the pivot, and when a paid revenue management system (RMS) is worth the money.

Disclosure: Data Gleaner, the hosted scraper described in the middle of this page, is us. The manual method and the pivot work without it, and the spreadsheet part is free whatever you use to collect prices.

## What a rate shop has to do

A rate shop answers one question, repeatedly: for the same night, same length of stay and same number of guests, what do my competitors charge, and where do I sit? The details that make the answer usable:

- **A fixed compset.** Keep it small and deliberate. The [PriceLabs guide to rate shopping](https://hotels.pricelabs.co/blog/hotel-rate-shopping-tool/), for example, recommends keeping the comp set between 5 and 15 hotels. Choosing by location, star class and guest type matters more than choosing many.
- **Fixed stay dates.** A price is only comparable if the check-in date, check-out date and number of adults are identical for every hotel. Choose the nights you want to watch (next Friday, the next holiday weekend, 30 and 60 days out) and keep them stable.
- **Every channel, not one.** The same hotel can be cheapest on one booking site and dearer on another. A single-channel view can mislead. (The PriceLabs page describes its Hotel Rate Shopper as powered by publicly available data from Booking.com.)
- **A history.** One day's snapshot says little. Prices saved every day show when competitors move and how fast.

## Free option 1: do it by hand on Google Hotels

For two or three competitors and a weekly check, a manual look costs nothing:

1. Open [google.com/travel/hotels](https://www.google.com/travel/hotels) and search for a competitor by name.
2. Set the check-in and check-out dates and the number of guests, and open the hotel's prices to see which booking sites list it and at what price.
3. Type the numbers into a sheet with the date you looked, the stay date, the hotel and the site.

It teaches you your market. It stops working at about five hotels, three stay dates and daily checks, because that is 15 lookups a day with copy and paste errors. If you only want a notification when one hotel's price changes, Google Hotels has a price tracking toggle; see [track hotel prices on Google Hotels](track-hotel-prices-google-hotels). It alerts you to a change on one set of dates, but it does not build a compset history for you.

## Free option 2: your own script

The prices on Google Hotels come from internal calls that you can make with plain HTTP in Python, with no browser. The full working code, and its honest limits (an undocumented endpoint, possible blocks from cloud IPs, consent pages), are in [scrape Google Hotels with Python](scrape-google-hotels-python). That guide prices one hotel across several dates, so a compset shop is a loop over your hotels as well as your stay dates. You own the code and pay nothing, and you also own the fixing when the endpoint changes. For a small hotel team that is the real cost, so read that guide's limits before you commit.

## Option 3: the Data Gleaner Google Hotels Scraper

Disclosure: Google Hotels Scraper is ours (Data Gleaner).

[Google Hotels Scraper](https://apify.com/datagleaner/google-hotels-scraper) is an Apify Actor. In its detail mode you give it `hotelUrls`, which are Google Hotels hotel page URLs or the bare `entityId` from earlier results, together with `checkIn`, `checkOut` and `adults`. It returns the full record for each hotel and your dates: `lowestPrice`, and an `offers` list with each booking site's `provider`, `pricePerNight`, `totalPrice` and `link`. It reads Google Travel over plain HTTP, with no browser, no login and no Google account.

From its documentation:

- **Price:** pay per result, **$3.00 per 1,000 hotels** ($0.003 per hotel), billed only for hotels pushed to the dataset. A 10-hotel test run costs $0.03.
- **Dates are per run.** `checkIn` and `checkOut` are single values for a run, so each stay date is one run.
- **`pricesForDates`** is `true` when Google priced the exact dates you asked for. A hotel with no availability on those dates is still returned, with `lowestPrice: null` and empty `offers`.
- **Providers differ by market.** Set `countryCode` to the market your guests book from.
- **Not in this version:** reviews and room types. You get the prices Google Hotels shows, not your competitors' internal rate plans or room-by-room matching.

### What it costs for a daily shop

The cost is hotels times stay dates times days. Count your own hotel too, because the pivot needs your rate beside the compset:

| Hotels shopped | Stay dates | Days | Hotel results | Cost at $0.003 |
|---|---|---|---|---|
| You + 5 competitors | 1 | 30 | 180 | $0.54 |
| You + 5 competitors | 3 | 30 | 540 | $1.62 |
| You + 9 competitors | 3 | 30 | 900 | $2.70 |

That is the Actor's per-result price. Check your Apify plan page for anything else on your account.

### Step 1: collect the hotel URLs

Search each hotel by name on Google Hotels, open its page and copy the address from the browser bar. It has the form `https://www.google.com/travel/hotels/entity/Ch...`, and the part after `entity/` is the `entityId`. Do this once for your own hotel and for each competitor, and keep the list in a file.

### Step 2: the daily script

This script runs the Actor once per stay date for your hotel list and appends one row per hotel and booking site to `rates.csv`. It uses the input fields `hotelUrls`, `checkIn`, `checkOut`, `adults`, `currency`, `countryCode` and `includeDetails`. Set `APIFY_TOKEN` in your environment (you need a free Apify account for the token), fill in the `entityId` values, and keep your own hotel first.

```python
# pip install apify-client
import csv
import os
from datetime import date, timedelta
from pathlib import Path

from apify_client import ApifyClient

# Bare entityIds (the part after /entity/ in the Google Hotels URL). Your hotel first.
MY_ID = "ChYOUR_HOTEL_ENTITY_ID"
COMPSET = ["ChCOMPETITOR_1_ENTITY_ID", "ChCOMPETITOR_2_ENTITY_ID", "ChCOMPETITOR_3_ENTITY_ID"]
HOTELS = [MY_ID] + COMPSET

STAY_DATES = [date(2026, 11, 13), date(2026, 11, 20), date(2026, 12, 24)]  # check-in nights
NIGHTS = 1
OUT = Path("rates.csv")
FIELDS = ["scraped_on", "stay_date", "hotel", "entity_id", "is_mine",
          "provider", "price_per_night", "currency"]

client = ApifyClient(os.environ["APIFY_TOKEN"])
today = date.today().isoformat()
rows = []

for stay in STAY_DATES:
    if stay < date.today():
        continue  # past nights cannot be shopped
    run = client.actor("datagleaner/google-hotels-scraper").call(run_input={
        "hotelUrls": HOTELS,
        "checkIn": stay.isoformat(),
        "checkOut": (stay + timedelta(days=NIGHTS)).isoformat(),
        "adults": 2,
        "currency": "USD",
        "countryCode": "us",
        "includeDetails": True,
    })
    if run is None:
        print(f"[ERROR] No run returned for {stay}")
        continue
    for hotel in client.dataset(run.default_dataset_id).iterate_items():
        mine = hotel["entityId"] == MY_ID
        offers = hotel.get("offers") or []
        currency = hotel.get("currency") or ""
        if not offers:  # sold out or no price: keep a row so gaps stay visible
            rows.append([today, stay, hotel["name"], hotel["entityId"], mine, "", "", currency])
        for o in offers:
            rows.append([today, stay, hotel["name"], hotel["entityId"], mine,
                         o["provider"], o["pricePerNight"], currency])

new_file = not OUT.exists()
with OUT.open("a", newline="", encoding="utf-8") as f:
    w = csv.writer(f)
    if new_file:
        w.writerow(FIELDS)
    w.writerows(rows)
print(f"[OK] {len(rows)} rows appended to {OUT}")
```

`NIGHTS = 1` is a choice: use the length of stay your guests usually book, and keep it the same every day. If you prefer no code for the collection step, open the Actor in the Apify Console, paste the hotel URLs into the "Hotel URLs or entity IDs" input, set the dates, and export the dataset as CSV or Excel.

### Step 3: schedule it

Pick one:

- **cron or Task Scheduler.** Run the script once a day on any computer or small server that is on at that hour. A crontab line for 06:00 daily: `0 6 * * * cd /path/to/rateshop && python rateshop.py`.
- **Apify Schedules.** Apify requires an Actor to have run at least once before it can be scheduled. Then open **Schedules** in the Console and click **Create new**, open the **Schedule setup** card, click **Add**, select **Actor**, choose the Actor, set its input, and click **Enable** (new schedules start disabled). Schedules use a cron expression, and `@daily` is accepted. One schedule can hold up to 10 Actor entries, so each stay date can have its own entry with its own input. This route leaves the results in Apify datasets, so you export them to your sheet yourself; it does not write `rates.csv` for you.

### Step 4: the pivot, my rate against the compset median

Load `rates.csv` and look at the latest day. For each stay date you want three numbers: your lowest rate, the median of your competitors' lowest rates, and the gap between them.

In pandas, which also makes a good daily check:

```python
# pip install pandas
import pandas as pd

df = pd.read_csv("rates.csv", parse_dates=["scraped_on", "stay_date"])
df = df[df["price_per_night"].notna()]
latest = df[df["scraped_on"] == df["scraped_on"].max()]

# lowest price per hotel across all booking sites
low = latest.groupby(["stay_date", "hotel", "is_mine"], as_index=False)["price_per_night"].min()

mine = low[low["is_mine"]].set_index("stay_date")["price_per_night"].rename("my_rate")
comp = (low[~low["is_mine"]].groupby("stay_date")["price_per_night"]
        .agg(compset_median="median", compset_min="min", compset_max="max"))

pivot = comp.join(mine)
pivot["vs_median_pct"] = (pivot["my_rate"] / pivot["compset_median"] - 1) * 100
print(pivot.round(1))
```

In a spreadsheet, build a pivot table from the same columns: rows `stay_date`, columns `hotel`, values the minimum of `price_per_night`, filtered to the latest `scraped_on`. Beside it, add `=MEDIAN(...)` over the competitor columns for the compset median, your own column as `my rate`, and `=my_rate/median-1` for the gap. A second pivot with `scraped_on` on the rows, filtered to one stay date, gives you the history: how each hotel's price for that night moved from day to day.

Read the result with care. The gap says you are above or below the median, not that you are wrong. A hotel with a sea view may rightly sit above a median of simpler rooms. Use the gap as a prompt to ask why, not as an instruction.

## When a paid RMS is worth it

A rate shop shows prices. It does not tell you what to do with them. A paid revenue management system or rate shopping product is worth considering when:

- **You need a much larger market view.** PriceLabs states its Hotel Rate Shopper can monitor up to 350 nearby properties. A daily sheet of ten hotels is a different tool.
- **You want recommendations, not a table.** A paid system can combine your own booking pace and demand with competitor prices and suggest a rate. A sheet does none of that.
- **You need to push rates to your PMS or channel manager.** A script that reads prices does not write them back.
- **You need room-level comparison.** This guide's data is the lowest price per hotel and each booking site's price shown on Google Hotels. It is not matched room type by room type.

A sheet and a daily script are enough when you are a small property, set rates yourself, have a small comp set and want a market view for a few dollars. A fair test for buying a tool: if a better price on the nights you most often get wrong would pay for it several times over, buy it. If you cannot name those nights yet, run the sheet for a month first and you will find them. Paid products vary in price and trials, so check each vendor's current pricing page; the PriceLabs page referenced here mentions a 30-day free trial and does not list plan prices.

## Limits of this method

- **Public prices only.** You see what Google Hotels shows a guest for those dates and that market, not corporate rates or member-only rates.
- **Market and currency matter.** Providers and deals can differ by market, so fix `countryCode` and `currency` and do not change them mid-series.
- **The lowest price is not always your comparable room.** It is the lowest the listing shows for your dates and guests, not necessarily the same room type as yours.
- **Dates must be in the future.** The Actor takes dates from today onwards, with check-out after check-in. Past nights cannot be re-shopped, so you only have the history you saved.
- **Terms of use.** Google's terms limit automated access and each booking site has its own terms. Keep the volume low, as a daily shop of a handful of hotels is, and read the terms for your own use.

## FAQ

**What is a free hotel rate shopper for small hotels?**
It is any way of collecting competitors' room prices for fixed dates without a software subscription. The simplest is checking Google Hotels by hand and typing prices into a sheet. A step up is a short script or a pay-per-result scraper that fills the sheet for you every day. Free options stop where the manual work becomes too much, so decide how many hotels and dates you need to watch before you choose.

**How many competitors should be in my compset?**
Five to ten, picked by location, star class and the guests you compete for. The PriceLabs guide to rate shopping recommends a set of 5 to 15 hotels, and bigger sets mostly add hotels that are not your real competitors. More hotels also raise the cost of a scraper, which charges per hotel result.

**How much does a daily rate shop cost with Google Hotels Scraper?**
$0.003 per hotel result. For your hotel plus five competitors, three stay dates and 30 days, that is 540 results, or $1.62 a month; for your hotel plus nine competitors it is $2.70. The cost scales with hotels times stay dates times days, so dropping a stay date or a hotel cuts it in proportion.

**Can I see every OTA's price, not just the lowest?**
Yes. With details on, which is the default, each hotel record has an `offers` list with the provider, price per night, total price and link for every booking site Google Hotels lists for that hotel in your market. The script above stores one row per site so you can compare channels.

**How do I compare my rate with the compset median?**
Take the lowest price for each hotel on each stay date, then the median of your competitors' values, and compare it with yours. The pandas snippet and the spreadsheet recipe above both do this, and the percentage gap shows how far above or below the market you are.

**Is a free rate shopper enough, or do I need an RMS?**
For a small property that sets its own rates, a daily sheet is often enough to see the market. You need an RMS when you want pricing recommendations based on your own pace and demand, rate pushes to your systems, or a much larger compset with room-level comparison. Run the sheet for a month first: it shows which decisions you actually get wrong.

## Related guides

- [Track hotel prices on Google Hotels](track-hotel-prices-google-hotels): the free alert toggle, its limits, and tracking at scale.
- [Hotel price comparison API](hotel-price-comparison-api): compare booking sites' prices for the same hotel as data.
- [Scrape Google Hotels with Python](scrape-google-hotels-python): the plain-HTTP script behind the free route.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What is a free hotel rate shopper for small hotels?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It is any way of collecting competitors' room prices for fixed dates without a software subscription. The simplest is checking Google Hotels by hand and typing prices into a sheet. A step up is a short script or a pay-per-result scraper that fills the sheet for you every day. Free options stop where the manual work becomes too much, so decide how many hotels and dates you need to watch before you choose."
      }
    },
    {
      "@type": "Question",
      "name": "How many competitors should be in my compset?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Five to ten, picked by location, star class and the guests you compete for. The PriceLabs guide to rate shopping recommends a set of 5 to 15 hotels, and bigger sets mostly add hotels that are not your real competitors. More hotels also raise the cost of a scraper, which charges per hotel result."
      }
    },
    {
      "@type": "Question",
      "name": "How much does a daily rate shop cost with Google Hotels Scraper?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "$0.003 per hotel result. For your hotel plus five competitors, three stay dates and 30 days, that is 540 results, or $1.62 a month; for your hotel plus nine competitors it is $2.70. The cost scales with hotels times stay dates times days, so dropping a stay date or a hotel cuts it in proportion."
      }
    },
    {
      "@type": "Question",
      "name": "Can I see every OTA's price, not just the lowest?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. With details on, which is the default, each hotel record has an offers list with the provider, price per night, total price and link for every booking site Google Hotels lists for that hotel in your market. The script above stores one row per site so you can compare channels."
      }
    },
    {
      "@type": "Question",
      "name": "How do I compare my rate with the compset median?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Take the lowest price for each hotel on each stay date, then the median of your competitors' values, and compare it with yours. The pandas snippet and the spreadsheet recipe above both do this, and the percentage gap shows how far above or below the market you are."
      }
    },
    {
      "@type": "Question",
      "name": "Is a free rate shopper enough, or do I need an RMS?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "For a small property that sets its own rates, a daily sheet is often enough to see the market. You need an RMS when you want pricing recommendations based on your own pace and demand, rate pushes to your systems, or a much larger compset with room-level comparison. Run the sheet for a month first: it shows which decisions you actually get wrong."
      }
    }
  ]
}
</script>
