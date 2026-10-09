---
title: Find Social Media Profiles From a List of Company Websites
description: "Find LinkedIn, Facebook and X profiles from company websites: read JSON-LD sameAs and footer links, drop share buttons. Python script included."
---

# How to find social media profiles from a list of company websites

To find a company's social media profiles from its website, read two places on the home page: the `sameAs` list in the page's JSON-LD structured data, and the `<a href>` links in the header and footer. Keep only links that point to a profile (for example `linkedin.com/company/stripe`) and drop share buttons and links to individual posts. Do that for every domain in your list and you get LinkedIn, Facebook, X, Instagram, YouTube and TikTok from one fetch per site. Below: both sources, the filtering rules, a free Python script, and a bulk option.

Disclosure: Data Gleaner, mentioned at the end as one option for bulk extraction, is us. The manual method and the script on this page are free and need no account.

## Why one pass for every network

Some tools for this job handle one network at a time, such as a "website to LinkedIn" mapper. That is fine if you only need LinkedIn, but a company's home page already links to all of its profiles. Fetching the page once and sorting the links by network is cheaper than running a separate lookup per network, and it makes the result a single row per company.

## Source 1: JSON-LD sameAs

Many sites describe themselves to search engines with schema.org structured data, in a `<script type="application/ld+json">` block. An `Organization` object can carry a `sameAs` property: a list of URLs of the same entity on other sites. Site owners put their social profiles there deliberately, so it is the cleanest source:

```json
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Example Co",
  "url": "https://example.com",
  "sameAs": [
    "https://www.linkedin.com/company/example-co",
    "https://www.facebook.com/exampleco",
    "https://twitter.com/exampleco"
  ]
}
```

Things to know:

- `sameAs` can be a single string instead of a list, and the object can sit inside an `@graph` array or inside another object (for example a `WebSite` that has a `publisher`). A good parser walks the whole JSON tree rather than reading only the top level.
- Some sites list Wikipedia, Wikidata or Crunchbase in `sameAs` too. Those are not social networks, so filter by domain.
- Not every site has it. Small business sites and older templates often have no JSON-LD at all, so you need the second source.
- Old links stay behind. A site can still list `twitter.com/handle` years after the company moved to `x.com`, so treat both hosts as X.

## Source 2: footer and header links

Almost every business site links its profiles with the familiar row of icons in the footer or header. These are ordinary `<a href>` links, so the script reads every link on the page and sorts them by host. This finds profiles on sites that have no JSON-LD, but it also finds a lot of links that are not the company's profile.

## The main source of wrong results: share buttons and post links

If you collect every link to a social host, you will get wrong answers. The same page often contains:

- **Share buttons**: `facebook.com/sharer/sharer.php?u=...`, `twitter.com/intent/tweet?url=...`, `linkedin.com/shareArticle?...`. These point to the network's share dialog, not to a profile.
- **Links to individual posts**: `x.com/handle/status/123456`, `instagram.com/p/abc123`, `facebook.com/handle/posts/123`. They belong to a profile but are not the profile URL. A blog post may also link to somebody else's account.
- **Embed and tracking endpoints**: `facebook.com/tr?id=...` (the tracking pixel), `platform.twitter.com/widgets.js`, `facebook.com/plugins/...`.
- **Login and help pages**, and in-product links when the site is itself a social network.

The script below uses three rules to avoid these:

1. Match the host exactly (`linkedin.com`, `facebook.com`, `twitter.com`, `x.com`, and so on, after removing `www.`, `m.` and language prefixes), not a substring. `notlinkedin.com` does not count.
2. Reject known non-profile first path segments, such as `sharer`, `intent`, `share`, `plugins`, `tr`, `login`, `p`, `reel` and `hashtag`.
3. Keep only the part of the URL that identifies the account: `/company/<slug>` for LinkedIn, the first path segment for the other networks, and drop the query string. A path with more segments, such as `/handle/status/123`, is a post and is skipped.

JSON-LD results are tried first, because they were put there on purpose, and a network keeps the first valid match it finds.

## 1. Check one site by hand

Open the home page, scroll to the footer, and click the icons. To see structured data, view the page source and search for `sameAs`. This takes a minute per site and is fine for a handful of companies.

