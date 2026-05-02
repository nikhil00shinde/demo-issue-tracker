# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `bun install` — install dependencies
- `bun run dev` — start the Next.js dev server on http://localhost:3000
- `bun run build` / `bun run start` — production build and serve
- `bun run typecheck` — run `tsc --noEmit` (no test runner is configured)

## Architecture

Next.js 15 App Router + React 19 kanban board. Single-process, in-memory only — there is no database.

**State lives in `lib/store.ts`.** A singleton `IssueStore` class holds issues in a `Map` and is cached on `globalThis.__issueStore` so the instance survives Next.js dev hot-reloads. Data is seeded on construction and **resets on every server restart**. All mutations go through this store.

**Ordering model.** Each `Issue` has a `status` (`backlog | todo | in_progress | done` from `lib/types.ts`) and a numeric `order` within its column. `store.update` auto-assigns a new `order` when an issue moves columns without an explicit `order`. `store.reorder(status, orderedIds)` rewrites both `status` and `order` for every id in the list — this is the path used by drag-and-drop.

**API routes** (under `app/api/`) are thin wrappers around the store:
- `GET/POST /api/issues`
- `GET/PATCH/DELETE /api/issues/:id`
- `PUT /api/columns/:status/reorder` with body `{ orderedIds: string[] }`

**Client (`components/Board.tsx`)** uses `@dnd-kit/core` + `@dnd-kit/sortable` with `closestCorners` collision detection. Drag flow:
1. `onDragOver` optimistically moves an issue between columns by mutating local `status`/`order` (no network call).
2. `onDragEnd` reorders within the destination column using `arrayMove`, then `PUT`s the full `orderedIds` to `/api/columns/:status/reorder`.

When changing drag behavior, keep these two phases in sync — `handleDragOver` handles cross-column moves and `handleDragEnd` handles intra-column ordering plus the server sync.

**Path alias.** `@/*` resolves to the repo root (see `tsconfig.json`), e.g. `@/lib/types`.
