---
name: Triage the external attack surface
description: Inventory CybelAngel-discovered assets, read their open ports, services and CVEs, then write asset and threat statuses back — for your own org or, as a partner, for a client org.
api: openapi/cybelangel-adm-inventory-openapi.yml
operations:
  - get-inventory-assets
  - get-inventory-assets-hostnames
  - get-inventory-threats-with-asset-info
  - put-inventory-assets-status
  - put-inventory-assets-threats-status
  - get-organization-inventory-assets
  - get-organization-inventory-assets-hostnames
  - get-organization-inventory-threats-with-asset-info
  - put-organization-inventory-assets-status
  - put-organization-inventory-assets-threats-status
  - get-organization-workspaces
generated: '2026-08-17'
method: generated
source: >-
  openapi/cybelangel-adm-inventory-openapi.yml, openapi/cybelangel-partner-openapi.yml,
  https://developers.cybelangel.com/docs/adm-inventory-api/7e88d945427a4-fetch-assets-details-from-adm-inventory,
  https://developers.cybelangel.com/docs/partner-api/72b66de24898e-cybel-angel-partners-api
---

# Triage the external attack surface

Base URL `https://api.cybelangel.com`. Authenticate first
(`skills/cybelangel-authenticate.md`).

Two parallel surfaces, identical shapes:

| Working on | Path form | Operations |
|---|---|---|
| Your own organization | `/v1/inventory/...` | `get-inventory-*`, `put-inventory-*` |
| A client organization (MSSP/partner) | `/v1/{organization_id}/inventory/...` | `get-organization-inventory-*`, `put-organization-inventory-*` |

Everything below uses the own-org operationIds; swap in the `-organization-` variants and add the
`{organization_id}` path parameter to run it as a partner. Use
`GET /v1/{organization_id}/workspaces` (`get-organization-workspaces`) to enumerate what a client
org contains before you start.

## Steps

1. **List the assets.** `GET /v1/inventory/assets` (`get-inventory-assets`), cursor-paginated
   (`cursor`; response `GetAssetsResponseDTO` with `items`, `next_cursor`, `total`). Each
   `InventoryAssetDTO` carries `id`, `value`, `type`, `status`, `asset_status`, `first_seen_at`,
   `last_seen_at`, `created_at`, `platform_link`, `registrar`, `organization`, `localization`,
   `tags` and `fingerprints`.

2. **Read the fingerprint.** `fingerprints.open_ports[]` → `OpenPortDTO` (`port`, `transport`,
   `service`) → `ServiceDTO` (`protocol`, `cpes[]`) → `OpenServiceCpeDTO` (`name`,
   `first_seen_at`, `last_seen_at`). This is how you tell a newly-exposed service from a
   long-standing one without diffing snapshots yourself.

3. **Enumerate hostnames.** `GET /v1/inventory/assets/hostnames`
   (`get-inventory-assets-hostnames`) when you need the hostname view rather than the asset view.

4. **Pull threats with their asset context.** `GET /v1/inventory/assets/threats`
   (`get-inventory-threats-with-asset-info`) returns
   `InventoryThreatWithAssetInfoDTO`: `hash`, `severity`, `type`, `status`, `port`, `first_seen`,
   `last_seen`, `data`, plus `asset_id` and `asset_value`. **A threat's identity is its `hash`,
   not a surrogate id** — the same finding keeps the same hash across re-detections, so hash is
   the right de-duplication key.

5. **Switch on the threat `type` to read `data`.** The payload is polymorphic:
   `DomainThreatDataDTO`, `SubDomainTakeoverThreatDataDTO`,
   `ExposedSensitiveFileThreatDataDTO`, `TLSCertificateThreatDataDTO`, `SpfThreatDataDTO`,
   `VulnerableTechnologyThreatDataDTO`, `OtherThreatDataDTO`. Most carry `summary`,
   `how_we_found_it`, `risks` and `details` as `TranslationElementDTO` objects with `en` and `fr`
   keys — pick your locale in code, there is no Accept-Language negotiation.

6. **Prioritize with the vulnerability signals.** `VulnerableTechnologyThreatDataDTO` gives you
   `cpe_id`, `cpe_vendor`, `cpe_product`, `cpe_product_version`, `cve_count`,
   `cve_in_kev_count`, `highest_cvss_score`, `highest_epss_score`, and `cves[]` of
   `VulnerableTechnologyCveDTO` (`cve_id`, `cvss_score`, `cvss_version`, `epss_score`,
   `have_known_exploit`, `required_action`). `cve_in_kev_count` and `have_known_exploit` map to
   CISA KEV — rank on those before raw CVSS.

7. **Write asset status back.** `PUT /v1/inventory/assets/status`
   (`put-inventory-assets-status`) with `UpdateAssetsStatusRequestBodyDTO`: select by
   `asset_ids` **or** `asset_values`, set `status`, and pass `asset_status` as the expected
   previous state.

8. **Write threat status back.** `PUT /v1/inventory/assets/threats/status`
   (`put-inventory-assets-threats-status`) with `UpdateAssetThreatStatusRequestBodyDTO`:
   `threats[]` of `{threat_hash, asset_value | asset_id}` plus the target `status`.

## Rules

- **Handle `409 Conflict` on both writes.** `put-inventory-assets-status` and its partner twin
  are the only operations in the estate that declare a 409. With no idempotency key on the write
  path, a concurrent change in the platform UI is the usual cause: re-read the asset, recompute
  `asset_status`, write again. Do not blind-retry.
- Cursor pagination only — there is no `skip`/`limit` on these operations.
- `400` and `403` return `{"error":{"message":…}}`; a `403` on a partner path means your grant does
  not cover that `organization_id`.
- Follow `platform_link` when a human needs to see the asset in the CybelAngel UI.
- `report_ids` on `InventoryThreatWithReportIds` joins a threat to the incident reports in the
  Reports API — a different base URL and a different scope set. See
  `skills/cybelangel-work-incident-reports.md`.
