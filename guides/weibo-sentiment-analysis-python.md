---
title: "Weibo Sentiment Analysis in Python: jieba, SnowNLP and BERT"
description: "Weibo sentiment analysis in Python end to end: collect posts with region and likes, segment with jieba, score with SnowNLP and a Chinese BERT, chart by province."
---

# Weibo sentiment analysis in Python

To run sentiment analysis on Weibo in Python, collect the posts for a keyword with their timestamp, province and like counts, clean the text, then score each post twice: once with SnowNLP, which is fast but was trained mainly on shopping reviews, and once with a Chinese BERT-family classifier from Hugging Face, which is slower and usually reads informal text better. Use jieba to segment the text for word counts and for reading what drives a score. Group the scores by day and province in pandas and chart them. This guide gives a script for each stage and states the limits of each model, because a sentiment number from the wrong model looks exactly as confident as one from the right model.

Disclosure: Data Gleaner, mentioned near the end as one way to collect the posts, is us. Every library and model on this page is free, and the analysis works on Weibo data from any source.

## The pipeline

1. **Collect** posts for a keyword: text, `createdAt`, `region`, `likesCount`, `repostsCount`, `commentsCount`.
2. **Clean** the text: remove links, `@mentions`, hashtag marks and the `[name]` emoji codes Weibo uses.
3. **Segment** with jieba, so you can count words and read the vocabulary behind a score.
4. **Score** with SnowNLP (baseline) and a Chinese classifier (stronger).
5. **Aggregate** by day and by province, then chart.

Which keywords to track, how to run a search daily and how to triage by engagement is covered in [Weibo brand monitoring](weibo-brand-monitoring). This page covers the language processing after you have the posts.

## Step 1: get the posts

Sentiment by province and by day needs three fields next to the text: when the post was written, where it was sent from, and how much engagement it got. The options for getting them:

