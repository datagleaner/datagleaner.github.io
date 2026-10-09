---
title: "Job Listings Scraper API: Greenhouse, Lever, Workday, SEEK"
description: "A job listings scraper API, compared: free ATS feeds (Greenhouse, Lever, Ashby), official job APIs, and pay-per-job scrapers for career sites and SEEK."
---

# Job listings scraper API: where to get job postings as JSON

To get job listings through an API, start with the source. Most companies post their jobs through an applicant tracking system (ATS) such as Greenhouse, Lever, Ashby or Workday, and several of those publish a free, public JSON feed of every open job. For job boards, use an official API where one exists (Adzuna, USAJOBS). A scraper API is worth paying for when you need many companies or several ATSes in one format, or a board with no public search API, such as SEEK in Australia and New Zealand.

This page covers the free options first, with code, then the two job scrapers we publish on the Apify Store.

## Option 1: the ATS's own public job feed (free)

Companies on these systems publish their open jobs at a public URL that needs no key and no login. You need the company's board name, which is the last part of its careers URL (`stripe` in `boards.greenhouse.io/stripe`).

| ATS | Public endpoint | Notes |
|---|---|---|
| Greenhouse | `https://boards-api.greenhouse.io/v1/boards/<board>/jobs?content=true` | `content=true` adds the description |
| Lever | `https://api.lever.co/v0/postings/<company>?mode=json` | EU-hosted boards use `api.eu.lever.co` |
| Ashby | `https://api.ashbyhq.com/posting-api/job-board/<board>?includeCompensation=true` | Includes pay ranges the company publishes |

A complete example for one Greenhouse board, using only `requests`:

```python
# pip install requests
import requests

board = "stripe"
url = f"https://boards-api.greenhouse.io/v1/boards/{board}/jobs"
jobs = requests.get(url, params={"content": "true"}, timeout=30).json()["jobs"]

for job in jobs:
    location = (job.get("location") or {}).get("name", "")
    print(job["title"], "|", location, "|", job["absolute_url"])
```

- **Good for:** a handful of companies on one ATS. It is free, fast and the data comes straight from the employer.
- **Caveats:** every ATS returns a different shape, so tracking companies across Greenhouse, Lever, Ashby and Workday means writing and maintaining one parser per system. Workday, SmartRecruiters, Workable and others each need their own approach. You also have to find the right board name for every company yourself, and there is no search across companies: you can only ask "what is open at company X".

## Option 2: an official job search API (free or paid)

Some job boards and aggregators offer a real search API:

- **Adzuna** has a developer API with a free key that searches its job index by keyword and location in a number of countries, Australia included.
- **USAJOBS** has a free API (key required) for US federal government jobs.
- **Commercial job data providers** sell large historical and live job datasets through an API, usually on a subscription.

Two of the biggest boards are not on this list on purpose. LinkedIn's job APIs are for approved partners who post jobs, not for searching listings. SEEK's developer platform is built for hirers and recruitment software that post and manage ads, not for searching SEEK's listings.

- **Good for:** keyword search across many employers without building anything.
- **Caveats:** an aggregator's index is not the employer's full list, fields like salary and full descriptions vary, and free tiers have rate limits. Read each API's terms on storing and republishing the data.

## Option 3: open-source scraping libraries (free, self-run)

Open-source Python libraries such as JobSpy collect listings from several large job boards into one table. They cost nothing, but you run and maintain them, they break when a site changes its pages, and the sites' terms of service and rate limits are yours to respect.

## Option 4: a hosted job scraper API (pay per job)

A hosted scraper runs the collection for you and returns one consistent format. This is the gap our two Actors fill. Disclosure: Data Gleaner is us, and these are our products.

| Actor | What it returns | Price | Store |
|---|---|---|---|
| **Career Site Jobs Scraper** (`career-jobs-scraper`) | Open jobs from company career sites on Greenhouse, Lever, Ashby, Workday, SmartRecruiters, Workable, Recruitee and Personio, in one normalized format: title, department, locations, remote flag, employment type, salary when published, posted date, apply link and full description | $0.0015 per job ($1.50 per 1,000) | Coming to the Store, see [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: career-jobs-scraper) --> |
| **SEEK Jobs Scraper** (`seek-jobs-scraper`) | Job ads from seek.com.au and seek.co.nz by keyword, location and SEEK's filters: title, company, advertiser, suburb, city, state and postcode, category, work type, on-site/hybrid/remote, advertised salary text plus parsed min and max, listing and expiry dates, full description and screening questions | $0.0008 per job ($0.80 per 1,000), full descriptions included | Coming to the Store, see [apify.com/datagleaner](https://apify.com/datagleaner) <!-- TODO(store-link: seek-jobs-scraper) --> |

