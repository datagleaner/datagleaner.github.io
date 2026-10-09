---
title: "SEEK Job Listings API: How to Get SEEK Jobs as Data"
description: "There is no public SEEK job listings API: SEEK's API is for hirers posting ads. Here is how to get SEEK job ads from Australia and NZ as JSON or CSV."
---

# SEEK job listings API: how to get SEEK jobs as data

SEEK does not offer a public API for reading job listings. The SEEK API documented at developer.seek.com is for recruitment software providers that post job ads and receive applications on behalf of hirers, and access needs SEEK's approval. If you want SEEK job ads from Australia (seek.com.au) or New Zealand (seek.co.nz) as data, your realistic options are: SEEK's own site and job alerts for small manual work, free government and SEEK statistics if you only need counts and trends, your own scraper, or a hosted SEEK scraper that you call through an API. This page covers what each gives you and where it falls short.

## What the official SEEK API does and does not do

SEEK's developer site describes the SEEK API as a way for software providers to "integrate your software with SEEK's employment marketplace". The main use cases it lists are:

| SEEK API use case | What it does |
|---|---|
| Job Posting | Your software posts and manages a hirer's own job ads on SEEK. |
| Optimised Apply | Exports candidate applications from SEEK into recruitment software. |
| Apply with SEEK | Lets a candidate pre-fill an external application form from their SEEK Profile. |
| Ad Performance | Shows how a hirer's ad performs against the market. |

None of these is job search. You cannot query the SEEK API for "all data analyst jobs in Sydney", and you cannot read other employers' ads through it. Every call is made on behalf of a hirer who has agreed to work with your software, and the site states that "Access to the SEEK API requires approval from SEEK" through its Integration Request form.

So if you searched for a SEEK API key to pull job listings, there is none to get. The SEEK API is the right tool if you build an applicant tracking system or a multiposting tool for employers. It is the wrong tool for job market data.

## Ways to get SEEK job ads as data

### 1. SEEK's website and job alerts (free, manual)

For a handful of searches, the website is enough. Search by keyword, location, category, work type and date listed, then save the search to get new matches by email. There is no export button, so anything beyond reading means copying ads by hand. This works for a job seeker or a recruiter watching one niche; it does not give you a dataset.

### 2. Published labour-market statistics (free, aggregate)

If what you need is the shape of the market rather than individual ads, start with free published figures before collecting anything:

- **Internet Vacancy Index (Australia)**, published monthly by Jobs and Skills Australia. Counts of online job ads by occupation, state and region, as a time series you can download.
- **SEEK Employment Report and SEEK Advertised Salary Index**, published by SEEK. Monthly changes in job ad volume, applications per ad and advertised salaries by industry and state.
- **Jobs Online (New Zealand)**, published quarterly by the Ministry of Business, Innovation and Employment, with a monthly index series. Online job ad volumes by industry, occupation and region.

These are free and consistent over years, which makes them better than raw ads for long trend lines. They do not give you job titles, employers, salary text or descriptions, and you cannot cut them by a skill or keyword of your own.

### 3. Your own scraper (free in fees, you maintain it)

SEEK's search pages are public, so you can write a script that runs a search and reads the results. Expect to handle paging, the search's result cap, deduplication across searches, and a separate request per ad if you want the full description. The page structure changes without notice and you maintain the parser. SEEK's terms of use limit automated access to the site; read them and decide for your use case, keep request rates low, and do not collect more personal data than you need.

### 4. A hosted SEEK scraper you call through an API

Several SEEK scrapers are listed on the Apify Store, each billed per result through your Apify account. You send a search and get back JSON or CSV, without writing or maintaining the scraper. Compare them on price per job, whether the full description is included at that price, and which filters they accept.

Disclosure: SEEK Jobs Scraper is ours (Data Gleaner).

