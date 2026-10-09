---
title: How to Find a YouTuber's Email (YouTube Channel Email)
description: "How to find a YouTuber's email: the About panel's View email address button, the description, channel links and the creator's website, plus bulk tools."
---

# How to find a YouTuber's email address

To find a YouTuber's email, open their channel on a desktop browser, open the About panel (click "more" next to the channel description) and look for a **View email address** button under business inquiries. You must be signed in and pass a CAPTCHA to reveal it. If there is no button, read the channel description and recent video descriptions, then follow the links in the About panel to the creator's website, link-in-bio page or Instagram, where many creators publish a contact address. If none of those shows an email, the creator has chosen not to publish one, and a social DM or a contact form is the remaining route.

This guide walks through each place to look, what goes wrong with each, and how to do it for a list of channels. Disclosure: Data Gleaner is us. Everything up to the bulk section works without our product.

## Where YouTubers publish their email

| Where | How to check | What you get | Caveat |
|---|---|---|---|
| About panel, "View email address" | Desktop browser, signed in, solve the CAPTCHA | The business email the creator entered in YouTube Studio | One channel at a time; YouTube limits how many you can reveal per day |
| Channel description | Read the About panel text | Any email the creator typed into the description | Often written as `name [at] domain` to deter scrapers |
| Video descriptions | Open two or three recent videos, expand the description | Sponsorship or business emails, sometimes a manager's | Varies per video; older videos may show an old address |
| Links in the About panel | Click the website, Linktree-style page, Instagram, X, TikTok | Contact page, link-in-bio email button, bio email | Each platform has its own login walls and rules |
| The creator's own website | Look for Contact, About, Work with me, Press | Business or management email, contact form | Sites that load content with JavaScript may hide it from simple tools |

## Step 1: the "View email address" button

1. Open the channel in a desktop browser, for example `youtube.com/@handle`.
2. Click the channel description (or "more") under the channel name to open the About panel.
3. Under the details section, look for **View email address**. If it is there, sign in to any Google account, click it and complete the CAPTCHA.
4. The address appears in place of the button. Copy it.

What to know about this button:

- **It only exists if the creator added a business email** in YouTube Studio. No button means they did not, not that you are doing something wrong.
- **You must be signed in.** Signed out, you see the button but cannot reveal the address.
- **There is a daily limit.** After a number of reveals, YouTube stops showing addresses for that account for the day. It is meant for contacting a few creators, not for building lists.
- **The phone app may not show it.** If you cannot find the button in the app, use a desktop browser or "request desktop site" in a mobile browser.
- **The address is often a manager's or agency's,** especially on large channels. That is the right inbox for sponsorship, but not for a personal message.

## Step 2: descriptions

If there is no button, read the channel description in the About panel, then expand the description of a few recent videos. Creators who want sponsorships often write "Business inquiries:" followed by an address. Search the page for `@` or the words "business", "inquiries", "contact" or "sponsorship" (Ctrl+F or Cmd+F).

Addresses written as `name (at) domain (dot) com` are still valid; rewrite them as a normal address before you send.

## Step 3: the links on the channel

The About panel lists the creator's links: a website, a link-in-bio page (Linktree, Beacons, Stan and similar), and their social profiles. Check them in this order:

1. **Link-in-bio page.** Many have an email button or a "contact" link.
2. **Website.** Look in the footer and on Contact, About, "Work with me" or Press pages. A media kit PDF often has the address.
3. **Instagram, TikTok and X bios.** Instagram business profiles can show an Email button in the app; X and TikTok bios sometimes list one.

If the site has a contact form but no address, use the form. It reaches the same person.

## Step 4: when there is no public email

Some creators publish no address at all. Your options then:

- **Social DMs** on Instagram or X, where many creators read business messages.
- **Their management or network,** if the description names one. Search the agency's site for a talent or partnerships contact.
- **A comment** on a recent video, short and specific, asking where to send a business inquiry. Search the comments first; someone may already have asked.
- **Creator marketplaces** where the creator has signed up to receive brand offers.

Do not try to guess a personal address or get around the CAPTCHA and the daily limit. A creator who does not publish an address is telling you how they want to be contacted.

## Finding emails for many channels at once

The manual steps take a few minutes per channel. For a list of fifty or five hundred creators, there are three kinds of tool:

| Approach | What it returns | Cost model | Fits |
|---|---|---|---|
| Your own script on the YouTube Data API | Channel description and stats; you parse emails from the text | Free within the API's daily quota | Developers who only need description emails |
| Influencer databases (Modash, HypeAuditor and similar) | Searchable creator database with contacts, audience data and outreach tools | Monthly subscription | Agencies running ongoing campaigns |
| Pay-per-result scrapers on the Apify Store | Emails, links and socials read from public pages for the channels you give | Pay per result | One-off lists, pipelines, AI agents |

Note that the YouTube Data API does not return the business email behind the "View email address" button; only the creator's published text is available to it.

### Free: channel descriptions through the YouTube Data API

Create an API key in the Google Cloud console (enable "YouTube Data API v3"), then read each channel's description and pull the emails out of it. A `channels.list` call costs 1 unit of the default 10,000-unit daily quota, so a few hundred channels a day is free.