You pay only for jobs returned, and you can set a maximum charge per run.

### Career Site Jobs Scraper, in more detail

It reads the same public ATS feeds as Option 1 and handles the parts that take work by hand:

- **Three ways in.** Paste a careers-page URL and it detects the ATS; write a company name or website (`Anthropic`, `datadog.com`) and it tries the Greenhouse, Lever and Ashby boards for it; or switch on **Search all bundled companies** to filter by job title, location or remote across 8,177 Greenhouse, Lever and Ashby boards that ship with the Actor.
- **One format for every ATS.** Jobs are de-duplicated on the ATS and its job ID, with filters for title keywords, locations, remote only and posted within N days.
- **Limits.** Salary comes through only where the ATS publishes it in structured form (Ashby, Lever when filled in, Recruitee and Greenhouse); it is `null` for Workday, SmartRecruiters, Workable and Personio. Lookup by plain name only tries Greenhouse, Lever and Ashby, so paste the URL for the others. The bundled board list was checked on 2026-10-08, so boards created later need to be added by URL. Workday returns at most 2,000 jobs per company. Only open, public jobs are visible.

### SEEK Jobs Scraper, in more detail

- Search by keyword, country (`AU` or `NZ`) and location, with SEEK's own filters for category, work type, work arrangement, salary and date range, or paste SEEK search URLs.
- `salaryMin` and `salaryMax` are filled only when the advertised text holds exactly one amount or range; "Competitive" gives `null`.
- SEEK serves at most about 500 results per search. To collect more, split by location, category, work type or date range, or run it daily with the "last 24 hours" filter.
- It does not return applicant counts or apply links, because SEEK does not publish them in this data.

## Who uses job listing data, and for what

- **Job boards and niche aggregators** that need fresh listings from many employers in one format.
- **Recruiters and sales teams** watching which companies are hiring for a role, as a hiring signal for outreach.
- **Salary research**, from the jobs that publish pay ranges.
- **Labour-market and skills analysis**: job counts by company, department, location, category or ATS, tracked over time with a scheduled daily run.
- **AI agents** that answer "who is hiring for X" with live data instead of a stale index.

## Python quick start

Install the Apify client and set your Apify API token. This run fetches engineering jobs from two career sites and stops at 10 jobs, so it costs at most $0.015.

```python
# pip install apify-client
import os
import sys

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/career-jobs-scraper").call(run_input={
    "companies": ["https://boards.greenhouse.io/stripe", "https://jobs.ashbyhq.com/openai"],
    "titleKeywords": ["engineer"],
    "maxJobsPerCompany": 5,
    "maxItems": 10,
    "includeDescription": False,
})
if run is None:
    sys.exit("[ERROR] The Actor run did not return.")

for job in client.dataset(run.default_dataset_id).iterate_items():
    salary = job.get("salary") or {}
    pay = f"{salary.get('min')}-{salary.get('max')} {salary.get('currency')}" if salary else "n/a"
    print(f"{job['company']} | {job['title']} | {', '.join(job.get('locations') or [])} | {pay} | {job['jobUrl']}")
```

For SEEK, the input is a keyword search instead:

```python
run = client.actor("datagleaner/seek-jobs-scraper").call(run_input={
    "keywords": ["data analyst"],
    "country": "AU",
    "location": "All Sydney NSW",
    "maxJobsPerSearch": 10,
})
if run is None:
    sys.exit("[ERROR] The Actor run did not return.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item["title"], "|", item["companyName"], "|", item["salaryLabel"])
```

## Use from AI agents (Apify MCP)

Claude, Cursor and other MCP clients can call either Actor through Apify's hosted MCP server at `https://mcp.apify.com`. Add the Actor to the URL to preload it as a tool.

Claude Code:

