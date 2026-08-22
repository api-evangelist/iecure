---
name: iecure-harvest-media-library
description: >-
  Enumerate the iECURE media library — logo, pipeline artwork, leadership headshots and document
  attachments — with correct renditions, via the WordPress REST media collection.
api: iecure:iecure-media-api
operations: [listMedia, getMediaItem]
---

# Harvest the iECURE media library

226 attachments were published at harvest time. This is where the corporate logo, the pipeline
graphics and the leadership headshots live.

Base URL: `https://iecure.com/wp-json`. No credentials.

## 1. Page the collection

    GET /wp/v2/media?per_page=100&orderby=date&order=desc&_fields=id,date,slug,link,source_url,mime_type,media_type,alt_text,post,media_details

Follow the `Link` header's `rel="next"`; `X-WP-Total` tells you the size up front (226 on
2026-08-22). Three pages at `per_page=100`.

## 2. Filter by kind

    GET /wp/v2/media?media_type=image&per_page=100
    GET /wp/v2/media?mime_type=application/pdf&per_page=100

`media_type` takes the coarse bucket (`image`, `file`); `mime_type` takes the exact type.

## 3. Pick the right rendition

`media_details.sizes` carries every generated size with its own `source_url`, `width` and `height`.
Use it — do not scale the full-size original client-side, and do not assume a size exists. The
corporate logo, for example, is published at
`https://iecure.com/wp-content/uploads/iECURE_White_Logo.png` with `-300x118` and `-705x277`
renditions alongside it. Note the name: the primary logo asset is the **white** variant, intended for
the site's dark header, so it will be invisible on a white background. Check `alt_text` and the slug
before assuming a rendition is the neutral mark.

## 4. Trace an attachment back to its post

    GET /wp/v2/media/{id}?_fields=id,post,source_url,alt_text
    GET /wp/v2/posts/{post}?_fields=id,link,title

The `post` field is the parent object the file was uploaded to — usually the press release that
carries it. It is `0` for library uploads with no parent.

## Rules

- **Read only.** Upload requires an authenticated WordPress user.
- **`per_page` max 100**, else `400 rest_invalid_param` / `rest_out_of_bounds`.
- **`source_url` is a direct CDN-fronted asset URL** on `iecure.com/wp-content/uploads/`; it is not
  behind the REST API and is not rate-limited separately (nothing here is — see
  `rate-limits/iecure-rate-limits.yml`).
- **Respect the copyright.** These are iECURE's corporate assets, published for use in coverage of
  the company. Enumerating them is not a licence to republish them.
- **`alt_text` is frequently empty** on this deployment. Do not rely on it for image classification.
