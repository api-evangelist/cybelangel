---
name: Manage monitored keywords
description: Read, create, update and enable/disable the keywords that drive CybelAngel detection, handling HTTP 207 partial success correctly.
api: openapi/cybelangel-keywords-openapi.yml
operations:
  - get-keywords
  - create-keywords
  - update-keywords
  - update-keywords-status
  - get-workspaces
  - get-organization-keywords
  - create-organization-keywords
  - delete-organization-keywords
  - update-organization-keywords-status
generated: '2026-08-17'
method: generated
source: >-
  openapi/cybelangel-keywords-openapi.yml, openapi/cybelangel-partner-openapi.yml,
  https://developers.cybelangel.com/docs/keywords-api/c812fc6b544b0-manipulate-your-keywords
---

# Manage monitored keywords

Keywords are what CybelAngel actually hunts for — brand names, domains, internal project
codenames, executive email addresses. Changing them changes what gets detected, so this is the
highest-consequence write surface in the estate.

Base URL `https://api.cybelangel.com`. Authenticate first
(`skills/cybelangel-authenticate.md`).

## Steps

1. **Enumerate workspaces first.** `GET /v1/workspaces` (`get-workspaces`) returns
   `WorkspaceListDTO` of `WorkspaceDTO` (`id`, `name`, `description`). Every keyword is scoped to
   one or more workspaces, and creating a keyword without the right workspace id puts it where
   nobody is watching.

2. **List current keywords.** `GET /v1/keywords` (`get-keywords`), cursor-paginated. Each
   `ClientKeywordDTO`: `id`, `name`, `type`, `status`, `description`, `workspaces`,
   `creation_date`, `last_modification_date`.

3. **Create.** `POST /v1/keywords` (`create-keywords`) with
   `ClientCreateKeywordsBodyDTO.keywords[]` of `ClientCreateKeywordInputDTO`
   (`name`, `type`, `description`, `workspaces`).

4. **Update.** `PATCH /v1/keywords` (`update-keywords`) with
   `ClientUpdateKeywordsBodyDTO.keywords[]` of `ClientUpdateKeywordInputDTO`
   (`id`, `type`, `description`, `workspaces`). Note there is no `name` field on the update input —
   a keyword's text is immutable; to change it, create a new one and disable the old one.

5. **Enable / disable.** `PUT /v1/keywords/status` (`update-keywords-status`) with
   `UpdateKeywordsStatusBodyDTO.keywords[]` of `{id, status}`.

## HANDLE HTTP 207 — this is the trap

Create and update are **batch** operations with **partial success** semantics. They return:

- `201` (create) or `200` (update) — everything succeeded, but the body **still** has both arrays.
- **`207`** — some items succeeded and some failed.

Both response shapes carry `created_keywords[]` / `updated_keywords[]` **and**
`failed_keywords[]`. A failure entry is `FailedToCreateKeywordDTO` (`name`, `error_code`,
`error_details`) or `FailedToUpdateKeywordDTO` (`id`, `error_code`, `error_details`), with
`error_code` drawn from `CreateKeywordErrorCode` / `UpdateKeywordErrorCode`.

**A client that only branches on `response.ok` will report success while silently dropping
keywords.** Always read `failed_keywords[]` and surface it, on every status code.

## Partner variant

As an MSSP, use the `{organization_id}`-scoped operations: `get-organization-keywords`,
`create-organization-keywords`, `update-organization-keywords-status`, and
`delete-organization-keywords` (`DELETE /v1/{organization_id}/keywords` with
`DeleteKeywordBodyDTO.keyword_ids[]`, also 200/207 with `deleted_keywords[]` +
`failed_keywords[]`).

**The partner surface is strictly more capable than the customer one.** `PartnerKeywordDTO` and
`PartnerCreateKeywordInputDTO` expose four detection-tuning fields the customer-facing Keywords
API does not: `rule`, `sources` (`KeywordSource`), `active_search` and `infostealer_search`.
There is also **no DELETE on the customer-facing API** — a customer can only disable a keyword,
while a partner can remove it. Do not assume feature parity between the two.

## Rules

- Cursor pagination (`cursor` / `next_cursor`), not `skip`/`limit`.
- No idempotency key: re-POSTing the same batch after a timeout can create duplicate keywords.
  Read back with `get-keywords` before retrying a create.
- `400` → `{"error":{"message":…}}`. `403` → the plan or grant does not cover keyword management.
- These writes change detection scope. In an automated pipeline, gate creates and deletes behind
  a human approval step rather than letting an agent widen or narrow monitoring unattended.
