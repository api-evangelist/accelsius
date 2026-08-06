---
name: Research the Accelsius technical library
description: >-
  Find and read Accelsius white papers, case studies, benchmark posts, infographics and podcasts on
  two-phase direct-to-chip liquid cooling, from the public WordPress content API, resolving content
  classes and topic tags correctly.
api: openapi/accelsius-content-api-openapi.yml
operations:
  - listCategories
  - listTags
  - listContent
  - getContentItem
  - listMedia
generated: '2026-08-06'
method: generated
---

# Research the Accelsius technical library

Accelsius makes two-phase, direct-to-chip liquid cooling for AI and HPC data centers. Its resource
library at `https://accelsius.com/resources/` is where the substantive material lives — thermal
benchmarks against single-phase warm-water cooling, TCO studies, deployment case studies and
recorded conversations. All of it is readable through the public WordPress REST API with **no
credential of any kind**. Base URL: `https://accelsius.com/wp-json`.

## Before you start — read this, it decides whether the skill works

- **No auth.** Send no `Authorization` header. There is no key to obtain and no signup.
- **This host bites.** `accelsius.com` runs the MalCare WordPress firewall. It answers **HTTP 403
  with an HTML page** — not the JSON error envelope — once it decides a client is hostile, and the
  block persists. It publishes **no rate-limit header at all**, so you cannot discover the
  threshold; you can only stay under it. `robots.txt` asks for `Crawl-delay: 10`. **One request per
  ten seconds. No parallelism.**
- **Branch on content-type.** A 403 here is `text/html`. If you assume JSON you will crash on the
  interstitial instead of backing off.
- **Trim the payload.** A single post object is roughly 17KB, almost all of it the `yoast_head`
  string and the `acf` block. Always pass `_fields`.
- **Cache.** Responses carry `Cache-Control: max-age=600`. Do not re-poll inside that window.

## Step 1 — Learn the content classes (`listCategories`)

```
GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count
```

This matters more here than on a typical site, because **Accelsius does not use post types to
separate content classes — it uses categories.** A white paper and a blog post are the same
`post` type; only the category tells them apart. The live set, with published counts observed on
2026-08-06:

| id | slug | count |
|---|---|---|
| 26 | blog | 39 |
| 1  | press-release | 34 |
| 31 | news-article | 26 |
| 27 | white-paper | 24 |
| 37 | video | 13 |
| 28 | infographic | 9 |
| 38 | podcast | 8 |
| 29 | case-study | 1 |

## Step 2 — Learn the topic axis (`listTags`)

```
GET /wp/v2/tags?per_page=100&_fields=id,name,slug,count
```

Tags are orthogonal to categories and there are only five:

| id | slug | count | what it means |
|---|---|---|---|
| 34 | 2pd2c-education | 70 | explainer material on two-phase direct-to-chip cooling |
| 33 | neucool-products | 53 | the IR150, MR250, TSR/LSS and cold-plate product line |
| 36 | accelsius-news | 51 | company announcements |
| 35 | scientific-research | 18 | benchmarks, thermal studies, peer material |
| 20 | featured | 8 | editorially promoted |

The research-grade material is the intersection you want: `categories=27` (white-paper) or
`categories=29` (case-study), crossed with `tags=35` (scientific-research).

## Step 3 — List, filtered (`listContent`)

```
GET /wp/v2/posts?categories=27&tags=35&per_page=20&page=1&_fields=id,slug,link,title,excerpt,date,modified,categories,tags,featured_media
```

- Filter with `categories`, `categories_exclude`, `tags`, `tags_exclude`, `slug`, `search`,
  `after`/`before` and `modified_after`/`modified_before` (ISO 8601).
- Order with `order` (`asc`|`desc`) and `orderby` (`date`, `modified`, `title`, `slug`,
  `relevance`, `include`, `id`).
- `per_page` is bounded 1–100 by the route descriptor; the default is 10.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers, and follow the RFC 8288
  `Link: <...>; rel="next"` header rather than incrementing `page` blindly.

At time of capture the whole collection held **154** published items, so a full walk at
`per_page=100` is two requests — do it that way rather than as 154 item fetches, which is exactly
the pattern that got this pass firewalled.

## Step 4 — Read one item (`getContentItem`)

```
GET /wp/v2/posts/{id}?_fields=id,slug,link,title,content,excerpt,date,modified,categories,tags,featured_media,acf
```

`content.rendered` is the full article as HTML. `acf` carries the Advanced Custom Fields payload
the site attaches to the type — plugin-provided, so treat its presence as optional and never key on
it. Do not request the object without `_fields` unless you actually want the SEO head blob.

**Cheaper alternative:** add `_embed=true` to Step 3 instead of doing per-item fetches. It inlines
the author, the terms and the featured image under `_embedded` in the same response — three round
trips collapsed into zero, which is the difference between staying under the firewall and not.

## Step 5 — Get the PDFs (`listMedia`)

The white papers, infographics and studies linked from `https://accelsius.com/papers-studies/` are
attachments, not posts:

```
GET /wp/v2/media?mime_type=application/pdf&per_page=100&_fields=id,slug,title,source_url,mime_type,date
```

`source_url` is the direct file URL. The library held **903** attachments in total; the PDF subset
is what you want for document ingestion. Media files are served from
`https://accelsius.com/wp-content/uploads/...` and are not behind the REST firewall path, but the
same courtesy applies.

## Errors

| Status | Body | What to do |
|---|---|---|
| 400 | `{"code":"rest_invalid_param",...}` | Read `data.params`. Usually `per_page` outside 1–100. |
| 404 | `{"code":"rest_post_invalid_id",...}` | The ID does not exist or is not published. List first. |
| 401 | `{"code":"rest_forbidden",...}` | Administrative route. There is no public credential — do not try to authenticate. |
| **403** | **`text/html`, MalCare interstitial** | **Stop. Back off for minutes, not seconds. Retrying immediately extends the block.** |

Full catalog: `errors/accelsius-problem-types.yml`. Note the caveat recorded there — the JSON error
bodies were never captured on this host, because the firewall replaced them.

## What this surface is not

This is the content API of a marketing site. Accelsius sells cooling hardware; it has no developer
program, no product API, no telemetry or monitoring API for its CDUs, no SDK and no GitHub
organization. Nothing here reads a rack, a coolant loop or a junction temperature. If you need
NeuCool operational data, that is a sales conversation, not an API call.