## 2. A free Python script for a list of sites

This script uses `requests` and the standard library. It reads JSON-LD `sameAs` first, then page links, and applies the filtering rules above:

```python
# pip install requests
import json
import re
import sys
from html.parser import HTMLParser
from urllib.parse import urljoin, urlparse

import requests

NETWORKS = {
    "linkedin": {"hosts": {"linkedin.com"}, "pages": {"company", "school", "showcase"}},
    "facebook": {"hosts": {"facebook.com", "fb.com"}, "deny": {
        "sharer", "sharer.php", "share.php", "share", "dialog", "plugins", "tr",
        "login", "login.php", "events", "groups", "watch", "hashtag", "photo",
        "photo.php", "permalink.php", "story.php", "policies", "help", "pages"}},
    "x": {"hosts": {"twitter.com", "x.com"}, "deny": {
        "intent", "share", "home", "i", "hashtag", "search", "login", "widgets.js",
        "privacy", "tos", "settings", "explore", "messages"}},
    "instagram": {"hosts": {"instagram.com"}, "deny": {"p", "reel", "reels", "explore", "accounts", "stories", "tv"}},
    "youtube": {"hosts": {"youtube.com"}, "pages": {"channel", "c", "user"}},
    "tiktok": {"hosts": {"tiktok.com"}, "handle": True},
    "github": {"hosts": {"github.com"}, "deny": {"features", "pricing", "login", "join", "about", "sponsors", "orgs"}},
}


class Collector(HTMLParser):
    def __init__(self):
        super().__init__()
        self.links, self.jsonld, self._in_ld = [], [], False

    def handle_starttag(self, tag, attrs):
        a = dict(attrs)
        if tag == "a" and a.get("href"):
            self.links.append(a["href"])
        self._in_ld = tag == "script" and a.get("type") == "application/ld+json"

    def handle_endtag(self, tag):
        self._in_ld = False

    def handle_data(self, data):
        if self._in_ld:
            self.jsonld.append(data)


def same_as(node):
    """Yield every sameAs URL in a parsed JSON-LD value, however deeply nested."""
    if isinstance(node, dict):
        v = node.get("sameAs")
        if isinstance(v, str):
            yield v
        elif isinstance(v, list):
            yield from (x for x in v if isinstance(x, str))
        for child in node.values():
            yield from same_as(child)
    elif isinstance(node, list):
        for child in node:
            yield from same_as(child)


def classify(url):
    """Return (network, clean_url) for a profile link, or None for share and post links."""
    p = urlparse(url)
    host = (p.hostname or "").lower()
    if host.count(".") > 1:                       # drop www., m., de-de. and similar
        host = re.sub(r"^(www|m|mobile|[a-z]{2}|[a-z]{2}-[a-z]{2})\.", "", host)
    parts = [s for s in p.path.split("/") if s]
    for name, rule in NETWORKS.items():
        if host not in rule["hosts"] or not parts:
            continue
        first = parts[0].lower()
        if "pages" in rule:                      # /company/<slug>, /channel/<id>
            if first in rule["pages"] and len(parts) >= 2:
                return name, f"https://{host}/{first}/{parts[1]}"
            if name == "youtube" and first.startswith("@"):
                return name, f"https://{host}/{parts[0]}"
            continue
        if rule.get("handle") and not first.startswith("@"):
            continue
        if first in rule.get("deny", set()) or first.endswith((".php", ".js")):
            continue
        if len(parts) > 1 and name in ("x", "facebook", "instagram", "github", "tiktok"):
            continue                              # /handle/status/123 is a post, not a profile
        return name, f"https://{host}/{parts[0]}"
    return None


def profiles(site):
    url = site if site.startswith("http") else "https://" + site
    r = requests.get(url, timeout=20, headers={"User-Agent": "Mozilla/5.0 (profile-finder; contact: you@example.com)"})
    r.raise_for_status()
    c = Collector()
    c.feed(r.text)
    candidates = []
    for block in c.jsonld:                       # 1. structured data first
        try:
            candidates += [(u, "jsonld") for u in same_as(json.loads(block))]
        except ValueError:
            pass
    candidates += [(urljoin(r.url, h), "link") for h in c.links]   # 2. then <a href>
    found = {}
    for u, src in candidates:
        hit = classify(u)
        if hit and hit[0] not in found:
            found[hit[0]] = {"url": hit[1], "source": src}
    return found


if __name__ == "__main__":
    for site in sys.argv[1:]:
        try:
            print(site, json.dumps(profiles(site), indent=2))
        except requests.RequestException as e:
            print(site, "ERROR", e)
```

