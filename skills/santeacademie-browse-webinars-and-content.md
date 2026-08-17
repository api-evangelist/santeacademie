---
name: santeacademie-browse-webinars-and-content
description: >-
  Find Santé Académie webinars, articles and other content resources for a healthcare profession — including which are
  most watched and which are scheduled — and read the FAQ. Use when a user wants free or upcoming content rather than a
  funded DPC course, or asks "what's popular", "what's coming up", or a question the published FAQ answers.
api: Santé Académie Frontstage API
base_url: https://frontstage.santeacademie.com
authentication: none
operations:
  - api_resources-searchfilters_get
  - api_resources-search_get_collection
  - api_resources_slug_get
  - api_resources_slugtopics_get
  - searchArticle
  - findArticle
  - listFaq
  - listMediaCategory
generated: '2026-08-17'
method: generated
source: >-
  openapi/santeacademie-frontstage-openapi.json + openapi/santeacademie-connector-openapi.json (real operationIds) +
  live responses verified 2026-08-17
---

# Browse webinars, articles and content resources

`api_resources-searchfilters_get`, `api_resources-search_get_collection`, `api_resources_slug_get` and
`api_resources_slugtopics_get` are real operationIds in `openapi/santeacademie-frontstage-openapi.json`.
`searchArticle`, `findArticle`, `listFaq` and `listMediaCategory` are real operationIds in
`openapi/santeacademie-connector-openapi.json`. No credential is required. Send `Accept: application/json`.

Resources are the free/editorial layer of the catalog — webinars, articles and course assets — as distinct from the
funded DPC topics covered by `santeacademie-find-funded-dpc-training`.

## Step 1 — read the facets (`api_resources-searchfilters_get`)

```
GET https://frontstage.santeacademie.com/api/resources-search/filters
Accept: application/json
```

Returns `categories[]` (`code`, `name`), `jobs[]` (`externalCode`, `appellation`) and `thematics[]` (`id`, `name`).
`WEBINAR` is a category code observed live. The thematic integer ids have no lookup endpoint of their own, so this call is
the only published source for them.

## Step 2 — search resources (`api_resources-search_get_collection`)

```
GET https://frontstage.santeacademie.com/api/resources-search
      ?resourceCategory.code[]=WEBINAR
      &resourceJobs.job.externalCode[]=INF
      &resourceJobs.space=B2C
      &order[viewCounter]=desc
      &itemsPerPage=20
Accept: application/json
```

Real parameters:

| parameter | meaning |
|---|---|
| `resourceCategory.code` / `[]` | category code from step 1 (e.g. `WEBINAR`) |
| `resourceJobs.job.externalCode` / `[]` | profession code |
| `resourceJobs.space` / `[]` | `B2C` or `B2B` |
| `resourceThematics.thematic.id` / `[]` | thematic id from step 1 |
| `status` / `[]` | publication status |
| `id` / `[]` | numeric resource id |
| `private` | boolean |
| `order[viewCounter]` | sort by real view count — this is the "what's popular" sort |
| `order[rank]` | editorial ordering |
| `page`, `itemsPerPage`, `pagination` | paging |

608 resources were live at the time of writing (`totalItems`, verified on
`/api/resources-search?itemsPerPage=1`). Always page; do not send `pagination=false`.

Envelope: `{ "elements": [...], "currentPage", "lastPage", "itemsPerPage", "totalItems" }`.

Each element carries `slug`, `title`, `caption`, `description`, `displayUrl`, `actionValue`, `rank`,
`viewCounter`, `scheduledAt` and its `resourceCategory` and `resourceJobs[]`.

- `viewCounter` is a real engagement count returned publicly — sort on it for popularity, and pair it with
  `resourceCategory.viewCounterLabel` (e.g. "participant") when phrasing a count to a user.
- `scheduledAt` is what makes this an events feed: filter to future values plus `resourceCategory.code=WEBINAR` to answer
  "what webinars are coming up". Note there is no `scheduledAt` filter or sort parameter, so fetch by category and filter
  client-side.
- `resourceCategory.ctaLabel` (e.g. "S'inscrire") is UI copy the API returns. Use it if you are rendering a link; do not
  present it as data.

## Step 3 — resolve one resource (`api_resources_slug_get`)

```
GET https://frontstage.santeacademie.com/api/resources/{slug}
Accept: application/json
```

Adds `contentUrl`, `extract`, `duration`, `keyIndicators`, `metaTitle`, `metaDescription`, `ogImage` and
`resourceTrainers[]` — each with `trainer` and a boolean `trainerHost` marking who hosts rather than teaches.

## Step 4 — link a resource back to funded training (`api_resources_slugtopics_get`)

```
GET https://frontstage.santeacademie.com/api/resources/{slug}/topics
Accept: application/json
```

The only relationship either API exposes as its own sub-resource endpoint. Use it to move a user from a free webinar to
the funded DPC topics it belongs to, then hand off to `santeacademie-find-funded-dpc-training` at its step 4.

## Articles (`searchArticle`, `findArticle`)

The editorial hub behind `santeacademie.com/media` lives on the Connector API:

```
GET /connector/api/search/article    # searchArticle
GET /connector/api/article/{slug}    # findArticle
GET /connector/api/media/category    # listMediaCategory
```

`searchArticle` has a real limitation you must plan around: its only parameter is a **required** query parameter named
`ArticleSearchQuery` — a PHP DTO class name that swagger-php serialised without expanding its fields. The individual
query fields are not described in the specification or anywhere else public. So `searchArticle` cannot be called
correctly from the published contract. Reach articles by slug via `findArticle`, taking slugs from
`GET /connector/api/sitemap` (see `santeacademie-enumerate-catalog`), until the parameter is documented.

## FAQ (`listFaq`)

```
GET https://frontstage.santeacademie.com/connector/api/faq
Accept: application/json
```

Returns published FAQ entries as `question`, `answer`, plus placement booleans `homePage`, `displayCpts` and
`displayDrh` (CPTS and DRH audience pages). The answers are HTML. This is the authoritative published answer set for
funding, enrollment and platform questions — prefer quoting it over paraphrasing, and note the entries are French.

## Errors

- Frontstage: `404` `application/problem+json`, with a constant `type` and `title` — branch on status only.
- Connector: `404` with a **bare JSON string** body (`"Topic not found"`) and no declared error response in the spec.
  Do not parse it as an object.
- No `429` declared, no rate-limit header returned. Page, cache, and use `If-None-Match` against the Frontstage `ETag`.

## Cautions

- `resourceTrainers[]` and the testimonial list contain named, photographed, identified healthcare professionals served
  without a credential. Surface an individual when it is relevant to a content recommendation; do not bulk-collect.
- The `private` filter exists on `/api/resources-search`. Do not set it looking for hidden content — if unlisted
  resources are reachable that way it is a provider misconfiguration, not a feature to exploit.
- Both APIs are unversioned with no deprecation policy. Fail soft on missing fields.
