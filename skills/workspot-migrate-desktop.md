---
name: Move or upgrade a persistent desktop between pools
description: Migrate a persistent Workspot desktop to another pool using moveDesktop or upgradeDesktop, honouring the documented preconditions, the enterprise-wide concurrency cap, and the cancel paths that recover a failed migration.
api: openapi/workspot-control-openapi.json
operations: [desktopDetailsUsingGET, poolsUsingGET, moveDesktopUsingPOST, cancelMoveDesktopUsingPOST, upgradeDesktopUsingPOST, cancelUpgradeDesktopUsingPOST, rebootDesktopUsingPOST, statusCheckUsingGET]
method: generated
generated: '2026-09-04'
source: https://docs.workspot.com/docs/using-the-workspot-control-api
---

# Move or upgrade a persistent desktop between pools

Authenticate first — see `workspot-authenticate-and-poll.md`.

This is the most disruptive routine operation in the Workspot API. It shuts down a user's desktop, copies it, and **deletes the original on success**. Read all of it before running any of it.

## Preconditions — check every one

`moveDesktopUsingPOST` and `cancelMoveDesktopUsingPOST` are available only when **all** of these hold:

- The cloud is **Azure**. Both commands are unavailable on GCP.
- The desktop is **persistent**.
- The desktop is not suspended, not part of a regional disaster-recovery event, and has **no connected user**.
- Source and target pools are in the **same Azure region**.
- The target pool's VM is consistent with the source — the SKUs must match. You cannot move a GPU workstation into a non-GPU pool.

Confirm current state with `desktopDetailsUsingGET` and the target with `poolsUsingGET` before submitting. A precondition failure surfaces as a failed async operation, after the desktop has already been shut down.

## What the move actually does

1. The source desktop is shut down.
2. A snapshot is taken of it.
3. A copy is created in the target pool.
4. The new desktop is powered on.
5. The Workspot Agent registers it.
6. The user's assignment transfers. The DNS name is preserved, so the move is invisible to the end user.
7. **The source desktop is deleted.**

## Running it

- `moveDesktopUsingPOST` — `POST /v1.0/pools/{poolId}/desktops/{desktopId}/moveDesktop`
- `upgradeDesktopUsingPOST` — `POST /v1.0/desktops/{desktopId}/upgradeDesktop` — moves a persistent desktop to a specified pool as an upgrade

Both are asynchronous. Poll `statusCheckUsingGET` to completion.

**Pacing, from Workspot's own best practices:**

- Cap outstanding create-or-move calls at **ten for the entire enterprise**, not per pool and not per caller.
- Leave **60 seconds or more between `moveDesktop` calls**.
- Move during off-hours, and tell users first. The desktop is inaccessible during migration and a user who finds it down will try to reboot it.

**Never reboot or redeploy a desktop while its migration is in progress.**

## Recovery

The cancel operations are `cancelMoveDesktopUsingPOST` (`.../cancelMoveDesktop`) and `cancelUpgradeDesktopUsingPOST` (`POST /v1.0/desktops/{desktopId}/cancelUpgradeDesktop`).

Workspot documents **no time window** for either. Treat them as best-effort and invoke them promptly rather than assuming they stay available.

Which recovery applies depends on how far the move got:

- **Failed early** (powering down or backing up the source): the source is powered back on automatically and stays assigned to the user. Nothing to do.
- **Failed later, assignment not yet transferred:** reboot the source desktop with `rebootDesktopUsingPOST`. That is sufficient.
- **Failed later, assignment already transferred:** call `cancelMoveDesktopUsingPOST` to bring the source desktop back.
- **Agent never registers** (more than a few hours): auto-registration has failed and the VM must be re-registered manually by a human running `reregister_poolvm.bat` with the pool's Agent Token on the desktop. Retrieve the token with `poolAgentTokenUsingGET` (`GET /v1.0/pools/{poolId}/agentToken`) — it is a credential; hand it over accordingly. This is not something to automate silently.

## The snapshot is not a self-service rollback

A snapshot of the original desktop is retained after a successful move and **can be restored only by Workspot Support**. There is no API operation for it and Workspot publishes no retention period. Do not promise a user that a completed move can be reversed — escalate to Workspot Support and let them state what is recoverable.
