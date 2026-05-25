# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

UIGen — AI-powered React component generator. The user describes a component in chat; Claude (via the Vercel AI SDK) uses two tools to mutate an **in-memory `VirtualFileSystem`** (no real disk writes for generated code); the client transpiles the JSX with `@babel/standalone` and renders a live preview. Authenticated users get their project (VFS + chat messages) persisted to SQLite via Prisma; anonymous users can still generate.

Stack: Next.js 15 App Router (turbopack), React 19, Tailwind v4, Prisma + SQLite, `@ai-sdk/anthropic`, `ai` v4, jose (JWT), vitest + jsdom.

## Commands

```bash
npm run setup        # install + prisma generate + prisma migrate dev (run before first start)
npm run dev          # next dev --turbopack
npm run dev:daemon   # same, backgrounded, output to ./logs.txt — useful inside agent sessions
npm run build        # next build
npm run lint         # next lint
npm test             # vitest (jsdom)
npm run db:reset     # prisma migrate reset --force  (DESTRUCTIVE)
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`
Filter by name:        `npx vitest run -t "creates a file"`

## Environment (`.env`)

- `ANTHROPIC_API_KEY` — **optional**. When unset/empty, `getLanguageModel()` in `src/lib/provider.ts` returns a `MockLanguageModel` that streams a canned counter/form/card component (based on keywords in the prompt). Useful for UI work without burning tokens; do not assume real model output in tests.
- `JWT_SECRET` — optional; falls back to `"development-secret-key"` (see `src/lib/auth.ts`). Set in any non-dev environment.
- DB is SQLite at `prisma/dev.db`. **Prisma client is generated to `src/generated/prisma`**, not `@prisma/client` — import accordingly (see `src/lib/prisma.ts`).

## Architecture

Chat request flow:
1. Client `ChatInterface` (in `src/components/chat/`) drives the Vercel AI SDK `useChat` hook from `ChatContext` (`src/lib/contexts/chat-context.tsx`). The current `VirtualFileSystem` is serialized and sent in the request body alongside messages.
2. `POST /api/chat` (`src/app/api/chat/route.ts`) calls `streamText` with:
   - the system prompt from `src/lib/prompts/generation.tsx` (root entrypoint must be `/App.jsx`, imports use the `@/` alias for non-library files, Tailwind for styling),
   - `providerOptions: { anthropic: { cacheControl: { type: "ephemeral" } } }` — preserve this when editing the system message; it's the prompt-cache hook,
   - two tools: `str_replace_editor` (`src/lib/tools/str-replace.ts` — `view` / `create` / `str_replace` / `insert`) and `file_manager` (`src/lib/tools/file-manager.ts` — `rename` / `delete`). Both operate on the per-request `VirtualFileSystem` instance.
3. Tool calls stream back; `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) applies them, which re-renders both the file tree / Monaco editor (`src/components/editor/`) and `PreviewFrame` (`src/components/preview/`). The preview transpiles `/App.jsx` and its `@/`-prefixed imports with `@babel/standalone` client-side.
4. For authenticated sessions, `src/actions/` server actions persist `VFS.serialize()` to `Project.data` and the chat transcript to `Project.messages` — both are JSON strings on the `Project` model in `prisma/schema.prisma`.

Auth: JWT in an httpOnly `auth-token` cookie, 7-day expiry. Helpers in `src/lib/auth.ts`. `src/middleware.ts` 401s any unauthenticated request to `/api/projects` or `/api/filesystem` — public routes (including `/api/chat`) are not gated, which is intentional (anonymous generation works).

## Critical files

- `src/lib/file-system.ts` — `VirtualFileSystem` class. Its `serialize()` / `deserialize()` shape is the on-disk contract for `Project.data`; changing it without a migration breaks every saved project.
- `src/lib/prompts/generation.tsx` — system prompt that defines the contract the model must follow (entrypoint, `@/` alias, Tailwind).
- `src/lib/tools/str-replace.ts`, `src/lib/tools/file-manager.ts` — tool schemas the model sees. Renaming a tool or parameter requires updating the system prompt and the mock model in lockstep.
- `src/app/api/chat/route.ts` — wires the above together; the place to touch for model/temperature/tool changes.
- `src/lib/provider.ts` — real-vs-mock toggle and the exact model id (`claude-haiku-4-5`).
- `prisma/schema.prisma` — **canonical source of truth for DB structure; consult it whenever you need to understand stored data shape.** `Project.messages` and `Project.data` are `String` columns holding JSON. Migrations under `prisma/migrations/`.

## Conventions

- Tests live next to the code they cover under `__tests__/` directories, named `*.test.ts(x)`. Environment is jsdom (`vitest.config.mts`) — no extra setup file.
- The mock model in `src/lib/provider.ts` branches on the **tool-message count** in the prompt to decide which step to emit. If you change the tool surface, update its branches too or mock mode will desync.
- Server-only code is marked with `import "server-only"` (e.g. `src/lib/auth.ts`). Don't import these from client components.
- Use comments sparingly. Only comment complex / non-obvious code (hidden constraints, subtle invariants, workarounds). Do not narrate what the code does — names should carry that.
