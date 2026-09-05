---
name: Provision a Cloud PC and assign it to a user
description: Create a desktop in a Workspot pool, wait for provisioning, and assign it to an end user — respecting Workspot's one-at-a-time provisioning limit and the absence of any idempotency contract.
api: openapi/workspot-control-openapi.json
operations: [poolsUsingGET, poolInfoUsingGET, getpoolDesktopsUsingGET, createDesktopUsingPOST, statusCheckUsingGET, addUserUsingPOST, userDetailsUsingGET, assignDesktopUsingPOST, assignPoolUsingPOST, unassignDesktopUsingDELETE]
method: generated
generated: '2026-09-04'
source: https://docs.workspot.com/docs/using-the-workspot-control-api
---

# Provision a Cloud PC and assign it to a user

Authenticate first — see `workspot-authenticate-and-poll.md`.

This flow creates infrastructure that costs money and cannot be undone. Read the safety section before running it.

## Steps

1. **Choose the pool.** `poolsUsingGET` — `GET /v1.0/pools` — lists all desktop pools. Inspect one with `poolInfoUsingGET` (`GET /v1.0/pools/{poolId}`). The pool fixes the template, VM class, cloud and region; you cannot override them per desktop.

2. **Check what already exists — this is your idempotency substitute.** `getpoolDesktopsUsingGET` — `GET /v1.0/pools/{poolId}/desktops`. Because Workspot has no idempotency key, this GET is the only thing standing between a retry and a duplicate VM. Do it every time, not just on retries.

   Note this endpoint has **no pagination** — no page, cursor, limit or offset parameter exists on it. On a large tenant you get the whole list or nothing.

3. **Create the desktop.** `createDesktopUsingPOST` — `POST /v1.0/pools/{poolId}/desktops`. For a custom-cloud pool use `createCustomDesktopUsingPOST` (`POST /v1.0/pools/{poolId}/customdesktop`) instead.

   This returns 202. Poll `statusCheckUsingGET` until `Succeeded` or `Failed`.

   If `errorInfo.error` is `maxConcurrentProvisioningError`, you have hit the enterprise-wide cap on outstanding provisioning requests. Wait and resubmit — the desktop was **not** created. Workspot provisions desktops strictly one at a time, never in parallel, so a queue of creates is inherently serial.

4. **Ensure the user exists.** `userDetailsUsingGET` — `GET /v1.0/users/{email}/`. If absent, `addUserUsingPOST` — `POST /v1.0/users`.

5. **Assign.** `assignDesktopUsingPOST` — `POST /v1.0/users/{email}/desktops` — binds a specific desktop to the user. To entitle a user to a whole pool instead, use `assignPoolUsingPOST` (`POST /v1.0/users/{email}/pools`).

6. **Verify.** Re-read `userDetailsUsingGET`. Do not report success from the 202 alone.

## Safety

- **The assignment is reversible; the desktop is not.** `unassignDesktopUsingDELETE` (`DELETE /v1.0/users/{email}/desktops/{desktopId}`) cleanly reverses step 5 at any time. But `deleteDesktopUsingDELETE` destroys the VM and there is **no undo, restore, trash or soft-delete anywhere in this API**, and no restore window is published. If you provisioned in error, unassign first and confirm with a human before deleting.
- **Never blind-retry step 3.** A retried create makes a second VM that bills. Re-run step 2 instead.
- **Watch the write budget.** 15 write calls per minute per customer. A bulk provisioning run must pace itself; on 429 read `X-Retry-After-Secs` from the response *body*.
- **Cost is real.** Each desktop is a VM in the customer's own cloud subscription. Confirm the count with a human before looping.
