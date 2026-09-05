---
name: Generate a Workspot usage report
description: Produce a Workspot Control usage report for a date range using the asynchronous submit-poll-download flow, respecting the 30-day window limit.
api: openapi/workspot-control-openapi.json
operations: [usageReportUsingPOST, statusCheckUsingGET, generateAllUsersReportUsingGET, getLicensesUsingGET]
method: generated
generated: '2026-09-04'
source: https://docs.workspot.com/docs/using-the-workspot-control-api
---

# Generate a Workspot usage report

Authenticate first — see `workspot-authenticate-and-poll.md`. This is a read-only flow and the safest place to validate a new integration.

## Flow

1. **Submit.** `usageReportUsingPOST` — `POST /v1.0/reports/generateusagereport` with `Content-Type: application/json` and a body of:

   - `start` — first date of the range, `YYYY-MM-DD`
   - `end` — last date of the range, `YYYY-MM-DD`
   - `format` — `"JSON"`. This is the **only supported value**; convert to CSV yourself afterwards.

   **The range may not start more than 30 days in the past.** For a longer history, issue several requests over consecutive windows and stitch the results — and pace them, since reads are capped at 20/min.

   The response carries a `StatusUrl`.

2. **Poll.** GET the `StatusUrl` every **5 seconds** until `status` is no longer `InProgress`. Workspot's own example uses exactly this interval. `statusCheckUsingGET` (`GET /v1.0/operation/{operationId}`) is the same polling surface.

3. **Download.** When ready, the status response carries `details.downloadUrl`. GET it. The payload's `UsersUsage` array is the report.

## Reading the results

Most fields are self-explanatory. Three are not:

- **`costCenter`** — taken from a chosen field on the AD user record. If the cost-center feature is not enabled for the tenant this is **always blank**, which means empty here signals "feature off", not "user unassigned". Do not report it as missing data.
- **`timeSpent`** — hours the user was actively signed in to desktops and apps. Active time, not elapsed time.
- **`sessions`** — the number of times the user *initiated* a new desktop or application session.

## Related read-only reports

- `generateAllUsersReportUsingGET` — `GET /v1.0/reports/users` — the full user roster.
- `getLicensesUsingGET` — `GET /v1.0/licenses` — purchased license counts per VM type. Pair it with the usage report to compare entitlement against actual consumption; this is the closest thing Workspot exposes to spend visibility, since it publishes no public pricing.

## Notes

- Report generation is asynchronous even though nothing is mutated — do not expect data on the first response.
- Reads are capped at 20/min per customer. On 429, read `X-Retry-After-Secs` from the response **body**.
- No endpoint in this flow paginates. A wide date range on a large tenant returns one large document.
