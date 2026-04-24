# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

UIGen is an AI-powered React component generator. Users describe a component in natural language; Claude generates it using tool calls, and the result renders live in an iframe. Supports both anonymous and authenticated (JWT) users with SQLite persistence.

## Commands

```bash
npm run dev          # Dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (all tests)
npm run setup        # Install deps + Prisma generate + migrate

# Database
npm run db:reset     # Reset SQLite database

# Single test
npx vitest run src/path/to/__tests__/file.test.ts
```

Requires `ANTHROPIC_API_KEY` in `.env`. Without it, the app falls back to a `MockLanguageModel` defined in `src/lib/provider.ts`.

## Architecture

**Stack:** Next.js 15 App Router, React 19, Vercel AI SDK + Anthropic, Prisma + SQLite, Tailwind v4 + shadcn UI, Monaco Editor, Vitest.

**Core data flow:**

1. User message → `MessageInput` → POST `/api/chat`
2. `/api/chat/route.ts` streams a response from Claude with tool use (`str_replace_editor`, `file_manager`)
3. Tool calls update an in-memory `VirtualFileSystem` instance (no disk writes)
4. Updated files flow back to the Monaco editor and the preview iframe via React context
5. On save/project creation, the serialized file system + messages are stored as JSON strings in the `Project` DB row

**Virtual File System (`src/lib/file-system.ts`):** The central abstraction. A `Map`-based in-memory FS that supports create/read/update/delete for files and directories, plus `serialize()`/`deserialize()` for database persistence. Every file operation in the AI tools goes through this class.

**AI Tools (`src/lib/tools/`):**
- `str-replace.ts` — `str_replace_editor` tool: create or patch files via string replacement
- `file-manager.ts` — `file_manager` tool: rename, delete, create directories

**React Contexts (`src/lib/contexts/`):**
- `ChatContext` — chat messages, streaming state, project ID
- `FileSystemContext` — current VirtualFileSystem instance, selected file

**Generation prompt (`src/lib/prompts/`):** System prompt instructing Claude to output JSX. Uses Anthropic cache control (`"cache_control": { "type": "ephemeral" }`) on the system message to reduce cost.

**JSX Transform (`src/lib/transform/jsx-transformer.ts`):** Transforms raw JSX from the AI into something the browser iframe can execute (Babel-based in-browser transform for preview).

**Auth (`src/lib/auth.ts`):** JWT sessions via `jose`. No OAuth — email/password only, bcrypt-hashed. Session cookie set on sign-in.

**Anonymous mode (`src/lib/anon-work-tracker.ts`):** Tracks usage limits for unauthenticated users. Projects with `userId: null` are anonymous.

## Database Schema

`User` (id, email, password, timestamps) ↔ one-to-many ↔ `Project` (id, name, userId?, messages JSON, data JSON, timestamps).

`messages` stores the full chat history as a JSON array. `data` stores the serialized `VirtualFileSystem` state. Both are plain strings in SQLite.

## Key File Locations

| Concern | Path |
|---|---|
| Chat API endpoint | `src/app/api/chat/route.ts` |
| Main page layout | `src/app/main-content.tsx` |
| Virtual FS | `src/lib/file-system.ts` |
| AI tool definitions | `src/lib/tools/` |
| System prompt | `src/lib/prompts/` |
| LLM provider setup | `src/lib/provider.ts` |
| Auth utilities | `src/lib/auth.ts` |
| Server actions | `src/actions/` |
| Preview iframe | `src/components/preview/PreviewFrame.tsx` |
| Monaco wrapper | `src/components/editor/CodeEditor.tsx` |
