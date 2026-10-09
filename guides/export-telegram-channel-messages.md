---
title: How to Export Telegram Channel Messages (JSON, HTML, CSV)
description: Export Telegram channel messages to JSON, HTML or CSV with Telegram Desktop, a Telethon script or the public t.me/s/ preview, with steps, code and limits.
---

# How to export Telegram channel messages

The quickest free way to export a Telegram channel's messages is Telegram Desktop: open the channel, click the three-dot menu, choose **Export chat history**, pick HTML or JSON, and it saves every message (and optionally the media) to a folder on your computer. If you need the export to run on a schedule or feed a script, use Telethon, a Python library that talks to Telegram's API with your own account. For a public channel you do not want to sign in for, the web preview at `t.me/s/<channel>` shows the posts to anyone, and a scraper can turn it into JSON or CSV.

The right method depends on three questions: do you have a Telegram account you are willing to use, is the channel public, and do you need this once or repeatedly.

## The options at a glance

| Method | Needs a Telegram account | Works on private channels | Output | Best for |
|---|---|---|---|---|
| Telegram Desktop export | Yes | Yes, if you are a member | HTML or JSON, plus media files | A one-off archive of a channel |
| Telethon script | Yes, plus api_id and api_hash | Yes, if you are a member | Whatever you write (JSON, CSV, database) | Repeated or automated exports |
| t.me/s/ web preview | No | No, public channels only | A web page you read or parse | Reading recent posts without signing in |
| Hosted scraper on the preview | No | No, public channels only | JSON, CSV or Excel | Many public channels, scheduled, no code to maintain |

## Option 1: Telegram Desktop (free, no code)

The export lives only in the desktop app (Windows, macOS, Linux) from telegram.org. The phone apps and Telegram Web do not have it.

1. Install Telegram Desktop and sign in.
2. Open the channel. If the menu below does not show the export option, join the channel first.
3. Click the three-dot menu in the top right corner and choose **Export chat history**.
4. Choose what to include: photos, videos, voice messages, video messages, stickers, GIFs and files. Set a size limit for media if you only want text and small images.
5. Choose the format:
   - **HTML** gives readable pages that look like the chat, good for archiving or reading offline.
   - **Machine-readable JSON** gives one `result.json` file with every message, its date, text, sender, and links to the exported media files. Use this if you want to analyse the messages.
6. Optionally set a date range with the **From** and **To** fields, pick a download folder, then click **Export**.

Caveats:

- A channel whose owner turned on **Restrict saving content** cannot be exported; the app refuses the export for protected chats.
- Large channels with media take a long time and a lot of disk space. Export text first and add media only if you need it.
- To export everything in your account at once (all chats, contacts and so on), use **Settings > Advanced > Export Telegram data** instead.
- JSON text is not always a plain string: formatted messages come as a list of pieces (plain text, bold, links). Join them when you process the file.

## Option 2: a Telethon script (free, for developers)

Telethon is an open-source Python library for Telegram's MTProto API. It signs in as you, so it can read any channel your account can see, including private ones you are a member of, and public ones you have not joined.

