---
name: Authenticate to Workspot Control and poll an async operation
description: Obtain an OAuth 2.0 access token for the Workspot Control REST API on the correct regional host, then correctly drive the submit-then-poll pattern that most Workspot write commands use.
api: openapi/workspot-control-openapi.json
operations: [statusCheckUsingGET, getLicensesUsingGET]
method: generated
generated: '2026-09-04'
source: https://docs.workspot.com/docs/using-the-workspot-control-api
---

# Authenticate to Workspot Control and poll an async operation

Every other Workspot skill depends on this one. Do this first.

## 1. Pick the right host

Workspot Control is regional and a tenant exists in exactly one region.

- `https://api.us.workspot.com` — US Control deployments (preferred)
- `https://api.eu.workspot.com` — EU Control deployments (preferred)
- `https://api.workspot.com` — the older address, still served

If you do not know the tenant's region, ask. Do not try both — a wrong-region call fails in ways that look like a permissions problem.

## 2. Get a token

Two mutually exclusive modes. Check which the tenant uses before building the request.

**Default (Client & Secret).** OAuth 2.0 resource-owner password grant:

- `POST https://api.<region>.workspot.com/oauth/token`
- Header `Authorization: Basic base64("<ClientID>:<ClientSecret>")` — the pair from Control under **Setup > API**
- Form body `grant_type=password`, `username=<Control admin email>`, `password=<Control admin password>`
- Read `access_token` from the response

**Entra-ID-only tenants.** The Client ID and Client Secret are not used and no Control password is sent. Obtain an Entra ID access token for the "Workspot" Enterprise Application registered in the tenant and use it directly as the bearer token. This is mandatory when Control's Setup > API Auth Type is "Azure AD Token (Entra ID)".

Send every subsequent call with `Authorization: Bearer <access_token>`.

**The token expires one hour after issue.** There is no refresh token on the password grant — repeat step 2. Budget for this on any long-running loop; a 401 mid-run usually means expiry, not a permissions change.

If API access has never been enabled for the tenant there is no Setup > API tab at all, and the tenant must contact Workspot Support. That is a 403, not something you can fix by retrying.

## 3. Verify the token cheaply

Call `getLicensesUsingGET` — `GET /v1.0/licenses`. It is read-only, cheap, and returns purchased license counts per VM type. A 200 confirms the token, region and permissions in one call.

## 4. Drive the async pattern

23 operations return **202 Accepted** with a `StatusURL` rather than doing the work inline. Workspot's guidance is explicit: success is not guaranteed, so check **both** the HTTP status code and the response body.

1. Submit the command. If you get 202, read `StatusURL` from the body.
2. Poll `statusCheckUsingGET` — `GET /v1.0/operation/{operationId}` — every **5 seconds** (Workspot's own example interval).
3. Stop when `status` is no longer `InProgress`. It becomes `Succeeded` or `Failed`.
4. On `Failed`, read `errorInfo.error` and `errorInfo.description`. These are the real failure; the HTTP 202 only ever meant "accepted".

A synchronous command returns its result directly and has no StatusURL. Gateway reboot (`rebootGwHostUsingPOST`) is currently synchronous but Workspot has said it will become asynchronous — handle both shapes.

## Rules that apply to every Workspot call

- **There is no idempotency.** No Workspot operation accepts an idempotency key. A retried write executes a second time. Never retry a POST or DELETE blind — re-read state with the matching GET first and reconcile.
- **Throttling is body-signalled, not header-signalled.** 20 GET/min and 15 write/min per customer. On 429 the wait hint is `X-Retry-After-Secs` *inside the JSON body*, not in a response header. Reading headers alone will find nothing.
- **Errors are not RFC 9457.** The envelope is `{"error": ..., "description": ...}` as `application/json`.
- **The spec omits its own security.** The published Swagger declares no securityDefinitions and no 429, 404 or 5xx responses. Do not conclude from the contract that those cannot occur.
- **Users are addressed by email in the path** (`/v1.0/users/{email}/...`). URL-encode it.
