---
name: Run day-two desktop operations and assist a user
description: Pause, resume, reboot, redeploy, tag and screenshot a Workspot Cloud PC, log a user off, message them, and open a remote assistance session — with the reversibility of each action stated plainly.
api: openapi/workspot-control-openapi.json
operations: [desktopDetailsUsingGET, pauseDesktopUsingPOST, resumeDesktopUsingPOST, retainDesktopUsingPOST, rebootDesktopUsingPOST, redeployDesktopUsingPOST, screenshotDesktopUsingPOST, tagDesktopUsingPOST, statusCheckUsingGET, logoffUserUsingPOST, sendMessageToUserUsingPOST, remoteAssistUserUsingPOST]
method: generated
generated: '2026-09-04'
source: https://docs.workspot.com/docs/using-the-workspot-control-api
---

# Day-two desktop operations and user assistance

Authenticate first — see `workspot-authenticate-and-poll.md`. Start from `desktopDetailsUsingGET` (`GET /v1.0/pools/{poolId}/desktops/{desktopId}`) to confirm current state before acting.

## Actions, ordered by how hard they are to take back

**Fully reversible**

- `pauseDesktopUsingPOST` — `POST /v1.0/pools/{poolId}/desktops/{desktopId}/pause`. Reversed by `resumeDesktopUsingPOST` (`.../resume`). Related: `retainDesktopUsingPOST` (`POST /v1.0/desktops/{desktopId}/retainDesktop`) sets the retention period applied to a desktop on suspension — a tenant-chosen value Workspot does not publish a default for.
- `tagDesktopUsingPOST` — `POST /v1.0/pools/{poolId}/desktops/{desktopId}/tags`. Set tags again to change them. See the [desktop tags guide](https://docs.workspot.com/docs/using-desktop-tags-in-the-workspot-control-api).

**Disruptive but not destructive**

- `rebootDesktopUsingPOST` — `.../reboot`. Ends the user's session. There is no cancel; check for an active session first.
- `logoffUserUsingPOST` — `POST /v1.0/users/{email}/logoff`. Signs the user out of their session. Unsaved work is theirs to lose — message them first.

**Destructive, no reversal operation exists**

- `redeployDesktopUsingPOST` — `.../redeploy`. Rebuilds the VM from its template. Anything not redirected off the VM is gone. **No cancel and no undo operation exists for this.** Require explicit human confirmation naming the desktop.
  *Not available on GCP.*

## Diagnostics

- `screenshotDesktopUsingPOST` — `.../screenshotDesktop`. Captures the **login screen only**; it returns nothing if a session is active. Useful for boot problems such as a stuck Windows Update. The link arrives in the async StatusURL response and the image **expires after 15 minutes** — fetch it promptly.
  *Not available on GCP.*
- `sendMessageToUserUsingPOST` — `POST /v1.0/users/{email}/sendMessage`. Pushes a message to the user's desktop. Use it before any disruptive action above.
- `remoteAssistUserUsingPOST` — `POST /v1.0/users/{email}/remoteAssistanceInvitation`. Takes the user's email, `desktopID`, `poolID`, and a temporary password set by the administrator for this session only. A `.msrincident` file is produced and linked from the StatusURL response; it **expires one hour after creation**.

  Treat the file and password as a credential pair: anyone holding both, with a desktop on the same network, can take the Expert role. Do not log them, do not paste them into a ticket body, and hand them to the intended engineer over a channel appropriate to that.

## Cloud parity

`redeployDesktopUsingPOST`, `screenshotDesktopUsingPOST`, `moveDesktopUsingPOST` and `cancelMoveDesktopUsingPOST` are **not available on GCP**. Determine the tenant's cloud before offering any of them, rather than discovering it from a failure.

## Reminders

- All of these are asynchronous. Poll `statusCheckUsingGET` and read `errorInfo` on `Failed`.
- No idempotency. A duplicate reboot reboots twice.
- 15 write calls per minute per customer.