- **The official Open Platform API.** Free, but it needs a verified developer account and mainly covers your own and your authorized users' data. See [Weibo API in Python](weibo-api-python).
- **Your own script against the public site.** Free, and you maintain it when Weibo changes. See [Weibo API in Python](weibo-api-python).
- **A hosted scraper**, described near the end of this page.
- **An open-source toolkit.** [Weibo-Analyst](https://gitee.com/g_12/Weibo-Analyst) on Gitee is a beginner-level Chinese-language toolbox that crawls Weibo comments and runs segmentation, word clouds, sentiment and LDA topics on them.

Whatever you use, save the posts as a JSON list with these field names, which is what the scripts below read:

```json
[
  {
    "createdAt": "2026-10-07T23:49:02+08:00",
    "text": "#瑞幸# 新品真的好喝，下次还买 [good]",
    "region": "江苏",
    "likesCount": 12,
    "repostsCount": 1,
    "commentsCount": 3
  }
]
```

Two cautions before you score anything. A keyword search returns the newest posts first and is capped, so for a busy keyword a one-week window may be a sample, not everything. And `region` is the province or country Weibo displays on the post; some posts have none, so drop those from the province chart rather than guess.

## Step 2: clean the text

Weibo text has noise that models were not trained on. This function removes links, mentions, hashtag marks and emoji codes. It keeps the words inside a hashtag, because `#瑞幸#` is often the subject of the post.

```python
# pip install pandas
import json
import re

import pandas as pd

URL = re.compile(r"https?://\S+")
MENTION = re.compile(r"@[^\s@:：,，。!！?？]+[:：]?")
EMOJI_CODE = re.compile(r"\[[^\[\]\s]{1,8}\]")  # Weibo writes emoji as [name]


def clean(text):
    text = URL.sub(" ", text)
    text = MENTION.sub(" ", text)
    text = EMOJI_CODE.sub(" ", text)
    text = text.replace("#", " ")
    return re.sub(r"\s+", " ", text).strip()


with open("posts.json", encoding="utf-8") as f:
    df = pd.DataFrame(json.load(f))

df["clean"] = df["text"].fillna("").map(clean)
df = df[df["clean"].str.len() >= 2].copy()
df["date"] = pd.to_datetime(df["createdAt"], utc=True).dt.tz_convert("Asia/Shanghai").dt.date
print(len(df), "posts after cleaning")
```

Removing emoji codes loses information: `[哭]` (crying) is a strong signal. If your topic is emotional, replace a few of them with words (for example `[哭]` with `难过`) before removing the rest. Dates are converted to Beijing time so a post written at 23:49 stays on its own day.

## Step 3: segment with jieba

Chinese is written without spaces, so word counts need segmentation. jieba (`pip install jieba`) is the common choice and supports a custom dictionary. Its default precise mode, `jieba.lcut`, is the one meant for text analysis.

```python
# pip install jieba
from collections import Counter

import jieba

# Teach jieba your brand and product names, or it may split them.
for word in ["瑞幸", "生椰拿铁", "酱香拿铁"]:
    jieba.add_word(word)

STOP = set("的 了 是 我 你 他 也 都 就 在 和 有 不 很 这 那 吗 啊 吧 呢 还 要 去 说 看 一个 什么 可以 真的 觉得".split())


def words(text):
    return [w for w in jieba.lcut(text) if len(w) > 1 and w not in STOP]


df["words"] = df["clean"].map(words)
print(Counter(w for ws in df["words"] for w in ws).most_common(20))
```

The stop word list is a short example, not a standard one. For research, start from a published Chinese stop word list and add the terms that are noise in your topic. Segmentation errors on new product names and slang are the main reason to use `jieba.add_word` and to check the top words by eye.

## Step 4a: score with SnowNLP

SnowNLP (`pip install snownlp`) returns `sentiments`, a number between 0 and 1 that its README describes as the probability that the text is positive. It needs no GPU and no download, so it is the right first pass.

```python
# pip install snownlp
from snownlp import SnowNLP

df["snow"] = df["clean"].map(lambda t: SnowNLP(t).sentiments)
```

SnowNLP's README is direct about the limit: its sentiment training data is mainly reviews written when buying things, so other kinds of text may work poorly. Weibo is the other kind of text. Sarcasm, news-style posts, fan talk and neutral posts all get pushed toward 0 or 1. Treat the score as a rough direction, check a sample by hand, and do not report a mean like 0.62 as a measured opinion. SnowNLP segments internally, so it does not use the jieba output.

## Step 4b: score with a Chinese BERT

A transformer classifier reads the whole sentence rather than counting words. The example uses `uer/roberta-base-finetuned-jd-binary-chinese` on Hugging Face, one of five Chinese RoBERTa-base classifiers from the UER-py project. Its model card says the JD datasets consist of user reviews of different sentiment polarities, so its training domain is close to SnowNLP's, but it handles context far better than word counts. Install `transformers` and `torch` first.

```python
# pip install transformers torch
from transformers import pipeline

MODEL = "uer/roberta-base-finetuned-jd-binary-chinese"
clf = pipeline("text-classification", model=MODEL, top_k=None, truncation=True, max_length=256)


def p_positive(batch):
    out = []
    for scores in clf(batch):
        pos = [s["score"] for s in scores if "pos" in s["label"].lower()]
        out.append(pos[0] if pos else None)
    return out


texts = df["clean"].tolist()
scores = []
for i in range(0, len(texts), 32):
    scores += p_positive(texts[i:i + 32])
df["bert"] = scores
print(df[["clean", "snow", "bert"]].head())
```

The script finds the positive label by name, not by position, because label order differs between models. This model's labels are `positive (stars 4 and 5)` and `negative (stars 1, 2 and 3)`, so it has no neutral class and a lukewarm post can land on either side. If you switch models, run `clf("这本书真的很不错")` once and read the label names. The first run downloads the model, and on a CPU a few thousand short posts take minutes.

The most reliable improvement is to label a few hundred of your own posts by hand. They let you measure how wrong each model is on your topic, which no model card can, and they are training data if you later fine-tune a model on social media text.

## Step 5: chart sentiment by day and by province

Group the scores by day and region. This script shows the plain mean and a mean weighted by likes plus reposts plus comments, because a post with 5,000 likes says more about public mood than one with none. Weighting is a choice, so state it in your write-up.

```python
import matplotlib.pyplot as plt

plt.rcParams["font.sans-serif"] = ["PingFang SC", "Microsoft YaHei", "Noto Sans CJK SC", "SimHei"]
plt.rcParams["axes.unicode_minus"] = False  # needed for Chinese fonts

df["score"] = df["bert"].fillna(df["snow"])
df["weight"] = 1 + df[["likesCount", "repostsCount", "commentsCount"]].fillna(0).sum(axis=1)

df["wscore"] = df["score"] * df["weight"]

by_day = df.groupby("date").agg(
    posts=("score", "size"), mean=("score", "mean"), wsum=("wscore", "sum"), wtotal=("weight", "sum")
)
by_day["weighted"] = by_day["wsum"] / by_day["wtotal"]

by_region = (
    df[df["region"].fillna("") != ""]
    .groupby("region")
    .agg(posts=("score", "size"), mean=("score", "mean"))
)
by_region = by_region[by_region["posts"] >= 20].sort_values("mean")  # ignore tiny samples

fig, (a, b) = plt.subplots(1, 2, figsize=(13, 5))
by_day[["mean", "weighted"]].plot(ax=a, marker="o", title="Sentiment by day (1 = positive)")
by_region["mean"].plot.barh(ax=b, title="Sentiment by province (20+ posts)")
plt.tight_layout()
plt.savefig("weibo_sentiment.png", dpi=150)
```

The minimum of 20 posts per province is a judgment call: with five posts, one angry user moves a province's mean a lot. For a map you need a GeoJSON file of Chinese provinces and a library such as pyecharts or plotly, and the names in `region` must match the names in that file. Check for the ones that do not, such as a region that is a foreign country.

## Honest limits

- **Binary scores hide neutral posts.** Most posts about a brand are news, ads, giveaways or reposts. A model forced to choose will put them somewhere. Filter reposts and ads first, or report a neutral band such as 0.4 to 0.6 separately.
- **Posts are not people.** One user can post many times, and marketing accounts look like enthusiasm. Deduplicate or weight by author if you have the author field.
- **Region is where the post was sent from**, not where the person lives or buys. It is a proxy.
- **Weibo users are not the population**, and a capped keyword search is a sample. A result describes the posts you collected, not China.
- **Sarcasm and in-group slang defeat every model here.** Read the lowest and highest scored posts on every run.

## Option: collect the posts with the Data Gleaner Weibo scraper

The Data Gleaner Weibo Scraper (`datagleaner/weibo-scraper` on Apify) returns public posts found by keyword with no Weibo login, cookie or API key. It costs $3 per 1,000 posts, charged only for posts saved to the dataset, so a 1,000-post sample for one keyword is $3. Each item has `text` as plain text (HTML removed, emoji as `[name]`), `createdAt` as ISO 8601 with timezone, `likesCount`, `repostsCount`, `commentsCount`, `region` (the province Weibo shows) and `author`.

Its main limit is your sample size: Weibo serves about 1,000 posts per keyword search, however large the topic. For more, split the topic into narrower keywords, or collect one date range at a time with `sinceDate` and `untilDate`. Only public posts are returned. Weibo can also throttle its guest access, so a run can return fewer posts than requested. To score the replies under posts as well, set `includeComments` (and `maxCommentsPerPost`): each post then carries a `comments` list with `text`, `likesCount` and `region`, billed at $2 per 1,000 comments.

The input fields used here are `searchQueries`, `maxItemsPerQuery`, `sinceDate` and `untilDate`. This script saves the dataset in the `posts.json` format the analysis reads:

```python
# pip install apify-client   (version 3.x)
import json
from decimal import Decimal

from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("datagleaner/weibo-scraper").call(
    run_input={
        "searchQueries": ["瑞幸"],
        "maxItemsPerQuery": 1000,
        "sinceDate": "2026-10-01",
        "untilDate": "2026-10-07",
    },
    max_total_charge_usd=Decimal("3"),  # hard cap on what this run can cost
)
if run is None:
    raise SystemExit("The run did not finish")

posts = list(client.dataset(run.default_dataset_id).iterate_items())
with open("posts.json", "w", encoding="utf-8") as f:
    json.dump(posts, f, ensure_ascii=False)
print(len(posts), "posts saved")
```

This run costs at most $3: `max_total_charge_usd` stops it there even if you later raise `maxItemsPerQuery` or add keywords.

## FAQ

**How do I do sentiment analysis on Weibo posts in Python?** Collect the posts, remove links, mentions and emoji codes, then score each post with a model that handles Chinese: SnowNLP for a quick baseline or a Chinese BERT-family classifier from Hugging Face for better accuracy. Use jieba to segment the text if you also want word counts. Then group the scores by date and province in pandas.

**Is SnowNLP good enough for Weibo?** It is good enough for a first look and not for a published result. Its README says the sentiment training data is mainly reviews written when buying things, and that other kinds of text may work poorly. Compare it with a transformer model and with a few hundred posts you label by hand before you trust it.

**Which Chinese BERT model should I use for sentiment?** Start with a Chinese classifier that has a model card you can read, such as the UER RoBERTa classifiers on Hugging Face, and test it on your own labelled sample. A model trained on social media text should fit Weibo better than one trained on product reviews, but the way to choose is to measure accuracy on your own posts, not to pick by name.

**Do I need jieba if I use SnowNLP or BERT?** Not for scoring: SnowNLP segments internally and the BERT tokenizer works on characters. You need jieba for word frequency, keyword extraction, topic modelling and for explaining which words appear in the lowest-scoring posts. Add your brand names with `jieba.add_word` so they are not split.

**How do I get the province of a Weibo post?** Weibo displays where a post was sent from, and a scraper can read it into a field. In the Data Gleaner Actor it is `region`. It is not where the person lives, and some posts have none, so drop empty values from a regional chart.

**Can I use this for academic research?** Yes as a method, but record how you collected the posts (keyword, date range, tool, collection date), validate the sentiment model against hand-labelled data and report its accuracy. Posts are personal data, so follow your ethics board, the platform's terms and laws such as China's PIPL or the GDPR that apply to you. This is not legal advice.

## Related guides

- [Weibo brand monitoring: a low-cost social listening setup](weibo-brand-monitoring): the keyword list, daily runs and triage that come before this analysis.
- [Weibo API in Python](weibo-api-python): the official API, the website endpoints and hosted options compared.

<!-- jsonld:auto -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How do I do sentiment analysis on Weibo posts in Python?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Collect the posts, remove links, mentions and emoji codes, then score each post with a model that handles Chinese: SnowNLP for a quick baseline or a Chinese BERT-family classifier from Hugging Face for better accuracy. Use jieba to segment the text if you also want word counts. Then group the scores by date and province in pandas."
      }
    },
    {
      "@type": "Question",
      "name": "Is SnowNLP good enough for Weibo?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "It is good enough for a first look and not for a published result. Its README says the sentiment training data is mainly reviews written when buying things, and that other kinds of text may work poorly. Compare it with a transformer model and with a few hundred posts you label by hand before you trust it."
      }
    },
    {
      "@type": "Question",
      "name": "Which Chinese BERT model should I use for sentiment?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Start with a Chinese classifier that has a model card you can read, such as the UER RoBERTa classifiers on Hugging Face, and test it on your own labelled sample. A model trained on social media text should fit Weibo better than one trained on product reviews, but the way to choose is to measure accuracy on your own posts, not to pick by name."
      }
    },
    {
      "@type": "Question",
      "name": "Do I need jieba if I use SnowNLP or BERT?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Not for scoring: SnowNLP segments internally and the BERT tokenizer works on characters. You need jieba for word frequency, keyword extraction, topic modelling and for explaining which words appear in the lowest-scoring posts. Add your brand names with jieba.add_word so they are not split."
      }
    },
    {
      "@type": "Question",
      "name": "How do I get the province of a Weibo post?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Weibo displays where a post was sent from, and a scraper can read it into a field. In the Data Gleaner Actor it is region. It is not where the person lives, and some posts have none, so drop empty values from a regional chart."
      }
    },
    {
      "@type": "Question",
      "name": "Can I use this for academic research?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes as a method, but record how you collected the posts (keyword, date range, tool, collection date), validate the sentiment model against hand-labelled data and report its accuracy. Posts are personal data, so follow your ethics board, the platform's terms and laws such as China's PIPL or the GDPR that apply to you. This is not legal advice."
      }
    }
  ]
}
</script>
