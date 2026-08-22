---
name: iecure-track-news-archive
description: >-
  Monitor iECURE's news archive — press releases, media coverage, awards and scientific
  publications — through the company's own WordPress REST content API, including filtering by
  category and by publication window.
api: iecure:iecure-posts-api
operations: [listPosts, getPost, listCategories]
---

# Track the iECURE news archive

iECURE is a clinical-stage genetic medicines company. It runs no developer program, but the
WordPress REST API behind `iecure.com` is open to anonymous callers and is the fastest machine-readable
way to follow the company's ECUR-506 program announcements. 60 posts were published at harvest time.

Base URL: `https://iecure.com/wp-json`. No credentials.

## 1. Learn the categories first

    GET /wp/v2/categories?per_page=100

Six terms are registered. As of 2026-08-22: `news` (51), `press release` (31), `in the news` (12),
`awards` (9), `pubs & pres` (8), `uncategorized` (0). A post commonly carries more than one, which is
why the counts exceed the 60 published posts. Keep the numeric `id` — the posts collection filters on
IDs, not slugs.

## 2. List posts, newest first

    GET /wp/v2/posts?per_page=20&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,categories

Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to size the crawl, and follow the
RFC 8288 `Link` header's `rel="next"` rather than incrementing `page` blindly.

`_fields` matters here: the full post payload inlines rendered HTML and All in One SEO metadata, so
an unfiltered list request is large. Ask for what you need.

## 3. Filter to the announcements that matter

Press releases only:

    GET /wp/v2/posts?categories=7&per_page=20

Anything published or edited since your last sweep — this is the correct incremental pattern, and
`modified_after` is what catches a corrected release you have already seen:

    GET /wp/v2/posts?after=2026-08-01T00:00:00&orderby=date&order=asc
    GET /wp/v2/posts?modified_after=2026-08-01T00:00:00&orderby=modified&order=asc

Program-specific search across title, content and excerpt:

    GET /wp/v2/posts?search=ECUR-506&search_columns=post_title,post_content

## 4. Fetch one item with its media and author inlined

    GET /wp/v2/posts/{id}?_embed

`_embed` returns the featured image and author under `_embedded` in a single request instead of
three. `content.rendered` is HTML, not text — strip it before feeding it to a model.

## Rules

- **Read only.** Every write route requires an authenticated WordPress user. Do not attempt one.
- **`per_page` is capped at 100.** Exceeding it returns `400 rest_invalid_param` with
  `data.details.per_page.code = rest_out_of_bounds`. It does not clamp. See
  `errors/iecure-problem-types.yml`.
- **A missing ID returns `404 rest_post_invalid_id`,** not an empty body. Resolve IDs through the
  collection or `/wp/v2/search` before fetching directly.
- **There is no rate-limit signal.** No `RateLimit-*`, no `X-RateLimit-*`, no `Retry-After`
  (`rate-limits/iecure-rate-limits.yml`). You get no backpressure warning, so pace yourself: this is
  a small company's marketing site, not a metered API.
- **Errors are not RFC 9457.** The envelope is `{code, message, data:{status}}` as
  `application/json`. Branch on `code`.
- **This is not iECURE's product.** It is the CMS behind their website. Nothing here is clinical or
  regulatory data; for that, use ClinicalTrials.gov.
