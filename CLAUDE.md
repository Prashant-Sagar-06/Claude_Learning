# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates the code, which is displayed in a Monaco editor and rendered live in an iframe via Babel transpilation.

Built with: Next.js 15, React 19, TypeScript, Tailwind CSS v4, Prisma (SQLite), Vercel AI SDK, shadcn/ui.

## Commands

```bash
npm run setup        # First-time setup: install deps + Prisma generate + migrate
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run all Vitest tests
npm run db:reset     # Reset database to initial state
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Architecture

### Data Flow

1. User sends a message → `ChatInterface` → `ChatProvider` (via `useChat`) → `POST /api/chat`
2. `/api/chat` streams a response from Claude (claude-haiku-4-5) using Vercel AI SDK with two tools: `str_replace_editor` and `file_manager`
3. Tool calls mutate the `VirtualFileSystem` (in-memory) and are sent back to the browser
4. The updated file system is reflected in `FileTree` + `CodeEditor` (Monaco)
5. `PreviewFrame` uses `jsx-transformer.ts` (Babel standalone) to transpile and render the component in an iframe via blob URLs + import maps
6. Project state (messages + file system) is persisted to SQLite via Prisma on each API call

### Key Modules

- **`src/lib/file-system.ts`** — `VirtualFileSystem` class: in-memory file operations with serialization for persistence to DB
- **`src/lib/contexts/`** — `ChatProvider` and `FileSystemProvider` are the two main state managers; nearly all components consume these via `useChat` / `useFileSystem`
- **`src/app/api/chat/route.ts`** — The core AI endpoint; handles streaming, tool execution, and project persistence
- **`src/lib/tools/`** — Zod-validated tool definitions (`str-replace.ts`, `file-manager.ts`) that Claude calls to create/edit files
- **`src/lib/transform/jsx-transformer.ts`** — Transpiles JSX/TSX to browser-runnable JS; creates import maps and blob URLs for the preview iframe
- **`src/lib/provider.ts`** — Returns either the real Claude model or a `MockLanguageModel` (used when `ANTHROPIC_API_KEY` is absent)
- **`src/lib/auth.ts`** — JWT session management with jose; 7-day HttpOnly cookies
- **`src/actions/index.ts`** — Server actions for auth (signUp, signIn, signOut, getUser)
- **`src/lib/anon-work-tracker.ts`** — Saves anonymous session work to `sessionStorage` so it survives login/signup without data loss

### Routing

- `/` — Home; redirects authenticated users to their projects list
- `/[projectId]` — Project workspace (chat + editor + preview)
- `/api/chat` — Streaming AI endpoint
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem` routes with JWT verification

### Authentication

JWT-based with bcrypt-hashed passwords stored in SQLite. Sessions live in HttpOnly cookies. Projects can be created anonymously and optionally associated with a user account.

### AI Integration

The system prompt (`src/lib/prompts/generation.tsx`) instructs Claude to generate self-contained React components. Claude uses two tools:
- `str_replace_editor` — create/view/edit files with str_replace or insert operations
- `file_manager` — rename/delete files

Prompt caching (`experimental_providerMetadata` with `cacheControl: 'ephemeral'`) is applied to the system prompt for cost efficiency.
