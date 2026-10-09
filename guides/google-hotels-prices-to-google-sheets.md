---
title: "Google Hotels Prices Into Google Sheets (Daily, No Extension)"
description: "Put Google Hotels prices into Google Sheets on a schedule: an Apps Script that appends dated rows, IMPORTDATA, the Apify route, and a price-history chart."
---

# How to get Google Hotels prices into Google Sheets

To get Google Hotels prices into Google Sheets, run a scraper on a schedule and have an Apps Script append one dated row per hotel each day; no browser extension is needed. Sheets has no built-in function that reads Google Hotels, and Google Hotels has no export button, so something outside the sheet has to fetch the prices. This page covers three ways to do the Sheets half: an Apps Script that appends rows daily (the only one that keeps a history), `IMPORTDATA` for a quick snapshot, and Apify's scheduler with a Google Sheets Actor. It ends with a sheet layout that charts price history and shows the cheapest provider. For the tracking method itself, see the [price tracking guide](track-hotel-prices-google-hotels).

Disclosure: Data Gleaner, mentioned in the second half as the source of the price data, is us. Google Sheets and Apps Script are free to use; the scraper that supplies the prices costs $0.003 per hotel.

## What a sheet can and cannot do on its own

- **Formulas cannot read Google Hotels.** `IMPORTXML` and `IMPORTHTML` fetch a page's HTML, and Google Hotels builds its results with scripts, so a formula does not reliably return the prices you see in a browser.
- **Prices belong to a stay.** A price is for one check-in date, one check-out date, a number of adults, a currency and a market. A history only makes sense if those stay fixed, so keep one stay per data tab.
- **A formula shows the present; a history needs rows.** `IMPORTDATA` re-reads its URL, so the cells show whatever the file says now. To keep yesterday's prices, something has to append rows, which is what Method 1 does.

Every method below uses the same price source: a scraper that returns one record per hotel with the lowest price and each booking site's offer. To write that part yourself, [the Python guide](scrape-google-hotels-python) has a plain-HTTP script. The Actor described later is the same thing as a hosted job, which lets Sheets call it with one web request.

## Method 1: an Apps Script that appends dated rows

Apps Script is built into Google Sheets (Extensions, then Apps Script). Its `UrlFetchApp` can call a web API, and a time-driven trigger can run a function every day. The script below calls the Apify API, waits for the run and appends one row per hotel, so the sheet becomes the price history. It uses Apify's "run Actor synchronously and get dataset items" endpoint, and Apify's documentation says a synchronous run longer than 300 seconds returns HTTP 408, so keep the hotel count small. For a bigger list, use Method 3.

1. Create a sheet, rename the first tab `Data`, and put these headers in row 1: `scrapedAt`, `hotel`, `checkIn`, `checkOut`, `currency`, `lowestPrice`, `cheapestProvider`, `rating`.
2. Open Extensions, then Apps Script, and paste the code below.
3. In Project Settings, add a script property named `APIFY_TOKEN` with your Apify API token, so the token is not written in the code.
4. Run `appendPrices` once by hand and approve the permissions Google asks for. Check that rows appear.
5. Run `createDailyTrigger` once. From then on the function runs every day at a random minute between 06:00 and 07:00 in the script's time zone.

