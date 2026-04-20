# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Dev server with Turbopack at localhost:3000
npm run dev:daemon  # Dev server in background (logs to logs.txt)
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Vitest (all tests)
npx vitest run src/lib/__tests__/file-system.test.ts  # Single test file
npm run db:reset    # Force-reset Prisma migrations
```

## Environment

- `ANTHROPIC_API_KEY` — optional; falls back to mock provider if absent
- `JWT_SECRET` — optional; defaults to `"development-secret-key"`

## Architecture

**UIGen** is a Next.js 15 App Router app where users describe React components in natural language and Claude generates them live.

### Request flow

1. User message → `ChatInterface` → `MessageInput`
2. POST `/api/chat` (streaming) with message history + serialized virtual file system
3. Claude streams a response using Vercel AI SDK; it calls two tools:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — surgical edits to files
   - `file_manager` (`src/lib/tools/file-manager.ts`) — create/rename/delete files
4. Virtual file system updates propagate via `FileSystemContext`
5. On finish, if user is authenticated and a `projectId` exists, messages + FS state are persisted to SQLite via server actions

### Virtual file system (`src/lib/file-system.ts`)

In-memory only — no disk writes. Serializes to/from JSON for database storage. This is the source of truth for generated code while a session is active.

### AI generation (`src/app/api/chat/route.ts`, `src/lib/prompts/generation.tsx`)

- Model: Claude Haiku 4.5 via `@ai-sdk/anthropic`
- System prompt instructs Claude to generate `/App.jsx` as the entry point and use Tailwind CSS for all styling (no inline styles)
- Prompt caching enabled via Anthropic ephemeral cache control on the system message
- `src/lib/provider.ts` manages model selection and falls back to a mock provider when no API key is set

### Authentication

JWT sessions (7-day TTL) stored in `httpOnly` secure cookies. `src/middleware.ts` guards `/api/projects` and `/api/filesystem`. Server actions (`src/actions/`) re-validate the session before any DB operation. Anonymous usage is supported — projects can exist without a `userId`.

### Database

SQLite via Prisma. Two models: `User` and `Project`. Projects store `messages` and `data` (file system) as JSON strings. Schema at `prisma/schema.prisma`.

### UI layout

Split-panel: chat on the left, Monaco code editor + live preview on the right (`react-resizable-panels`). `FileSystemContext` and `ChatContext` wire state across the panel boundary. shadcn/ui (Radix + Tailwind, New York style) provides the component library.

### JSX transformation (`src/lib/transform/jsx-transformer.ts`)

Custom parser/validator for Claude-generated JSX before it reaches the live preview.
