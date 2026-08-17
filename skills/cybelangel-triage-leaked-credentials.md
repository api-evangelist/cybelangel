---
name: Triage leaked credentials
description: Pull leaked credentials from CybelAngel — from a leak alert or from the account-wide watchlist — then resolve them and export the evidence.
api: openapi/cybelangel-alerts-openapi.yml
operations:
  - alerts_get_alert_credentials_alerts__alert_id__leak_credentials_get
  - alerts_get_leak_credentials_leak_credentials_get
  - get-credential-watchlist
  - get-reports-credentials-count
  - get-volume-of-credentials
  - get-export-credential-watchlist
  - update-status-of-credential
  - alerts_patch_an_alert_alerts__alert_id__customer_patch
generated: '2026-08-17'
method: generated
source: >-
  openapi/cybelangel-alerts-openapi.yml, openapi/cybelangel-platform-reports-openapi.yml,
  https://developers.cybelangel.com/docs/alerts-api/6e548d500150e-cybel-angel-api-toolbox-tutorial
---

# Triage leaked credentials

Credentials are exposed on **two** surfaces with different shapes and different base URLs. Pick
the one that matches your workflow; do not assume they return the same object.

- Alert-driven (`https://api.cybelangel.com`) — credentials attached to a specific leak alert.
- Watchlist-driven (`https://platform.cybelangel.com/api`) — the account's standing
  credential watchlist, with statuses and a CSV export.

Authenticate first — see `skills/cybelangel-authenticate.md`.

## A. From a leak alert

1. Find leak alerts: `GET /v1/alerts` with `categories=leak`
   (see `skills/cybelangel-poll-alert-feed.md`).
2. For each alert id, `GET /v1/alerts/{alert_id}/leak-credentials`
   (`alerts_get_alert_credentials_alerts__alert_id__leak_credentials_get`). The response is
   `CredentialsResponse` → `credentials[]` of `Cred` objects: `login`, `password`, and an
   optional `infostealer` block (`victim_ip`, `machine_name`, `target`, `leak_date`,
   `malware_name`).
   **Do not call `GET /v1/alerts/{alert_id}/credentials`** — it is `deprecated: true` in the
   specification and can return `410 Gone`.
3. Or sweep across alerts with `GET /v1/leak-credentials`
   (`alerts_get_leak_credentials_leak_credentials_get`), which also declares a `410`.
4. Record your verdict back on the alert: `PATCH /v1/alerts/{alert_id}/customer`
   (`alerts_patch_an_alert_alerts__alert_id__customer_patch`) with a `CustomerAssessment`
   (`AlertRequestCustomerFieldsPatch.assessment`) — true positive, false positive, unknown.
   This is the only customer-owned write on an alert.

## B. From the credential watchlist

1. Size the problem before you page it: `GET /v1/credentials/count`
   (`get-volume-of-credentials`) and `GET /v1/reports_credentials/count`
   (`get-reports-credentials-count`).
2. `GET /v1/credentials` (`get-credential-watchlist`) — offset paginated with **`skip` + `limit`**,
   not `cursor`. This is the older API generation; the cursor convention does not apply here.
   Each `Credential` carries `email`, `password`, `domain`, `ip_address`, `malware_name`,
   `malware_location`, `user_machine_name`, `user_session`, `extracted_at`,
   `last_detection_date`, `is_new_to_user`, `status`, and three cross-reference arrays:
   `cred_ids`, `alert_ids`, `report_ids`.
3. Use `alert_ids` / `report_ids` to join a watchlist credential back to the alert or incident
   report it came from. There is no single operation that resolves the join for you.
4. Export evidence: `GET /v1/credentials/export`
   (`get-export-credential-watchlist`) returns CSV.
5. Resolve or reopen: `POST /v1/credentials/status`
   (`update-status-of-credential`) with a `CredentialStatus`.

## Rules

- Required scopes on the watchlist surface: `credentials.read` to list, `credentials.move` to
  change status, `credentials.export` for the CSV. A `403` here means a missing scope, not a bad
  token — re-minting will not fix it.
- **Handle this data as a breach payload.** Responses contain live plaintext passwords, victim
  machine names and infostealer telemetry. Do not log responses, do not write them to a shared
  cache, and prefer streaming into the destination over spooling to disk. The provider's own
  tutorial script writes `credentials.json` to the working directory — do not copy that pattern
  into production.
- There is no idempotency key on `POST /v1/credentials/status`. It is a status set rather than an
  append, so a retry converges, but never assume that for the comment or remediation endpoints.
