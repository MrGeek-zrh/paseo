# On-Demand Checkout Diff Design

## Goal

Make checkout diff opt-in for git workspaces and keep remote session management usable when the host cannot afford more file watchers.

## Problem

Git workspaces currently default to the `Changes` tab. That default mounts `GitDiffPane`, which immediately requests a one-shot checkout diff and then starts a diff subscription. On the daemon side, the first diff response waits for `requestWorkingTreeWatch()` to finish. On large Linux workspaces, recursive per-directory watcher setup can hit `ENOSPC` after a long traversal, so the user waits on a diff feature before the main remote session feels available.

The user value is uneven here. Session management, agent history, and workspace browsing matter more than inline diff for this workflow. Diff should not be on the hot path for opening a remote host.

## Constraints

- Older clients must remain compatible with newer daemons.
- The change must not depend on host-level tuning, file deletion, or proxy changes.
- The daemon must not keep attempting diff refresh when the host has already reported watcher exhaustion.
- Existing per-workspace tab memory should keep working.

## Non-Goals

- No redesign of checkout status, PR status, or workspace registration.
- No new required WebSocket fields.
- No change to how diff is rendered once it is successfully loaded.

## Design

### Default tab behavior

Git workspaces should default to the `Files` tab instead of `Changes`. Existing remembered tab choices stay authoritative, so users who already picked `Changes` for a specific checkout keep that behavior. This change removes checkout diff from the default open flow without adding a new control. Clicking the `Changes` tab is already an explicit diff action.

### Resource exhaustion handling

`WorkspaceGitServiceImpl` should treat watcher-capacity failures as a distinct internal condition. When `fs.watch()` fails with `ENOSPC`, or the platform reports the same “file watchers reached” condition through an equivalent error message, the Linux directory traversal should stop immediately instead of continuing through the remaining tree.

`requestWorkingTreeWatch()` should reject for that specific condition. `CheckoutDiffManager.subscribe()` should catch it and return an immediate payload with empty files plus a user-facing error message. The payload should keep the existing `CheckoutError` shape and use `code: "UNKNOWN"` so older clients continue to parse it. The user-facing message should make the state clear: diff is unavailable on this host because file watcher capacity is exhausted.

This design intentionally avoids the timed refresh fallback for this condition. The user request is not “try a slower diff mode.” The user request is “do not execute diff when the host lacks the required resources.”

### UI behavior

`GitDiffPane` already renders diff payload errors. The new daemon error can flow through the existing path with no schema break and no special transport logic. The pane should show the message and leave the rest of the host online. File browsing, session history, agent lists, and timelines stay untouched.

### Compatibility

The wire format stays backward-compatible. The daemon keeps sending the existing `CheckoutError` fields. The error `code` stays within the old enum by using `UNKNOWN`. Only the human-readable `message` changes. Older clients should continue to parse and display the error. Newer clients can also display it without any protocol migration.

## Verification

1. Update panel-store tests so git workspaces default to `Files`.
2. Add a workspace-git-service test that simulates `ENOSPC` and verifies traversal aborts early.
3. Add a checkout-diff-manager test that turns a watcher-capacity failure into a structured diff error payload.
4. Run targeted Vitest files for the touched areas.
5. Run `npm run typecheck`.