```javascript
const INPUT = {
  locations: ["Lisbon"],
  checkIn: "2026-11-10",     // fixed dates, not "30 days from today"
  checkOut: "2026-11-12",
  adults: 2,
  currency: "USD",
  maxHotelsPerLocation: 20,
  includeDetails: true,
  countryCode: "us",
};

function appendPrices() {
  const token = PropertiesService.getScriptProperties().getProperty("APIFY_TOKEN");
  const url = "https://api.apify.com/v2/acts/datagleaner~google-hotels-scraper/run-sync-get-dataset-items";
  const res = UrlFetchApp.fetch(url, {
    method: "post",
    contentType: "application/json",
    headers: { Authorization: "Bearer " + token },
    payload: JSON.stringify(INPUT),
    muteHttpExceptions: true,
  });
  if (res.getResponseCode() >= 300) {
    throw new Error("Apify returned " + res.getResponseCode() + ": " + res.getContentText().slice(0, 300));
  }
  const hotels = JSON.parse(res.getContentText());
  const now = new Date();
  const rows = hotels.map(function (h) {
    return [
      now, h.name, h.checkIn, h.checkOut, h.currency,
      h.lowestPrice == null ? "" : h.lowestPrice,
      h.cheapestProvider || "",
      h.rating == null ? "" : h.rating,
    ];
  });
  if (!rows.length) return;
  const sheet = SpreadsheetApp.getActive().getSheetByName("Data");
  sheet.getRange(sheet.getLastRow() + 1, 1, rows.length, rows[0].length).setValues(rows);
}

function createDailyTrigger() {
  ScriptApp.newTrigger("appendPrices").timeBased().everyDays(1).atHour(6).create();
}
```

This script was checked against a sample Actor response, not a live sheet, so test it with one hotel first (`maxHotelsPerLocation: 1`) and read the row it writes before you leave it on a trigger. Apps Script stops any single execution after 6 minutes, which is longer than Apify's 300-second synchronous limit, so the Apify limit is the one you hit first.

Things to know: the token is a password, so keep it in script properties and not in a cell. After the check-in date passes, Google no longer prices that stay and the run fails, so change the dates for the next trip or start a new tab. Hotels without availability come back with an empty price, which the script writes as an empty cell, leaving a gap in the chart instead of a zero.

## Method 2: IMPORTDATA for a quick snapshot

`IMPORTDATA(url)` imports a `.csv` or `.tsv` file from a URL into a sheet, according to Google's documentation, and the URL can sit in a cell. Apify can return a dataset as CSV with `format=csv`, and a `fields` list keeps only the columns you ask for. Apify also has a shortcut to the dataset of an Actor's most recent run, so the URL does not change from day to day:

```
https://api.apify.com/v2/actors/datagleaner~google-hotels-scraper/runs/last/dataset/items?format=csv&fields=name,checkIn,checkOut,currency,lowestPrice,cheapestProvider,rating,scrapedAt&clean=1&status=SUCCEEDED&token=YOUR_APIFY_TOKEN
```

Put that URL in a cell, for example `A1`, and in `A3` write `=IMPORTDATA(A1)`. Schedule the Actor in Apify (see Method 3) so a fresh run exists, and the sheet shows the latest hotels.

Limits you should plan around:

- **No history.** The cells show the last run, and the next run replaces them. To keep a record you must copy the values somewhere, or use Method 1.
- **The token sits in the URL.** Anyone who can open the sheet can read the cell and use your token. Share such a sheet only with people you would trust with the token.
- **Choose flat fields.** Each hotel record also holds a nested `offers` list, which does not fit a flat CSV. List the plain fields you need in `fields`; the cheapest booking site is already a plain field, `cheapestProvider`.
- **Refresh timing is Google's.** Google does not document how often `IMPORTDATA` re-fetches, so you cannot set the time of day. Method 1 runs when you choose.

## Method 3: Apify's scheduler plus a Google Sheets Actor

If you would rather not maintain a script, Apify can run the scraper on a schedule and a community Actor can write the results to your sheet. Apify's documentation describes schedules like this:

1. Run the Actor once by hand. An Actor must have been run at least once before you can schedule it.
2. In the Apify Console, go to **Schedules** and click **Create new**. Name it (3 to 63 characters), choose the frequency with the setup tool, and use **Add** to attach the Actor and its input. A schedule uses a cron expression whose first field, seconds, is optional, and `@daily` (midnight) is one of the shortcuts. Each schedule has its own time zone.
3. New schedules start disabled, so click **Enable**.

