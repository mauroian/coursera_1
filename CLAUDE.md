# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates code into a virtual file system that renders instantly in an iframe sandbox.

## Commands

```bash
npm run setup          # First-time: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server with Turbopack at localhost:3000
npm run build          # Production build
npm run lint           # ESLint
npm test               # Vitest (all tests)
npx vitest run src/path/to/file.test.ts  # Single test file
npm run db:reset       # Reset SQLite database (destructive)
npx prisma studio      # Browse database via GUI
```

Set `ANTHROPIC_API_KEY` in `.env` to enable real AI generation. Without it, a mock provider returns static code.

## Architecture

### Virtual File System

The core abstraction is `VirtualFileSystem` (`src/lib/file-system.ts`) — an in-memory tree of `FileNode` objects that never touches disk. The AI writes files here via two Vercel AI SDK tools:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view/create/edit files with string replacement
- `file_manager` (`src/lib/tools/file-manager.ts`) — delete/rename/list files

State flows: chat API reconstructs VFS from serialized JSON → AI tool calls mutate it → final state is serialized back and saved to the project's `data` column in SQLite.

### Live Preview

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders an iframe with `srcdoc`. The JSX transformer (`src/lib/transform/jsx-transformer.ts`) uses `@babel/standalone` to transpile JSX in-browser, constructs an ES module import map with blob URLs for each virtual file, and injects it into the iframe. No server round-trip for preview updates.

Entry point resolution order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → `/src/App.tsx` → first `.jsx`/`.tsx` found.

### Chat & State Management

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — provides VFS access to all client components; `refreshTrigger` counter causes preview to re-render on file changes
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK `useChat`, sends file state with every message, updates VFS from tool call results in the stream
- Chat API route (`src/app/api/chat/route.ts`) — uses `streamText` with `maxSteps: 40` for multi-turn tool use; saves final state to Prisma on finish

### Auth & Persistence

Custom JWT auth via `jose` (`src/lib/auth.ts`). Sessions stored in httpOnly cookies. Registered users get project persistence in SQLite via Prisma; anonymous users get local state only tracked via `src/lib/anon-work-tracker.ts`.

### Database

SQLite with Prisma. Schema in `prisma/schema.prisma` — two models: `User` (id, email, password) and `Project` (id, name, optional userId, `messages` JSON array, `data` serialized VFS JSON).

### Provider Abstraction

`src/lib/provider.ts` returns a mock or real Anthropic language model depending on whether `ANTHROPIC_API_KEY` is set. The mock provider prevents errors during development without an API key.

## Code Style

Use comments sparingly. Only comment complex code.

## Key Conventions

- App Router with server components by default; add `"use client"` only where needed
- Server actions live in `src/actions/`; mark files with `import "server-only"` where appropriate
- Tailwind CSS v4 (no `tailwind.config.js` — configured via `globals.css`)
- shadcn/ui components in `src/components/ui/`
- Tests use Vitest + Testing Library with jsdom; test files colocated in `__tests__/` subdirectories
