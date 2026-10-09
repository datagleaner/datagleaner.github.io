---
title: "WeChat Official Account Articles Scraper: 4 Ways (公众号)"
description: "How to scrape WeChat Official Account (公众号) articles: fetch mp.weixin.qq.com links, search Sogou WeChat, use open-source exporters, and the limits of each."
---

# WeChat Official Account articles scraper: how to collect 公众号 articles

To scrape WeChat Official Account (公众号) articles, start from what you already have. If you have article links (`mp.weixin.qq.com/s/...`), a plain HTTP request returns the full page with no login, and a short script can pull the title, account, publish time and body. If you need to find articles by topic, Sogou WeChat search (`weixin.sogou.com`) is the only public, no-login search index, and it returns recent articles only, about 100 per query. If you need an account's full history, there is no public archive, and the open-source exporters that once got it through a logged-in Official Account backend stopped working when WeChat closed that interface in 2026. The sections below cover each route, a free Python script, and what none of them can do.

Disclosure: Data Gleaner, mentioned below as one option for hosted runs, is us. Every other method on this page is free.

## What WeChat makes public, and what it does not

WeChat is a closed app, and that shapes every tool on this page:

- **Article pages are public.** Anyone with a link can open an article in a normal browser at `mp.weixin.qq.com`, and the page HTML contains the full text and image URLs.
- **There is no public search API and no public account archive.** Inside the app you can browse an account's history, but WeChat offers no open web page or API that lists every article an account published.
- **Read and like counts are mostly hidden** from anonymous visitors. They are shown inside the WeChat app to logged-in users.
- **Search-result links expire.** Links that come out of Sogou search are signed, temporary URLs (`mp.weixin.qq.com/s?src=11&timestamp=...&signature=...`). Short links (`/s/AbCd...`) and `__biz=...&mid=...&sn=...` links are the stable forms.
- **Articles get deleted.** Authors and WeChat remove articles often, so if you need a copy, save it when you find it.

## Comparison: which way fits your job

| Method | Finds articles by | Login needed | Coverage | Cost |
|---|---|---|---|---|
| 1. Fetch article links directly | Links you already have | No | Exactly the links you give | Free |
| 2. Sogou WeChat search | Keyword | No | Recent articles, about 10 pages (100 results) per query | Free, but rate-limited |
| 3. Open-source exporters (WeChat login) | Account | Yes, your own WeChat account | The main backend route was closed by WeChat in 2026; what remains is manual | Free, but fragile |
| 4. Hosted scraper (for example Data Gleaner) | Links, keyword, account name | No | Same as 1 and 2, run in the cloud | Pay per article |

## 1. Fetch article links directly (free, no login)

If your list of articles comes from somewhere else (a newsletter, a spreadsheet, links people shared with you), you do not need search at all. Each article page is server-rendered HTML. The main elements, as the page is structured today:

- Title: `<h1 id="activity-name">` (class `rich_media_title`)
- Account name: the element with `id="js_name"`
- Body: `<div id="js_content">`, with images in `data-src` attributes rather than `src`
- Publish time: a Unix timestamp in the page's inline JavaScript, in a variable named `ct`

This free Python script reads a list of links and prints those fields. WeChat changes its page markup from time to time, so check the selectors if a field comes back empty.

```python
# pip install requests beautifulsoup4
import re
import time
from datetime import datetime, timezone

import requests
from bs4 import BeautifulSoup

URLS = [
    "https://mp.weixin.qq.com/s/_2kC-fXw7UjneZSrsC9CVQ",
]

session = requests.Session()
session.headers["User-Agent"] = "Mozilla/5.0 (research script; contact: you@example.com)"


def text_of(soup, selector):
    el = soup.select_one(selector)
    return el.get_text(strip=True) if el else None


for url in URLS:
    html = session.get(url, timeout=20).text
    soup = BeautifulSoup(html, "html.parser")
    body = soup.select_one("#js_content")
    if body is None:
        print(url, "-> no article body (deleted, restricted, or a non-text post)")
        continue
    ct = re.search(r'var ct\s*=\s*"(\d+)"', html)
    published = (datetime.fromtimestamp(int(ct.group(1)), timezone.utc).isoformat()
                 if ct else None)
    images = [img.get("data-src") for img in body.select("img[data-src]")]
    print(text_of(soup, "#activity-name"), "|", text_of(soup, "#js_name"), "|", published)
    print("  ", body.get_text("\n", strip=True)[:200].replace("\n", " "))
    print("  ", len(images), "images")
    time.sleep(2)  # keep a pause between requests
```

