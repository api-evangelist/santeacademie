---
name: santeacademie-enumerate-catalog
description: >-
  Enumerate the whole Santé Académie catalog — every topic and article slug — and then resolve each one, using the
  Connector API's sitemap feed as the index. Use when a task needs coverage rather than a single answer: building a
  local mirror, counting what exists per profession, or checking whether a specific course is still published.
api: Santé Académie Connector API
base_url: https://frontstage.santeacademie.com
authentication: none
operations:
  - getSitemap
  - findTopic
  - findArticle
  - findResource
  - findCustomCatalog
  - listJob
  - listJobSpaces
  - listMediaCategory
  - api_topicsjobscounts-by-space_get
generated: '2026-08-17'
method: generated
source: >-
  openapi/santeacademie-connector-openapi.json (real operationIds) +
  openapi/santeacademie-frontstage-openapi.json + live responses verified 2026-08-17
---

# Enumerate the Santé Académie catalog

`getSitemap`, `findTopic`, `findArticle`, `findResource`, `findCustomCatalog`, `listJob`, `listJobSpaces` and
`listMediaCategory` are real operationIds in `openapi/santeacademie-connector-openapi.json`.
`api_topicsjobscounts-by-space_get` is a real operationId in `openapi/santeacademie-frontstage-openapi.json`.

Both APIs are served from the same host and require no credential. They are two separate routers —
`/api/*` (Frontstage) and `/connector/api/*` (Connector) — with **different error contracts**. Read the section at the
bottom before writing retry logic.

## Step 1 — get the index (`getSitemap`)

```
GET https://frontstage.santeacademie.com/connector/api/sitemap
Accept: application/json
```

This is the cheapest enumeration path Santé Académie publishes. It returns sitemap source data — the slugs the public
site links — in one call, with no paging. Everything downstream is a per-slug resolve against that list.

The alternative is walking `GET /api/topics-search` and `GET /api/resources-search` page by page, which works but costs
one request per page and gives you the search projection of each entity rather than the full one.

## Step 2 — get the fixed reference sets

Three small, stable lists worth fetching once and caching, because every other payload references them by code:

```
GET /connector/api/job              # listJob            — professions (externalCode, appellation)
GET /connector/api/job/space        # listJobSpaces      — B2C / B2B
GET /connector/api/media/category   # listMediaCategory  — article/media categories
```

None of them takes a parameter and none of them pages — each returns the whole set as a bare array.

For a coverage summary without resolving anything, the Frontstage API has a purpose-built counter:

```
GET /api/topics/jobs/counts-by-space   # api_topicsjobscounts-by-space_get
```

which returns `{ jobCode, space, countTopics }` rows — how many topics exist per profession per space. Use this to
sanity-check an enumeration before and after.

## Step 3 — resolve each slug

| operationId | request |
|---|---|
| `findTopic` | `GET /connector/api/topic/{slug}` |
| `findArticle` | `GET /connector/api/article/{slug}` |
| `findResource` | `GET /connector/api/resource/{slug}` |
| `findCustomCatalog` | `GET /connector/api/custom-catalog/{slug}` |

For topics and resources you have a choice of API. Prefer the **Frontstage** equivalents —
`api_topics_slug_get` (`GET /api/topics/{slug}`) and `api_resources_slug_get` (`GET /api/resources/{slug}`) — when you
need depth: those return the embedded courses, trainers, funding schemes and priced products. The Connector versions are
the ones the marketing site uses and are shaped for page rendering.

## Step 4 — pace yourself

There is no published rate limit and no rate-limit header on any response, so there is no signal telling you when to
slow down. That is a reason for more care, not less:

- Serialise the per-slug resolves; do not fan out concurrently.
- Frontstage responses carry an `ETag`. Store it and revalidate with `If-None-Match` on later passes — a re-enumeration
  should be mostly `304`s.
- Connector responses carry no `ETag` and `cache-control: max-age=0, must-revalidate, private`, so cache them on your
  own key and re-check on a schedule you choose.
- Both APIs return `x-version-id` (observed `v134`). It is an internal deploy id, not an API version, but a change in it
  between passes is the only hint you will get that the service was redeployed.

## Errors — the two routers disagree

This is the single most important thing to get right when walking both APIs:

- **Frontstage** (`/api/*`) returns `application/problem+json`:
  `{"type":"https://tools.ietf.org/html/rfc2616#section-10","title":"An error occurred","detail":"Not Found"}`.
  `type` and `title` are constant on every error, so they carry no information — branch on the HTTP status.
- **Connector** (`/connector/api/*`) returns a **bare JSON string**: `"Topic not found"`, with
  `content-type: application/json`. Parsing it as an object will throw. There is no error code to read.
- The Connector specification declares only `200` responses for all fourteen operations, so a generated client will
  treat every `404` as an unexpected response. Handle it explicitly.

A `404` during enumeration usually means the slug was unpublished between the sitemap fetch and the resolve. Drop it and
carry on; do not retry.

## Cautions

- **Do not enumerate custom-catalog slugs.** `findCustomCatalog` resolves B2B customer-specific catalog pages, and the
  payload carries `externalOrganizationId`, the customer's logo and its theme. Any slug resolves anonymously. Resolve a
  catalog only when a user supplies its exact slug; never guess, crawl or list them.
- Testimonial and trainer payloads contain named, photographed healthcare professionals. If you are mirroring the
  catalog, decide deliberately whether to retain that personal data.
- `www.santeacademie.com/robots.txt` disallows `/register/` and `/c-job/`, and
  `simulateur.santeacademie.com/robots.txt` disallows `/financements/`, `/idel/`, `/medecin-generaliste/`,
  `/pharmacien/` and `/preparateur-en-pharmacie/`. Those are HTML paths, not API paths, but they signal what the company
  does not want harvested — respect the equivalent intent.
- Nothing in these APIs is writable, so an enumeration cannot change state. It can, however, be indistinguishable from
  abuse on an unauthenticated, unrated endpoint that also serves the company's own production websites. Keep the request
  rate low.
