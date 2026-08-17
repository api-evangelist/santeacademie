---
name: santeacademie-find-funded-dpc-training
description: >-
  Find the Santé Académie DPC training available to a specific French healthcare profession, and read what funds it
  and what the professional is paid for taking it. Use when a user names a profession (médecin, infirmier, pharmacien,
  préparateur, aide-soignant, cadre de santé) and asks what continuing-education courses exist, whether they are
  funded, or what they are worth.
api: Santé Académie Frontstage API
base_url: https://frontstage.santeacademie.com
authentication: none
operations:
  - api_jobs_get_collection
  - api_topics-searchfilters_get
  - api_topics-search_get_collection
  - api_topics_slug_get
generated: '2026-08-17'
method: generated
source: openapi/santeacademie-frontstage-openapi.json + live responses verified 2026-08-17
---

# Find funded DPC training for a profession

Every operation below is a real operationId from `openapi/santeacademie-frontstage-openapi.json`. They are the API
Platform auto-generated ids; `overlays/santeacademie-frontstage-overlay.yaml` maps each to a readable alias
(`listJobs`, `searchTopics`, `getTopic`, `getTopicSearchFilters`) if you are generating a client from the overlay.

No credential is required. Send `Accept: application/json` on every request — the Frontstage API also produces
`text/html` for the same URLs and a browser-shaped Accept header will get you a documentation page instead of data.

## Step 1 — resolve the profession code (`api_jobs_get_collection`)

```
GET https://frontstage.santeacademie.com/api/jobs
Accept: application/json
```

Returns the profession list. Each record carries `externalCode`, `appellation`, `jobParent` and `jobSpaces` (a per-space
appellation and description for `B2C` and `B2B`). Codes observed live: `MED` (Médecin), `INF` (Infirmier),
`PHA` (Pharmacien), `PPH` (Préparateur en pharmacie), `AIS` (Aide-soignant), `CADSANTE` (Cadre de santé).

Do not hard-code the codes. They are the join key for every filter in the rest of this skill, and this endpoint is the
only published source of truth for them.

Pick the space too: a self-employed practitioner is `B2C`, a salaried professional or a facility training manager is
`B2B`. The same profession is described differently in each.

## Step 2 — read the legal facet values (`api_topics-searchfilters_get`)

```
GET https://frontstage.santeacademie.com/api/topics-search/filters
Accept: application/json
```

Returns `jobs[]` (code + appellation) and `thematics[]` (id + name). The thematic **ids are integers with no lookup
endpoint of their own**, so this call is the only way to learn them. Read them here before filtering by theme.

## Step 3 — search topics (`api_topics-search_get_collection`)

```
GET https://frontstage.santeacademie.com/api/topics-search
      ?topicJobs.job.externalCode[]=MED
      &topicThematics.thematic.id[]=<id from step 2>
      &topicCourses.course.qualifying=true
      &order[rank]=asc
      &itemsPerPage=25
Accept: application/json
```

Filters, all optional and all real parameters in the spec:

| parameter | meaning |
|---|---|
| `externalCode` / `externalCode[]` | the topic's own business code |
| `topicJobs.job.externalCode` / `[]` | profession code from step 1 |
| `topicThematics.thematic.id` / `[]` | thematic id from step 2 |
| `topicCourses.course.virtualClassroom` | `true` for live virtual-classroom delivery |
| `topicCourses.course.qualifying` | `true` for qualifying programmes |
| `order[rank]`, `order[rankB2b]` | editorial ordering — use `rankB2b` for a B2B audience |
| `page`, `itemsPerPage`, `pagination` | paging |

Every filter exists in both a scalar and a repeatable `[]` form; use the `[]` form to OR several values of one filter.

Response envelope (not Hydra, despite the API Platform stack):

```json
{ "elements": [ ... ], "currentPage": 1, "lastPage": 12, "itemsPerPage": 25, "totalItems": 287 }
```

Page with `page`; read `lastPage` and `totalItems` from the envelope. Setting `pagination=false` returns the whole
collection — do not do this by default; the sibling resource collection has 608 items and this one is unbounded when
paging is disabled.

Each element is a topic with `slug`, `externalCode`, `metaTitle`, `metaDescription`, `rank`, `rankB2b`, and its
embedded `topicCourses[]` and `topicJobs[]`.

## Step 4 — read the funded offer (`api_topics_slug_get`)

```
GET https://frontstage.santeacademie.com/api/topics/{slug}
Accept: application/json
```

This is the payoff. Take the `slug` from step 3 and this returns the topic with everything embedded:

- `topicCourses[].course` — `name`, `shortName`, `description`, `duration`, `format`, `status`, `qualifying`,
  `virtualClassroom`, and **`refDpc`**, the ANDPC programme reference. Quote `refDpc` when a user needs to find the
  same programme on the agence's own site; enrollment for funded DPC is finalised on `agencedpc.fr`, not here.
- `topicCourses[].course.trainers[]` — the named experts, with `honorificTitle`, `institution` and `shortBiography`.
- `topicCourses[].course.fundings[]` — the schemes that cover this course, by `externalCode` and `name`
  (ANDPC, FIF-PL, FAF-PM, ANFH, OPCO).
- `topicCourses[].course.products[]` — the priced offer, one entry per `courseCode` × `jobCode` × `fundingCode`,
  carrying `price` (what the scheme pays) and `compensation` (the indemnity paid **to** the professional for the time
  spent), plus `status`.

`products[]` is why the public website can say training is "100% pris en charge" and show no price: the money moves
between Santé Académie and the funding scheme. When a user asks "what does it cost me", the honest answer read from
this API is `price` is settled by the scheme named in `fundingCode` and `compensation` is what they receive — filter
`products[]` to the `jobCode` you resolved in step 1 before quoting either number, because they differ by profession.

## Errors

- `404` `application/problem+json` — `{"type":"https://tools.ietf.org/html/rfc2616#section-10","title":"An error
  occurred","detail":"Not Found"}`. The slug did not resolve. Branch on the **status**, not on `type` or `title`: both
  are constant across every error this API returns. Never construct a slug — always take it from a search or sitemap
  response.
- No `429` is declared and no rate-limit header is returned, so there is no back-off signal. Be conservative: page
  rather than disabling pagination, and cache. `ETag` is returned on Frontstage collections, so revalidate with
  `If-None-Match` instead of re-fetching.

## Cautions

- All 24 operations across both Santé Académie APIs are `GET`. There is no write path, no enrollment call and no
  idempotency concern. Do not offer to enroll a user — you cannot, and enrollment happens on `agencedpc.fr`.
- Trainer records are named, photographed, identified healthcare experts returned without a credential. Quote a
  trainer's name and institution when it is relevant to a training choice; do not bulk-collect them.
- The API is unversioned (`info.version` is a static `1.0.0`, the only runtime marker is an informational
  `x-version-id` header) and there is no deprecation policy, so treat field names as unguaranteed and fail soft on a
  missing property.