For the write step, Apify's Store has [Google Sheets Import & Export](https://apify.com/lukaskrivka/google-sheets) (`lukaskrivka/google-sheets`, not ours). Its listing describes an `append` mode that "adds new data as additional rows below the old rows already present in the sheet", a `replace` mode, and inputs including `datasetId`, `spreadsheetId`, `range`, `deduplicateByField` and `transformFunction`. You connect your Google account in its input. It has no fee of its own beyond Apify platform usage, and it lists limits of 60 sheet reads or updates per minute and 5 million cells per spreadsheet. I did not find a page describing how to chain it after another Actor, so check its listing for the current steps. The records contain a nested `offers` list, so test a small run and look at what lands in the sheet before you schedule it.

## A sheet layout with a price-history chart and lowest provider

With Method 1 the `Data` tab fills up by itself. Add a second tab called `Chart`:

1. In `Chart!B1` type the exact hotel name from the `hotel` column. Optionally add a drop-down: select the cell, open Data, then Data validation, and choose the range `Data!B2:B`.
2. In `Chart!A3` enter this formula. It pulls one hotel's dates, lowest prices and cheapest provider, oldest first:

```
=QUERY(Data!A2:H, "select A, F, G where B = '" & B1 & "' order by A", 0)
```

3. Select the first two columns of the result (date and price), insert a chart (Insert, then Chart) and set the type to a line chart. The cheapest provider for each day stays beside it as a table.
4. For the lowest price seen so far, add `=MIN(FILTER(Data!F2:F, Data!B2:B = B1))`.

A hotel name with an apostrophe breaks the query string above. Adjust the column letters if you add columns.

## The Data Gleaner Actor

[Google Hotels Scraper](https://apify.com/datagleaner/google-hotels-scraper) is our Apify Actor. It reads Google Hotels over plain HTTP, with no browser, login or Google account, and returns one record per hotel: the lowest price for your dates, the cheapest booking site (`cheapestProvider`), every booking site's per-night and total price with a link, rating, review count, star class, coordinates and a `scrapedAt` timestamp. Its proxy setting is off by default.

It costs **$3.00 per 1,000 hotels** ($0.003 per hotel), billed only for hotels pushed to the dataset. Tracking 20 hotels once a day for 30 days is 600 hotels, or $1.80. A 10-hotel test run costs $0.03. You also need a free Apify account and its API token.

The input fields are `locations`, `hotelUrls`, `checkIn` and `checkOut` (`YYYY-MM-DD`), `adults` (1 to 8), `currency`, `maxHotelsPerLocation` (default 20), `includeDetails` (default on, and needed for provider offers), `minRating`, `hotelClass`, `minPrice`, `maxPrice`, `sortBy`, `language`, `countryCode`, `requestDelaySecs` and `proxyConfiguration`. To track specific hotels instead of a city, put their Google Hotels URLs or `entityId` values in `hotelUrls`.

If you prefer to fetch the data from Python and write the sheet yourself:

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
    "maxHotelsPerLocation": 20,
    "includeDetails": True,
})
if run is None:
    raise SystemExit("[ERROR] The Actor run did not return.")

for hotel in client.dataset(run.default_dataset_id).iterate_items():
    print(hotel["scrapedAt"], hotel["name"], hotel.get("lowestPrice"), hotel.get("currency"), hotel.get("cheapestProvider"))
