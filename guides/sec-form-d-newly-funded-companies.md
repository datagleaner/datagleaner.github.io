---
title: "SEC Form D: How to Find Companies That Just Raised Money (Python and API)"
description: "How to find companies that just raised money from SEC Form D filings: EDGAR search, the daily index, a tested Python script, and an API with phone and named executives."
---

# SEC Form D: how to find companies that just raised money

To find companies that just raised money, read the SEC's Form D filings. A US company that sells securities in a private raise files a Form D with the SEC within 15 days of the first sale, and the filing is public on EDGAR, usually the same day. It names the company, the amount sold, the number of investors, a phone number and the executives and directors. This guide shows three ways to read it: EDGAR's own search, a short Python script that uses EDGAR's daily index, and an API for when you want filtered rows with no code.

Disclosure: Data Gleaner is us. The first two methods are free and need nothing from us.

## What a Form D contains

A Form D is a short notice, filed as an XML document (`primary_doc.xml`) on EDGAR. It is not a prospectus and it has no business description.

| Section | What it gives you |
|---|---|
| Issuer | Company name, CIK, state and year of incorporation, entity type, address and **phone number** |
| Related persons | Executive officers, directors and promoters, with their relationship to the company and a city |
| Offering | Industry group, total offering amount, **amount sold**, amount remaining, number of investors, date of first sale, type of securities, the exemption claimed (for example Rule 506(b) or 506(c)) |
| Sales compensation | Brokers paid to sell the round, if any |
| Signature | The person who signed, with a title, usually the CEO or CFO |

What it does not contain: an email address, a valuation, investor names, or a description of what the company does. The only sector signal is the industry group, a fixed list that includes "Other Technology", "Biotechnology", "Commercial" (real estate) and "Pooled Investment Fund". Treat the amount sold as the money raised so far, not the round size, and remember that a company can leave some fields blank or mark the offering amount as indefinite.

## Method 1: EDGAR full-text search (free, by hand)

