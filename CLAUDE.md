# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack (uses node-compat.cjs shim)
npm run dev:daemon   # Start dev server in background, logs written to logs.txt
npm run build        # Production build
npm run start        # Start production server

# Testing
npm run test         # Run all tests with Vitest
npx vitest run src/path/to/test.ts  # Run a single test file

# Linting
npm run lint         # ESLint

# Database
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database (destructive)
```

## Architecture

UIGen is an AI-powered React component generator with live preview. The user chats with Claude, which generates/modifies React component code that is instantly previewed in an iframe.

### Request Flow

1. User types in the chat → `ChatContext` sends messages to `/api/chat`
2. `/api/chat/route.ts` adds a system prompt and calls Anthropic Claude (or a mock provider if `ANTHROPIC_API_KEY` is unset)
3. Claude uses two tools to modify files: `str_replace_editor` (targeted edits) and `file_manager` (create/delete)
4. Tool calls update the `FileSystemContext` (in-memory virtual FS, never writes to disk)
5. `PreviewFrame` watches the file system and re-renders the live iframe preview via `jsx-transformer`, which uses `@babel/standalone` to transpile JSX in the browser

### Key Architectural Concepts

**Virtual File System** (`src/lib/file-system.ts`): All generated code lives in an in-memory `VirtualFileSystem`. It is serialized to JSON for database persistence. There is no disk I/O for generated files.

**In-browser Transpilation** (`src/lib/transform/jsx-transformer.ts`): The preview iframe receives a self-contained HTML blob. Babel runs in the browser to transpile JSX/TSX at preview time—no build step needed for previews.

**AI Tools** (`src/lib/tools/`): The chat API exposes two Vercel AI SDK tools to the model: `str_replace_editor` (string replace + view file) and `file_manager` (create/delete/list). These are the only way the model can change files.

**Authentication** (`src/lib/auth.ts`): JWT stored in HTTP-only cookies, 7-day expiry. Middleware at `src/middleware.ts` protects `/api/projects` and `/api/filesystem`. Projects work for anonymous users too—they're linked by `userId: null`. Anonymous work is tracked in `src/lib/anon-work-tracker.ts` so it can be claimed after sign-up.

**Provider abstraction** (`src/lib/provider.ts`): Returns Anthropic model (`claude-haiku-4-5`) if `ANTHROPIC_API_KEY` is set, otherwise returns a mock that generates static placeholder code.

### Directory Map

```
src/
  app/
    api/chat/route.ts       # Main AI chat endpoint
    [projectId]/page.tsx    # Project page (loads DB data)
    main-content.tsx        # Root layout: chat panel (35%) + preview/code panel (65%)
  lib/
    file-system.ts          # VirtualFileSystem class
    contexts/               # FileSystemContext, ChatContext
    tools/                  # str_replace_editor, file_manager (AI tools)
    transform/              # jsx-transformer (Babel-based preview renderer)
    prompts/                # System prompt for code generation
    provider.ts             # Anthropic / mock model provider
    auth.ts                 # JWT session helpers
  components/
    chat/                   # Chat UI + markdown renderer
    editor/                 # Monaco editor + file tree
    preview/                # PreviewFrame (iframe)
    auth/                   # SignInForm, SignUpForm, AuthDialog
    ui/                     # shadcn/ui primitives
  hooks/                    # use-auth (client-side auth state)
  actions/                  # Next.js server actions (DB CRUD for projects)
prisma/
  schema.prisma             # User + Project models (SQLite)
```

### Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json`).
