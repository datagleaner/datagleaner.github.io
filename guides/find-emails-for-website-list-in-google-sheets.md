---
title: "Find Emails for a List of Websites in Google Sheets (Apps Script)"
description: "Google Sheets email finder for a column of domains: an Apps Script that calls a scraper API and fills best email, deliverable flag, phones and LinkedIn, with batching and cost."
---

# Find emails for a list of websites in Google Sheets

To find emails from a website list in Google Sheets, put the domains in column A and run a short Google Apps Script that sends them in batches to a scraper API, then writes the best email, a deliverable flag, phone numbers and the LinkedIn page into the next columns. The script below does that with the [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper), and the API token stays in Script Properties, never in a cell.

Disclosure: that Actor is ours (Data Gleaner). The script is a plain HTTP call, so you can point it at any similar API by changing the URL and the field mapping. If your list is small and the sites are simple, the free [IMPORTXML formula](google-sheets-extract-email-from-website-url) may be enough.

## What you get

Row 1 is a header. Column A holds one domain or URL per row (`example.com` and `https://www.example.com/` both work). The script fills:

| Column | Header | Source field | Example |
| --- | --- | --- | --- |
| B | bestEmail | `bestEmail` | `sales@stripe.com` |
| C | deliverable | `bestEmailDeliverable` | `yes`, `no` or `unknown` (the domain's mail setup, not a mailbox check) |
| D | phones | `phoneList` (first 3, E.164) | `+18889262289` |
| E | linkedin | `linkedin` (first one) | `https://www.linkedin.com/company/stripe` |
| F | status | `status` | `ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl`, `error` |

`deliverable` says whether the email's domain accepts mail. It does not prove the specific mailbox exists.

## Step 1: get an Apify token

Create a free Apify account and copy your API token from Settings, Integrations. New accounts get a small monthly free credit, which covers a few hundred sites (see the cost table below).

## Step 2: add the script

In your sheet open Extensions, Apps Script, delete the starter code and paste this. Save it, then reload the sheet.

```javascript
var ACTOR_URL = 'https://api.apify.com/v2/acts/datagleaner~website-contact-details-scraper/run-sync-get-dataset-items';
var BATCH_SIZE = 25;

function onOpen() {
  SpreadsheetApp.getUi().createMenu('Email finder')
    .addItem('Set Apify token', 'setToken')
    .addItem('Find emails for column A', 'findEmails')
    .addToUi();
}

function setToken() {
  var ui = SpreadsheetApp.getUi();
  var res = ui.prompt('Paste your Apify API token');
  if (res.getSelectedButton() === ui.Button.OK && res.getResponseText().trim()) {
    PropertiesService.getScriptProperties().setProperty('APIFY_TOKEN', res.getResponseText().trim());
  }
}

function buildPayload(sites) {
  return { websites: sites, maxPagesPerSite: 5, respectRobotsTxt: true };
}

function rowFor(site, item) {
  if (!item) return ['', '', '', '', 'no result'];
  var email = item.bestEmail || '';
  var deliverable = email ? (item.bestEmailDeliverable === true ? 'yes' : item.bestEmailDeliverable === false ? 'no' : 'unknown') : '';
  var phones = (item.phoneList || []).slice(0, 3).join(', ');
  var linkedin = (item.linkedin || [])[0] || '';
  return [email, deliverable, phones, linkedin, item.status];
}

function rowsFromItems(sites, items) {
  var byWebsite = {};
  items.forEach(function (it) { byWebsite[it.website] = it; });
  return sites.map(function (s) { return rowFor(s, byWebsite[s]); });
}

function findEmails() {
  var token = PropertiesService.getScriptProperties().getProperty('APIFY_TOKEN');
  if (!token) { SpreadsheetApp.getUi().alert('Run "Set Apify token" first.'); return; }
  var sheet = SpreadsheetApp.getActiveSheet();
  var last = sheet.getLastRow();
  if (last < 2) return;
  sheet.getRange(1, 2, 1, 5).setValues([['bestEmail', 'deliverable', 'phones', 'linkedin', 'status']]);

  var started = Date.now();
  var todo = [];  // rows with a site in A and no status in F yet
  var data = sheet.getRange(2, 1, last - 1, 6).getValues();
  data.forEach(function (r, i) {
    var site = String(r[0]).trim();
    if (site && !r[5]) todo.push({ row: i + 2, site: site });
  });

  for (var i = 0; i < todo.length; i += BATCH_SIZE) {
    if (Date.now() - started > 4.5 * 60 * 1000) break;  // Apps Script stops a run at 6 minutes; run the menu item again to continue
    var batch = todo.slice(i, i + BATCH_SIZE);
    var sites = batch.map(function (b) { return b.site; });
    var resp = UrlFetchApp.fetch(ACTOR_URL, {
      method: 'post',
      contentType: 'application/json',
      headers: { Authorization: 'Bearer ' + token },
      payload: JSON.stringify(buildPayload(sites)),
      muteHttpExceptions: true
    });
    var code = resp.getResponseCode();
    if (code !== 200 && code !== 201) {
      batch.forEach(function (b) { sheet.getRange(b.row, 6).setValue('HTTP ' + code); });
      continue;
    }
    var rows = rowsFromItems(sites, JSON.parse(resp.getContentText()));
    batch.forEach(function (b, k) { sheet.getRange(b.row, 2, 1, 5).setValues([rows[k]]); });
    SpreadsheetApp.flush();
  }
}
```

The first time you run **Email finder, Find emails for column A**, Google asks you to authorize the script (it needs your sheet and permission to call an external URL).

### The request body, verbatim

`buildPayload` sends exactly this JSON for the sites in a batch (shown for five test domains):

```json
{"websites":["appier.com","stripe.com","sakura.ad.jp","https://www.hubspot.com/","nonexistent-domain-xyz123.com"],"maxPagesPerSite":5,"respectRobotsTxt":true}
```

The field names come from the Actor's input schema. `maxPagesPerSite` is 1 to 30 (default 8); 5 is enough for the home page plus the contact, about and legal pages. Add `"defaultCountry": "US"` if your list is mostly one country and local phone numbers are being missed.

## How the script behaves

- **Batching.** It sends 25 domains per call. The `run-sync-get-dataset-items` endpoint holds the connection open until the run ends and Apify stops waiting after 300 seconds, so a small batch keeps each call well under that limit. In our test, 5 sites came back in about 14 seconds.
- **Resuming.** Rows with a value in column F are skipped. If a run hits the time guard, a network error or an HTTP error (written as `HTTP 408` and so on in F), clear those F cells and run the menu item again.
- **Matching rows.** The Actor returns one row per website with the `website` value exactly as you gave it, and the script matches on that string, not on order. Do not edit column A while it runs.
- **Token.** It lives in Script Properties (`APIFY_TOKEN`) and is sent in an `Authorization` header, so it is not in a cell, a URL or the sheet's version history. Do not share the Apps Script project with people who should not use your Apify credit.

## What it costs

The Actor charges US$0.002 per website where at least one email is found. Sites with no email, unreachable sites, sites blocked by robots.txt and invalid URLs are free, so you pay only for rows that fill column B.

| Sites in column A | Sites with an email found | Cost |
| --- | --- | --- |
| 100 | 40 | US$0.08 |
| 500 | 200 | US$0.40 |
| 1,000 | 400 | US$0.80 |
| 1,000 | 1,000 (best case) | US$2.00 |
| 10,000 | 4,000 | US$8.00 |

The "40%" hit rate is an assumption for illustration; the real rate depends on your niche and country, so run 100 rows first and count column B. Apify's monthly free credit applies before you pay anything. In our 5-domain test, 2 sites returned an email, so only those 2 were billed (US$0.004).

## Test result

We could not run Apps Script itself in this test. Instead we ran the same `buildPayload` and `rowsFromItems` functions, unchanged, in Node.js against the live API with the request above (HTTP 201, 5 rows back). The mapped columns:

| Site (A) | bestEmail (B) | deliverable (C) | phones (D) | linkedin (E) | status (F) |
| --- | --- | --- | --- | --- | --- |
| appier.com | contactus-hk@appier.com | yes | +886287802800, +81364356617, +82221358266 | linkedin.com/company/appier | ok |
| stripe.com | sales@stripe.com | yes | +18889262289 | linkedin.com/company/stripe | ok |
| sakura.ad.jp | | | +81664768790 | | ok |
| https://www.hubspot.com/ | | | +48800013088, +886277374415, +82220236280 | linkedin.com/company/hubspot | ok |
| nonexistent-domain-xyz123.com | | | | | unreachable |

Two of five sites gave an email. sakura.ad.jp and hubspot.com returned phones (and a LinkedIn page for HubSpot) but no email address in the pages read, which is a normal outcome and is not billed. The `UrlFetchApp` call and sheet writes are standard Apps Script; if you hit a problem there, the examples repo has the same request in Python, Node.js and curl: [datagleaner/website-email-finder](https://github.com/datagleaner/website-email-finder).

## Honest limits

- **Empty means empty.** A site that shows no email, or only a contact form, returns an empty `bestEmail`. The `contactForms` field in the full output lists forms, but this script does not copy it.
- **Bots are blocked.** The Actor uses plain HTTP, no browser. Sites that answer 403 or 429, sit behind a challenge page, or build their contact details with JavaScript come back as `unreachable` or with nothing found. Emails drawn as images are not read.
- **Generic inboxes.** Many results are `info@` or `sales@` addresses the company publishes, not a named person.
- **`deliverable` is domain-level.** `yes` means the domain accepts mail; it is not a mailbox check, so keep bounce handling on.
- **Apps Script quotas.** A run stops at 6 minutes and consumer accounts have a daily URL-fetch quota (see Google's current quotas page), so very long lists take several runs or days. Above a few thousand rows, run the Actor from Python or the Apify Console and import the CSV.
- **Use the data responsibly.** Check GDPR, CCPA and anti-spam rules for your country and recipients before emailing anyone.

## FAQ

**How do I find emails from a list of websites in Google Sheets?**
Put the domains in column A, add the Apps Script above, store your Apify token with the menu item, and run "Find emails for column A". It fills best email, deliverable flag, phones, LinkedIn and status in columns B to F.

**Is there a Google Sheets email finder add-on for this?**
This guide uses a script instead of an add-on, so you can read all of the code and the token never leaves your own Google account and Apify account.

**Can Apps Script scrape emails from a website without an API?**
Yes, for simple pages: `UrlFetchApp.fetch(url)` plus a regular expression finds `mailto:` links, but it reads one page, does not run JavaScript and often gets blocked. The [IMPORTXML formula guide](google-sheets-extract-email-from-website-url) covers the no-script options.

**What does it cost to find emails for 1,000 websites?**
Up to US$2.00, and less in practice, because only sites where an email is found are billed (US$0.002 each).

**Why are some rows empty?**
The site published no email in the pages read, blocked the request, was unreachable or has a bad URL. Column F says which.

## Related guides

- [Google Sheets: extract email from a website URL](google-sheets-extract-email-from-website-url): the free formula route.
- [Find emails for Google Maps leads from the website column](enrich-google-maps-leads-with-emails)
- [Website contact and email scrapers compared: price per 1,000 sites](website-email-scraper-price-comparison)
- [Find emails from a list of websites (API and Python)](../contact-and-lead-scrapers)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I find emails from a list of websites in Google Sheets?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Put the domains in column A, add an Apps Script that posts them in batches to a scraper API, store your Apify token in Script Properties, and run the menu item. It fills best email, deliverable flag, phones, LinkedIn and status in columns B to F."
      }
    },
    {
      "@type": "Question",
      "name": "Can Apps Script scrape emails from a website without an API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, for simple pages: UrlFetchApp.fetch(url) plus a regular expression finds mailto: links, but it reads one page, does not run JavaScript and often gets blocked."
      }
    },
    {
      "@type": "Question",
      "name": "What does it cost to find emails for 1,000 websites?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Up to US$2.00, and less in practice, because only sites where an email is found are billed at US$0.002 each."
      }
    },
    {
      "@type": "Question",
      "name": "Why are some rows empty?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The site published no email in the pages read, blocked the request, was unreachable or has a bad URL. The status column says which."
      }
    }
  ]
}
</script>