**SEEK Jobs Scraper** (`seek-jobs-scraper`) searches seek.com.au or seek.co.nz by keyword and location, or takes SEEK search URLs you paste in. It can filter by category, work type, on-site/hybrid/remote, salary range and date listed, and returns one record per job with the full description. It runs over plain HTTP, with no browser and no SEEK account. It costs **$0.80 per 1,000 jobs** ($0.0008 per job), full descriptions included, billed only for jobs returned. It is not listed on the Apify Store yet; it will appear on the [Data Gleaner Apify page](https://apify.com/datagleaner).
<!-- TODO(store-link: seek-jobs-scraper) -->

Once it is listed, this is how you would call it from Python:

```bash
pip install apify-client
export APIFY_TOKEN=your_apify_token
```

```python
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/seek-jobs-scraper").call(run_input={
    "keywords": ["data analyst"],
    "country": "AU",
    "location": "All Sydney NSW",
    "dateRange": "7",
    "maxJobsPerSearch": 10,
})
if run is None:
    raise SystemExit("The Actor run did not return.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(f'{item.get("title")} | {item.get("companyName")} | '
          f'{item.get("salaryLabel")} | {item.get("location")} | {item.get("url")}')
```

Ten jobs cost under one cent. The main input fields are:

| Field | Meaning |
|---|---|
| `keywords` | One search per keyword, for example `nurse` or `data analyst`. |
| `country` | `AU` (seek.com.au, the default) or `NZ` (seek.co.nz). |
| `location` | A place as SEEK writes it: `All Sydney NSW`, `Melbourne VIC 3000`, `Auckland`, `Remote`. Default is the whole country. A place SEEK does not recognise returns no jobs. |
| `startUrls` | SEEK search result URLs; the keyword, location and filters are read from the URL. |
| `workType` | Any of `fullTime`, `partTime`, `contract`, `casual`. |
| `workArrangement` | Any of `onSite`, `hybrid`, `remote`. |
| `classification` | SEEK category IDs, for example `6281` for Information & Communication Technology or `1211` for Healthcare & Medical. |
| `dateRange` | Listed within the last `1`, `3`, `7`, `14` or `31` days. |
| `salaryMin`, `salaryMax`, `salaryType` | SEEK's own salary filter, `annual`, `monthly` or `hourly`. It also keeps ads that show no salary. |
| `sortBy` | `relevance` or `date`. |
| `maxJobsPerSearch` | Cap per keyword or URL, default 100, maximum 500. This is also your cost cap. |
| `includeDetails` | Full description, expiry date and screening questions. On by default, same price either way. |

## Fields returned

One record per job ad. This is an illustrative record, shortened:

```json
{
  "jobId": "95047618",
  "url": "https://www.seek.com.au/job/95047618",
  "title": "Registered Nurse, Sleep Disorders Unit",
  "companyName": "Example Hospital Group",
  "location": "Kogarah, Sydney NSW",
  "suburb": "Kogarah",
  "city": "Sydney",
  "state": "NSW",
  "postcode": "2217",
  "classification": "Healthcare & Medical",
  "subClassification": "Nursing - General Medical & Surgical",
  "workType": "Part time",
  "workArrangement": "on-site",
  "salaryLabel": "$45 – $50 per hour",
  "salaryMin": 45,
  "salaryMax": 50,
  "salaryPeriod": "hour",
  "currency": "AUD",
  "listedAt": "2026-10-08T23:07:29Z",
  "expiresAt": "2026-11-08T12:59:59.999Z",
  "bulletPoints": ["Flexible roster", "Career growth"],
  "teaser": "Part-time role with career growth.",
  "descriptionText": "About the role\n...",
  "applicationQuestions": ["How many years' experience do you have as a nurse?"],
  "searchKeywords": "nurse",
  "country": "AU",
  "scrapedAt": "2026-10-09T05:00:00+00:00"
}
```

Records also carry `advertiserId`, `advertiserName`, `descriptionHtml`, `isFeatured`, `isPromoted` and the search that found them. Fields SEEK does not publish for a job are `null`: many ads show no salary, and private advertisers hide the company name. `salaryMin` and `salaryMax` are filled only when the salary text holds exactly one amount or range, so "Competitive" gives `null`. Applicant counts and apply links are not in the data.

## Using SEEK data for labour-market research

Job ads are a fast, fine-grained signal of hiring demand, but they are not a census of vacancies. A few practices make the numbers hold up:

- **Track flow, not stock.** Run the same searches daily with `dateRange` set to `1` (last 24 hours) and store the results. Counting new ads per day by role, city or category gives you a demand series; deduplicate on `jobId` across days.
- **Split searches to stay under the cap.** SEEK serves at most about 500 results per search. For a broad query, split by state, category, work type or date range rather than relying on one search.
- **Treat salary data as advertised, not paid.** Only part of the ads show a salary, the period varies (hour, year), and ads with a salary may differ from those without. Normalise to one period and report how many ads had a parseable range.
- **Read skills from the description.** `descriptionText` lets you count mentions of tools or qualifications (for example "SQL", "AHPRA registration") by city or over time.
- **Watch for repeats and agencies.** Recruitment agencies and large employers may post the same role in several locations. Group by title, advertiser and description before counting roles.
- **Cross-check with the official series.** Compare your counts with the Internet Vacancy Index or Jobs Online for the same period; large gaps usually point to a search that is too narrow or a cap you hit.
- **Mind personal data.** Descriptions sometimes include a recruiter's name or phone number. Drop fields you do not need and handle the rest under the privacy law that applies to you.

## Which option to choose

| Need | Best fit |
|---|---|
| Posting your clients' jobs to SEEK from your software | The official SEEK API, after SEEK approves your integration |
| Monthly job ad trends by occupation or region, free | Internet Vacancy Index (AU), Jobs Online (NZ), SEEK Employment Report |
| A few searches you read yourself | SEEK website with saved-search email alerts |
| Ad-level data (titles, employers, salary text, descriptions) without maintaining code | A hosted SEEK scraper on Apify, such as ours or the others listed there |
| Full control and no fees, and you will maintain it | Your own scraper, within SEEK's terms |

## FAQ

**Does SEEK have a public API for job listings?**
No. The SEEK API is for approved software providers that post job ads and handle applications for hirers. It has no job search and does not return other employers' ads.

**How do I get a SEEK API key?**
Software providers request access through the Integration Request form on developer.seek.com, and SEEK must approve it. There is no self-serve key, and it would not give you job search anyway. Hosted scrapers use their own platform's token instead, such as an Apify API token.

**Can I export SEEK job search results to Excel or CSV?**
Not from SEEK's website. A scraper returns the results as JSON or CSV; on Apify, any run's dataset can be downloaded as CSV or Excel.

**Does this work for SEEK New Zealand?**
The official SEEK API covers both markets for hirers. For listings data, our scraper takes `country: "NZ"` or a seek.co.nz search URL.

**How many jobs can I get from one SEEK search?**
SEEK serves at most about 500 results per search. To cover more, split the search by location, category, work type or date range, or collect new ads daily.

## Related guides

- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp)
- [Google Trends API: the options and how to use them](google-trends-api)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Does SEEK have a public API for job listings?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. The SEEK API is for approved software providers that post job ads and handle applications for hirers. It has no job search and does not return other employers' ads."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get a SEEK API key?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Software providers request access through the Integration Request form on developer.seek.com, and SEEK must approve it. There is no self-serve key, and it would not give you job search anyway. Hosted scrapers use their own platform's token instead, such as an Apify API token."
      }
    },
    {
      "@type": "Question",
      "name": "Can I export SEEK job search results to Excel or CSV?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not from SEEK's website. A scraper returns the results as JSON or CSV; on Apify, any run's dataset can be downloaded as CSV or Excel."
      }
    },
    {
      "@type": "Question",
      "name": "Does this work for SEEK New Zealand?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The official SEEK API covers both markets for hirers. For listings data, our scraper takes country: \"NZ\" or a seek.co.nz search URL."
      }
    },
    {
      "@type": "Question",
      "name": "How many jobs can I get from one SEEK search?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "SEEK serves at most about 500 results per search. To cover more, split the search by location, category, work type or date range, or collect new ads daily."
      }
    }
  ]
}
</script>