Run it with `python profiles.py stripe.com wordpress.org`. Each site gives one dictionary with the network as key and the profile URL plus where it was found (`jsonld` or `link`). On a test run, stripe.com returned X, YouTube, LinkedIn, Facebook, GitHub and Instagram profiles, all from JSON-LD, and wordpress.org returned Facebook and X from JSON-LD plus Instagram, LinkedIn and TikTok from page links.

To use it on a list, loop over your domains, write the results to a CSV, and add a delay between requests to the same host if you fetch more than one page per site.

### Limits of the script

- **No JavaScript.** It reads the HTML the server sends. If the footer is built by scripts in the browser, the links are not in the HTML and the script finds nothing for that site.
- **Home page only.** Some companies put social links only on a contact or about page. Add those paths if you need them.
- **Blocked requests.** Sites with bot protection can answer 403 or 429 to a plain script.
- **Sites that are social networks.** Running it on github.com returns `github.com/mcp`, an in-product link that passes the first-path-segment rule.
- **One profile per network.** A company that links several accounts on one network (a parent brand with regional profiles) gives you only the first match.
- **Stale and wrong links.** A link on a site is what the site owner chose to publish. It can point to an abandoned page, a personal account or a profile of a partner. Spot-check a sample before you rely on the list.
- **Terms and privacy.** Reading a public web page is not the same as having a right to contact the people behind a profile. Check the rules that apply to you before you store or message them.

## 3. LinkedIn, Facebook and X specifics

- **LinkedIn**: company pages live at `linkedin.com/company/<slug>`. Schools use `/school/<slug>`, and sub-brands `/showcase/<slug>`. Personal profiles are `/in/<slug>`; the script deliberately ignores them, because a link to a person is rarely the company page.
- **Facebook**: a page is usually `facebook.com/<name>`. Older pages can be `facebook.com/pages/<name>/<id>`, which the script skips; add a rule if your list has many of them.
- **X (Twitter)**: `twitter.com` and `x.com` are the same network. The script returns the host as found, so normalise to one host if you need to deduplicate.

## 4. Bulk option: the Data Gleaner Actor

Disclosure: Data Gleaner is us. The [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper) runs on the Apify Store. It is a contact scraper rather than a social-only tool: for each website it fetches the home page and the pages most likely to hold contact details, and returns social profiles alongside emails, phone numbers, contact forms and addresses.

What it does for this task:

- Returns social profiles on 14 platforms (LinkedIn, X (Twitter), Facebook, Instagram, YouTube, TikTok, GitHub, Pinterest, Threads, Telegram, WhatsApp, Discord, Reddit and Snapchat), each with the page where it was found. Share buttons and post links are ignored.
- Separates the company's own accounts (`socials`: linked from the header, footer or navigation, or matching the brand) from other people's profiles that the site merely links, such as testimonials, team members or press (`mentionedSocials`). This solves the "personal account or partner profile" problem the script above leaves to you.
- Uses plain HTTP requests (no browser), reads up to `maxPagesPerSite` pages per site (8 by default, 30 at most), stays on the same domain and respects `robots.txt` by default.
- Returns one row per input website with a `status` of `ok`, `noContacts`, `unreachable`, `blockedByRobots`, `invalidUrl` or `error`. Results export as CSV, Excel or JSON.
- Costs US$4.00 per 1,000 websites with contacts (US$0.004 each). You are charged once per website that returns at least one contact, which can be an email, phone, contact form, address or one of the site's own social profiles (other people's profiles do not count); unreachable sites and sites with no contacts are free. For 5,000 sites of which 3,500 return a contact, that is $14.00.

