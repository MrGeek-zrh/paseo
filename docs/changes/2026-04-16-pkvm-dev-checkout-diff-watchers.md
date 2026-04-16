# pkvm-dev Checkout Diff Watcher Exhaustion

## Summary

Desktop connections to `pkvm-dev` could remain in a long `connecting` state, while phone connections to the same host still succeeded.

The affected path was the checkout diff flow behind the `Changes` tab. The fix changed two behaviors: git workspaces now open on `Files` by default, and watcher-capacity exhaustion now returns an immediate visible error instead of keeping the desktop session blocked behind diff initialization.

Related design: `docs/plans/2026-04-16-on-demand-checkout-diff-design.md`

## Background

The reported problem came from a real host named `pkvm-dev`.

The reported symptom was asymmetric. The phone client connected to `pkvm-dev` successfully. The desktop client often stayed on `connecting`.

The user stated that session management was the main product value. Inline diff was useful but not required for first connection.

## Evidence

Daemon logs on `pkvm-dev` reported watcher-capacity exhaustion during working-tree watch setup.

The key failure shape was `ENOSPC` together with log text indicating working tree watcher capacity exhaustion under `/home/mrgeek/pkvm-x86`.

The visible desktop-side message after the fix was:

`Changes view unavailable on this host because file watcher capacity is exhausted.`

That message confirmed that the daemon returned a structured diff error instead of leaving the client waiting for the diff path.

## Behavior Change

Git workspaces now default to the `Files` tab instead of the `Changes` tab.

If the user explicitly opens `Changes`, the daemon still attempts checkout diff setup. If watcher capacity is exhausted, the daemon now returns an immediate error payload with an empty file list and a human-readable message.

The daemon does not keep retrying the diff subscription path for this resource condition.

## Compatibility

The change keeps the existing checkout diff error shape.

The daemon continues to use the existing `CheckoutError` fields and uses `code: "UNKNOWN"` for the watcher-capacity condition. That choice keeps the response compatible with older clients.

No breaking WebSocket schema change was introduced for this behavior change.

## Code Index

App changes:

- `packages/app/src/stores/explorer-tab-memory.ts`
- `packages/app/src/stores/panel-store.ts`
- `packages/app/src/stores/panel-store.test.ts`

Server changes:

- `packages/server/src/server/workspace-git-service.ts`
- `packages/server/src/server/workspace-git-service.test.ts`
- `packages/server/src/server/checkout-diff-manager.ts`
- `packages/server/src/server/checkout-diff-manager.test.ts`

Planning documents:

- `docs/plans/2026-04-16-on-demand-checkout-diff-design.md`
- `docs/plans/2026-04-16-on-demand-checkout-diff.md`

Published branch and commit recorded during deployment work:

- Branch: `mrgeek/on-demand-checkout-diff`
- Commit: `1e551007e3b78fd2c0c4c7b2131f3d5a97e8b8d6`
- Commit message: `fix: make checkout diff on demand`

## Verification

The implementation work previously included targeted tests for the app tab default, the workspace watcher exhaustion path, and the checkout diff manager error conversion path.

The host was also validated through an actual reconnect flow against `pkvm-dev`. After re-pairing, the client connected and the `Changes` page showed the expected resource-exhaustion message instead of leaving the session in `connecting`.

For this documentation update, repository validation still needs the standard formatter and type check in the local workspace.

## Deployment Status

The `pkvm-dev` main daemon was already switched to the updated code during the earlier deployment session.

The recorded daemon status at that time was:

- Version: `0.1.57-rc.1`
- Listen: `127.0.0.1:6767`
- Server ID: `srv_gShBgxMQvIcc`

## Rollback References

The functional rollback point is the pre-fix daemon and client behavior before the on-demand checkout diff change.

This document does not record a pre-fix commit hash in the local workspace. Any rollback should therefore reference the deployment history for branch `mrgeek/on-demand-checkout-diff` together with the daemon version history on `pkvm-dev`.

## Known Limits

This change does not increase host watcher capacity.

This change does not provide a degraded diff mode when the host lacks watcher resources.

The local repository state in this workspace did not contain the recorded deployment commit object when this document was written, so the commit metadata above is a deployment record rather than a locally verified object reference.
