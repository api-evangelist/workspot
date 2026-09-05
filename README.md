# Workspot

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
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

# Workspot

Workspot is a cloud-native virtual desktop infrastructure (VDI) and Cloud PC provider delivering
Desktop-as-a-Service through its Workspot Control SaaS management plane and the Workspot Desktop
Control Fabric, a globally distributed architecture that provisions and manages Windows desktops
and published applications across Microsoft Azure, Google Cloud and Amazon WorkSpaces Core.

- Website: https://www.workspot.com/
- Documentation: https://docs.workspot.com/
- API reference: https://api.workspot.com/swagger-ui.html
- Status: https://status.workspot.com/

## APIs profiled

| API | Contract | Auth | Base |
|---|---|---|---|
| Workspot Control REST API | Swagger 2.0, 85 paths / 105 operations / 120 definitions | OAuth 2.0 password grant, or Entra ID token | `https://api.us.workspot.com`, `https://api.eu.workspot.com`, `https://api.workspot.com` |
| Workspot SIEM (Splunk) Events API | documented only (no published spec) | HMAC-SHA256, `Authorization: WSEvents` | same host family |

## What this profile found

- **A real, anonymously-served contract.** The Workspot Control Swagger 2.0 document is live at
  `https://api.workspot.com/v2/api-docs` and served identically from all three regional hosts.
  Saved verbatim to `openapi/workspot-control-openapi-original.json`.
- **The published document is not valid JSON.** Two `example` arrays serialize bare unquoted
  tokens, so a strict parser rejects Workspot's contract as served.
  `openapi/workspot-control-openapi.json` is the same document with only those tokens quoted;
  nothing else was changed. Recorded in `overlays/`.
- **The contract omits its own security model.** Zero `securityDefinitions` across 105 operations,
  yet every operation declares 401 and 403. The real auth model exists only in prose.
- **No 429, 404 or 5xx is declared** anywhere in the spec, even though the docs document
  per-customer throttling that returns 429.
- **No idempotency**, across 63 mutating operations.
- **Pagination on exactly one operation** of 105 (`staleDevicesUsingGET`).
- **A remote MCP server** is advertised via RFC 8414 and RFC 9728 discovery documents on
  `www.workspot.com` — but it is the WordPress adapter on the marketing site, scope `mcp`,
  unrelated to the Control API. `tools/list` is OAuth-gated. See `mcp/`.
- **A real event surface** — the checkpointed SIEM/Splunk pull feed — but no AsyncAPI and no
  webhooks anywhere in 580 documentation pages. See `asyncapi/`.
- **No SDKs, no CLI, no GitHub organization, no Postman collection, no public pricing.**
- **No vulnerability disclosure channel**: no `security.txt`, no `/security/` page, no bug bounty.
- **A stale pointer in Workspot's own spec**: `info.termsOfService` resolves to the legal index,
  not the agreement it names. Recorded in `lifecycle/`.

## Provenance

Every artifact carries `generated`, `method` and `source` frontmatter. Every `operationId`
referenced in `skills/` and `conventions/` was verified to exist verbatim in the harvested
specification. Absences recorded here are measured, not assumed.
