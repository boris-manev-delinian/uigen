# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Install deps, generate Prisma client, run migrations (first-time setup)
npm run dev          # Start dev server (Next.js + Turbopack)
npm run dev:daemon   # Run dev server in background (logs to logs.txt)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (tests live in src/components/**/__tests__/)
npm run db:reset     # Reset SQLite database (dev only)
```

All npm scripts inject `NODE_OPTIONS='--require ./node-compat.cjs'` — this is required to fix Node 25+ Web Storage SSR compatibility issues (the shim deletes global `localStorage`/`sessionStorage` on the server). Do not remove it from scripts.

Environment: create a `.env` file with:
- `ANTHROPIC_API_KEY=...` — without it, the app falls back to a static mock response
- `JWT_SECRET=...` — optional; defaults to `"development-secret-key"` (use a real secret in production)

## Architecture

UIGen is a Next.js 15 (App Router) application that uses Claude to generate React components in a browser-based IDE with live preview.

### Core Data Flow

1. User types in **ChatInterface** → POST to `/api/chat`
2. API route streams Claude responses via Vercel AI SDK (`@ai-sdk/anthropic`)
3. Claude calls tools (`str_replace_editor`, `file_manager`) to mutate the **VirtualFileSystem**
4. File changes are streamed back and applied to **FileSystemContext** on the client
5. **PreviewFrame** compiles JSX in an iframe using Babel Standalone + import maps and re-renders on changes
6. Projects (messages + file system state) are persisted to SQLite via Prisma server actions

### Key Directories

- `src/app/` — Next.js App Router pages and `/api/chat` streaming endpoint
- `src/components/` — UI split into `chat/`, `editor/`, `preview/`, `ui/` (shadcn), `auth/`
- `src/lib/` — Core logic:
  - `file-system.ts` — In-memory virtual FS (no disk writes); serializable for persistence
  - `provider.ts` — Claude Haiku model (or mock fallback)
  - `auth.ts` — JWT sessions (HTTP-only cookies, 7-day expiry, `jose`)
  - `prisma.ts` — Prisma client singleton
  - `contexts/` — `ChatContext` and `FileSystemContext` (client state)
  - `tools/` — AI tool definitions: `str-replace.ts`, `file-manager.ts`
  - `prompts/generation.tsx` — System prompt for component generation
  - `transform/jsx-transformer.ts` — JSX → preview-ready HTML
- `src/actions/` — Next.js server actions for auth and project CRUD
- `prisma/schema.prisma` — SQLite schema: `User` (email, bcrypt password) + `Project` (messages/data as JSON)

### Generation Constraints (enforced by system prompt)

- Entry point must be `/App.jsx`
- Tailwind CSS only — no inline styles, no CSS modules
- Import aliases use `@/` prefix within the virtual FS
- Claude model: `claude-haiku-4-5-20251001` (cost-efficient for fast generation)

### API Route (`/api/chat`)

- `maxDuration: 120` — 2-minute serverless timeout
- `maxSteps: 40` for real API, `4` for mock (controls multi-turn tool use loops)
- Prompt caching is active via `providerOptions.anthropic.cacheControl: { type: 'ephemeral' }` on the system prompt
- The serialized VirtualFileSystem is sent with every request in the `files` field and deserialized server-side; the updated FS is returned in the stream for client reconciliation

### Preview Frame

- **Entry point detection order**: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → `/src/App.tsx` → first `.jsx/.tsx` found
- Local files are Babel-compiled to blob URLs and injected via an import map
- Third-party packages (React, etc.) are resolved via `https://esm.sh/{package}` CDN
- CSS imports are silently stripped from JS/TS files and collected into a `<style>` tag in the iframe; they do not cause errors but won't work if referenced in JS logic
- Files with Babel syntax errors are skipped (reported in the preview, not thrown)

### Tools

- `str_replace_editor` — commands: `view`, `create`, `str_replace`, `insert`. The `undo_edit` command is defined but explicitly unsupported (returns an error).
- `file_manager` — commands: `rename`, `delete`

### Mock Provider (`src/lib/provider.ts`)

When `ANTHROPIC_API_KEY` is absent, `MockLanguageModel` runs a fixed 4-step sequence: create component file → enhance it → create `App.jsx` → emit summary. Component template is chosen by keyword matching the prompt (`"form"`, `"card"`, `"counter"`). Useful for UI development without spending API credits.

### Auth

JWT tokens stored in HTTP-only cookies (`auth-token`). Anonymous users can create projects (no `userId`). Middleware in `src/middleware.ts` gates `/api/projects` and `/api/filesystem`. On sign-in, anonymous work stored in `sessionStorage` is auto-saved as a new project.

### Database

Prisma with SQLite (`prisma/dev.db`). Client output goes to `src/generated/prisma`. Always read `prisma/schema.prisma` before writing any DB-related code.

### Testing

Vitest with jsdom. Run a single test file: `npx vitest run src/components/path/to/__tests__/file.test.tsx`.
