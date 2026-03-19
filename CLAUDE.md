# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Setup (first time)
npm run setup          # installs deps, generates Prisma client, runs migrations

# Development
npm run dev            # Next.js dev server with Turbopack
npm run build          # Production build
npm run lint           # ESLint

# Testing
npm test               # Run all tests with Vitest
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx  # Run a single test file

# Database
npx prisma studio      # Open Prisma DB browser
npm run db:reset       # Reset DB (destructive)
npx prisma migrate dev # Apply pending migrations
```

## Environment

Create a `.env` file with:
```
ANTHROPIC_API_KEY=...
```
Without it, the app falls back to a `MockLanguageModel` that returns static code.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude generates/edits files in a virtual file system; a live preview iframe renders the result.

### Request Flow

1. User sends a message in `ChatInterface.tsx`
2. `ChatProvider` (via `chat-context.tsx`) POSTs to `/api/chat` with the current messages and serialized virtual file system
3. The API route streams a response using Vercel AI SDK's `streamText()` with two tools:
   - `str_replace_editor` — create, read, view, str_replace, insert operations on virtual files
   - `file_manager` — rename/delete files
4. Tool calls update the virtual file system in `FileSystemProvider`
5. On stream completion, the updated files and messages are saved to the DB (if authenticated)
6. `PreviewFrame.tsx` watches the file system, transpiles JSX via Babel standalone, and renders in a sandboxed iframe using esm.sh CDN for React

### Key Directories

- `src/app/` — Next.js App Router pages and `/api/chat` route
- `src/actions/` — Server actions: auth (`index.ts`) and project CRUD
- `src/components/chat/` — Chat UI (ChatInterface, MessageList, MessageInput)
- `src/components/editor/` — Monaco-based code editor and FileTree
- `src/components/preview/` — Iframe live preview
- `src/lib/contexts/` — React Context providers: `FileSystemProvider` and `ChatProvider`
- `src/lib/tools/` — AI tool implementations (`str-replace.ts`, `file-manager.ts`)
- `src/lib/transform/` — Babel-based JSX/TS transpilation for preview
- `src/lib/prompts/` — System prompt for Claude (`generation.tsx`)
- `prisma/` — Prisma schema (SQLite), migrations, and `dev.db`

### Authentication

JWT sessions via `jose` stored in HttpOnly cookies (7-day expiration). Server actions in `src/actions/index.ts` handle sign up/in/out. Passwords are bcrypt-hashed. Anonymous users get LocalStorage-backed state; on sign-in, their work is migrated into a new project (`anon-work-tracker.ts`).

### Data Models

```prisma
User   { id, email, password, projects[] }
Project { id, name, userId?, messages (JSON string), data (JSON string) }
```

`messages` and `data` are serialized JSON stored as strings — `data` is the virtual file system snapshot.

### AI Provider

`src/lib/provider.ts` returns a Claude Haiku 4.5 model (`claude-haiku-4-5`) when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel`. The system prompt uses Anthropic cache control for efficiency.

### Testing

Vitest with React Testing Library, JSDOM environment. Tests live alongside source in `__tests__/` subdirectories.
