---
name: Read Nacuity Pharmaceuticals corporate and clinical-programme content
description: Enumerate Nacuity Pharmaceuticals' 29 published pages — About Us, Our Science, Pipeline, Development Programs, Investors & Collaborators, Medical Information, Publications, Expanded Access Policy, References, Contact — over the anonymous WordPress REST content API, and work around the fact that the API returns EMPTY page bodies.
api: openapi/nacuity-pharmaceuticals-content-openapi.yml
base_url: https://www.nacuity.com/wp-json
operations:
  - listTypes
  - listPages
  - getPage
generated: '2026-08-04'
method: generated
---

# Read Nacuity Pharmaceuticals corporate and clinical-programme content

Nacuity Pharmaceuticals is a clinical-stage biopharmaceutical company developing NPI-001
(N-acetylcysteine amide tablets) for retinitis pigmentosa and NPI-002 (an intravitreal implant) for
the delay of cataract progression. Its entire public content set lives in **29 WordPress pages**,
all listable with no credentials.

## Before you start — read this or you will waste requests

- **The API does not return page text.** `content.rendered` and `excerpt.rendered` are **empty
  strings on every one of the 29 pages**, verified on pages 23, 80 and 630 on 2026-08-04. The bodies
  are authored in Slider Revolution / page-builder post meta that WordPress never projects into
  REST. If you need the prose, you must fetch the page's HTML `link`. Do not retry, do not add
  `context=edit` (it will 401), and do not treat the empty string as a fetch error.
- **No authentication, and none obtainable.** Every operation here returns 200 with no header.
  Nacuity issues no credentials to third parties. The credentialed routes (`/wp/v2/users`,
  `/wp/v2/settings`, `wp-abilities/v1`, `wordfence/v1`) return `401`.
- **This is a CMS content API, not a product API.** Nacuity does not market it, version it, or
  document it. There is no SLA, no status page, no deprecation policy and no rate-limit header.
  Treat it as best-effort and keep volume low.
- **Read-only.** Every operation in the packaged surface is a GET.

## Steps

### 1. Sanity-check the surface

Call `listTypes` (`GET /wp/v2/types`) and confirm `page.rest_base` is still `pages`. The shape of
this API is governed by WordPress and plugin upgrades, not by any Nacuity policy, so verify rather
than assume — the theme-registered `portfolio` type and the Yoast and Slider Revolution namespaces
can appear or disappear without notice.

### 2. List the pages

Call `listPages` (`GET /wp/v2/pages`) with `per_page=100`. There were 29 published pages at harvest
time, so one request suffices — but read `X-WP-Total` and `X-WP-TotalPages` rather than assuming,
and follow `Link: rel="next"` if it appears.

Trim the payload, because the unfiltered object carries a large `yoast_head` markup string:

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent,menu_order,modified
```

The 13 top-level pages (`parent: 0`), by slug:

| slug | id | what it holds |
|---|---|---|
| `home` | 80 | Company positioning — "Leader in Innovative Treatments for Oxidative Stress" |
| `about` | 23 | Company background, executive leadership, scientific advisory board |
| `our-science` | 337 | The oxidative-stress thesis and mechanism |
| `pipeline` | 193 | Programme pipeline — retinitis pigmentosa, cataract, cystinosis |
| `development-programs` | 425 | Clinical development detail |
| `investors-collaborators` | 156 | Investors and collaborators |
| `news` | 99 | Press-release index — the parent of 14 child pages |
| `medical-information` | 560 | Medical information landing page — parent of 2 children |
| `references` | 478 | Scientific references |
| `contact` | 29 | Fort Worth TX and Carlton South VIC addresses |
| `privacy-policy` | 186 | Privacy Policy |
| `disclaimer` | 188 | Disclaimer |

Children: 14 press releases under `parent: 99`, and `publications` (562) plus
`expanded-access-policy` (564) under `parent: 560`.

### 3. Fetch a page record

Call `getPage` (`GET /wp/v2/pages/{id}`) using an id from step 2 — never a constructed one. Ids are
non-contiguous (they span 23 to 630).

What you get back that is actually useful:

- `title.rendered`, `link`, `slug`, `date`, `modified`, `parent`, `menu_order`, `class_list`
- `yoast_head_json` — canonical URL, robots directives, Open Graph and Twitter tags, and the
  schema.org `@graph` (`WebPage`, `BreadcrumbList`, `WebSite`, `Organization`)

What you do **not** get: `content.rendered` (empty), `excerpt.rendered` (empty), `featured_media`
(always 0 on the pages sampled), and a resolvable `author` (`/wp/v2/users` returns
`401 rest_user_cannot_view`).

### 4. Get the prose

Fetch the `link` URL over plain HTTP and parse the HTML. The page-builder markup is heavy — strip
Slider Revolution containers and inline builder CSS before extracting text. Cite the HTML page as
your source, not the API, and record that the text came from rendered HTML rather than from a field.

### 5. Use the JSON-LD for structured facts

Rather than scraping the company description, take it from `yoast_head_json.schema['@graph']`. The
`Organization` node gives the legal name (`Nacuity Pharmaceuticals, Inc.`), canonical URL, a
244x60 logo `ImageObject`, and `sameAs` links to X and LinkedIn. The same graph is on every page.
A verbatim copy is in `json-ld/nacuity-pharmaceuticals-organization.jsonld`.

## Errors you will actually see

| Status | code | Meaning | What to do |
|---|---|---|---|
| 400 | `rest_invalid_param` | A parameter failed validation, usually `per_page` outside 1-100 | Read `data.params`, fix, retry |
| 404 | `rest_post_invalid_id` | The id does not exist or is not published | Resolve ids from `listPages`, never construct them |
| 401 | `rest_user_cannot_view` | You tried `/wp/v2/users` | Stop — author records are not exposed here |
| 401 | `rest_forbidden` | The route needs a credential | Stop — no credential is obtainable |

Full catalog: `errors/nacuity-pharmaceuticals-problem-types.yml`. These are **not** RFC 9457
`application/problem+json`; the envelope is `{code, message, data:{status, params?}}`.

## Etiquette

No `Cache-Control`, `ETag` or `Last-Modified` is returned, so conditional GET is unavailable. Cache
locally and poll with `modified_after` instead of refetching whole collections. No rate-limit
headers are advertised; Wordfence is installed and may throttle silently.
