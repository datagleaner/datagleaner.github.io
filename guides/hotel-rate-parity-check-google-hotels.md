---
title: "Hotel Rate Parity Check on Google Hotels: Find Which OTA Undercuts You"
description: "Run a hotel rate parity check on Google Hotels: compare each OTA's offer with your direct rate by date, flag gaps over 2%, and build a weekly report."
---

# Hotel rate parity check: which OTA undercuts your direct rate

To check rate parity on Google Hotels, search your hotel for the exact stay dates and compare the price your official site shows with the price each online travel agency (OTA) shows in the same list of booking options. Any OTA that is cheaper than your direct rate for the same dates, guests and market is undercutting you. Doing that by eye works for one date. To know which OTA undercuts you, by how much and on which dates, you need the per-provider prices for many dates as data, a rule that flags gaps above a threshold such as 2%, and a weekly summary you can send to the OTA's market manager. This guide gives you each of those, starting with the free ones.

Disclosure: Data Gleaner, mentioned in the second half as one way to collect the prices, is us. The manual check and the analysis script below are free, and the script works on prices from any source.

## Why this is worth checking

Triptease published an analysis of its Google Hotel Ads data for January to May 2022 ([How often are you being undercut on metasearch?](https://triptease.com/resources/new-data-how-often-are-you-being-undercut-on-metasearch)). It found that on average 61% of the time, a hotel's direct price was beaten by at least one other OTA, and that 24% of impressions showed matching prices. By region, the undercut share was 71% in APAC, 62% in North America and 52% in EMEA. The same page reports a click-through rate of 2.46% when the hotel was undercut against 3.61% at parity, and a conversion rate of 0.07% against 0.108%. Two cautions: it is one vendor's data from 2022, and it counts a hotel as undercut if just one OTA in the results is cheaper. Treat it as a reason to measure your own property, not as your number.

Whether a price gap breaches your OTA contract, and what you can do about it, depends on that contract and on local rules. This page does not give legal advice.

## Option 1: check by hand on Google Hotels (free)

For a first look at one date, you need no tools:

1. Open [google.com/travel/hotels](https://www.google.com/travel/hotels) and search for your hotel's name.
2. Set the check-in and check-out dates and the number of guests you want to test.
3. Open your hotel's listing and look at the prices from the booking sites. Google shows a list of providers with a price each.
4. Note your official site's price and the lowest OTA price.

Search in the same market you care about. Google can show different providers and prices depending on the market and the language, so a check from your office in one country may not match what a guest in another country sees. Use the same dates, guests and currency every time you repeat it.

The limits are plain. A manual check takes a minute per date, so a month of dates is half a day, and nothing is recorded unless you write it down. Your official site only appears in the list if your hotel is connected to Google's booking links, and the provider's name may not look like your hotel's name. Check how your own listing is named before you build anything on top of it.

## Option 2: record the prices in a sheet and flag gaps with a formula (free)

If you only have a few dates, type or paste the prices into a spreadsheet and let a formula do the comparison. Put the stay date in column A, your direct price per night in column B, and one OTA's price per night in column C. Then in D and E:

```
D2: =IF(OR(B2="",C2=""),"",(B2-C2)/B2)
E2: =IF(D2="","",IF(D2>0.02,"UNDERCUT","ok"))
```

Format column D as a percentage. A value of 0.05 means the OTA is 5% below your direct rate. Fill the formulas down, and add one block of columns per OTA. Change `0.02` to the threshold you want. To count how often an OTA undercuts you, use `=COUNTIF(E2:E32,"UNDERCUT")`.

Compare like with like. Check that both prices are per night or both are the total for the stay, in the same currency, and that you know whether taxes and fees are included on each side, because providers do not always show them the same way. A small gap may come from that and not from a real discount.

## Option 3: pull the offers as data (Python)

A spreadsheet stops being practical past a few dates. Our guide [How to scrape Google Hotels with Python](scrape-google-hotels-python) has a free script that returns every provider's price for a hotel and dates using only `requests`. Its limits apply here: the endpoint is undocumented and may change, and Google may refuse requests from datacenter IPs at higher volume. The script in the next section shows the parity rule itself and works with any source of per-provider prices.

## The parity script: flag gaps above 2% and print the weekly report

This script takes one hotel, runs the price lookup for each check-in date in a range, finds your direct offer by a piece of text in its provider name, and compares every other provider with it. It takes each provider's lowest price per date, since Google can list one provider twice. It prints a Markdown report grouped by OTA, with the number of dates undercut, the average gap, the worst gap and a table of the dates. It uses the hosted lookup described below, so it needs `pip install apify-client` and an `APIFY_TOKEN`; to use another source, keep the comparison part and replace the lookup.

```python
# pip install apify-client
import os
import sys
from collections import defaultdict
from datetime import date, timedelta

from apify_client import ApifyClient

HOTEL = sys.argv[1]                        # entityId or Google Hotels hotel URL
DIRECT = sys.argv[2].lower()               # text in YOUR provider name, e.g. "tsubahotel"
START = date.fromisoformat(sys.argv[3])    # first check-in date
DAYS = int(sys.argv[4])                    # how many check-in dates to test
NIGHTS = 1
THRESHOLD = 0.02                           # flag OTAs more than 2% below direct

client = ApifyClient(os.environ["APIFY_TOKEN"])
rows, checked = [], 0
for i in range(DAYS):
    d_in = START + timedelta(days=i)
    run = client.actor("datagleaner/google-hotels-scraper").call(run_input={
        "hotelUrls": [HOTEL],
        "checkIn": d_in.isoformat(),
        "checkOut": (d_in + timedelta(days=NIGHTS)).isoformat(),
        "adults": 2,
        "currency": "USD",
        "countryCode": "us",
        "includeDetails": True,
    })
    if run is None:
        print(f"[WARN] no run for {d_in}", file=sys.stderr)
        continue
    for hotel in client.dataset(run.default_dataset_id).iterate_items():
        # lowest price per provider: Google can list one provider twice (a sponsored and a regular row)
        best = {}
        for o in hotel.get("offers") or []:
            if o.get("pricePerNight"):
                best[o["provider"]] = min(o["pricePerNight"], best.get(o["provider"], float("inf")))
        direct = [p for name, p in best.items() if DIRECT in name.lower()]
        if not direct:
            print(f"[WARN] {d_in}: no direct offer; providers: {list(best)}", file=sys.stderr)
            continue
        checked += 1
        base = min(direct)
        for name, p in best.items():
            if DIRECT in name.lower():
                continue
            gap = (base - p) / base
            if gap > THRESHOLD:
                rows.append((d_in.isoformat(), name, base, p, gap))

by_ota = defaultdict(list)
for r in rows:
    by_ota[r[1]].append(r)

print("# Rate parity report\n")
print(f"Hotel: {HOTEL} | dates with a direct rate: {checked} of {DAYS} | threshold: {THRESHOLD:.0%}\n")
if checked == 0:
    print("Direct rate not found on any date, so parity was not checked.")
elif not rows:
    print("No OTA undercut the direct rate by more than the threshold.")
for ota, rs in sorted(by_ota.items(), key=lambda kv: -len(kv[1])):
    worst = max(rs, key=lambda r: r[4])
    avg = sum(r[4] for r in rs) / len(rs)
    print(f"## {ota}: undercut on {len(rs)} of {checked} dates, average {avg:.1%}, worst {worst[4]:.1%} on {worst[0]}")
    print("| Check-in | Direct | OTA | Gap |\n|---|---|---|---|")
    for r in sorted(rs):
        print(f"| {r[0]} | {r[2]:.2f} | {r[3]:.2f} | {r[4]:.1%} |")
    print()
```

Run it as `python parity.py <entityId> <text-in-your-provider-name> 2026-11-10 30 > report.md`. The first run is the one to read closely. If it warns "no direct offer" (on the terminal, not in the report file) for every date and the report says the direct rate was not found, your official site is not in the provider list for that market, or its name does not contain the text you gave, and the printed provider names tell you which. Fix that before you trust a clean report, because a report that cannot see your direct rate says nothing about parity.

To get a weekly report, run the same command on a schedule, for example with cron or a scheduled workflow, over the next 30 to 60 check-in dates, and save each output with the run date. Comparing two weeks shows whether a gap is new, repeated or fixed.

## What to put in the report for the OTA market manager

Send facts the manager can check and act on, not a complaint:

- The date range, guests, currency and market you tested, and the date you ran it.
- For each date, your direct price, the OTA's price and the gap in percent, as in the tables the script prints.
- How many of the tested dates were undercut, the average gap and the worst case.
- A link or screenshot of the Google Hotels listing for two or three of the worst dates, taken at the time.
- A clear ask: which rates you want corrected, and by when.

The OTA may have a reason for a lower price, such as a package or a different cancellation policy, so say what you compared and let them explain it.

## Option 4: Data Gleaner Google Hotels Scraper

Disclosure: Google Hotels Scraper is ours (Data Gleaner).

[Google Hotels Scraper](https://apify.com/datagleaner/google-hotels-scraper) is an Apify Actor that returns one flat JSON record per hotel from Google Hotels, with the lowest price for your dates and each booking site's per-night and total price with its link. For parity work, the useful fields are `offers` (a list with `provider`, `pricePerNight`, `totalPrice` and `link`), `website`, `lowestPrice`, `currency` and `pricesForDates`, which confirms Google priced the exact dates you asked for. It calls Google Travel over plain HTTP with no browser, no login and no Google account.

It costs **$3.00 per 1,000 hotels** ($0.003 per hotel), billed only for hotels pushed to the dataset, with no monthly plan. A parity check is one hotel per date, so checking one hotel for 30 check-in dates is 30 records, or $0.09. You need a free Apify account and its API token.

The input fields you need are `hotelUrls` (Google Hotels hotel page URLs or the bare `entityId` from an earlier result), `checkIn` and `checkOut` (`YYYY-MM-DD`), `adults` (1 to 8), `currency`, `countryCode` (the Google market, default `us`), `language` and `includeDetails` (default on, and required here: it adds the per-provider offers). To find your hotel's `entityId` the first time, search with `locations` set to your hotel's name or city, take the matching record's `entityId`, and use it in `hotelUrls` afterwards. This is the short version of the call used in the script above:

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-hotels-scraper").call(run_input={
    "hotelUrls": ["ChkI8bPIkrOozLonGg0vZy8xMWNseWdzeDVmEAE"],
    "checkIn": "2026-11-10",
    "checkOut": "2026-11-11",
    "adults": 2,
    "currency": "USD",
    "countryCode": "us",
    "includeDetails": True,
})
if run is None:
    raise SystemExit("[ERROR] The Actor run did not return.")

for hotel in client.dataset(run.default_dataset_id).iterate_items():
    for offer in hotel.get("offers", []):
        print(offer["provider"], offer["pricePerNight"], offer["totalPrice"])
```

Its limits are the limits of the source. It sees what Google lists for that hotel in that market on that day, so your official site shows up only if Google lists it; the Actor does not know which provider is "you", which is why the script matches on the provider name. A hotel with no availability on the dates still comes back, with `lowestPrice: null` and empty `offers`. Prices are for the dates, guests, currency and market you set. The Actor's proxy setting is off by default because Google answered datacenter IPs in our testing, and you can turn it on if blocks start at high volume. It reads Google's price list only: it does not see your channel manager, the rate plans behind each price, taxes shown after a click-through or member-only rates.

## FAQ

**How do I check rate parity on Google Hotels?**

Search your hotel with the exact check-in and check-out dates, guests and market, then compare your official site's price with each OTA's price in the provider list. To do it for many dates, collect the per-provider prices as data and flag any OTA that is below your direct rate by more than your threshold, such as 2%.

**What counts as an OTA undercutting my direct rate?**

Here it means an OTA price per night that is lower than your official site's price for the same dates, guests, currency and market. Check that both prices include the same taxes and fees, and note that the OTA may be selling a different rate plan, so a gap is a reason to look, not proof of a breach.

**Why use a 2% threshold?**

It is a starting point, not a standard. A small gap can come from rounding, currency conversion or how taxes are shown. A threshold of 2% leaves out most of that noise while still catching real discounts, and you can raise or lower it in the script by changing `THRESHOLD`.

**Why does the report say my direct offer was not found?**

Either your official site is not in Google's provider list for that market and date, or its provider name does not contain the text you passed to the script. Read the provider names the script prints, adjust the text, and test another market with `countryCode` if your listing only shows there.

**How often should I run a parity check?**

Weekly is a practical rhythm for a report you take to an OTA, because it gives enough dates to show a pattern without flooding the manager with noise. Run it over the next 30 to 60 check-in dates, keep each week's output, and compare weeks to see whether a gap was fixed.

**Is the 61% figure my hotel's undercut rate?**

No. It is an average from Triptease's own Google Hotel Ads data for January to May 2022, counting a hotel as undercut if at least one OTA was cheaper. Your own rate depends on your market, your OTAs and your dates, which is why the useful number is the one you measure with the script above.

## Related guides

- [How to track hotel prices on Google Hotels](track-hotel-prices-google-hotels): the price tracking toggle and scheduled runs for many hotels and dates.
- [Hotel price comparison API](hotel-price-comparison-api): get every provider's price for a hotel as JSON.
- [How to scrape Google Hotels with Python](scrape-google-hotels-python): the plain-HTTP script for provider offers.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I check rate parity on Google Hotels?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Search your hotel with the exact check-in and check-out dates, guests and market, then compare your official site's price with each OTA's price in the provider list. To do it for many dates, collect the per-provider prices as data and flag any OTA that is below your direct rate by more than your threshold, such as 2%."
      }
    },
    {
      "@type": "Question",
      "name": "What counts as an OTA undercutting my direct rate?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Here it means an OTA price per night that is lower than your official site's price for the same dates, guests, currency and market. Check that both prices include the same taxes and fees, and note that the OTA may be selling a different rate plan, so a gap is a reason to look, not proof of a breach."
      }
    },
    {
      "@type": "Question",
      "name": "Why use a 2% threshold?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It is a starting point, not a standard. A small gap can come from rounding, currency conversion or how taxes are shown. A threshold of 2% leaves out most of that noise while still catching real discounts, and you can raise or lower it in the script by changing THRESHOLD."
      }
    },
    {
      "@type": "Question",
      "name": "Why does the report say my direct offer was not found?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Either your official site is not in Google's provider list for that market and date, or its provider name does not contain the text you passed to the script. Read the provider names the script prints, adjust the text, and test another market with countryCode if your listing only shows there."
      }
    },
    {
      "@type": "Question",
      "name": "How often should I run a parity check?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Weekly is a practical rhythm for a report you take to an OTA, because it gives enough dates to show a pattern without flooding the manager with noise. Run it over the next 30 to 60 check-in dates, keep each week's output, and compare weeks to see whether a gap was fixed."
      }
    },
    {
      "@type": "Question",
      "name": "Is the 61% figure my hotel's undercut rate?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. It is an average from Triptease's own Google Hotel Ads data for January to May 2022, counting a hotel as undercut if at least one OTA was cheaper. Your own rate depends on your market, your OTAs and your dates, which is why the useful number is the one you measure with the script above."
      }
    }
  ]
}
</script>
