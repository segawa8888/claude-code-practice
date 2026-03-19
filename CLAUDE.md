# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start Next.js dev server (Turbopack)
npm run build        # Build for production
npm run lint         # Run ESLint
npm run test         # Run tests with Vitest
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database to initial state
```

Run a single test file:
```bash
npx vitest run path/to/test.ts
```

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates and edits them via tool calls against a virtual file system.

### Core Flow

1. User sends a chat message → `/api/chat` route
2. `streamText` (Vercel AI SDK) calls Claude with `str_replace_editor` and `file_manager` tools
3. AI tool calls mutate the **virtual file system** (in-memory, never touches disk)
4. The preview iframe picks up file changes and re-renders using Babel (client-side JSX transform) + esm.sh CDN for imports

### Key Directories

- `src/app/` — Next.js App Router pages. `[projectId]/` is the main workspace route.
- `src/app/api/chat/` — AI streaming endpoint using Vercel AI SDK `streamText`.
- `src/lib/` — Core logic:
  - `file-system.ts` — `VirtualFileSystem` class (in-memory tree, serializable)
  - `contexts/` — `FileSystemContext` and `ChatContext` (React state management)
  - `tools/` — AI tool definitions (`str_replace_editor`, `file_manager`) that operate on the virtual FS
  - `transform/` — Babel JSX transformer + esm.sh import map generator for browser preview
  - `prompts/` — System prompts for Claude
  - `provider.ts` — Returns Anthropic model or `MockLanguageModel` (used when no API key)
  - `auth.ts` — JWT session management (jose + httpOnly cookies, 7-day expiry)
- `src/actions/` — Next.js server actions for project CRUD
- `src/components/preview/` — `PreviewFrame` renders components in an isolated iframe

### State Management

Two React contexts handle all state:
- **FileSystemContext**: Virtual file system state + methods (create/update/delete files)
- **ChatContext**: Chat messages + AI interaction via Vercel AI SDK `useChat`

### Authentication & Data

- JWT-based auth stored in httpOnly cookies
- **Dual mode**: authenticated users persist projects to SQLite (Prisma); anonymous users work in-memory
- `prisma/schema.prisma` defines `User` and `Project` models; `Project.messages` and `Project.data` are JSON-stringified fields
- `middleware.ts` protects `/api/projects` and `/api/filesystem` routes

### Preview System

- `PreviewFrame` generates an HTML document injected into an iframe
- Babel runs client-side in the browser to transform JSX
- npm imports are resolved via esm.sh CDN through a dynamically generated import map
- Tailwind CSS loads via CDN in the preview iframe

### AI Integration

- Model: Anthropic Claude via `@ai-sdk/anthropic`
- `max_steps: 40` for real API, `4` for mock provider
- System prompt uses Anthropic cache control to reduce token costs
- When `ANTHROPIC_API_KEY` is unset, `MockLanguageModel` returns static examples (demo mode)

## Environment

`.env` file needs `ANTHROPIC_API_KEY=""` (optional — app works in demo mode without it).

Database: SQLite at `prisma/dev.db`.

## Tech Stack

- **Next.js 15** (App Router, Turbopack), **React 19**, **TypeScript 5**
- **Tailwind CSS v4**, **shadcn/ui** (Radix UI, new-york style, neutral base color)
- **Vercel AI SDK** (`ai` + `@ai-sdk/anthropic`)
- **Prisma 6** with SQLite
- **Vitest** + React Testing Library + jsdom
