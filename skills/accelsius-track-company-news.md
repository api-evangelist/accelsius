---
name: Track Accelsius announcements and coverage
description: >-
  Watch for new Accelsius press releases, product announcements and third-party press coverage,
  correctly separating the two different content types the site uses for them.
api: openapi/accelsius-news-api-openapi.yml
operations:
  - searchSite
  - listContent
  - listNews
  - getNewsItem
  - getContentItem
generated: '2026-08-06'
method: generated
---

# Track Accelsius announcements and coverage

Accelsius is an Austin-based maker of two-phase, direct-to-chip liquid cooling for AI data centers.
It announces frequently — product GA, benchmark results, partner and investor news — and it splits
that flow across **two different WordPress content types**, which is the one thing that makes this
non-obvious.

Base URL: `https://accelsius.com/wp-json`. No credential.

## The split you have to get right

| What | Where it lives | How to query |
|---|---|---|
| Press releases Accelsius issues itself | `post` type, category **1** (`press-release`), 34 items | `GET /wp/v2/posts?categories=1` |
| Company news / announcements | `post` type, category **31** (`news-article`), 26 items | `GET /wp/v2/posts?categories=31` |
| Third-party press **coverage** of Accelsius | the custom **`news` type**, 8 items | `GET /wp/v2/news` |
| Company announcements by topic | tag **36** (`accelsius-news`), 51 items | `GET /wp/v2/posts?tags=36` |

A crawler that only knows about `/wp/v2/posts` silently misses the entire `news` type, and one that
only reads `/wp/v2/news` misses the 60 first-party announcements. Poll both.

## Before you start

- **No auth.** No key, no signup, no header.
- **The origin firewalls bursts.** MalCare answers **HTTP 403 with `text/html`** — not the JSON
  error envelope — with no rate-limit header and no documented threshold. `robots.txt` asks for
  `Crawl-delay: 10`. One request per ten seconds; on a 403, stop and wait minutes.
- **Cache window is 600 seconds.** Polling faster than that gains you nothing but risk. For a news
  watcher, **once an hour is generous** — this is a company that publishes a few times a week.

## Step 1 — Poll first-party announcements (`listContent`)

```
GET /wp/v2/posts?categories=1,31&after=<last_seen_utc_iso8601>&orderby=date&order=desc&per_page=20&_fields=id,slug,link,title,excerpt,date_gmt,categories,tags
```

- `categories=1,31` unions press releases and news articles.
- `after` takes ISO 8601. Pass the `date_gmt` of the newest item you already have — **use the
  `_gmt` field**, because the naked `date` is in the site's `-5` offset and will drift your window
  by five hours.
- Use `modified_after` instead of `after` if you also want to catch edits to already-seen items.

## Step 2 — Poll third-party coverage (`listNews`)

```
GET /wp/v2/news?after=<last_seen_utc_iso8601>&orderby=date&order=desc&per_page=20&_fields=id,slug,link,title,excerpt,date_gmt,categories,tags
```

Only 8 items exist today, so this is cheap. It is the outlet-coverage feed — trade press writing
about Accelsius, curated onto `https://accelsius.com/in-the-news/`.

## Step 3 — Read the item (`getContentItem` / `getNewsItem`)

```
GET /wp/v2/posts/{id}?_fields=id,slug,link,title,content,excerpt,date_gmt,categories,tags
GET /wp/v2/news/{id}?_fields=id,slug,link,title,content,excerpt,date_gmt,categories,tags
```

`content.rendered` is the full body as HTML. Prefer adding `_embed=true` to Step 1 and Step 2 over
per-item fetches — item-level request bursts are precisely what triggered the firewall during
profiling.

## Step 4 — Resolve a headline you saw elsewhere (`searchSite`)

If you have a title or a URL and need the numeric ID:

```
GET /wp/v2/search?search=<terms>&per_page=10&_fields=id,title,url,type,subtype
```

Returns a lightweight uniform record across every searchable object (192 at time of capture).
`subtype` tells you which collection to go back to — `post`, `page` or `news`.

## Cheaper alternative

If you only need "did anything new appear", the site's RSS feed at `https://accelsius.com/feed/`
(HTTP 200) is one request and no JSON parsing. Use the REST API when you need the category/tag
classification, the `news` type, or the full HTML body — the feed carries neither the custom type
nor the taxonomy IDs.

## What you will not find here

There is no changelog, no status page, no roadmap and no deprecation policy — Accelsius operates no
API program, and this content surface is a side effect of its marketing site rather than a product.
Announcements about hardware availability (for example the NeuCool IR150 reaching general
availability) land in the resource library as ordinary posts, not as release notes. See
`lifecycle/accelsius-lifecycle.yml`.
