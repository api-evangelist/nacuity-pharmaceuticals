---
name: Harvest the Nacuity Pharmaceuticals media library
description: Page through the 140-item WordPress media library at www.nacuity.com to retrieve the company logo, leadership headshots, scientific figures and page imagery with source URLs, MIME types, alt text and pre-rendered size variants, including how to run an incremental harvest.
api: openapi/nacuity-pharmaceuticals-content-openapi.yml
base_url: https://www.nacuity.com/wp-json
operations:
  - listMedia
  - getMediaItem
  - listPages
generated: '2026-08-04'
method: generated
---

# Harvest the Nacuity Pharmaceuticals media library

The media library is **the only collection on this deployment that returns substantive content over
the API**. Pages come back with empty bodies; media items come back complete. If you want machine-
readable assets from Nacuity Pharmaceuticals, this is the surface.

140 attachments were registered at harvest time (`X-WP-Total: 140` on 2026-08-04).

## Before you start

- No credentials. `GET /wp/v2/media` returns 200 anonymously.
- `author` on each item does **not** resolve — `/wp/v2/users` returns `401 rest_user_cannot_view`.
  Treat it as an opaque integer.
- `post` (the object the attachment is attached to) may be null for unattached uploads.
- Assets are served from `https://www.nacuity.com/wp-content/uploads/YYYY/MM/...`. Respect the
  company's copyright; this skill retrieves metadata and URLs, it does not license the imagery.

## Steps

### 1. Page the collection

Call `listMedia` (`GET /wp/v2/media`) at the maximum page size and follow the pagination headers:

```
GET /wp/v2/media?per_page=100&page=1&_fields=id,slug,title,alt_text,caption,media_type,mime_type,source_url,post,date,modified
```

Read `X-WP-Total` (140) and `X-WP-TotalPages` (2 at `per_page=100`) rather than assuming, and follow
`Link: rel="next"`. `per_page` outside 1-100 returns `400 rest_invalid_param`.

### 2. Filter to what you need

Useful parameters on this collection:

- `media_type=image` — restrict to images.
- `mime_type=image/png` — restrict to a MIME type.
- `search=` — matches title, slug and alt text.
- `parent=` — attachments belonging to a specific page.
- `after` / `before` — filter by upload date.

### 3. Fetch full details for an item

Call `getMediaItem` (`GET /wp/v2/media/{id}`). The full record adds `media_details`, a map of the
size variants WordPress pre-rendered (`thumbnail`, `medium`, `large`, `full`, plus theme-registered
sizes), each with its own width, height, file name and `source_url`. Pick the variant that fits your
use rather than downloading `full` and resizing.

### 4. Take the company logo from JSON-LD, not from a guess

The canonical logo is declared in the schema.org `Organization` node on every page:

```
https://www.nacuity.com/wp-content/uploads/2017/03/nacuity-pharmaceuticals-inc_web.png  (244x60)
```

Read it from `yoast_head_json.schema['@graph']` on any page, or from the verbatim copy in
`json-ld/nacuity-pharmaceuticals-organization.jsonld`. That is the asset the company itself
designates as its logo — do not infer one from the media library by filename.

### 5. Run the harvest incrementally

Store the highest `modified` timestamp you have seen, then poll:

```
GET /wp/v2/media?per_page=100&modified_after=2026-08-04T00:00:00&_fields=id,modified,source_url
```

New and edited attachments come back; everything else is skipped. There is no `ETag` or
`Last-Modified` header on this API, so `modified_after` is the only efficient re-sync mechanism.

### 6. Cross-reference to pages

If you need to know where an asset is used, call `listPages` and match on the attachment's `post`
field. Note that `featured_media` is `0` on every page sampled, so the featured-image relation is
unpopulated on this site — `post` is the only usable link.

## Errors you will actually see

| Status | code | What to do |
|---|---|---|
| 400 | `rest_invalid_param` | `per_page` must be 1-100 |
| 404 | `rest_post_invalid_id` | The attachment id does not exist — resolve from `listMedia` |
| 401 | `rest_user_cannot_view` | You tried to resolve `author` — stop, that route is closed |

Full catalog: `errors/nacuity-pharmaceuticals-problem-types.yml`.

## Etiquette

No rate-limit headers are published and Wordfence is installed on the origin. Two requests get you
the whole index; keep the harvest to that and space out any bulk asset downloads.
