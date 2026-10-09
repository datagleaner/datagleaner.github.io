---
title: "How to Track Hotel Prices on Google Hotels (and at Scale)"
description: "Track hotel prices on Google Hotels: turn on the price tracking toggle for email alerts, its limits, and how to track many hotels and dates with scheduled runs."
---

# How to track hotel prices on Google Hotels

To track hotel prices on Google Hotels, sign in to your Google account, search for a hotel with your check-in and check-out dates set, and turn on the price tracking toggle on the hotel's prices. Google then emails you when the price for those dates changes significantly. Tracking a single hotel was added in 2026; before that, Google Hotels tracked only whole city searches, and you may still see that option on a results list. That covers one trip. If you need a price history for many hotels, many date ranges or several markets, you need the prices as data, which this guide also covers.

## Option 1: the Google Hotels price tracking toggle (free)

### Track one hotel

1. Sign in to your Google account. Tracking does not work signed out.
2. Go to [google.com/travel/hotels](https://www.google.com/travel/hotels), or search the hotel's name on Google.
3. Set your check-in and check-out dates and the number of guests. Alerts are for those exact dates.
4. Open the hotel's listing and go to its prices. The toggle sits near the pricing section; on desktop you may need to scroll past the list of booking options.
5. Turn on the price tracking toggle. The exact label has changed between versions, so look for "track" next to the prices.

### Track every hotel in a city

1. Search a place, for example "hotels in Lisbon", with your dates set.
2. At the top of the results list, turn on the price tracking toggle for that search ("Track prices" or "Track all hotels", depending on the version you see).

Some users report that the city-level toggle disappears once a filter (price range, star class, amenities) is applied, so set it on the plain search.

### How the alerts work

- Alerts arrive by email when the price for your dates changes significantly. Google has not said what counts as significant.
- You can track several hotels and searches, and turn each one off from the same toggle.
- Prices come from the hotel and from booking sites together. You cannot choose which sources the alert watches.
- Tracking is tied to fixed dates. There is no "any weekend in March" tracking for hotels.
- Older guides report that the toggle did not appear in some countries. If you do not see it, check that you are signed in and that dates are set.

### What to do with an alert

- If you booked a free-cancellation rate, you can book the lower price and cancel the first booking. Check the cancellation deadline first.
- If your booking came with a best-price guarantee from the hotel or booking site, contact them with the lower price.
- Prepaid, non-refundable rates usually cannot be repriced, so tracking matters most before you book or when you hold a refundable rate.

## Where the toggle stops being enough

The toggle answers "has my trip got cheaper?" It does not give you:

- **A price history.** You get an email when something changes, not a table of prices per day you can chart.
- **Many dates at once.** Comparing the same hotel across 10 different weekends means 10 separate tracked searches.
- **Provider-level prices.** The alert reports a headline price, not each booking site's rate side by side.
- **Many hotels as data.** A revenue manager watching 40 competitor hotels, or a travel app needing prices for a whole city, needs rows in a spreadsheet or database, not emails.

Other free options have the same shape: booking sites and apps offer their own price alerts, and some browser extensions watch the page you are on. All of them notify; none of them hand you the numbers.

For a dataset, the only source is the public Google Hotels results themselves. Google has no public API for reading hotel prices (its hotel APIs are for hotels and booking sites that send prices to Google). So you either record prices by hand on a schedule, or use a scraper that reads the same results a visitor sees and returns them as rows.

## Option 2: collect Google Hotels prices on a schedule

Whatever tool you use, the method is the same:

1. **Fix the variables.** A hotel price only means something next to its check-in and check-out dates, number of adults, currency and Google market (country). Keep all of them the same between runs, or record them with every price.
2. **Pick fixed dates, not relative ones.** "30 days from today" moves every day, so you would be comparing different stays. Use the actual dates you care about, for example 2026-12-18 to 2026-12-20.
3. **Run at the same time each day** and store each run with a timestamp.
4. **Compare per hotel and per provider.** Join runs on a stable hotel ID, then chart the lowest price per night and the spread between booking sites.
5. **Pace your requests.** At high volume Google may start refusing requests, so space them out and collect only what you need.

### Using Data Gleaner's Google Hotels Scraper

Disclosure: Data Gleaner is us. [Google Hotels Scraper](https://apify.com/datagleaner/google-hotels-scraper) is an Actor on the Apify Store that does the collection step. You give it places (`Tokyo`, `Paris 8e`) or Google Hotels hotel URLs, plus your dates, guests, currency and market, and it returns one JSON record per hotel with:

- `lowestPrice` (per night) and `totalPriceDisplay` (whole stay), in your currency
- `offers`: each booking site Google lists, with `provider`, `pricePerNight`, `totalPrice` and a link
- `entityId`, a stable Google Hotels ID to join runs on
- rating, review count, star class, highlight amenities, address and coordinates
- `checkIn`, `checkOut`, `adults`, `pricesForDates` (confirms Google priced your exact dates) and `scrapedAt`

A hotel with no availability on your dates is still returned, with `lowestPrice` empty and no offers, which is itself a useful signal when tracking.

It costs **$3.00 per 1,000 hotels** ($0.003 per hotel), billed only for hotels written to the dataset. From the Actor's pricing: one hotel checked daily for 30 days is 30 results, $0.09; 5 cities at 100 hotels each is 500 hotels, $1.50 a run, or $45 for 30 daily runs. You need an Apify account and its API token, but no Google account.

### Python: one tracking run

`pip install apify-client`, set the `APIFY_TOKEN` environment variable, and run. This tracks 20 hotels in Lisbon for a fixed weekend and prints each hotel's cheapest rate. Change the dates to your own; they must be in the future.

```python
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/google-hotels-scraper").call(run_input={
    "locations": ["Lisbon"],
    "checkIn": "2026-12-18",
    "checkOut": "2026-12-20",
    "adults": 2,
    "currency": "EUR",
    "countryCode": "pt",
    "maxHotelsPerLocation": 20,
    "includeDetails": True,
})
if run is None:
    raise SystemExit("[ERROR] The Actor run did not return.")

for hotel in client.dataset(run.default_dataset_id).iterate_items():
    offers = hotel.get("offers") or []
    providers = ", ".join(f'{o.get("provider")} {o.get("pricePerNight")}' for o in offers[:3])
    print(f'{hotel.get("entityId")} | {hotel.get("name")} | '
          f'{hotel.get("lowestPrice")} {hotel.get("currency")}/night | {providers}')
```

That run returns at most 20 hotels, about $0.06. To watch specific hotels instead of a whole city, put their Google Hotels URLs or the `entityId` values from a previous run in `hotelUrls` and drop `locations`.

### Run it every day

You have two ways to repeat the run:

- **On Apify:** in the Apify Console, save your input as a task for the Actor, then create a Schedule (for example, daily at 07:00) that runs the task. Each run keeps its own dataset, which you can download as CSV or JSON, or read from the API.
- **On your own machine or server:** run the Python script above from cron or Task Scheduler, and append each run's rows to a CSV or database table with the run date.

To track several date ranges, run once per range (each run takes one `checkIn` and `checkOut`) or create one task per range and schedule each.

### Turn the runs into a price history

A simple layout that works in a spreadsheet or any database: one row per hotel per run, with `scrapedAt`, `entityId`, `name`, `checkIn`, `checkOut`, `lowestPrice`, `currency`, and the cheapest provider. Then:

- chart `lowestPrice` over `scrapedAt` for each `entityId` to see the trend for one stay;
- compare `offers` within a run to see which booking site is cheapest and by how much;
- flag rows where `lowestPrice` drops below a threshold you choose, which recreates Google's email alert on your own terms.

## Toggle or scheduled data: which to use

| | Google Hotels toggle | Scheduled scraping (any tool) |
|---|---|---|
| Cost | Free | Paid per result or your own time and servers |
| Setup | Seconds, signed in to Google | A script or a scheduled task |
| Output | Email when the price changes | Rows: price per hotel per run |
| Price history | No | Yes |
| Booking-site prices side by side | No | Yes, with provider offers |
| Many hotels, dates or markets | One tracked search each | One run each, automated |
| Best for | Booking your own trip | Revenue management, research, travel apps |

## Caveats

- **Prices are what Google shows**, for your dates, currency and market. Taxes and fees may or may not be included depending on the market and settings, so compare like with like. Member-only and mobile-only rates may not appear.
- **Booking sites differ by market.** Set `countryCode` (or your Google region) to the market you care about and keep it fixed.
- **Google lists about 500 distinct hotels per query at most.** For a large city, search neighbourhoods separately.
- **Pages change.** Google can change Google Hotels at any time, and any scraper, ours included, can break when it does.
- **Terms.** Scraping reads public pages without logging in. Check that your use fits Google's terms and the providers' terms.

## FAQ

**Does Google Hotels have price tracking?** Yes. Signed in to a Google account, with dates set, you can turn on tracking for one hotel or for a city search, and Google emails you when the price changes for those dates.

**How do I set a price alert on Google Hotels?** Search the hotel or the city with your dates, then turn on the price tracking toggle on the hotel's prices or at the top of the results list. Alerts go to your Google account's email address.

**Can Google Hotels show a hotel's price history?** Not as data you can download. The tracking toggle emails you about changes; it does not keep a table of past prices. For that, record prices on a schedule yourself or with a scraper and keep each run.

**Is there a Google Hotels API for prices?** Not one you can read prices from. Google's hotel APIs are for hotels and booking sites that send prices to Google. Reading prices means using the public Google Hotels results, by hand or with a scraper.

**How often should I check hotel prices?** Once a day is enough for most trips and most competitor tracking. Checking more often than daily rarely changes a booking decision, and checking less often keeps the cost and the load on Google low.

## Related pages

- [Google Hotels, Trends and News scraper APIs](../google-data-scrapers)
- [Scraping Google News with Python](google-news-scraper-python)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Does Google Hotels have price tracking?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. Signed in to a Google account, with dates set, you can turn on tracking for one hotel or for a city search, and Google emails you when the price changes for those dates."
      }
    },
    {
      "@type": "Question",
      "name": "How do I set a price alert on Google Hotels?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Search the hotel or the city with your dates, then turn on the price tracking toggle on the hotel's prices or at the top of the results list. Alerts go to your Google account's email address."
      }
    },
    {
      "@type": "Question",
      "name": "Can Google Hotels show a hotel's price history?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not as data you can download. The tracking toggle emails you about changes; it does not keep a table of past prices. For that, record prices on a schedule yourself or with a scraper and keep each run."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a Google Hotels API for prices?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not one you can read prices from. Google's hotel APIs are for hotels and booking sites that send prices to Google. Reading prices means using the public Google Hotels results, by hand or with a scraper."
      }
    },
    {
      "@type": "Question",
      "name": "How often should I check hotel prices?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Once a day is enough for most trips and most competitor tracking. Checking more often than daily rarely changes a booking decision, and checking less often keeps the cost and the load on Google low."
      }
    }
  ]
}
</script>
