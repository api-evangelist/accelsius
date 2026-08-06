---
name: Harvest the Accelsius site
description: >-
  Mirror the product pages, the resource library, the press-coverage type and the media library of
  accelsius.com, driven by the site's own discovery routes rather than a hardcoded content model,
  and paced so the origin firewall never triggers.
api: openapi/accelsius-discovery-api-openapi.yml
operations:
  - getRootIndex
  - listPostTypes
  - listTaxonomies
  - listPages
  - getPage
  - listContent
  - listNews
  - listMedia
generated: '2026-08-06'
method: generated
---

# Harvest the Accelsius site

Everything published on `accelsius.com` is reachable through the public WordPress REST API at
`https://accelsius.com/wp-json` with no credential. This skill mirrors it **model-first**: ask the
site what types and taxonomies it has, then walk them — so a plugin change or a new custom post
type does not silently drop content from your copy.

## The constraint that shapes the whole plan

`accelsius.com` runs the **MalCare WordPress firewall** and publishes **no rate-limit header**.
During profiling it began returning **HTTP 403 with an HTML interstitial** after a short burst of
per-item requests, while collection requests kept succeeding, and the block persisted across a
user-agent change and several minutes of waiting.

The harvest plan below is therefore built around one rule: **collections, not items.** The whole
site is 154 resource-library posts + 8 news items + 30 pages + 903 attachments = 1,095 objects,
which is **12 collection requests at `per_page=100`**. Fetching them one by one would be 1,095
requests and would be blocked long before it finished.

- One request per **10 seconds** (`robots.txt` `Crawl-delay: 10`). No parallelism.
- Always `_fields`. Always `_embed=true` instead of follow-up item fetches.
- Branch on `Content-Type`. A 403 is `text/html`, not the JSON error envelope.
- On 403: stop, wait minutes, resume from the last completed page. Do not retry tightly.
- Respect the 600-second `Cache-Control` window on re-runs.

## Step 1 — Read the route index once (`getRootIndex`)

```
GET /wp-json/?_fields=name,url,home,gmt_offset,namespaces,authentication
```

**Always pass `_fields` here.** The full document is ~285KB (387 routes across 16 namespaces); the
trimmed version is a few hundred bytes. What you want from it:

- `name` / `url` / `home` — confirm you are on the right site.
- `gmt_offset` — `-5`. Every naked `date`/`modified` field is in this offset; the `*_gmt` twins are
  UTC. Key your incremental sync on the `_gmt` fields.
- `namespaces` — 16 on this host. `wp/v2` is the content surface; the rest are plugin routes
  (`yoast/v1`, `ws-form/v1`, `wpforms/v1`, `bricks/v1`, `redirection/v1`, `leadin/v1`,
  `cleantalk-antispam/v1`, `wpe/cache-plugin/v1`, ...) and are mostly credential-gated.
- `authentication` — advertises `application-passwords` only. Confirms there is no public
  credential path.

## Step 2 — Ask what types exist (`listPostTypes`)

```
GET /wp/v2/types?_fields=slug,name,rest_base,rest_namespace,taxonomies,hierarchical
```

Do not hardcode `posts` and `pages`. This site registers a **custom `news` type** for third-party
press coverage that a stock crawler misses entirely. The publicly readable content types are:

| slug | rest_base | taxonomies | note |
|---|---|---|---|
| `post` | `posts` | category, post_tag | the resource library |
| `page` | `pages` | — | static pages, hierarchical |
| `attachment` | `media` | happyfiles_category | the media library |
| `news` | `news` | category, post_tag, recent-news | third-party press coverage |

## Step 3 — Ask what taxonomies exist (`listTaxonomies`)

```
GET /wp/v2/taxonomies?_fields=slug,name,rest_base,types,hierarchical
```

Three are anonymously readable and populated: `category` (8 terms — the content classes),
`post_tag` (5 terms — the topic axis), `happyfiles_category` (19 terms — media folders, plugin
provided). `recent-news` is registered against the news type but held zero terms. Fetch each term
list once and cache it; every content object references terms by bare integer ID.

## Step 4 — Walk the content collections

Two requests each, at `per_page=100`, with the fields you actually store:

```
GET /wp/v2/posts?per_page=100&page=1&_embed=true&_fields=id,slug,link,type,title,content,excerpt,date_gmt,modified_gmt,categories,tags,featured_media,_embedded
GET /wp/v2/news?per_page=100&page=1&_embed=true&_fields=id,slug,link,type,title,content,excerpt,date_gmt,modified_gmt,categories,tags,featured_media,_embedded
GET /wp/v2/pages?per_page=100&page=1&_fields=id,slug,link,type,title,content,parent,menu_order,date_gmt,modified_gmt
```

- Follow the `Link: <...>; rel="next"` header to page. Read `X-WP-Total` to know when you are done.
- `_embed=true` inlines the featured image and the resolved terms under `_embedded`, which is what
  keeps this a 12-request job instead of a 1,095-request one.
- Pages are hierarchical — keep `parent` and `menu_order` so you can rebuild the nav tree. The 30
  pages include the IR150 and MR250 product pages, the TSR/LSS page, the multi-GPU page, the
  two-phase and direct-to-chip technology explainers, solutions, services, partners, company,
  careers, FAQ, customer support, contact and the privacy policy.

## Step 5 — Walk the media library (`listMedia`)

```
GET /wp/v2/media?per_page=100&page=1&_fields=id,slug,title,source_url,mime_type,media_type,alt_text,post,date_gmt,media_details.sizes
```

903 attachments, so 10 pages. Split your ingestion by `mime_type`: `application/pdf` are the white
papers, studies and infographics from `/papers-studies/`; the images are product renders, diagrams
and partner logos. `media_details.sizes` gives the generated variants — pick one rather than always
taking the original.

## Step 6 — Incremental re-sync

Do not re-walk. On every later run:

```
GET /wp/v2/posts?modified_after=<last_run_utc_iso8601>&per_page=100&_fields=id,slug,modified_gmt
GET /wp/v2/news?modified_after=<last_run_utc_iso8601>&per_page=100&_fields=id,slug,modified_gmt
GET /wp/v2/pages?modified_after=<last_run_utc_iso8601>&per_page=100&_fields=id,slug,modified_gmt
```

Use the UTC `modified_gmt` value from your last successful run, not `modified`, and not local time.
Deletions are invisible on this surface — nothing reports a removed object — so periodically
reconcile your ID set against a `_fields=id` full listing rather than trusting the incremental
feed alone.

## Do not harvest

- **`/wp/v2/users`.** It answers anonymously with 14 author records. It is a directory of named
  individuals and it is excluded from every artifact in this repository on purpose. Store the
  `author` integer as an opaque key or drop it.
- **The credential-gated routes.** `settings`, `menus`, `menu-items`, `templates`, `themes`,
  `plugins`, `widgets`, `block-types` and the plugin admin namespaces all return 401 anonymously.
  They are administrative. Do not attempt to authenticate against this site.
