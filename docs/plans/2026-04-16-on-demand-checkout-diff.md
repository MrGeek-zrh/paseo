# On-Demand Checkout Diff Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Make checkout diff opt-in by default and return a fast, user-visible error when the host lacks file watcher capacity.

**Architecture:** The app side changes only the default explorer tab selection for git workspaces. The daemon side stops watcher traversal on watcher-capacity failures, lets `requestWorkingTreeWatch()` reject for that condition, and converts that failure into a normal checkout diff payload error so the existing UI can render it.

**Tech Stack:** TypeScript, Zustand, Expo app UI, Node daemon, Vitest

---

### Task 1: Default git workspaces to `Files`

**Files:**
- Modify: `packages/app/src/stores/explorer-tab-memory.ts`
- Modify: `packages/app/src/stores/panel-store.ts`
- Test: `packages/app/src/stores/panel-store.test.ts`

**Step 1: Write the failing test**

Change `packages/app/src/stores/panel-store.test.ts` so git workspaces expect `files` as the default tab, while non-git workspaces still expect `files`.

**Step 2: Run test to verify it fails**

Run: `npx vitest run packages/app/src/stores/panel-store.test.ts --bail=1`

Expected: the git default test fails because the current code still returns `changes`.

**Step 3: Write minimal implementation**

Update `resolveExplorerTabForCheckout()` in `packages/app/src/stores/explorer-tab-memory.ts` so git workspaces default to `files`. Update the store bootstrap default in `packages/app/src/stores/panel-store.ts` from `changes` to `files`.

**Step 4: Run test to verify it passes**

Run: `npx vitest run packages/app/src/stores/panel-store.test.ts --bail=1`

Expected: PASS

**Step 5: Commit**

```bash
git add packages/app/src/stores/explorer-tab-memory.ts packages/app/src/stores/panel-store.ts packages/app/src/stores/panel-store.test.ts
git commit -m "feat: default git explorer tab to files"
```

### Task 2: Stop watcher traversal on capacity exhaustion

**Files:**
- Modify: `packages/server/src/server/workspace-git-service.ts`
- Test: `packages/server/src/server/workspace-git-service.test.ts`

**Step 1: Write the failing test**

Add a test in `packages/server/src/server/workspace-git-service.test.ts` that forces Linux mode, simulates an `ENOSPC` watcher failure after a small number of directory watches, and asserts that `requestWorkingTreeWatch()` rejects instead of continuing through the rest of the tree.

**Step 2: Run test to verify it fails**

Run: `npx vitest run packages/server/src/server/workspace-git-service.test.ts --bail=1`

Expected: the new test fails because the current implementation keeps traversing and only logs warnings.

**Step 3: Write minimal implementation**

Add an internal watcher-capacity error path in `packages/server/src/server/workspace-git-service.ts`. Detect `ENOSPC` and equivalent watcher-capacity messages, stop traversal immediately, and rethrow the condition from `requestWorkingTreeWatch()`.

**Step 4: Run test to verify it passes**

Run: `npx vitest run packages/server/src/server/workspace-git-service.test.ts --bail=1`

Expected: PASS

**Step 5: Commit**

```bash
git add packages/server/src/server/workspace-git-service.ts packages/server/src/server/workspace-git-service.test.ts
git commit -m "fix: fail fast on checkout watcher exhaustion"
```

### Task 3: Return a structured diff error instead of hanging

**Files:**
- Modify: `packages/server/src/server/checkout-diff-manager.ts`
- Test: `packages/server/src/server/checkout-diff-manager.test.ts`

**Step 1: Write the failing test**

Add a test in `packages/server/src/server/checkout-diff-manager.test.ts` where `requestWorkingTreeWatch()` rejects with the watcher-capacity condition. Assert that `subscribe()` resolves with an initial payload containing `files: []` and a user-facing error message, plus a safe unsubscribe function.

**Step 2: Run test to verify it fails**

Run: `npx vitest run packages/server/src/server/checkout-diff-manager.test.ts --bail=1`

Expected: the new test fails because the current implementation lets the rejection escape.

**Step 3: Write minimal implementation**

Catch the watcher-capacity condition in `packages/server/src/server/checkout-diff-manager.ts` and convert it into the existing checkout diff payload error format. Keep `code` within the existing compatible set by using `UNKNOWN`.

**Step 4: Run test to verify it passes**

Run: `npx vitest run packages/server/src/server/checkout-diff-manager.test.ts --bail=1`

Expected: PASS

**Step 5: Commit**

```bash
git add packages/server/src/server/checkout-diff-manager.ts packages/server/src/server/checkout-diff-manager.test.ts
git commit -m "fix: surface checkout diff watcher exhaustion"
```

### Task 4: Verify the integrated behavior

**Files:**
- Modify: `docs/plans/2026-04-16-on-demand-checkout-diff-design.md`
- Modify: `docs/plans/2026-04-16-on-demand-checkout-diff.md`

**Step 1: Run targeted app and server tests**

Run: `npx vitest run packages/app/src/stores/panel-store.test.ts packages/server/src/server/workspace-git-service.test.ts packages/server/src/server/checkout-diff-manager.test.ts --bail=1`

Expected: PASS

**Step 2: Run typecheck**

Run: `npm run typecheck`

Expected: PASS

**Step 3: Review docs and touched files**

Confirm the behavior matches the approved design and no compatibility-sensitive schema field changed in a breaking way.

**Step 4: Commit**

```bash
git add docs/plans/2026-04-16-on-demand-checkout-diff-design.md docs/plans/2026-04-16-on-demand-checkout-diff.md
git commit -m "docs: record on-demand checkout diff design"
```
