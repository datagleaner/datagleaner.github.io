---
title: "Xiaohongshu Scraper: How to Scrape RedNote Notes and Profiles"
description: "Xiaohongshu scraper options for RedNote notes, profiles and feeds: what is public without a login, a free Python script, MediaCrawler's risks, and hosted APIs."
---

# Xiaohongshu scraper: how to scrape RedNote notes and profiles

A Xiaohongshu scraper can read, without logging in, what a guest sees on the website: a single note (when the link carries the `xsec_token` parameter Xiaohongshu adds to shared links), a user's profile with their newest notes, and the public discovery feeds. Keyword search and comment threads sit behind a login wall, so any tool that offers them is using a logged-in account, usually yours. Xiaohongshu (小红书, also called RedNote, XHS or Little Red Book) has no public data API, so your choices are a short script of your own, an open-source crawler such as MediaCrawler, or a hosted scraper you call through an API.

This guide covers what each route can reach, the free script, the risks of the logged-in tools, and how to choose.

## What Xiaohongshu shows without a login

Xiaohongshu's web pages are server-rendered: the page HTML contains a JSON object, `window.__INITIAL_STATE__`, with the data the page displays. What that object holds depends on the page and on whether you are logged in.

| Data | Guest (no login) | Notes |
|---|---|---|
| A single note: title, text, tags, images, video, like / collect / comment / share counts, publish time | Yes, if the link has `xsec_token` | Without the token Xiaohongshu refuses the note. Copy the full link from the Share button or the browser address bar. |
| A user profile: nickname, RED ID, bio, IP location, followers, following, likes and collects, note count | Yes | Token optional. |
| A user's notes | Partly | Only the newest notes (about 30) as cards: title, cover, like count. Guests do not get the IDs needed to open each one in full. |
| Discovery feeds (recommended, food, fashion, beauty, travel and other categories) | Yes | About 35 notes per page load. These are recommendations: there is no date range or keyword filter, and reloads repeat some notes. |
| Keyword search results | No | Needs a logged-in session and Xiaohongshu's request signing. |
| Comments | No | The comment count is public; the comment text needs a login. |
| Follower and following lists | No | |

Two practical details: counts are displayed in Chinese units ("1.7万" is 17,000, "亿" is 100 million), and Xiaohongshu redirects to a login page when it decides a visitor has made too many requests. If www.xiaohongshu.com does not load from your network, the international domain www.rednote.com serves the same pages.

## Option 1: by hand (free, a few notes)

For a handful of notes or profiles, open each one in a browser and copy what you need into a spreadsheet. The app and website both show likes, collects and comment counts on the note. Use the Share button and "copy link" to keep a link that still works for other people, because it includes the `xsec_token`.

## Option 2: a free Python script (public notes)

This script fetches one note and prints its main fields. It reads the same `__INITIAL_STATE__` object the page uses. It needs `pip install requests`.

```python
import json
import re

import requests

# Paste a full note link, including ?xsec_token=...
NOTE_URL = "https://www.xiaohongshu.com/explore/<noteId>?xsec_token=<token>&xsec_source=pc_feed"

html = requests.get(
    NOTE_URL,
    headers={
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
                      "(KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36",
        "Accept-Language": "zh-CN,zh;q=0.9",
    },
    timeout=25,
).text

m = re.search(r"__INITIAL_STATE__\s*=\s*(\{.*?\})\s*</script>", html, re.S)
if not m:
    raise SystemExit("No page data found: the link may lack xsec_token, or you were sent to a login page.")

# The object is JavaScript, not strict JSON: it contains `undefined`.
state = json.loads(m.group(1).replace(":undefined", ":null"))
detail = (state.get("note") or {}).get("noteDetailMap") or {}
note = next(iter(detail.values()), {}).get("note")
if not note:
    raise SystemExit("The page has no note data: the token may be expired, or the note is private or deleted.")

stats = note.get("interactInfo", {})
print("Title:   ", note.get("title"))
print("Author:  ", note.get("user", {}).get("nickname"))
print("Tags:    ", [t["name"] for t in note.get("tagList", [])])
print("Likes:   ", stats.get("likedCount"))
print("Collects:", stats.get("collectedCount"))
print("Comments:", stats.get("commentCount"))
print("Text:    ", note.get("desc"))
```

