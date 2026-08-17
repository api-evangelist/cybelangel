---
name: Poll the CybelAngel alert feed into a SIEM
description: Continuously ingest new CybelAngel alerts with cursor pagination, overlapping date windows, and the 1,000-record response cap handled correctly.
api: openapi/cybelangel-alerts-openapi.yml
operations:
  - alerts_search_alerts_alerts_get
  - alerts_search_an_alert_alerts__alert_id__get
  - stix_search_stix_alerts_stix_alerts_get
generated: '2026-08-17'
method: generated
source: >-
  openapi/cybelangel-alerts-openapi.yml,
  https://developers.cybelangel.com/docs/alerts-api/c53add3c8b16f-getting-started,
  https://developers.cybelangel.com/docs/alerts-api/304dab003eb13-limitations,
  https://developers.cybelangel.com/docs/alerts-api/3d22245755b86-alerts-in-stix-format
---

# Poll the CybelAngel alert feed into a SIEM

The canonical CybelAngel integration: keep a downstream system (Splunk, Sentinel, a data
warehouse) current with the alerts CybelAngel's detection pipeline produces.

Base URL `https://api.cybelangel.com`. Authenticate first — see
`skills/cybelangel-authenticate.md`.

## Steps

1. **Search a date window.** `GET /v1/alerts`
   (`alerts_search_alerts_alerts_get`) with your `stream_id` and a start/end date window.
   Useful filters on this operation: `keyword`, `ip`, `hostname`, `categories`, `status`,
   `customer_assessment`, `min_ml_score`, `limit`, `cursor`.

2. **Page with the cursor, not with offsets.** The response is an `AlertList` carrying
   `total`, `alerts[]`, `more` and `cursor`. While `more` is true, re-issue the same query with
   the returned `cursor`. **A single response never contains more than 1,000 alerts** — this is
   documented, and it is why a naive "read the first page and move on" integration silently
   loses data.

3. **Read the polymorphic payload by `category`.** Each `PublicAlert` carries `id`, `stream_id`,
   `status`, `category`, `ingestion_date`, `detection_date`, `ml.score`, `matches[]`, an optional
   `report.id`, and then exactly one category-shaped sibling object: `adm`, `board`,
   `clouddrive`, `codeshare`, `database`, `dns`, `docshare`, `fileserver`, `leak`, `paste`,
   `rss` or `social`. Switch on `category` and read the matching key. There is no `oneOf`
   discriminator to lean on.

4. **Overlap your windows.** From the provider's own limitations page: "Due to the asynchronous
   nature of our detection and ingestion process, it is possible that the alerts from the last
   minutes are not made available in the correct sequence." Do **not** advance a strict
   high-water mark on `ingestion_date`. Re-query the last few minutes on every pass and
   de-duplicate on alert `id`.

5. **Respect the retention boundary.** Alerts live for a rolling 12 months and **nothing exists
   before 2026-01-01**. A query outside the window returns an empty result set, not an error —
   so an empty page is not proof that a backfill succeeded.

6. **Fetch detail only when you need it.** `GET /v1/alerts/{alert_id}`
   (`alerts_search_an_alert_alerts__alert_id__get`) for one alert.

7. **Prefer STIX when the destination speaks it.** `GET /v1/stix/alerts`
   (`stix_search_stix_alerts_stix_alerts_get`) returns the same alerts as OASIS STIX bundles
   (`StixAlertsResponse` with `stix_bundle`, `more`, `cursor`). Note the page-size parameter on
   this operation is named **`alerts_limit`**, not `limit`. Every alert category except RSS has
   a published STIX mapping; RSS is explicitly "Not published in STIX".

## Error handling

- `400` → `APIErrorResponse[ValidationError]`: `{"error":{"message":…,"fields":[{"name","msg","input","valid_input"}]}}`.
  Iterate `fields[]` and fix each named parameter — almost always a malformed or out-of-window date.
- `403` → the stream is not provisioned for this client, or the plan lacks the Alerts API. Do not retry.
- `410` → the resource is permanently gone. Stop retrying and migrate; see the deprecation note below.
- `500` → retry with backoff. No request-id header is returned, so record time + URL + payload
  yourself if you need to escalate to support@cybelangel.com.

## Deprecation to avoid

`GET /v1/alerts/{alert_id}/credentials`
(`alerts_get_alert_credentials_deprecated_alerts__alert_id__credentials_get`) is marked
`deprecated: true` in the specification and can return `410`. Use
`GET /v1/alerts/{alert_id}/leak-credentials` instead — see
`skills/cybelangel-triage-leaked-credentials.md`.

## Throughput

15 requests/second, 20 concurrent, and one token per hour reused across the whole run.
