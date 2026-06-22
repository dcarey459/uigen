# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server (Turbopack) at localhost:3000
npm run dev:daemon     # Start in background, logs to logs.txt
npm run build          # Production build
npm run test           # Run all tests (Vitest)
npx vitest run src/path/to/file.test.ts  # Run a single test file
npm run db:reset       # Drop and recreate the database
npx prisma migrate dev # Apply schema changes to dev DB
```

**Do not run `npm audit fix`.** Dependencies are pinned intentionally; audit fix can break compatibility. Update pinned versions directly if a CVE needs addressing.

## Environment

Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. If the key is missing or still `your-api-key-here`, the app uses `MockLanguageModel` in `src/lib/provider.ts`, which returns canned components without calling Claude.

The real model is `claude-haiku-4-5` (set in `src/lib/provider.ts:8`).

## Architecture

### Request flow

1. User types a prompt → `ChatInterface` → POST `/api/chat` with messages + serialized VFS state
2. `src/app/api/chat/route.ts` reconstructs `VirtualFileSystem`, calls `streamText` (Vercel AI SDK) with two tools
3. Claude uses `str_replace_editor` (create/str_replace/insert commands) and `file_manager` (rename/delete) to write files into the VFS
4. The route streams the response; on finish, saves messages + VFS to Prisma if the user is authenticated
5. Client receives tool call events → `FileSystemContext.handleToolCall` applies them to the client-side VFS
6. `refreshTrigger` counter increments → `PreviewFrame` rebuilds and injects the preview iframe

### Virtual file system

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree of `FileNode`s. It lives on both server (reconstructed per request from JSON) and client (maintained in React context). Files are never written to disk. The VFS serializes to `Record<string, FileNode>` for the API request body and for Prisma storage (`Project.data`).

### Live preview pipeline

`PreviewFrame` calls `createImportMap` → `transformJSX` (Babel standalone, in-browser) per file, producing blob URLs. Third-party imports are resolved to `esm.sh`. Missing local imports get placeholder stub modules. The final HTML (with inline import map + Tailwind CDN) is set as `iframe.srcdoc`. Entry point discovery order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → first `.jsx`/`.tsx` found.

### Auth

JWT-based, implemented in `src/lib/auth.ts` with `jose`. Sessions stored in an `HttpOnly` cookie (`session`). Anonymous users can use the app; projects are only persisted for authenticated users. `src/middleware.ts` protects routes.

### Database

SQLite via Prisma. Two models: `User` (email + bcrypt password) and `Project` (messages as JSON string, VFS data as JSON string). Generated client lives in `src/generated/prisma/`. Schema is defined in `prisma/schema.prisma` — reference it anytime you need to understand the structure of the data stored in the database.

## Code Style

Use comments sparingly. Only comment complex code.

### Key files

| File | Role |
|---|---|
| `src/app/api/chat/route.ts` | Streaming AI endpoint; wires tools to VFS |
| `src/lib/provider.ts` | Selects real vs. mock Claude model |
| `src/lib/file-system.ts` | `VirtualFileSystem` class |
| `src/lib/transform/jsx-transformer.ts` | Babel transform + import map + preview HTML generation |
| `src/components/preview/PreviewFrame.tsx` | Iframe-based live preview |
| `src/lib/contexts/file-system-context.tsx` | Client-side VFS state + tool call handler |
| `src/lib/contexts/chat-context.tsx` | Chat message state + streaming |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude |
| `src/lib/tools/str-replace.ts` | `str_replace_editor` tool definition |
| `src/lib/tools/file-manager.ts` | `file_manager` tool definition |
