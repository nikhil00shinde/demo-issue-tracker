---
name: Project architecture snapshot
description: Key architectural facts about the demo-issue-tracker codebase that inform review standards
type: project
---

Next.js 15 App Router + React 19 kanban board. Single-process, in-memory only — no database, no persistence across server restarts.

**State:** `lib/store.ts` — singleton `IssueStore` cached on `globalThis.__issueStore` to survive dev hot-reloads. Seeded with 3 issues on construction. Production intentionally does NOT cache on globalThis (store resets on restart is expected behavior).

**Ordering model:** Each `Issue` has `status: Status` and numeric `order` within its column. `store.reorder(status, orderedIds)` is the drag-and-drop path (rewrites both status and order for all ids). `store.update` auto-assigns order when status changes and no explicit order is given.

**API routes** (`app/api/`):
- `GET/POST /api/issues` — list all, create one
- `GET/PATCH/DELETE /api/issues/[id]` — single-issue operations
- `PUT /api/columns/[status]/reorder` — drag-and-drop reorder path, body: `{ orderedIds: string[] }`

**Client drag flow** (`components/Board.tsx`):
- `handleDragOver` — optimistic cross-column move (updates local state only, no network)
- `handleDragEnd` — reads `byStatus[to]` (snapshot from before DragOver's setState settles), calls arrayMove for intra-column, then PUTs to reorder API
- These two handlers MUST be reviewed together; they share subtle state timing dependencies

**Types** (`lib/types.ts`): `Status = "backlog" | "todo" | "in_progress" | "done"`, `Issue` interface, `STATUSES` array with key+label pairs.

**Path alias:** `@/*` → repo root (tsconfig.json). All imports should use `@/lib/...`, `@/components/...` etc.

**Stack:** bun, TypeScript strict mode, @dnd-kit/core ^6.1.0, @dnd-kit/sortable ^8.0.0. No test runner configured — typecheck via `bun run typecheck`.
