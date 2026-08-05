---
name: Track Nacuity Pharmaceuticals press releases and regulatory milestones
description: Enumerate Nacuity Pharmaceuticals' press archive over the anonymous WordPress REST content API by walking the child pages of the News page, run an incremental poll for new releases, and handle the trap that the posts collection and the RSS feed are both empty.
api: openapi/nacuity-pharmaceuticals-content-openapi.yml
base_url: https://www.nacuity.com/wp-json
operations:
  - listPages
  - getPage
  - listPosts
  - searchContent
generated: '2026-08-04'
method: generated
---

# Track Nacuity Pharmaceuticals press releases and regulatory milestones

Nacuity Pharmaceuticals announces clinical and regulatory milestones — FDA Breakthrough Therapy and
Fast Track designations for NPI-001, Phase 1/2 trial progress for NPI-001 and NPI-002, financings,
licensing agreements and board appointments — as press releases on www.nacuity.com. This skill
retrieves them.

## The trap, stated first

**`listPosts` returns an empty array.** `GET /wp/v2/posts` answers `[]` with `X-WP-Total: 0`, and
the RSS feed at `/feed/` carries no items. That is not a failure and not a permissions problem —
Nacuity authors every press release as a **page**, parented to the News page (id `99`). If you
consume this API the way you would consume a normal WordPress site, you will conclude the company
has published nothing since 2014.

Second trap: **the release text is not in the API.** `content.rendered` is an empty string on every
page. The API gives you the release *title*, *URL*, *published date* and *modified date* — which is
enough to build a reliable index and detect new items — but the body must come from the HTML page.

## Steps

### 1. Enumerate the archive

Call `listPages` (`GET /wp/v2/pages`) and filter on the News parent:

```
GET /wp/v2/pages?per_page=100&parent=99&_fields=id,slug,title,link,date,modified&orderby=date&order=desc
```

At harvest time this returned 14 child pages. Three further releases sit at top level
(`nacuity-and-johns-hopkins-sign-exclusive-license-agreement`, id 223;
`nacuity-pharmaceuticals-announces-initiation-of-slo-rp-phase-1-2-clinical-trial`, id 361; plus the
Arctic Therapeutics joint announcement), so for a complete archive list all 29 pages and select
those whose `link` contains `/news/`.

Each row gives you a stable `id`, a descriptive `slug`, the full headline in `title.rendered`, the
canonical `link`, and an ISO `date`. That is a usable press index.

The most recent entries observed on 2026-08-04, newest first:

| id | headline |
|---|---|
| 630 | Granted U.S. FDA **Breakthrough Therapy** Designation for NPI-001 for Retinitis Pigmentosa |
| 624 | Positive Data from Clinical Trial Evaluating NPI-001 in RP Associated with Usher Syndrome |
| 570 | Granted U.S. FDA **Fast Track** Designation for NPI-001 for Retinitis Pigmentosa |
| 542 | First Patients Implanted in Final Cohort of Phase 1/2 Trial of NPI-002 Intravitreal Implant |
| 538 | Expands Board of Directors with Appointment of Dr. Emmett Cunningham Jr. |
| 534 | Arctic Therapeutics and Nacuity Announce EMA Approval of First Clinical Trial of AT-001 for HCCAA |

### 2. Poll incrementally

Re-run step 1 with `modified_after` set to your last successful poll:

```
GET /wp/v2/pages?per_page=100&parent=99&modified_after=2026-08-04T00:00:00&_fields=id,slug,title,link,date,modified
```

Compare returned ids against your stored set. A new id is a new release; a changed `modified` on a
known id is an edit. There is no `ETag` or `Last-Modified` header, so this field-level comparison is
the only change-detection mechanism available.

### 3. Search when you want a topic, not the archive

Call `searchContent` (`GET /wp/v2/search?search=...`). Search *does* reach into the page bodies even
though the REST content field is empty, so it is the one way to find a release by its subject
matter.

```
GET /wp/v2/search?search=retinitis&per_page=20
```

returned `X-WP-Total: 4` on 2026-08-04, top hit id 630. Every result has `type: post` and
`subtype: page`; resolve the hit through `_links.self.href`, which points at `/wp/v2/pages/{id}`.
Useful terms: `NPI-001`, `NPI-002`, `retinitis`, `Usher`, `cataract`, `cystinosis`,
`Foundation Fighting Blindness`, `Arctic Therapeutics`.

### 4. Retrieve the release text

Fetch the `link` URL over HTTP and parse the HTML. Attribute the text to the HTML page, and record
the date from the API field (`date`) rather than from prose — the API date is the reliable one.

### 5. Do not expect a changelog or a feed

There is no API changelog, no webhook, no AsyncAPI, no email-subscription endpoint and no populated
RSS. Polling step 2 is the mechanism.

## Errors you will actually see

| Status | code | What to do |
|---|---|---|
| 400 | `rest_invalid_param` | `per_page` must be 1-100; `modified_after` must be ISO 8601 |
| 404 | `rest_post_invalid_id` | Resolve ids from `listPages` or `searchContent` |
| 401 | `rest_forbidden` | You strayed onto a gated route — stop |

Full catalog: `errors/nacuity-pharmaceuticals-problem-types.yml`.