```python
# pip install requests
import os
import re

import requests

KEY = os.environ["YOUTUBE_API_KEY"]
EMAIL = re.compile(r"[\w.+-]+@[\w-]+(?:\.[\w-]+)+")

for handle in ["@mkbhd", "@aliabdaal"]:
    r = requests.get("https://www.googleapis.com/youtube/v3/channels", params={
        "part": "snippet", "forHandle": handle, "key": KEY}, timeout=20)
    items = r.json().get("items", [])
    desc = items[0]["snippet"]["description"] if items else ""
    print(handle, sorted(set(EMAIL.findall(desc))) or "no email in description")
```

This only sees the channel description. It misses addresses written as `name (at) domain`, and it does not open the creator's website or link-in-bio page; steps 2 and 3 above, or a scraper, cover those.

### YouTube Channel Email & Influencer Contacts Finder

Disclosure: Data Gleaner is us. Our YouTube Channel Email & Influencer Contacts Finder (`youtube-channel-contacts`) is not listed on the Apify Store yet; it will appear on the [Data Gleaner store page](https://apify.com/datagleaner) when it is.
<!-- TODO(store-link: youtube-channel-contacts) -->

It does steps 2 and 3 above for each channel:

- Takes channel URLs, `@handles` or `UC...` channel IDs, or a keyword such as `fitness coach` to find channels through YouTube search (up to 500 per keyword).
- Reads emails and phones from the channel description, and classifies the About links into socials, link-in-bio pages and websites.
- Visits up to 3 of the channel's own pages (link-in-bio first, then the website, then its contact page) and keeps only emails that plausibly belong to the creator.
- Returns each email with the page it was found on, plus subscribers, videos, views, country and the creator's Instagram, TikTok and X links.
- Filters by subscriber range and country, and can output only channels with an email.

It does **not** reveal the address behind the "View email address" button and never touches the CAPTCHA. Instead it reports `hasBusinessEmailButton` for each channel, so you know which creators you would have to look up by hand. On our test runs, about 45% of a mixed sample of 20 channels and 20 to 30% of a 30-channel "fitness coach" search had a public email; small channels rarely publish one.

**Price:** $0.015 per channel where at least one public email was found. Channels with no email, and channels that could not be read, are free.

Once it is listed, you call it like this:

```python
# pip install apify-client
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/youtube-channel-contacts").call(run_input={
    "channels": ["@mkbhd", "@aliabdaal"],
    "searchKeywords": ["fitness coach"],
    "maxChannelsPerKeyword": 5,
    "followLinks": True,
    "onlyWithEmail": False,
})
if run is None:
    raise SystemExit("The run did not return.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    if item.get("status") != "ok":
        print("error:", item.get("input"), item.get("error"))
        continue
    emails = [f'{e["email"]} ({e["source"]})' for e in item.get("emails", [])]
    print(f'{item.get("title")} ({item.get("handle")}), {item.get("subscribers")} subscribers')
    print("  emails:", ", ".join(emails) or "-")
    print("  button:", "yes" if item.get("hasBusinessEmailButton") else "no")
```

This run covers 2 named channels plus at most 5 from the keyword, so it costs at most 7 x $0.015 = $0.105.

If you already have the creators' websites rather than their channels, our page on [finding emails from a list of websites](../contact-and-lead-scrapers) covers that case, including the [Website Contact Details Scraper](https://apify.com/datagleaner/website-contact-details-scraper), which is live now.

## Using the emails responsibly

A YouTuber's email is personal data in many places even when it is public. Write to creators one by one about something relevant to their channel, say who you are, and honor a request to stop. Bulk cold email is regulated by GDPR in the EU and UK, CAN-SPAM in the US and similar laws elsewhere.

## FAQ

**Why is there no "View email address" button on a channel?**
The creator has not added a business email in YouTube Studio, or you are using the phone app, which may not show it. Check on a desktop browser; if it is still missing, look in the descriptions and the creator's links.

**Why did "View email address" stop working for me?**
YouTube limits how many addresses one account can reveal per day. Wait until the next day, and use the descriptions and links for the rest.

**How do I find a YouTube channel owner's email for free?**
Use the button in the About panel, then the channel and video descriptions, then the website and link-in-bio page in the About links. All of it is free; it only takes time.

**Is there a YouTube channel email finder tool?**
Yes: influencer databases sell creator contacts by subscription, and scrapers such as ours read the emails creators publish on their channel and websites. No legitimate tool can show an email the creator never published.

**How do I contact a YouTuber without an email?**
Send a DM on the social profiles linked from their channel, contact their management if they name one, or leave a short comment asking where business inquiries should go.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Why is there no \"View email address\" button on a channel?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The creator has not added a business email in YouTube Studio, or you are using the phone app, which may not show it. Check on a desktop browser; if it is still missing, look in the descriptions and the creator's links."
      }
    },
    {
      "@type": "Question",
      "name": "Why did \"View email address\" stop working for me?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "YouTube limits how many addresses one account can reveal per day. Wait until the next day, and use the descriptions and links for the rest."
      }
    },
    {
      "@type": "Question",
      "name": "How do I find a YouTube channel owner's email for free?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Use the button in the About panel, then the channel and video descriptions, then the website and link-in-bio page in the About links. All of it is free; it only takes time."
      }
    },
    {
      "@type": "Question",
      "name": "Is there a YouTube channel email finder tool?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes: influencer databases sell creator contacts by subscription, and scrapers such as ours read the emails creators publish on their channel and websites. No legitimate tool can show an email the creator never published."
      }
    },
    {
      "@type": "Question",
      "name": "How do I contact a YouTuber without an email?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Send a DM on the social profiles linked from their channel, contact their management if they name one, or leave a short comment asking where business inquiries should go."
      }
    }
  ]
}
</script>
