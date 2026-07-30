---
name: lish-blog-harvest
description: >-
  Harvest the complete Lish blog archive and page inventory from the public
  WordPress REST API behind lishfood.com, correctly paginated and trimmed. Use
  when building an index of Lish content, mirroring the archive, doing a content
  audit, or answering questions that need the full corpus rather than one page.
api: Lish WordPress REST API
base_url: https://www.lishfood.com/wp-json/wp/v2
auth: none
operations:
  - listPosts
  - listPages
  - listCategories
  - listTags
  - listUsers
  - getMediaItem
generated: '2026-07-19'
method: generated
source: openapi/lish-wordpress-openapi.json
---

# Harvest the Lish content archive

The corpus is small and fully anonymous-readable. As of 2026-07-19: **28 posts,
60 pages, 4 categories**. A complete harvest is a handful of requests.

## Step 1 — size the job before fetching

Issue a one-item request and read the count headers rather than guessing:

```
GET /posts?per_page=1   →  X-WP-Total: 28,  X-WP-TotalPages: 28
GET /pages?per_page=1   →  X-WP-Total: 60,  X-WP-TotalPages: 60
```

`X-WP-Total` is the number you want. Both headers are CORS-exposed.

## Step 2 — pull in pages of 100

`per_page` is capped at 100 — sending 101 returns 400 `rest_invalid_param` with
`data.params.per_page` explaining the bound. At this corpus size everything fits
in a single page per collection:

```
GET /posts?per_page=100&_fields=id,slug,date,modified,title,excerpt,link,categories,tags,author,featured_media
GET /pages?per_page=100&_fields=id,slug,date,modified,title,link,parent,menu_order
```

Always pass `_fields`. Without it each item carries fully rendered HTML for
content and excerpt, which inflates a 28-item response by orders of magnitude.

## Step 3 — terminate the loop correctly

For a larger install, loop — but terminate on the `Link` header, not on an empty
body:

```
1. GET /posts?per_page=100&page=1
2. Read the Link response header.
3. If it contains rel="next", follow that URL. Otherwise stop.
```

`Link: <https://wordpress.lishfood.com/wp-json/wp/v2/posts?per_page=2&page=2>; rel="next"`

**Never loop until you receive an empty array.** Paging past the last page returns
HTTP 400 `rest_post_invalid_page_number`, so a naive loop terminates on an error
and risks discarding a good final page. Either follow `rel="next"`, or compute the
bound from `X-WP-TotalPages` up front.

Note the `Link` header uses the `wordpress.lishfood.com` origin. Following it
verbatim works, but normalize the host to `www.lishfood.com` in anything you store
or display.

## Step 4 — fetch full bodies only for what you need

The trimmed list gives you ids. Pull full content per item, with relations inlined:

```
GET /posts/{id}?_embed
```

`_embed` populates `_embedded` with `author`, `wp:featuredmedia` and `wp:term`,
which resolves the author name, the featured image `source_url` and the category
and tag names without three extra calls each. For 28 posts this turns roughly 112
requests into 28.

## Step 5 — resolve the taxonomy and author dimensions

```
GET /categories?per_page=100&_fields=id,slug,name,count
GET /tags?per_page=100&_fields=id,slug,name,count
GET /users?per_page=100&_fields=id,slug,name
```

Expect a lopsided taxonomy: of the 4 categories, `corporate-catering` (id 32)
holds all 28 posts and `health-wellness`, `lish-company` and `uncategorized`
hold zero. Do not build a category-faceted index on this data — it has one real
facet. `listUsers` returns only authors of published content anonymously.

## Step 6 — normalize what you store

For each harvested item, record:

- `id`, `slug`, `date`, `modified` — `modified` is the field to diff on re-runs
- `title.rendered` and `content.rendered`, HTML-stripped
- `link`, host-rewritten from `wordpress.lishfood.com` to `www.lishfood.com`
- the `_embedded` author name, featured-image `source_url` and term names

## Re-running incrementally

`modified` supports server-side filtering, so a delta pull is one request:

```
GET /posts?modified_after=2026-07-19T00:00:00&per_page=100&_fields=id,slug,modified,title,link
```

Also compare `X-WP-Total` against your stored count to detect deletions, which
`modified_after` cannot surface.

## Constraints

- Read-only. Never issue a write method against a live company marketing site.
- No rate-limit headers exist. Responses cache for 600 seconds behind WP Engine
  and CloudFront. Keep concurrency low and space out full harvests — a complete
  pull is under 100 requests, so there is no reason to parallelize aggressively.
- Errors use `{"code","message","data":{"status"}}`, not RFC 9457. Branch on
  `code`. See `errors/lish-problem-types.yml` for the verified catalog.
- The route surface is a CMS byproduct with no stability contract. Re-read
  `https://www.lishfood.com/wp-json/` before a scheduled harvest and fail loudly
  if `wp/v2/posts` is no longer advertised.
