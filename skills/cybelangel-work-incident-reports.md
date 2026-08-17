---
name: Work CybelAngel incident reports end to end
description: Search incident reports, pull their evidence, comment, resolve them in bulk, and file an analyst remediation request.
api: openapi/cybelangel-platform-reports-openapi.yml
operations:
  - getReportPermissions
  - get-v2-reports
  - get-v2-reports-by-id
  - get-mirror-by-report-id
  - get-mirror-csv-by-report-id
  - get-mirror-archive-by-report-id
  - get-attachments-by-report-id
  - get-pdf-by-report-id
  - get-report-comments
  - post-report-comments
  - update-report-status-by-report-id
  - update-multiple-reports-statuses
  - create-remediation-request
  - get-v2-stats-reports
  - get-report-asset
generated: '2026-08-17'
method: generated
source: >-
  openapi/cybelangel-platform-reports-openapi.yml,
  https://developers.cybelangel.com/docs/cybelangel-platform-api/39d4926befc14-what-can-i-do-with-this-api,
  https://developers.cybelangel.com/docs/cybelangel-platform-api/129db10d84718-retrieve-the-last-incident-reports
---

# Work CybelAngel incident reports end to end

Base URL `https://platform.cybelangel.com/api` — note this is **not** `api.cybelangel.com`.
Authenticate first (`skills/cybelangel-authenticate.md`); the audience stays
`https://platform.cybelangel.com/`.

## Steps

1. **Check what you are allowed to do.** `GET /v1/reports/permissions`
   (`getReportPermissions`) returns the caller's report permissions. Call this once at startup
   rather than discovering a `403` mid-run.

2. **Search reports.** `GET /v2/reports` (`get-v2-reports`) with a `start-date` / `end-date`
   window (ISO 8601, e.g. `2026-08-01T00:00:00`). Note the **hyphenated** parameter names here —
   the newer APIs use underscores, this one does not.

3. **Read one report.** `GET /v2/reports/{report-id}` (`get-v2-reports-by-id`) returns a
   `Report-v2`: `id`, `incident_id`, `incident_type`, `category`, `status`, `severity`,
   `created_at`, `detected_at`, `updated_at`, `url`, `abstract`, `ip`, `keywords`, `attachments`,
   `workspace`, `module`, `report_content`.

4. **Pull the evidence.** Pick the shape your workflow needs:
   - `GET /v1/reports/{report-id}/mirror` (`get-mirror-by-report-id`) — the file listing
     (`ReportMirror`: `files_count`, `files_volume`, `available_files_count`, `status`,
     `stream_id`).
   - `GET /v1/reports/{report-id}/mirror/csv` (`get-mirror-csv-by-report-id`) — same as CSV.
   - `GET /v1/reports/{report-id}/mirror/archive` (`get-mirror-archive-by-report-id`) — zip.
   - `GET /v1/reports/{report-id}/attachments/{attachment-id}` (`get-attachments-by-report-id`)
     — a specific attachment (screenshot etc). Ids come from `Report-v2.attachments[]`.
   - `GET /v1/reports/{report-id}/pdf` (`get-pdf-by-report-id`) — the human-readable report.
   - `GET /v1/assets/{asset-type}/{asset-name}` (`get-report-asset`) — the asset behind the
     incident. Needs the `assets.download` scope.

5. **Collaborate.** `GET /v1/reports/{report-id}/comments` (`get-report-comments`) and
   `POST /v1/reports/{report-id}/comments` (`post-report-comments`). Comments are threaded via
   `discussion_id` + `parent_id`. Scopes: `reports_global_comments.read` /
   `reports_global_comments.write` — the provider's own scope description flags the write as
   "Endpoint not accessible, yet", so expect a `403` and treat write access as unconfirmed.

6. **Resolve.** One report: `PUT /v1/reports/{report-id}/status`
   (`update-report-status-by-report-id`). Many at once: `POST /v1/reports/status`
   (`update-multiple-reports-statuses`) — prefer the bulk call, it is the only batching in this
   API and it keeps you far from the 15 req/s ceiling. Both need `reports.move`.

7. **Escalate to CybelAngel analysts.** `POST /v1/reports/remediation-request`
   (`create-remediation-request`) files a takedown/remediation request.

8. **Report on the program.** `GET /v2/stats/reports` (`get-v2-stats-reports`) for report volume,
   with `split_by` to break the series down.

## Rules

- **`POST` is not retry-safe here.** `post-report-comments` and `create-remediation-request` have
  **no idempotency key** — a retry after a timeout creates a duplicate comment or a duplicate
  remediation request. Record the response before retrying, and prefer a read-back over a blind
  retry. The status writes (`PUT .../status`, `POST /v1/reports/status`) are status *sets* and do
  converge on retry.
- **This spec declares no error schemas.** Its `400/401/403/404/500` responses carry a
  description only, so you cannot rely on the contract for the failure shape. In practice the
  envelope is `{"error":{"message":"…"}}` — see `errors/cybelangel-problem-types.yml`.
- Pagination on this API is `skip` + `limit` (offset), not `cursor`.
- No request-id header is returned on any operation. Log your own correlation id alongside the
  URL and timestamp so support@cybelangel.com has something to work from.
- Scope map for this flow: `reports.read` (search/read), `reports.move` (status),
  `assets.download` (assets/attachments), `reports_global_comments.read` / `.write` (comments).