```bash
# Signs in through the browser on first use (run /mcp inside Claude Code)
claude mcp add --transport http apify "https://mcp.apify.com?tools=datagleaner/career-jobs-scraper"

# Or with an Apify token
claude mcp add --transport http apify "https://mcp.apify.com?tools=datagleaner/career-jobs-scraper" --header "Authorization: Bearer YOUR_APIFY_TOKEN"
```

Clients that take an `mcpServers` URL entry:

```json
{
  "mcpServers": {
    "apify": { "url": "https://mcp.apify.com?tools=datagleaner/career-jobs-scraper,datagleaner/seek-jobs-scraper" }
  }
}
```

Then ask in plain language, for example: "Find remote machine learning engineer jobs posted in the last 14 days at Stripe, OpenAI and Spotify, and give me a table with company, title, location, salary and apply link." The agent fills in the Actor's input and reads the jobs from the run's dataset.

## Which option should you use?

| You want | Use |
|---|---|
| Open jobs at a few companies on one ATS | That ATS's public feed (Option 1) |
| Keyword search across employers in a country an aggregator covers | An official job search API such as Adzuna (Option 2) |
| Many companies across several ATSes, in one format, with salary where published | Career Site Jobs Scraper |
| "Which companies are hiring for this title?" across thousands of boards | Career Site Jobs Scraper with the bundled-company search |
| Australian or New Zealand jobs from SEEK | SEEK Jobs Scraper |

## FAQ

**Is there a free job postings API?**
Yes, several. Greenhouse, Lever and Ashby publish free, public JSON feeds of each company's open jobs, with no key needed. Adzuna and USAJOBS offer free search APIs with a key. The trade-off is that each one covers only its own jobs and returns its own format.

**Is there a LinkedIn or Indeed job postings API?**
Not for general job search. LinkedIn's job APIs are for approved partners posting jobs, and Indeed's current APIs are for employers and ATS partners posting jobs and handling applications, not for searching its listings. For employer jobs, the employer's own ATS feed is often the better source, since it is where the job was posted in the first place.

**Is there a SEEK job search API?**
SEEK's developer platform is for posting and managing ads, not searching them. A SEEK scraper such as our SEEK Jobs Scraper reads the public search results and returns them as JSON, with no SEEK account needed.

**Is it legal to scrape job postings?**
Job postings are published for anyone to read, and reading them is widely done, but the rules depend on where you are, the site's terms and what you do with the data. Postings can name recruiters or hiring managers, so if you store or contact people from this data, data-protection and anti-spam rules apply to you. Keep request rates modest.

**How do I scrape job postings with Python?**
For one company, call its ATS feed with `requests`, as in the Greenhouse example above. For many companies or several ATSes, call a hosted scraper through the `apify-client` package, as in the quick start.

## Related

- [Contact and lead scrapers](contact-and-lead-scrapers): turn a list of hiring companies into contact details.
- [SEEK jobs API](guides/seek-jobs-api): get SEEK Australia and New Zealand job ads as JSON.
- [Web content scrapers](web-content-scrapers)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there a free job postings API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, several. Greenhouse, Lever and Ashby publish free, public JSON feeds of each company's open jobs, with no key needed. Adzuna and USAJOBS offer free search APIs with a key. The trade-off is that each one covers only its own jobs and returns its own format."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a LinkedIn or Indeed job postings API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not for general job search. LinkedIn's job APIs are for approved partners posting jobs, and Indeed's current APIs are for employers and ATS partners posting jobs and handling applications, not for searching its listings. For employer jobs, the employer's own ATS feed is often the better source, since it is where the job was posted in the first place."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a SEEK job search API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "SEEK's developer platform is for posting and managing ads, not searching them. A SEEK scraper such as our SEEK Jobs Scraper reads the public search results and returns them as JSON, with no SEEK account needed."
      }
    },
    {
      "@type": "Question",
      "name": "Is it legal to scrape job postings?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Job postings are published for anyone to read, and reading them is widely done, but the rules depend on where you are, the site's terms and what you do with the data. Postings can name recruiters or hiring managers, so if you store or contact people from this data, data-protection and anti-spam rules apply to you. Keep request rates modest."
      }
    },
    {
      "@type": "Question",
      "name": "How do I scrape job postings with Python?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "For one company, call its ATS feed with requests, as in the Greenhouse example above. For many companies or several ATSes, call a hosted scraper through the apify-client package, as in the quick start."
      }
    }
  ]
}
</script>