To save Markdown instead of plain text, pass the `#js_content` HTML to a converter such as `markdownify` (and copy each image's `data-src` into `src` first, or the images will be blank).

## 2. Search by keyword with Sogou WeChat search

Sogou runs the public web search for WeChat articles at [weixin.sogou.com](https://weixin.sogou.com) (search for "sogou wechat search" if the link moves). Type a keyword, choose "搜文章" (search articles), and you get recent public articles with title, account name, date and a link.

What to know before you script it:

- **It is a search engine, not an archive.** Results lean to recent articles and stop at about 10 pages (roughly 100 results) per query. Older articles are often missing.
- **Chinese keywords work best.** English queries return far fewer results.
- **Account search is limited.** Searching an account name returns articles that mention or come from it, not a complete list. Sogou's account directory is not available to anonymous visitors.
- **It rate-limits.** Request pages too quickly and Sogou shows a verification (CAPTCHA) page instead of results. Go slowly: a few seconds between pages, and stop when you see a verification page rather than retrying hard.
- **Links are temporary.** Each result link is a signed redirect that expires. Follow it soon after the search, fetch the article (method 1), and store the stable link the article page exposes, or the account's `__biz` value plus the title.

For a one-off research task, searching by hand on the site and copying links into the method 1 script is often enough.

## 3. Open-source exporters that use a WeChat login (mostly closed now)

Until mid-2026, the usual answer for "every article from one account" was an open-source exporter that logged into your own WeChat Official Account backend (the editor at `mp.weixin.qq.com` that account owners use) and searched other accounts' article lists from there. That route has largely closed:

- **[wechat-article-exporter](https://github.com/wechat-article/wechat-article-exporter)** (TypeScript, MIT license) was the best-known tool of this kind: QR-code login to your own Official Account, then export to HTML, JSON, Excel, TXT, Markdown or DOCX, as a hosted site, in Docker or on Cloudflare. Its maintainer stopped maintaining it on 2026-07-30 because WeChat shut down the upstream interface it depended on, and the README says one-click syncing of an account's articles no longer works. The repository stays readable, and the hosted site's domain expires on 2026-10-30.
- **Credential-based tools** remain: they use session credentials captured from your own logged-in WeChat client to read an account's history, and sometimes read counts and comments. The same maintainer describes this path as much less usable and heavily manual. It ties the export to your personal WeChat account, so read the project's current README and WeChat's terms before relying on it.
- **Smaller scripts and AI-agent skills** on GitHub that fetch single article links and convert them to Markdown are method 1 packaged, and still work.

In short, as of late 2026 there is no reliable free tool that lists an account's complete history. If you need one account's archive, collect links as articles appear (subscribe to the account, or search for it regularly) and fetch each one with method 1.

## 4. A hosted scraper: Data Gleaner WeChat Articles

If you would rather not run scripts or keep a WeChat account, Data Gleaner's [WeChat Articles Scraper](https://apify.com/datagleaner) (`wechat-articles`) runs methods 1 and 2 as a hosted job on Apify. <!-- TODO(store-link: wechat-articles) --> It is not yet listed publicly on the Apify Store; the link goes to our store page, where it will appear.

What it does, from its documentation:

- Takes **article URLs**, **search keywords** (through Sogou WeChat search) and **account names**, in any mix. No WeChat login and no cookies.
- Returns per article: title, account name, account ID (`gh_...`), the account's `biz` value, a stable `articleId` (`biz_mid_idx`, the field to de-duplicate on because search links expire), author, publish time (ISO 8601), the 原创 (original) flag, the IP region shown under the title, digest, cover image, every image URL, and the body as text, Markdown and/or HTML. Download as JSON, CSV or Excel.
- Takes `publishedWithin` (`day`, `week` or `month`) to keep only recent keyword and account results, and `maxArticles` (up to 100) per keyword or account.
- Costs **$5 per 1,000 articles**. Failed or deleted articles are not charged.

Its limits are the ones described above, because it uses the same public sources:

- Keyword and account discovery stops at about 100 articles per query and finds recent articles only.
- Account lookup is best effort: it keeps search results from an account with that name, so it cannot download an account's full history. Give links for full coverage.
- Read and like counts are not in the output, because anonymous pages do not show them.
- Deleted, restricted and some video or image-only posts have no body and are skipped.

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/wechat-articles").call(run_input={
    "searchKeywords": ["新能源汽车"],
    "maxArticles": 5,
    "includeContent": ["markdown"],
})
if run is None:
    raise SystemExit("Run did not start")

for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item.get("publishedAt"), "|", item.get("accountName"), "|", item.get("title"))
    print("  ", item.get("url"))
