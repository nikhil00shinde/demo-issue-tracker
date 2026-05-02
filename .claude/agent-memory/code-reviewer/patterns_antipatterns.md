---
name: Recurring patterns and anti-patterns
description: Known issues, conventions, and things to flag in future reviews of this codebase
type: feedback
---

## Conventions to enforce

- Always use `@/` path aliases (not relative paths) for cross-directory imports. Established in tsconfig.json.
- API routes must be thin wrappers — no business logic. Logic belongs in IssueStore methods.
- Client components must never import from `lib/store.ts` directly; only through API routes.
- Validate `status` against the `Status` union type before passing to store methods in API routes.
- `handleDragOver` and `handleDragEnd` in Board.tsx must always be reviewed together — they share state timing dependencies.

## Known issues found in initial review (2026-05-02)

### Critical / High
- `handleDragEnd` (Board.tsx:106) reads `byStatus[to]` which is a stale `useMemo` snapshot — it does NOT reflect the optimistic state update from `handleDragOver`'s `setIssues`. Cross-column drops will always send the pre-move column order to the server, corrupting drag results. Fix: derive the post-move column from the `issues` state directly inside `handleDragEnd`, not from `byStatus`.
- `app/api/issues/[id]/route.ts PATCH`: passes raw `body` directly to `store.update()` with no field validation. Allows callers to patch any field including `id`, `createdAt`, and `order` — fields that should be immutable or only mutated through controlled paths. Fix: whitelist only `title`, `description`, `status`, `order`.
- `app/api/columns/[status]/reorder/route.ts`: does not validate that `status` is a valid `Status` value before casting with `as Status` and passing to `store.reorder`. An arbitrary string can corrupt store state. Fix: check against the `STATUSES` array or a Set before casting.

### Major
- `app/api/issues/route.ts POST`: passes full `body` to `store.create()` instead of destructuring only known fields. Allows injection of `id`, `order`, `createdAt` from the request body, bypassing store construction logic.
- `handleDragEnd` (Board.tsx:120): no error handling on the `await api(...)` call. If the PUT fails, local state has already been updated optimistically and is now permanently desynced from server. Fix: wrap in try/catch and revert local state on failure (or refetch).
- `useEffect` in Board.tsx (line 39): fetch errors are silently swallowed — `.then()` with no `.catch()` and loading never gets set to false on failure, leaving the board in a permanent loading state.
- `store.reorder` does not validate that the provided IDs actually belong to the given `status` column. You can silently move issues between columns on the server by listing foreign IDs in the `orderedIds` array.

### Minor / Suggestions
- `handleStatusChange` (Board.tsx:75-81): performs a network call but has no error handling. On failure the local state is not reverted.
- `store.list()` sorts all issues globally by `order` — but `order` is only meaningful within a column (it's column-local). This works incidentally for the current seed data but could return confusing orderings if two issues in different columns share the same `order` value.
- `IssueCard` select onChange casts `e.target.value as Status` without runtime validation. A tampered DOM value would pass through unchecked (low risk in practice).
- `Board.tsx` `handleAdd` does not handle errors from the POST API call.
- The `api<T>` helper swallows HTTP error details — it only surfaces method+path, not the server's error message body.

**Why:** recorded to avoid re-discovering these in future review sessions and to track whether they've been fixed.
**How to apply:** when reviewing future PRs, check if the stale-byStatus bug was fixed, and whether PATCH/PUT routes now validate inputs properly.
