# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server at localhost:3000 (uses turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run tests with Vitest
npx prisma studio    # Browse database
npm run db:reset     # Reset and re-run all migrations
```

Note: All scripts prepend `NODE_OPTIONS='--require ./node-compat.cjs'` for Babel standalone compatibility.

## Environment

Requires `ANTHROPIC_API_KEY` in `.env.local`. Without it, the app falls back to a `MockLanguageModel` that returns static placeholder responses.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat; Claude generates code; a live preview renders the result in a sandboxed iframe — all without writing files to disk.

### AI Chat Flow (`/api/chat`)

`src/app/api/chat/route.ts` is the core endpoint. It:
1. Receives the conversation and current virtual file system state
2. Streams a response from Claude (via `@ai-sdk/anthropic`) with tool-calling enabled
3. Uses Anthropic prompt caching (ephemeral cache) on the system prompt for efficiency
4. Falls back to `MockLanguageModel` (`src/lib/provider.ts`) if no API key is set

The system prompt is in `src/lib/prompts/generation.tsx`.

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` class manages an in-memory Map-based file tree. Files never touch disk during generation. The state serializes to JSON for persistence and is passed to/from the API on each request.

`src/lib/contexts/file-system-context.tsx` — React context wrapping the VFS with UI-facing operations.

### AI Tools

The AI uses two tools to modify files:
- `src/lib/tools/str-replace.ts` — `str_replace_editor`: text-editor-style operations (view, create, str_replace, insert, delete)
- `src/lib/tools/file-manager.ts` — `file_manager`: higher-level file operations (rename, delete, move)

### Live Preview

`src/components/preview/PreviewFrame.tsx` renders an iframe. The transform pipeline:
1. `src/lib/transform/jsx-transformer.ts` — transforms JSX → JavaScript using `@babel/standalone` (runs in the browser)
2. Import map resolves `react`, `react-dom`, and local virtual file imports to esm.sh CDN URLs or Blob URLs
3. CSS files are collected and injected as `<style>` tags into the iframe

### Authentication & Persistence

- JWT sessions via `jose`, stored as HTTP-only cookies (`src/lib/auth.ts`)
- Passwords hashed with `bcrypt`
- Prisma + SQLite (`prisma/schema.prisma`): `User` and `Project` models; `Project.messages` and `Project.data` are JSON strings
- Server Actions in `src/actions/index.ts` handle sign-up, sign-in, sign-out, and project CRUD

### State Management

Two primary React contexts:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — VFS state and operations
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — AI conversation state via Vercel AI SDK `useChat`

### UI Layout

`src/app/main-content.tsx` — root layout with `react-resizable-panels`: left panel is chat, right panel toggles between live preview and Monaco code editor. File tree sits above the editor.
