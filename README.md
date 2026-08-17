# CybelAngel

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

CybelAngel is a Paris- and Boston-based external risk protection company. Its platform scans the
public internet, the deep and dark web, unsecured file servers and databases, cloud drives, code-
and paste-sharing sites, DNS and social platforms for a customer's exposed data and unmanaged
assets, across external attack surface (ADM) inventory, data-breach prevention, credential
intelligence, brand protection and impersonation, cyber threat intelligence, and analyst-led
remediation.

## What this profile covers

CybelAngel publishes **seven documented REST APIs** on a Stoplight developer portal at
[developers.cybelangel.com](https://developers.cybelangel.com/) — **52 operations across seven
OpenAPI 3.1.0 documents**, all harvested verbatim into [`openapi/`](openapi/):

| API | Ops | Base URL |
|---|---|---|
| [Reports](https://developers.cybelangel.com/docs/cybelangel-platform-api/39d4926befc14-what-can-i-do-with-this-api) | 21 | `https://platform.cybelangel.com/api` |
| [Alerts (Alerts in Feed)](https://developers.cybelangel.com/docs/alerts-api/72b66de24898e-cybel-angel-alerts-api-real-time-threat-intelligence) | 9 | `https://api.cybelangel.com` |
| [Partner](https://developers.cybelangel.com/docs/partner-api/72b66de24898e-cybel-angel-partners-api) | 10 | `https://api.cybelangel.com` |
| [ADM Inventory](https://developers.cybelangel.com/docs/adm-inventory-api/7e88d945427a4-fetch-assets-details-from-adm-inventory) | 5 | `https://api.cybelangel.com` |
| [Keywords](https://developers.cybelangel.com/docs/keywords-api/c812fc6b544b0-manipulate-your-keywords) | 5 | `https://api.cybelangel.com` |
| [Threat Intelligence](https://developers.cybelangel.com/docs/threat-intelligence-api/38e63ab3d5c42-fetch-threat-intelligence-claimed-attacks) | 1 | `https://api.cybelangel.com` |
| [Audit Logs](https://developers.cybelangel.com/docs/audit-logs-api/72b66de24898e-cybel-angel-audit-logs-api) | 1 | `https://api.cybelangel.com` |

Authentication is OAuth 2.0 client-credentials against an Auth0 tenant at
`auth.cybelangel.com`, which serves real OIDC discovery, RFC 8414 metadata and JWKS
(all three captured in [`well-known/`](well-known/)).

## Notable findings

- **Token minting is the real quota.** 2,000 tokens/month per `client_id` against a token the
  docs variously describe as lasting 1 hour or 24 hours. Caching is mandatory, not an
  optimization — see [`rate-limits/`](rate-limits/) and [`conventions/`](conventions/).
- **Two pagination styles in one estate**: cursor on the newer `api.cybelangel.com` APIs,
  `skip`/`limit` on the Reports watchlists and Threat Intelligence.
- **No idempotency key anywhere**, and two genuinely non-idempotent POSTs (report comments,
  remediation requests).
- **Published limits, no runtime signal**: 15 req/s and 20 concurrent requests are documented,
  but no `X-RateLimit-*`, `RateLimit-*` or `Retry-After` header exists and the exhaustion status
  code is never named.
- **The Reports API declares no error schemas** — its 4xx/5xx responses carry a description only,
  unlike the six sibling specs.
- **STIX is a real strength**: `GET /v1/stix/alerts` returns OASIS STIX bundles, with a published
  schema mapping per alert category.
- **The `/security.txt` is broken three ways** — wrong path, no `Expires`, and its only `Contact`
  is the marketing site's WordPress agency rather than CybelAngel. Detail in
  [`security/`](security/).
- **No SDKs, no CLI, no MCP server, no agent card, no webhooks/AsyncAPI, no status page, no
  published prices.** Every registry and well-known path probed is recorded with the status it
  actually returned.
- CybelAngel does hold **ISO/IEC 27001:2022** and **SOC 2 Type I** (both A-LIGN) — see
  [`conformance/`](conformance/).

Six packaged [Agent Skills](skills/) ground the marquee flows in real `operationId`s, and each
harvested spec has an [Overlay](overlays/) carrying API Evangelist's derived findings without
mutating the provider's document.

Source: portfolio company of [serena](https://github.com/api-evangelist/serena) — https://www.cybelangel.com/