It shares the script's main limits: no JavaScript rendering, so details added only by scripts are not found, and sites that block non-browser clients (HTTP 403 or 429) are not read. It also does not return a social profile for every site, because some sites do not publish one.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/website-contact-details-scraper").call(run_input={
    "websites": ["stripe.com", "wordpress.org", "https://www.appier.com"],
    "maxPagesPerSite": 5,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    socials = item.get("socials", {})
    print(item["website"], item["status"])
    for network in ("linkedin", "facebook", "twitter"):
        for profile in socials.get(network, []):
            print("  ", network, profile["url"], "(found on", profile["foundOn"] + ")")
```

The `socials` object has one list per network (`linkedin`, `twitter`, `facebook`, `instagram`, `youtube`, `tiktok`, `github`, `pinterest`, `threads`, `telegram`, `whatsapp`, `discord`, `reddit`, `snapchat`), and each entry has a `url` and a `foundOn` field. `mentionedSocials` has the same shape for profiles the site links that are not its own.

## FAQ

**How do I find a company's social media profiles from its website?**
Open the home page and look at the footer icons, or view the page source and search for `sameAs`. The structured data lists the official profiles when the site publishes it. For many companies, run a script or an extractor that does the same thing for every domain.

**Where do I get a company's LinkedIn URL from its website?**
First look for a link to `linkedin.com/company/...` in the footer or in the JSON-LD `sameAs` list. If the site has none, search LinkedIn for the company name and check that the website shown on the LinkedIn page matches the domain.

**Why do I get Facebook or X links that are not the company's page?**
Because the page contains share buttons (`sharer.php`, `intent/tweet`), tracking pixels or links to individual posts. Filter on the first path segment and drop anything with a post path such as `/status/` or `/p/`.

**Is twitter.com the same as x.com?**
Yes. The network was renamed, and sites still link to either host. Treat both as X and normalise the host if you want to remove duplicates.

**Can I get social profiles for thousands of companies?**
Yes. A simple script works through a list of domains one by one; for large lists add error handling, retries and a delay between requests to the same site. A hosted extractor runs many sites in parallel and retries failed requests for you.

**What if a site shows no social links?**
Then none are published in the HTML, or they are added by JavaScript after the page loads. Try the company's contact or about page, or look the company up directly on each network.

## Related guides

- [How to find email addresses from a list of websites](../contact-and-lead-scrapers): the same list of domains, for emails instead of social profiles.
- [How to extract emails from a website for free](extract-emails-from-website-free): free methods and a script for one site.
- [How to find the sitemap of a website](find-sitemap-of-website): list the pages of a site before you scrape it.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I find a company's social media profiles from its website?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Open the home page and look at the footer icons, or view the page source and search for sameAs. The structured data lists the official profiles when the site publishes it. For many companies, run a script or an extractor that does the same thing for every domain."
      }
    },
    {
      "@type": "Question",
      "name": "Where do I get a company's LinkedIn URL from its website?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "First look for a link to linkedin.com/company/... in the footer or in the JSON-LD sameAs list. If the site has none, search LinkedIn for the company name and check that the website shown on the LinkedIn page matches the domain."
      }
    },
    {
      "@type": "Question",
      "name": "Why do I get Facebook or X links that are not the company's page?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Because the page contains share buttons (sharer.php, intent/tweet), tracking pixels or links to individual posts. Filter on the first path segment and drop anything with a post path such as /status/ or /p/."
      }
    },
    {
      "@type": "Question",
      "name": "Is twitter.com the same as x.com?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. The network was renamed, and sites still link to either host. Treat both as X and normalise the host if you want to remove duplicates."
      }
    },
    {
      "@type": "Question",
      "name": "Can I get social profiles for thousands of companies?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes. A simple script works through a list of domains one by one; for large lists add error handling, retries and a delay between requests to the same site. A hosted extractor runs many sites in parallel and retries failed requests for you."
      }
    },
    {
      "@type": "Question",
      "name": "What if a site shows no social links?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Then none are published in the HTML, or they are added by JavaScript after the page loads. Try the company's contact or about page, or look the company up directly on each network."
      }
    }
  ]
}
</script>
