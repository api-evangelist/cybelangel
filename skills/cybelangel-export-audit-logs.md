---
name: Export CybelAngel audit logs and threat intelligence
description: Pull the platform audit trail for a compliance evidence store, and pull claimed-attack threat intelligence for a SIEM or TIP.
api: openapi/cybelangel-audit-logs-openapi.yml
operations:
  - audit_logs_search_audit_logs__organization_id__audit_logs_get
  - get-threat-intelligence-claimed-attacks
generated: '2026-08-17'
method: generated
source: >-
  openapi/cybelangel-audit-logs-openapi.yml, openapi/cybelangel-threat-intelligence-openapi.yml,
  https://developers.cybelangel.com/docs/audit-logs-api/f9723159f94ff-getting-started-guide-audit-logs-api,
  https://developers.cybelangel.com/docs/threat-intelligence-api/38e63ab3d5c42-fetch-threat-intelligence-claimed-attacks
---

# Export CybelAngel audit logs and threat intelligence

Two single-operation APIs that serve two different compliance and SOC needs. Both on
`https://api.cybelangel.com`. Authenticate first (`skills/cybelangel-authenticate.md`).

## A. Audit logs — who did what in the platform

`GET /v1/{organization_id}/audit-logs`
(`audit_logs_search_audit_logs__organization_id__audit_logs_get`)

1. **You need an `organization_id`.** It is a UUID and it is not self-serve — the provider's own
   guide says "To retrieve your organization ID, please contact your Customer Success Manager."
   Store it in configuration; it does not appear in any other API response.
2. Cursor-paginate with `cursor` + `limit`. The response is `AuditLogsList`
   (`total`, `events[]`, `more`, `cursor`).
3. Each `PublicAuditLog` carries `event_id`, `organization_id`, `action` (an `Action` enum),
   `object_type` (an `ObjectType` enum), `object_id`, `author_id`, `author_email`, `event_date`,
   `updated_fields`, and an **embedded snapshot** of the affected entity — `report`
   (`ReportMetadata`: `incident_id`, `incident_type`, `severity`, `status`), `keyword`
   (`KeywordMetadata`: `name`, `status`, `type`) or `user` (`UserMetadata`: `email`).
4. Because the metadata is embedded rather than referenced, an audit record stays readable after
   the underlying report or keyword changes — treat it as the point-in-time evidence it is, and
   do not re-resolve `object_id` when writing to your evidence store.
5. Filter by `author_email`, `object_type`, and a date window.
6. De-duplicate on `event_id` for idempotent ingestion.

**Error handling:** `400` returns `APIErrorResponse[ValidationError]` with a `fields[]` array
(`name`, `msg`, `input`) — iterate it. `403` means, in the provider's own words, "No rights to
access audit logs on the organization" — check that the API plan includes the scope; do not
re-mint the token.

## B. Threat intelligence — claimed attacks

`GET /v1/threat-intelligence/claimed-attacks`
(`get-threat-intelligence-claimed-attacks`)

1. **This feed is not customer-scoped.** Unlike every other CybelAngel API, it is a world-view of
   ransomware and extortion claims, not findings about your organization. Do not route it into an
   incident queue as if each row were your exposure.
2. Paginate with **`skip` + `limit`** (offset), not `cursor` — this operation follows the older
   convention.
3. Each `ClaimedAttackDTO`: `reference_id`, `claimed_at`, `category`
   (`ClaimedAttackCategory`), `title`, `summary`, `source` (`ClaimedAttackSourceDTO` with
   `network` (`ClaimedAttackNetwork`) and `url`), `threat_actors[]`, `countries[]`,
   `industries[]`, `victims[]`, `domains[]`.
4. Filter with `threat_actors`, `countries`, `industries`, `victims`, `domains`, `category` and a
   date window to narrow to your sector or supply chain before ingesting.
5. De-duplicate on `reference_id`. Join to your own vendor list on `victims[]` / `domains[]` — this
   is the practical way to use it as third-party breach monitoring today, ahead of the
   Vendor Exposure Monitoring feature on the Q3 2026 roadmap.

## Shared rules

- One cached token for both, 15 req/s, 20 concurrent, and no rate-limit response headers.
- No webhooks and no AsyncAPI anywhere in CybelAngel: both of these are **poll-only**. Schedule
  them; do not wait for a push.
- No request-id header on either operation.