1. Open [EDGAR full-text search](https://www.sec.gov/edgar/search/), choose **Form D** under the form filter and set a date range.
2. Type a keyword. It matches the text of the filing, which means company names, people and cities, not what the company does.
3. Open a result and read the filing page. The **Index** page links the structured XML.

This is right for checking a company you already know about or for a few filings. It does not scale: there is no export, no filter on amount or on the funds that dominate the list, and each result needs a click.

## Method 2: the daily index and a short Python script (free, tested)

EDGAR publishes a daily index of every filing by form type. The script below lists the day's Form D filings, then reads each filing's XML. It skips amendments (form type `D/A`), because the index lists them under a different form type, and it waits between requests because SEC's fair-access rules allow at most 10 requests a second and require a `User-Agent` that identifies you.

```python
import re, time, sys, urllib.request
import xml.etree.ElementTree as ET

UA = {"User-Agent": "YourName your-email@example.com"}  # SEC asks for a real contact

def get(url):
    with urllib.request.urlopen(urllib.request.Request(url, headers=UA), timeout=30) as r:
        return r.read().decode("utf-8", "replace")

def form_d_for_day(day):                       # day = "20261008"
    q = (int(day[4:6]) - 1) // 3 + 1
    idx = get(f"https://www.sec.gov/Archives/edgar/daily-index/{day[:4]}/QTR{q}/form.{day}.idx")
    for line in idx.splitlines():
        if line.startswith("D "):               # "D/A" amendments start with "D/A", so they are skipped
            m = re.search(r"edgar/data/(\d+)/(\d{10}-\d{2}-\d{6})\.txt", line)
            if m:
                yield m.group(1), m.group(2)

def parse(cik, acc):
    base = f"https://www.sec.gov/Archives/edgar/data/{cik}/{acc.replace('-', '')}"
    root = ET.fromstring(get(base + "/primary_doc.xml"))
    t = lambda path: (root.findtext(path) or "").strip()
    people = [
        " ".join(filter(None, [p.findtext("relatedPersonName/firstName"), p.findtext("relatedPersonName/lastName")]))
        for p in root.findall("relatedPersonsList/relatedPersonInfo")
    ]
    return {
        "company": t("primaryIssuer/entityName"),
        "industry": t("offeringData/industryGroup/industryGroupType"),
        "phone": t("primaryIssuer/issuerPhoneNumber"),
        "state": t("primaryIssuer/issuerAddress/stateOrCountry"),
        "sold": t("offeringData/offeringSalesAmounts/totalAmountSold"),
        "investors": t("offeringData/investors/totalNumberAlreadyInvested"),
        "people": people,
        "url": base + "/",
    }

if __name__ == "__main__":
    for cik, acc in list(form_d_for_day(sys.argv[1]))[:int(sys.argv[2])]:
        print(parse(cik, acc))
        time.sleep(0.2)                          # stay well under SEC's 10 requests a second
```

Run `python3 formd.py 20261008 4` to print four filings from 8 October 2026. On that day the index listed 215 Form D filings. Reading all of them gave this split:

| Group | Filings |
|---|---|
| Industry group "Pooled Investment Fund" | 143 (66%) |
| Everything else (operating companies and other issuers) | 72 (34%) |
| Of the 72, filings that name at least one related person | 72 |

So about two thirds of a normal day is hedge funds, venture and private-equity vehicles and SPVs. That is useful if you sell to fund managers and noise if you want startups. Filter on `industry` to drop them. Two details the script shows: phone numbers come in whatever format the filer typed, and `state` can be a country code (`F4` in the output above is Hong Kong), so normalise both before you load them into a CRM.

Limits of this approach: you write and maintain the parsing, a day with no filings (weekends, US holidays) returns an error for the index file, and the index has no keyword search, so you filter yourself. SEC also publishes quarterly Form D data sets as bulk files, which suit historical analysis better than a daily feed.

## Method 3: an API that returns filtered rows

If you want the filtering done for you, the [Form D Funding Leads](https://apify.com/datagleaner/form-d-funding-leads) Actor on the Apify Store runs EDGAR full-text searches for your keywords, states, industries or company names, reads each filing's XML, and returns one row per filing. It drops funds and amendments by default, matches sector words such as `software` or `fintech` against the Form D industry groups as well as the text, and can keep only raises above an amount. The same engine runs behind the three example tasks below.

Price: US$0.01 per filing that names an executive or director. A filing with no named person or usable phone, and any error row, is free. Ten qualified filings cost US$0.10.

Input for software and fintech companies in Texas and California that sold at least US$1 million in the last 14 days:

```json
{
  "keywords": ["software", "fintech"],
  "states": ["TX", "CA"],
  "daysBack": 14,
  "minAmountSold": 1000000,
  "maxFilings": 200
}
```

Each row has the company, CIK, phone, address, industry group, offering and sold amounts, number of investors, date of first sale, exemptions, the executives (name, relationship, title, city), the signer, and links to the filing and its XML. Fields a filing leaves blank are empty, never invented.

Run one of the public example tasks without writing any input:

- [Software startups that just raised money (SEC Form D)](https://apify.com/datagleaner/form-d-funding-leads/newly-funded-software-startups-this-week)
- [Biotech companies that just raised capital (Form D leads)](https://apify.com/datagleaner/form-d-funding-leads/biotech-funding-rounds-form-d-leads)
- [Texas companies that raised over 1 million USD (Form D)](https://apify.com/datagleaner/form-d-funding-leads/texas-companies-raising-over-1m-form-d)

## Which method to use

| Need | Use |
|---|---|
| Check one company's filings | EDGAR full-text search |
| A free daily list you will filter and enrich yourself | The Python script |
| Recent raises by sector, state and size, as CSV, JSON or an agent tool call | The Actor |
| Years of history for analysis | SEC's quarterly Form D data sets |
| Rounds that include named investors, valuations or press coverage | A funding database such as Crunchbase or PitchBook; Form D has none of these |

## Using the data responsibly

Form D is public record, but the people on it are individuals. Use the company-level facts freely. Contact a company at the phone number it filed or through its own website, give people a way to opt out, and follow the rules for cold email and calls in the countries you write to (CAN-SPAM, GDPR and PECR, and do-not-call rules). The filing does not tell you whether the raise succeeded, only that a sale happened, and a few companies file for an offering that never closes.

## FAQ

**How do I find companies that recently raised money?**
Filter SEC EDGAR for Form D filings in the last few days, drop the "Pooled Investment Fund" industry group, and read the amount sold, phone and named executives. The script above does it for free; the Actor does it with filters.

**How soon after a raise does the Form D appear?**
The deadline is 15 days after the first sale, and many companies file near it, so a Form D can lag the closing by up to two weeks. Once filed, EDGAR shows it the same day.

**Does a Form D include an email address?**
No. It has the company phone and address and the names of its executives. Find an email from the company's website, for example with the [Company Contact Details Actor](../contact-and-lead-scrapers).

**Is a Form D the same as a funding announcement?**
No. It is a regulatory notice that a private sale of securities happened. It covers equity, debt and funds, and it says nothing about valuation or who invested.

**Can I get Form D data through an official API?**
SEC offers EDGAR's full-text search, the daily and quarterly indexes and bulk Form D data sets, all free. It does not offer a filtered "recent startup raises" feed; that is what third-party tools build on top.

## Related

- [Find emails from a list of websites](../contact-and-lead-scrapers), to add an email to each company
- [Enrich a list of company domains](clearbit-alternative-company-enrichment-by-domain)
- [Company Leadership & Team Page Scraper](https://apify.com/datagleaner/team-page-contacts), for the people behind each company

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {"@type": "Question", "name": "How do I find companies that recently raised money?", "acceptedAnswer": {"@type": "Answer", "text": "Filter SEC EDGAR for Form D filings in the last few days, drop the Pooled Investment Fund industry group, and read the amount sold, phone and named executives. A short Python script on EDGAR's daily index does it for free; the Form D Funding Leads Actor does it with filters."}},
    {"@type": "Question", "name": "How soon after a raise does the Form D appear?", "acceptedAnswer": {"@type": "Answer", "text": "The deadline is 15 days after the first sale, so a Form D can lag the closing by up to two weeks. Once filed, EDGAR shows it the same day."}},
    {"@type": "Question", "name": "Does a Form D include an email address?", "acceptedAnswer": {"@type": "Answer", "text": "No. It has the company phone and address and the names of its executives and directors. Find an email from the company's own website."}},
    {"@type": "Question", "name": "Is a Form D the same as a funding announcement?", "acceptedAnswer": {"@type": "Answer", "text": "No. It is a regulatory notice that a private sale of securities happened. It covers equity, debt and funds, and says nothing about valuation or investors."}},
    {"@type": "Question", "name": "Can I get Form D data through an official API?", "acceptedAnswer": {"@type": "Answer", "text": "SEC offers EDGAR full-text search, daily and quarterly indexes and bulk Form D data sets, all free. It does not offer a filtered feed of recent startup raises."}}
  ]
}
</script>
