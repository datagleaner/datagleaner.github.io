---
title: "Web Scraping MCP Server for AI Agents: Apify Setup Guide"
description: "Set up a web scraping MCP server for AI agents in Claude, Cursor or VS Code: Apify's mcp.apify.com, OAuth or token, preloading Actors, and capping cost."
---

# Web scraping MCP server for AI agents: how to set one up

An MCP (Model Context Protocol) server gives an AI agent such as Claude, Cursor or VS Code's agent mode a set of tools it can call. A web scraping MCP server adds tools that fetch pages or run scrapers and hand the results back to the model. The quickest hosted option is Apify's server at `https://mcp.apify.com`: add that URL to your client, sign in with OAuth (or send an Apify API token), and the agent can search the Apify Store, run any public Actor (Apify's name for a scraper) and read its results. Add `?tools=<owner>/<actor-name>` to the URL to give the agent one specific scraper as its own tool. This guide covers the setup for each client, the free alternatives, how to keep an agent from running up a bill, and three worked examples.

Disclosure: Data Gleaner, whose Actors are used in the worked examples, is us. The setup steps work the same with any Actor on the Apify Store, and the free options below need no Apify account.

## Which kind of scraping server you need

"Web scraping MCP server" covers three different things. Pick by what the agent has to do:

| Option | What the agent gets | Cost | Good for | Limits |
|---|---|---|---|---|
| A fetch server (for example the `fetch` reference server from the MCP project) | One tool that downloads a URL and returns its text | Free, runs on your machine | Reading a few known pages | No JavaScript rendering, no crawling, one page per call |
| A browser server (for example Microsoft's Playwright MCP) | A real browser the agent clicks and types in | Free, runs on your machine | Pages that need a login or clicks, one-off tasks | Slow and token-heavy; every step costs a model call |
| A hosted scraper platform (Apify, Firecrawl and others) | Ready-made scrapers that return structured data | Pay per use, usually with a free allowance | Many pages, structured fields, sites you scrape repeatedly | Needs an account; each run costs money |

Many AI clients already have a built-in web fetch or web search tool. If the agent only needs to read one article or one documentation page, try that first; you do not need an MCP server for it.

A hosted platform earns its place when you want rows of data (every email on 200 company sites, every URL in a sitemap, 100 posts for a keyword) rather than one page of text. A scraper written for that site returns clean fields, so the model reads a small JSON result instead of parsing raw HTML, which saves tokens as well as time.

## Set up the Apify MCP server

The server URL is `https://mcp.apify.com`. There are two ways to authenticate:

- **OAuth (recommended).** Give your client the URL alone. On first connection a browser window opens, you sign in to Apify (a free account works) and approve access.
- **API token.** Send your Apify API token in an `Authorization: Bearer` header. You find the token in Apify Console under Settings, then API & Integrations. Use this when your client cannot do the OAuth flow, or in scripts and CI.

The generic configuration, used by Cursor (`~/.cursor/mcp.json` or `.cursor/mcp.json` in a project) and most clients that read an `mcpServers` block:

```json
{
  "mcpServers": {
    "apify": {
      "url": "https://mcp.apify.com"
    }
  }
}
```

With a token instead of OAuth:

```json
{
  "mcpServers": {
    "apify": {
      "url": "https://mcp.apify.com",
      "headers": {
        "Authorization": "Bearer <YOUR_APIFY_TOKEN>"
      }
    }
  }
}
```

Per client:

- **Claude Code:** `claude mcp add --transport http apify https://mcp.apify.com`, then run `/mcp` inside Claude Code to finish the sign-in. To use a token instead, add `--header "Authorization: Bearer <YOUR_APIFY_TOKEN>"`.
- **Claude Desktop and claude.ai:** add `https://mcp.apify.com` as a custom connector in Settings, under Connectors, and sign in when asked.
- **Cursor, VS Code, Windsurf and others:** the configurator at [mcp.apify.com](https://mcp.apify.com) generates the right snippet for each client.

If you prefer to run the server locally over stdio (needs Node.js installed):

```json
{
  "mcpServers": {
    "actors-mcp-server": {
      "command": "npx",
      "args": ["-y", "@apify/actors-mcp-server"],
      "env": {
        "APIFY_TOKEN": "<YOUR_APIFY_TOKEN>"
      }
    }
  }
}
```

Keep the token out of files you commit. Anyone with it can run Actors on your account.

## What tools the agent sees

With no options, the server loads a general toolkit:

- `search-actors` and `fetch-actor-details`: find a scraper in the Apify Store and read its input schema.
- `call-actor`: run an Actor with an input.
- Two ready Actors as tools: `apify/rag-web-browser` (search and read web pages) and `apify/web-fetch`.
- `search-apify-docs` and `fetch-apify-docs`.

`call-actor` returns the run's status and storage IDs, not the scraped rows. The agent then reads the rows with `get-dataset-items` and the run's dataset ID. The server adds `get-dataset-items` (with `get-actor-run` and `abort-actor-run`) automatically whenever `call-actor` or an Actor tool is loaded.

### Preload a specific scraper with `?tools=`

Letting the agent search the whole Store is flexible, but it costs extra calls and the agent may pick an Actor you did not intend. If you already know which scraper you want, name it in the URL. The server reads that Actor's input schema and turns it into a tool of its own, with the Actor's fields as the tool's parameters:

```
https://mcp.apify.com?tools=datagleaner/website-contact-details-scraper
```

Several at once, plus the tool categories you want, separated by commas. Once you pass `?tools=`, the default toolkit is no longer added, so list every category you still want (`actors` for search and `call-actor`):

```
https://mcp.apify.com?tools=actors,storage,datagleaner/website-contact-details-scraper,datagleaner/sitemap-extractor,datagleaner/weibo-scraper
```

The older parameter `?actors=` still works and means the same thing as `?tools=` for Actors (`?actors=apify/rag-web-browser` equals `?tools=apify/rag-web-browser`). The local server takes the same list as a flag: `npx @apify/actors-mcp-server --tools apify/rag-web-browser`.

Preloading has two practical effects. The agent's tool list stays short, which leaves more of the model's context for your task. And the agent can only run the scrapers you listed, which is the simplest way to know what it will spend money on.

## Keep an agent from running up a bill

An agent decides on its own how many times to call a tool and with what limits. With a scraper that bills per result, that decision is a spending decision. Four controls, from simplest to strictest:

1. **Use pay-per-result Actors.** Many Store Actors charge a fixed price per item returned, shown on the Actor's page, instead of charging for compute time. The cost of a run is then the number of results times the price, which you can work out before you start.
2. **Put the limit in the input.** Most scrapers have a field that caps results: `maxItemsPerQuery`, `maxUrlsPerSite`, `maxPagesPerSite` and so on. Tell the agent the cap in your prompt ("use at most 20 posts per keyword") and check the tool call it makes before you approve it. Most clients ask for approval before an MCP tool runs; leave that on for paid tools.
3. **Set a maximum cost per run.** Pay-per-event Actors accept a maximum total charge per run, set in Apify Console or through the API (`maxTotalChargeUsd`). An Actor built on the Apify SDK stops when it reaches that cap. This is the only control that holds even if the agent passes a large limit.
4. **Watch the account.** Apify Console lists every run with its cost, and you can set usage limits on the account. Apify's free plan includes a small monthly credit, so you can test a setup without adding a card.

Also note: the Apify MCP server allows up to 30 requests per second per user and returns HTTP 429 beyond that, and Actors that need full permissions or are rented by the month are not available through it.

## Worked examples

These three Actors are public on the Apify Store, charge per result, and stop at the maximum cost per run you set. Prices are the ones on the Store pages at the time of writing.

| Actor | What one result is | Price |
|---|---|---|
| [Company Contact Details & Website Email Finder](https://apify.com/datagleaner/website-contact-details-scraper) | One website where at least one email was found; every other site, including sites with only phones or social profiles, is free | US$2.00 per 1,000 ($0.002 each) |
| [Sitemap URL Extractor](https://apify.com/datagleaner/sitemap-extractor) | One page URL found in the site's sitemaps | US$0.20 per 1,000 ($0.0002 each) |
| [Weibo Scraper](https://apify.com/datagleaner/weibo-scraper) | One Weibo post from search or a user's timeline | US$3.00 per 1,000 ($0.003 each) |

Preload all three with the URL from the previous section, then prompt the agent in plain language.

### 1. Contact details for a list of companies

Prompt:

> Find the public contact email, phone and LinkedIn page for stripe.com, appier.com and sakura.ad.jp. Check at most 5 pages per site. Return a table.

The agent calls the contact details tool with an input like this:

```json
{
  "websites": ["stripe.com", "https://www.appier.com", "https://www.sakura.ad.jp"],
  "maxPagesPerSite": 5
}
```

The Actor returns one item per website with `emails`, `phones` (normalized to E.164), `socials`, contact form pages and the company name, each value with the page it came from. Three sites cost at most US$0.006, and a site with no email costs nothing. It respects `robots.txt` by default (`respectRobotsTxt`). If you only need a handful of sites and no API, the free methods in [how to extract emails from a website](extract-emails-from-website-free) work without any account.

### 2. Every URL in a website's sitemap

Prompt:

> List the blog post URLs on stripe.com from its sitemap, at most 200, and tell me which were modified most recently.

Input the agent sends:

```json
{
  "websites": ["stripe.com"],
  "maxUrlsPerSite": 200,
  "includeUrlPatterns": ["/blog/"]
}
```

Each item has the page `url`, its `lastmod` and `priority` where the sitemap gives them, and the `sitemapUrl` it came from. `maxUrlsPerSite` is the cost cap: 200 URLs cost at most US$0.04. This is a good first step before handing pages to a fetch tool, because the agent learns what exists on a site without crawling it. For doing this by hand, see [how to find the sitemap of a website](find-sitemap-of-website).

### 3. What Weibo says about a brand

Prompt:

> Get the 20 most recent Weibo posts mentioning 瑞幸 since 2026-10-01, translate them to English and summarize the main complaints.

Input:

```json
{
  "searchQueries": ["瑞幸"],
  "maxItemsPerQuery": 20,
  "sinceDate": "2026-10-01"
}
```

Each post comes back with its text, `createdAt`, like, comment and repost counts, the author and the post `url`. Twenty posts cost at most US$0.06. Weibo itself serves only about 1,000 posts per keyword search, whatever limit you set. For the same data from a script, see [the Weibo API in Python](weibo-api-python).

## The same run without MCP

If a workflow runs on a schedule rather than in a chat, call the Actor from code instead. Agents are good at deciding what to scrape; a script is cheaper and more predictable for the same job every day. With the `apify-client` package (`pip install apify-client`):

```python
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/sitemap-extractor").call(run_input={
    "websites": ["stripe.com"],
    "maxUrlsPerSite": 10,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item["url"], item.get("lastmod"), item.get("priority"), item.get("sitemapUrl"))
```

This run returns at most 10 URLs, so it costs at most US$0.002.

## Caveats

- **Scraping rules still apply.** An agent scraping on your behalf is you scraping. Check the site's terms, respect `robots.txt`, and take care with personal data (emails and names are personal data under GDPR and similar laws).
- **Agents can misread a schema.** Check the first tool call's input before approving it, especially limits and dates. Preloading a specific Actor helps, because the tool's parameters then come straight from that Actor's input schema.
- **Large results do not belong in the context window.** Ask the agent for a small sample or a summary, and pull the full dataset from Apify Console or the API.
- **Long field descriptions are cut.** The Apify server truncates input field descriptions over 500 characters when it builds a tool, so an Actor with a terse schema can be easier for an agent to use correctly than one with a long one.

## FAQ

### Is there a free web scraping MCP server?

Yes. The `fetch` reference server and Microsoft's Playwright MCP both run on your own machine for free. They fetch or browse one page at a time, which is enough for reading pages but slow for collecting data in bulk. Apify's server itself has no separate fee: you pay for the Actors you run, and the free plan's monthly credit covers small tests.

### What is the Apify MCP server URL?

`https://mcp.apify.com`. Add `?tools=` with a comma-separated list of tool categories or Actor names (`owner/actor-name`) to choose what the agent sees. The older `?actors=` parameter still works for Actors.

### Do I need an Apify API token for the MCP server?

Not if your client supports OAuth: give it the URL alone and sign in when the browser opens. Use a token, sent as `Authorization: Bearer <token>`, for clients without OAuth, for the local `npx` server (as the `APIFY_TOKEN` environment variable) and for automation.

### How do I use the Apify MCP server in Claude Code?

Run `claude mcp add --transport http apify https://mcp.apify.com`, then `/mcp` to sign in. To limit it to specific scrapers, use a URL such as `https://mcp.apify.com?tools=actors,storage,datagleaner/sitemap-extractor`.

### Which is better for an agent, a browser MCP or a scraper MCP?

Use a browser MCP when the task needs clicks, forms or a logged-in session on one site. Use a scraper when you want structured data from many pages: the scraper does the page-by-page work outside the model, so the agent spends one tool call instead of dozens.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there a free web scraping MCP server?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. The fetch reference server and Microsoft's Playwright MCP both run on your own machine for free. They fetch or browse one page at a time, which is enough for reading pages but slow for collecting data in bulk. Apify's server itself has no separate fee: you pay for the Actors you run, and the free plan's monthly credit covers small tests."
      }
    },
    {
      "@type": "Question",
      "name": "What is the Apify MCP server URL?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "https://mcp.apify.com. Add ?tools= with a comma-separated list of tool categories or Actor names (owner/actor-name) to choose what the agent sees. The older ?actors= parameter still works for Actors."
      }
    },
    {
      "@type": "Question",
      "name": "Do I need an Apify API token for the MCP server?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not if your client supports OAuth: give it the URL alone and sign in when the browser opens. Use a token, sent as Authorization: Bearer <token>, for clients without OAuth, for the local npx server (as the APIFY_TOKEN environment variable) and for automation."
      }
    },
    {
      "@type": "Question",
      "name": "How do I use the Apify MCP server in Claude Code?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Run claude mcp add --transport http apify https://mcp.apify.com, then /mcp to sign in. To limit it to specific scrapers, use a URL such as https://mcp.apify.com?tools=actors,storage,datagleaner/sitemap-extractor."
      }
    },
    {
      "@type": "Question",
      "name": "Which is better for an agent, a browser MCP or a scraper MCP?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Use a browser MCP when the task needs clicks, forms or a logged-in session on one site. Use a scraper when you want structured data from many pages: the scraper does the page-by-page work outside the model, so the agent spends one tool call instead of dozens."
      }
    }
  ]
}
</script>
