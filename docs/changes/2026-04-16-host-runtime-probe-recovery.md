# Summary

Desktop host runtime now allows a successful background probe to replace the active client when the current host connection is no longer `online`.

# Background

On `pkvm-dev`, the desktop client could keep showing `Connecting` after relay interruptions even though later relay probes could still connect.

The active host runtime client kept its `activeConnectionId` after disconnect and error states. A later successful probe for the same host could connect and ping, but the probe client was closed before it became the active client.

This record was written from branch `mrgeek/on-demand-checkout-diff` in `/Users/mrgeek/tmp/paseo-src` after syncing local Git, GitHub, and `pkvm-dev` source checkouts to commit `c94408ca9ed52e09ca0a64d28c7df9486ef5853e`.

# Evidence

Remote daemon logs on `pkvm-dev` showed repeated short-lived relay data sockets after the main session had already dropped. The daemon stayed alive while `relay_data_connected` was followed quickly by `relay_data_disconnected` with `reason: "Client disconnected"`.

Desktop-side analysis showed that `packages/app/src/runtime/host-runtime.ts` only promoted the first successful probe when `snapshot.activeConnectionId` was empty. Once the active client had disconnected, `activeConnectionId` still remained set, so later successful probes were treated as background probes and were closed.

A regression test was added to reproduce this exact state transition and verify that the next successful probe now becomes the active client again.

# Behavior Change

`HostRuntimeController.runProbeCycleNow()` now treats a host as eligible for probe handoff whenever the current snapshot is not `online`, even if `activeConnectionId` is still present.

This change keeps the existing behavior for healthy active connections. It only changes the handoff path for reconnect and recovery states.

# Compatibility

The change is local to desktop app host runtime state management.

No WebSocket schema, relay protocol, daemon request shape, or persisted host profile format was changed.

# Code Index

- `packages/app/src/runtime/host-runtime.ts`
- `packages/app/src/runtime/host-runtime.test.ts`

# Verification

The following commands passed:

`cd /Users/mrgeek/tmp/paseo-src/packages/app && npx vitest run src/runtime/host-runtime.test.ts --bail=1`

`cd /Users/mrgeek/tmp/paseo-src && npm run typecheck`

`cd /Users/mrgeek/tmp/paseo-src && npm run format`

# Deployment Status

Not deployed.

The `pkvm-dev` source checkout at `/home/mrgeek/tmp/paseo-fork` was synced to branch `mrgeek/on-demand-checkout-diff` at commit `c94408ca9ed52e09ca0a64d28c7df9486ef5853e` before this fix was applied locally.

# Rollback References

The pre-fix synced branch state was `c94408ca9ed52e09ca0a64d28c7df9486ef5853e` on branch `mrgeek/on-demand-checkout-diff`.

`pkvm-dev` still had local untracked path `.codex` at the time of sync. That path was not modified by this change record.

# Known Limits

This change does not remove the underlying relay instability that produced `1006`, stale control sockets, or TLS `ECONNRESET` events.

This change only makes desktop host runtime recovery use the next successful probe instead of staying attached to a disconnected active client.

The current `packages/app` runtime tests still print existing Expo and AsyncStorage warnings in the Node test environment, but the test command itself passed.