1. Go to [my.telegram.org](https://my.telegram.org), sign in with your phone number, open **API development tools** and create an application. You get an `api_id` and an `api_hash`. Keep them private.
2. Install the library: `pip install telethon`.
3. Run a script like this one, which writes every message of a channel to a JSON file:

```python
import json

from telethon.sync import TelegramClient

api_id = 1234567          # from my.telegram.org
api_hash = "your_api_hash"
channel = "durov"         # username, t.me link, or a channel you are a member of

with TelegramClient("export_session", api_id, api_hash) as client:
    rows = []
    for msg in client.iter_messages(channel, reverse=True):  # oldest first
        rows.append({
            "id": msg.id,
            "date": msg.date.isoformat(),
            "text": msg.message,
            "views": msg.views,
            "forwards": msg.forwards,
            "has_media": msg.media is not None,
        })

with open(f"{channel}.json", "w", encoding="utf-8") as f:
    json.dump(rows, f, ensure_ascii=False, indent=2)
print(len(rows), "messages saved")
```

On the first run Telethon asks for your phone number and the login code Telegram sends you, then stores the session in `export_session.session` so later runs do not ask again. Treat that file like a password: anyone who has it is signed in as you.

Caveats:

- Use `limit=` or `offset_date=` on `iter_messages` to export only part of a long channel.
- To download media, call `client.download_media(msg)` for the messages that have it. This is slow and large on media-heavy channels.
- Telegram limits how fast an account can make requests. If you hit a `FloodWaitError`, wait the number of seconds it gives and slow down; Telethon sleeps through short waits on its own.
- You are using your personal account, so follow Telegram's API terms.

## Option 3: the public t.me/s/ web preview (no account)

Most public channels have a web preview that anyone can open in a browser without signing in: add `/s/` to the link, as in `https://t.me/s/durov`. It shows the most recent posts with their text, dates, view counts, reactions and media, and loads older posts as you scroll up.

This is enough to read or copy a few dozen recent posts by hand, or to save the page. It has limits:

- Only public channels whose owner has not switched the preview off. If `t.me/s/<channel>` sends you to the plain `t.me/<channel>` page, there is no preview to read.
- No private channels, groups or invite links (`t.me/+...`).
- Counts are rounded the way Telegram shows them (`1.2K` views).
- Scrolling back through thousands of posts by hand is impractical. You can parse the page with your own code (it loads about 20 messages at a time), but then you maintain the parser when Telegram changes the page.

## Option 4: a hosted scraper for public channels you are not in

If you need messages from many public channels, on a schedule, without signing in and without maintaining your own parser, a hosted scraper that reads the same public preview is the low-effort route. Several exist on the Apify Store, each billed per result.

Disclosure: Data Gleaner Telegram Channel Scraper is us. It reads `t.me/s/<channel>` for each channel you give it and returns one row per message with the date, text (plain and HTML), views, reactions, media links, forwards, replies, polls, links, hashtags and mentions, plus one info row per channel with its title, description and subscriber count. It needs no Telegram account, API key or phone number. You can export the result as JSON, CSV or Excel, or read it through the API, and `onlyNewerThan` (for example `1 day`) limits each run to new posts for a scheduled feed.

Pricing is $0.50 per 1,000 messages ($0.0005 each) and $0.001 per channel info row. For example, 10 channels with 500 messages each and channel info on comes to 5,000 x $0.0005 + 10 x $0.001 = $2.51.

**Telegram Channel Scraper is coming to the Apify Store.** Until it is listed, see [apify.com/datagleaner](https://apify.com/datagleaner) for our public Actors.
<!-- TODO(store-link: telegram-channel-scraper) -->

Once it is live, this Python script prints the latest posts of a channel (install with `pip install apify-client` and set `APIFY_TOKEN`; this run returns at most 10 messages and 1 channel row, so it costs at most $0.006):

```python
import os

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
run = client.actor("datagleaner/telegram-channel-scraper").call(run_input={
    "channels": ["durov"],
    "maxMessagesPerChannel": 10,
})
if run is None:
    raise SystemExit("The Actor run did not start.")

for item in client.dataset(run.default_dataset_id).iterate_items():
    if item["type"] == "channel":
        print("Channel:", item.get("title"), "-", item.get("subscribers"), "subscribers")
    else:
        print(item["url"], item.get("date"), item.get("views"), (item.get("text") or "")[:80])
```

It shares the preview's limits: public channels with the preview on only, no private channels or groups, no member lists, no comments from a linked discussion group, rounded counts, and some documents and large videos come back without a media URL because the preview does not include them. If you are a member of a private channel, use Telegram Desktop or Telethon instead.

## FAQ

**Can I export a Telegram channel on my phone?**
No. The Android and iOS apps have no chat export. Use Telegram Desktop on a computer, or open `t.me/s/<channel>` in a mobile browser to read a public channel's recent posts.

**How do I export a Telegram channel I am not a member of?**
For a public channel, Telethon can read it by username without joining, and the `t.me/s/` preview (or a scraper built on it) needs no account at all. Telegram Desktop may ask you to join before it offers the export. A private channel cannot be exported unless you are a member.

**Why is "Export chat history" missing or greyed out?**
Either you are on the phone app or Telegram Web, which do not have it, or the channel has **Restrict saving content** turned on, which blocks exporting, forwarding and saving its content.

**Can I export a channel's members or subscribers?**
Not from a channel you do not administer. Telegram shows the subscriber count publicly, but the member list is visible only to the channel's admins. None of the methods above lists a public channel's subscribers.

**How do I get the export into CSV or Excel?**
Telegram Desktop gives HTML or JSON only. Load the JSON in Python (`pandas.json_normalize` on the `messages` list) and save it as CSV, or write CSV directly from a Telethon script. A hosted scraper's dataset downloads as CSV or Excel directly.

## Related guides

- [Web scraping for AI agents with MCP](web-scraping-for-ai-agents-mcp): run scrapers like this one from Claude, Cursor or other MCP clients.
- [Weibo API in Python](weibo-api-python): the same choice between official APIs, your own code and a hosted scraper, for Weibo.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Can I export a Telegram channel on my phone?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "No. The Android and iOS apps have no chat export. Use Telegram Desktop on a computer, or open t.me/s/<channel> in a mobile browser to read a public channel's recent posts."
      }
    },
    {
      "@type": "Question",
      "name": "How do I export a Telegram channel I am not a member of?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "For a public channel, Telethon can read it by username without joining, and the t.me/s/ preview (or a scraper built on it) needs no account at all. Telegram Desktop may ask you to join before it offers the export. A private channel cannot be exported unless you are a member."
      }
    },
    {
      "@type": "Question",
      "name": "Why is \"Export chat history\" missing or greyed out?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Either you are on the phone app or Telegram Web, which do not have it, or the channel has Restrict saving content turned on, which blocks exporting, forwarding and saving its content."
      }
    },
    {
      "@type": "Question",
      "name": "Can I export a channel's members or subscribers?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not from a channel you do not administer. Telegram shows the subscriber count publicly, but the member list is visible only to the channel's admins. None of the methods above lists a public channel's subscribers."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get the export into CSV or Excel?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Telegram Desktop gives HTML or JSON only. Load the JSON in Python (pandas.json_normalize on the messages list) and save it as CSV, or write CSV directly from a Telethon script. A hosted scraper's dataset downloads as CSV or Excel directly."
      }
    }
  ]
}
</script>