Profile pages (`https://www.xiaohongshu.com/user/profile/<userId>`) work the same way: the profile sits under `state["user"]["userPageData"]`.

Caveats:

- **This is undocumented.** The field names are whatever Xiaohongshu's front end uses this month. When the site changes, the script breaks and you fix it.
- **Go slowly.** Send one request every few seconds at most. Fast loops get redirected to the login page, and the script then finds no data.
- **Counts are strings** like "1.7万" that you convert yourself.
- **No search or comments.** Those are not in the guest page data.

## Option 3: open-source crawlers (MediaCrawler and others)

[MediaCrawler](https://github.com/NanmiCoder/MediaCrawler) is the best-known open-source project for this. It covers Xiaohongshu together with Douyin, Kuaishou, Bilibili, Weibo, Tieba and Zhihu. It drives a real browser with Playwright and keeps a logged-in session (you log in by scanning a QR code with the app), which is how it reaches keyword search, comments and creator pages that a guest cannot. GitHub also has smaller Xiaohongshu-only projects and Python libraries that call the signed web API with your cookie.

What that costs you:

- **Your account carries the risk.** Every request runs as your logged-in user. Xiaohongshu can restrict or ban accounts it sees doing automated collection, and you would be the one breaking its terms of service. Use an account you can afford to lose, and do not use your main one.
- **You run and repair it.** You need Python, a browser install and the project's dependencies, and the request signing these tools rely on breaks when Xiaohongshu updates its site. Check when a project was last updated before you depend on it.
- **License.** MediaCrawler's license limits it to non-commercial learning use. Read the license of any project before you use its output in paid work.
- **Personal data.** Comments and profiles name real people. Storing them puts you under China's PIPL, and the GDPR if any are in the EU.

If you need keyword search or comment text, a logged-in tool is currently the only way to get it, and these are the trade-offs that come with it.

## Option 4: hosted Xiaohongshu scrapers with an API

Hosted scrapers run on someone else's servers and return JSON, CSV or Excel through an HTTP API, usually priced per result. The Apify Store lists several Xiaohongshu scrapers from different developers. When comparing them, check:

- **Whether it needs your cookies or login.** Listings that offer keyword search or comments ask for your `web_session` cookie or a logged-in browser, so the account risk above applies to them too.
- **Which inputs it accepts:** note links, profile links, feeds, keyword search, comments.
- **Price per result,** and whether profiles and failed pages count.
- **The documented limits.** The guest limits in the table above apply to every no-login scraper, because Xiaohongshu sets them.

### Our Xiaohongshu (RedNote) Scraper

Disclosure: Data Gleaner is us. Our **Xiaohongshu (RedNote) Scraper** (`xiaohongshu-rednote-scraper`) is coming to the Apify Store. Until it is listed, see [apify.com/datagleaner](https://apify.com/datagleaner). <!-- TODO(store-link: xiaohongshu-rednote-scraper) -->

It reads only public pages, with no Xiaohongshu account, login or cookies:

- **Note links** (with `xsec_token`): title, full text, tags, image URLs, video length, like, collect, comment and share counts, and publish time.
- **Discovery feeds:** recommended plus ten categories (fashion, food, beauty, film and TV, career, relationships, home, gaming, travel, fitness). Feed notes can optionally be expanded into full notes, one extra request each.
- **Profiles:** nickname, RED ID, bio, IP location, gender, followers, following, likes and collects, note count, plus the newest notes (up to about 30) as cards.

Counts like "1.7万" come back as numbers. The price is **$0.003 per note**, and each profile record also counts as one note, so 1,000 feed notes cost $3.00. It waits 3 seconds between requests by default, so expanding 100 feed notes into full notes takes about 5 minutes.

What it does not do: keyword search, comments, or expanding a profile's notes into full notes. If Xiaohongshu puts up a login wall during a run, the run stops with a message and keeps what it already saved.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/xiaohongshu-rednote-scraper").call(
    run_input={
        "noteUrls": [],
        "profileUrls": ["https://www.xiaohongshu.com/user/profile/69129eb000000000370059f0"],
        "feedChannels": ["food"],
        "maxItems": 20,
        "fetchNoteDetails": False,
    }
)
if run is None:
    raise SystemExit("The run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    if item.get("recordType") == "profile":
        print("PROFILE", item.get("nickname"), item.get("followers"), item.get("url"))
    else:
        author = (item.get("author") or {}).get("nickname")
        print(item.get("title"), "|", author, "| likes:", item.get("likes"), "|", item.get("url"))
```

## Which route to use

| You need | Use |
|---|---|
| A few notes, once | Copy them by hand |
| Public notes or profiles, and you can maintain code | The free script (Option 2) |
| Keyword search or comment text | A logged-in tool such as MediaCrawler, on an account you can afford to lose |
| Public notes, profiles or trending feeds on a schedule, as JSON, without maintaining code or using an account | A hosted no-login scraper (Option 4) |

## FAQ

**Is there a Xiaohongshu API?**
Not for reading public data. Xiaohongshu's open platform serves merchants and advertisers working with their own stores and campaigns. To read notes and profiles you scrape the website, or call a hosted scraper that exposes the result as an API.

**Can I scrape Xiaohongshu without logging in?**
Yes, for single notes (with `xsec_token` in the link), profiles with their newest notes, and the discovery feeds. Keyword search and comments need a login.

**Is there a Xiaohongshu scraper on GitHub?**
Several. MediaCrawler is the most widely used and covers keyword search and comments through a logged-in browser session. Check each project's license (MediaCrawler's is non-commercial) and its last update date, and expect the account you log in with to carry the ban risk.

**Why does my Xiaohongshu note link return nothing?**
The link is missing its `xsec_token` parameter, or Xiaohongshu redirected you to a login page because of too many requests. Copy the full link again from the Share button and slow down.

**Is scraping Xiaohongshu legal?**
It depends on where you are, what you collect and what you do with it, and this is not legal advice. Reading public pages is a different act from logging in and automating an account against the terms of service. If you keep names, bios or locations, data-protection laws such as China's PIPL and the GDPR apply to how you store and use them.

## Related

- [Weibo, Bilibili and Xiaohongshu scraper APIs compared](../chinese-social-media-scrapers)
- [Weibo API in Python: get Weibo posts from Python](weibo-api-python)
- [How to scrape Bilibili comments and danmaku](scrape-bilibili-comments-danmaku)

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there a Xiaohongshu API?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not for reading public data. Xiaohongshu's open platform serves merchants and advertisers working with their own stores and campaigns. To read notes and profiles you scrape the website, or call a hosted scraper that exposes the result as an API."
      }
    },
    {
      "@type": "Question",
      "name": "Can I scrape Xiaohongshu without logging in?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, for single notes (with xsec_token in the link), profiles with their newest notes, and the discovery feeds. Keyword search and comments need a login."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a Xiaohongshu scraper on GitHub?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Several. MediaCrawler is the most widely used and covers keyword search and comments through a logged-in browser session. Check each project's license (MediaCrawler's is non-commercial) and its last update date, and expect the account you log in with to carry the ban risk."
      }
    },
    {
      "@type": "Question",
      "name": "Why does my Xiaohongshu note link return nothing?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The link is missing its xsec_token parameter, or Xiaohongshu redirected you to a login page because of too many requests. Copy the full link again from the Share button and slow down."
      }
    },
    {
      "@type": "Question",
      "name": "Is scraping Xiaohongshu legal?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It depends on where you are, what you collect and what you do with it, and this is not legal advice. Reading public pages is a different act from logging in and automating an account against the terms of service. If you keep names, bios or locations, data-protection laws such as China's PIPL and the GDPR apply to how you store and use them."
      }
    }
  ]
}
</script>