```

This run fetches at most 5 articles, about $0.025. To export specific articles, use `"articleUrls": ["https://mp.weixin.qq.com/s/..."]` instead of `searchKeywords`.

## Legal and ethical notes

Article text and images are copyrighted by their authors. Collecting public articles for research, monitoring or a personal archive is different from republishing them. If you store anything that identifies people, data-protection rules such as China's PIPL or the EU's GDPR may apply. Keep request rates low, and respect WeChat's and Sogou's terms.

## FAQ

**Is there a WeChat Official Account API for reading other accounts' articles?**
No. WeChat's official API lets an account manage its own content, not read other accounts. Collecting other accounts' articles goes through public article pages, Sogou search, or a logged-in backend as described above.

**How do I scrape all articles from one WeChat public account?**
There is no public archive, so a no-login tool can only find what search surfaces. The best-known full-history exporter, wechat-article-exporter, stopped working in 2026 after WeChat closed the backend interface it used (method 3). The dependable approach now is to collect links as the account publishes and fetch each one (method 1).

**Can I get read counts (阅读量) and likes?**
Not from anonymous pages; those numbers appear only inside the WeChat app. Some open-source tools get them by capturing traffic from a logged-in WeChat client, which needs a WeChat account and extra setup.

**Why does Sogou WeChat search show a verification page?**
Sogou rate-limits searches. Slow down, search by hand for small jobs, and stop when the verification page appears rather than retrying quickly.

**How do I save a WeChat article as Markdown?**
Fetch the article link, take the `#js_content` element, copy image `data-src` values into `src`, and convert the HTML with a library like `markdownify` (method 1).

## Related guides

- [Weibo API in Python](weibo-api-python): collect public posts from China's other big social platform.
- [Bilibili API in Python](bilibili-api-python): video, comment and danmaku data from Bilibili.
- [Scrape Medium articles](scrape-medium-articles): the same task for English-language articles.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Is there a WeChat Official Account API for reading other accounts' articles?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. WeChat's official API lets an account manage its own content, not read other accounts. Collecting other accounts' articles goes through public article pages, Sogou search, or a logged-in backend as described above."
      }
    },
    {
      "@type": "Question",
      "name": "How do I scrape all articles from one WeChat public account?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "There is no public archive, so a no-login tool can only find what search surfaces. The best-known full-history exporter, wechat-article-exporter, stopped working in 2026 after WeChat closed the backend interface it used (method 3). The dependable approach now is to collect links as the account publishes and fetch each one (method 1)."
      }
    },
    {
      "@type": "Question",
      "name": "Can I get read counts (阅读量) and likes?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not from anonymous pages; those numbers appear only inside the WeChat app. Some open-source tools get them by capturing traffic from a logged-in WeChat client, which needs a WeChat account and extra setup."
      }
    },
    {
      "@type": "Question",
      "name": "Why does Sogou WeChat search show a verification page?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Sogou rate-limits searches. Slow down, search by hand for small jobs, and stop when the verification page appears rather than retrying quickly."
      }
    },
    {
      "@type": "Question",
      "name": "How do I save a WeChat article as Markdown?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Fetch the article link, take the #js_content element, copy image data-src values into src, and convert the HTML with a library like markdownify (method 1)."
      }
    }
  ]
}
</script>
