# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run Vitest tests
npm run db:reset     # Reset SQLite database (destructive)
npx prisma studio    # Browse database in browser
```

**Windows note:** The `dev` script uses `NODE_OPTIONS` with `cross-env`. If it fails, ensure `cross-env` is installed (`npm install --save-dev cross-env`).

**Running a single test:**
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

## Architecture

This is an AI-powered React component generator. Users describe components via chat, Claude generates React code using tool calls, and the result is shown in a live preview.

### Request Flow

1. User message → `ChatContext` (`src/lib/contexts/chat-context.tsx`) → `POST /api/chat`
2. API route (`src/app/api/chat/route.ts`) streams a response from Claude (claude-haiku-4-5)
3. Claude calls tools to write files into a virtual file system:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create/patch file contents
   - `file_manager` (`src/lib/tools/file-manager.ts`) — create/delete files
4. On stream finish, the project (messages + virtual FS) is saved to SQLite via Prisma
5. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) updates the client-side virtual FS
6. The preview iframe renders `App.jsx` from the virtual FS

### Key Layers

| Layer | Location | Purpose |
|---|---|---|
| UI shell | `src/app/main-content.tsx` | 3-panel resizable layout (chat / file tree / editor + preview) |
| Chat | `src/components/chat/` | Message rendering, streaming state |
| Editor | `src/components/editor/` | Monaco editor + file tree |
| AI endpoint | `src/app/api/chat/route.ts` | Streams Claude, handles tool calls, saves project |
| LLM provider | `src/lib/provider.ts` | Wraps Anthropic SDK; falls back to mock if no API key |
| Virtual FS | `src/lib/file-system.ts` | In-memory file system synced from AI tool calls |
| Auth | `src/lib/auth.ts` | JWT sessions (jose) + bcrypt passwords, httpOnly cookies |
| Server actions | `src/actions/` | DB reads/writes for projects (called from RSC) |
| Database | `prisma/schema.prisma` | SQLite, two models: `User` and `Project` |

### Data Models

- **User**: `id`, `email`, `password` (bcrypt), timestamps
- **Project**: `id`, `name`, `userId`, `messages` (JSON — chat history), `data` (JSON — serialized virtual FS), timestamps

### Auth Flow

Middleware (`src/middleware.ts`) verifies JWT from cookie on all protected routes. Sign-in/up go through server actions that set an httpOnly secure cookie. The `use-auth` hook (`src/hooks/use-auth.ts`) manages client-side auth state.

### LLM Provider

`src/lib/provider.ts` checks for `ANTHROPIC_API_KEY` in `.env`. If absent, a mock provider returns placeholder responses — useful for UI development without API costs.

### System Prompt

`src/lib/prompts/generation.tsx` contains the system prompt that instructs Claude how to generate React components and use the file tools.

## Instructions Storage

Store all Claude instructions and preferences in this `CLAUDE.md` file, not in the auto-memory system.

## Coding Style

- Only comment complex or non-obvious code. Skip comments on self-evident logic.

## Environment

```
ANTHROPIC_API_KEY=   # Optional — app runs in mock mode if not set
```