```

Limits: Google lists around 500 distinct hotels per query at most, prices differ by market (set `countryCode`), dates must be today or later, and at very high volume Google may start refusing requests, in which case you can enable `proxyConfiguration`. Reviews and room types are not included.

## FAQ

**Can Google Sheets pull Google Hotels prices directly?**
Not with a built-in function. `IMPORTXML` and `IMPORTHTML` read page HTML, and Google Hotels builds its prices with scripts, so they are not a reliable route. The working approach is to get the prices from a scraper or API and have a script, an `IMPORTDATA` formula or an integration put them in the sheet.

**How do I keep a price history instead of overwriting the cells?**
Append rows instead of refreshing a formula. A scheduled Apps Script that adds one dated row per hotel each day does this, as does the Google Sheets Import & Export Actor in `append` mode. `IMPORTDATA` only shows the current contents of a file, so on its own it keeps no history.

**Do I need a browser extension?**
No. The methods here run on Google's and Apify's servers on a schedule, so the sheet updates while your computer is off. Nothing has to be open in your browser.

**What does it cost to track hotels in a sheet every day?**
Google Sheets and Apps Script are free to use within Google's quotas. The prices come from the scraper, which costs $0.003 per hotel with Data Gleaner's Actor, so 20 hotels once a day for 30 days is $1.80. The Google Sheets Actor in Method 3 adds its own Apify platform usage.

**How do I find the cheapest booking site for each hotel?**
Read the `cheapestProvider` field, which names the booking site with the lowest price for your dates, and write it in a column next to `lowestPrice`, as the script above does. For the full comparison, each record's `offers` list has every booking site Google shows with its per-night price, total price and link; it needs `includeDetails` on, which is the default.

**Why did my sheet stop updating?**
The usual cause is a check-in date that has passed, because Google no longer prices a stay in the past. Other causes are an expired or wrong token, a synchronous call that ran past 300 seconds, or a disabled Apify schedule. The error in the Apps Script execution log or the Apify run log says which.

## Related guides

- [How to track hotel prices on Google Hotels](track-hotel-prices-google-hotels): the free price tracking toggle, and when you need the prices as data.
- [How to scrape Google Hotels with Python](scrape-google-hotels-python): the plain-HTTP script behind the Actor.
- [Google Sheets: extract email from a website URL](google-sheets-extract-email-from-website-url): another Sheets formula and Apify route.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Can Google Sheets pull Google Hotels prices directly?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not with a built-in function. IMPORTXML and IMPORTHTML read page HTML, and Google Hotels builds its prices with scripts, so they are not a reliable route. The working approach is to get the prices from a scraper or API and have a script, an IMPORTDATA formula or an integration put them in the sheet."
      }
    },
    {
      "@type": "Question",
      "name": "How do I keep a price history instead of overwriting the cells?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Append rows instead of refreshing a formula. A scheduled Apps Script that adds one dated row per hotel each day does this, as does the Google Sheets Import & Export Actor in append mode. IMPORTDATA only shows the current contents of a file, so on its own it keeps no history."
      }
    },
    {
      "@type": "Question",
      "name": "Do I need a browser extension?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. The methods here run on Google's and Apify's servers on a schedule, so the sheet updates while your computer is off. Nothing has to be open in your browser."
      }
    },
    {
      "@type": "Question",
      "name": "What does it cost to track hotels in a sheet every day?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Google Sheets and Apps Script are free to use within Google's quotas. The prices come from the scraper, which costs $0.003 per hotel with Data Gleaner's Actor, so 20 hotels once a day for 30 days is $1.80. The Google Sheets Actor in Method 3 adds its own Apify platform usage."
      }
    },
    {
      "@type": "Question",
      "name": "How do I find the cheapest booking site for each hotel?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Read the cheapestProvider field, which names the booking site with the lowest price for your dates, and write it in a column next to lowestPrice, as the script above does. For the full comparison, each record's offers list has every booking site Google shows with its per-night price, total price and link; it needs includeDetails on, which is the default."
      }
    },
    {
      "@type": "Question",
      "name": "Why did my sheet stop updating?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The usual cause is a check-in date that has passed, because Google no longer prices a stay in the past. Other causes are an expired or wrong token, a synchronous call that ran past 300 seconds, or a disabled Apify schedule. The error in the Apps Script execution log or the Apify run log says which."
      }
    }
  ]
}
</script>
