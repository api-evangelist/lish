---
name: lish-content-search
description: >-
  Find and read content published by Lish, the Seattle corporate catering company —
  blog posts, marketing pages, service descriptions, FAQ and partner-chef
  information — using the public WordPress REST API behind lishfood.com. Use when
  asked what Lish offers, what a Lish service page says, or to search the Lish blog.
api: Lish WordPress REST API
base_url: https://www.lishfood.com/wp-json/wp/v2
auth: none
operations:
  - searchContent
  - listPosts
  - getPost
  - listPages
  - getPage
  - listCategories
  - listTags
generated: '2026-07-19'
method: generated
source: openapi/lish-wordpress-openapi.json
---

# Search and read Lish content

Lish is a corporate catering and workplace food service company serving Seattle
and the Bellevue/Eastside area. It has no product API. The only public API is the
WordPress REST API behind its marketing site, which is enough to answer questions
about what Lish sells, how its service works, and what it has published.

No credentials are required. Every call below is a plain anonymous GET.

## Step 1 — search when you do not know the id

`searchContent` is the entry point. It searches posts and pages together and
returns lightweight stubs.

```
GET https://www.lishfood.com/wp-json/wp/v2/search?search=catering&per_page=10
```

Each result looks like:

```json
{ "id": 10251,
  "title": "Indian Food Catering for Corporate Events: Bold Flavors and Vegetarian Options",
  "url": "https://wordpress.lishfood.com/blog/indian-food-catering-corporate",
  "type": "post", "subtype": "post" }
```

Read `type` to decide the follow-up call: `post` → `getPost`, `page` → `getPage`.

**Rewrite the origin before showing any URL to a user.** The install reports its
home as `wordpress.lishfood.com` and emits that host in `url`, `link`, `_links`
and `Link` headers. The public site is `www.lishfood.com`. Swap the host.

## Step 2 — fetch a known marketing page directly

Most substantive Lish business content is in **pages**, not posts (60 pages vs 28
posts). If you already know the page, skip search and filter by slug — one call,
no pagination:

```
GET /pages?slug=about&_fields=id,slug,link,title,content
GET /pages?slug=faq&_fields=id,slug,title,content
GET /pages?slug=lish-technology&_fields=id,slug,title,content
GET /pages?slug=our-chefs&_fields=id,slug,title,content
```

Useful slugs: `about`, `faq`, `lish-technology`, `our-chefs`, `why-workplace-food`,
`subscription-catering`, `popups`, `order-catering`, `lunch-catering`,
`breakfast-catering`, `happy-hour-catering`, `box-lunch-catering`,
`meeting-event-catering`, `testimonials`, `lish-cares`, `chef-signup`,
`catering-drivers`, `terms`, `privacy`, `contact-us`.

The slug filter returns an **array**, not a single object — take element 0, and
treat an empty array as "page not found".

## Step 3 — browse or filter the blog

```
GET /posts?per_page=20&_fields=id,slug,date,title,excerpt,link
GET /posts?search=vegetarian&_fields=id,title,link
GET /posts?after=2026-01-01T00:00:00&_fields=id,date,title,link
GET /posts?categories=32&_fields=id,title,link
```

Categories are few — `listCategories` returns 4, of which `corporate-catering`
(id 32) carries all 28 posts. Filtering by category is therefore rarely useful
here; prefer `search`.

Retrieve one post in full with `getPost`:

```
GET /posts/10251?_embed
```

## Step 4 — always trim and embed

- Send `_fields` on every list call. Full objects embed rendered HTML and will
  exhaust your context on a 20-item page.
- Send `_embed` on single-item calls to inline author, featured media and terms
  in one round trip instead of N+1 follow-ups. Confirmed `_embedded` keys:
  `author`, `wp:featuredmedia`, `wp:term`.
- Strip or sanitize `content.rendered` HTML before reasoning over it.

## Step 5 — paginate correctly

`per_page` maxes at 100 (default 10). Read `X-WP-Total` and `X-WP-TotalPages`,
both exposed to browsers via CORS.

**Requesting a page past the last returns HTTP 400, not an empty array:**

```json
{"code":"rest_post_invalid_page_number",
 "message":"The page number requested is larger than the number of pages available.",
 "data":{"status":400}}
```

Loop by following the `Link` header `rel="next"` and stop when it is absent, or
stop when `page` reaches `X-WP-TotalPages`. Do not loop until you get an empty
response — that condition never arrives.

## Handling errors

Errors use the WordPress envelope, not RFC 9457 problem+json:

```json
{ "code": "rest_post_invalid_id", "message": "Invalid post ID.", "data": { "status": 404 } }
```

Branch on `code`, never on `message`.

| code | status | what to do |
|---|---|---|
| `rest_post_invalid_id` | 404 | Id is wrong or the item is not public. Re-resolve via search. |
| `rest_no_route` | 404 | Path or method not registered. Check `/wp-json/` index. |
| `rest_invalid_param` | 400 | Read `data.details.<param>.code` for the exact failure and fix the value. |
| `rest_post_invalid_page_number` | 400 | You paginated past the end. Stop. |
| `rest_forbidden` | 401 | Needs authentication. Stay on the public read surface. |
| `rest_forbidden_context` | 401 | You sent `context=edit`. Use the default `context=view`. |

## Constraints

- Read-only. Never attempt POST, PUT, PATCH or DELETE — writes would modify the
  company's live marketing site and will be rejected with 401 anyway.
- Never send `context=edit`.
- No rate-limit headers are published. Responses are cached for 600 seconds
  behind WP Engine and CloudFront; keep request volume modest.
- There is no idempotency mechanism, so do not rely on safe retries for anything
  other than GET.
- This API carries **no stability contract**. It is a CMS byproduct, not a product.
  Do not build a durable dependency on it.

## What this API cannot tell you

There is no public API for menus, pricing, availability, ordering, delivery
status or accounts. Those live in the customer application at
`https://app.lishfood.com` behind a login, and are not exposed. If asked to place
or price an order programmatically, say plainly that Lish publishes no such API
and point to https://www.lishfood.com/pages/get-started or catering@lishfood.com.
