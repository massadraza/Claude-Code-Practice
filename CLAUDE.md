# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**UIGen** — AI-powered React component generator with live preview. Located in `uigen/`.

## Commands

Run all commands from `uigen/`:

```bash
npm run setup        # First-time setup: install deps + prisma generate + migrate
npm run dev          # Dev server (Turbopack) at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run Vitest tests
npm run db:reset     # Reset SQLite database (destructive)
```

To run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

Set `ANTHROPIC_API_KEY` in `.env` to use real AI generation; without it the app falls back to static mock responses.

## Architecture

### Key Technologies
- Next.js 15 App Router, React 19, TypeScript, Tailwind CSS v4
- Vercel AI SDK (`ai` package) + `@ai-sdk/anthropic` for streaming
- Prisma with SQLite (`prisma/dev.db`)
- JWT auth via `jose` stored in HTTP-only cookies

### Virtual File System
The core abstraction is `VirtualFileSystem` (`src/lib/file-system.ts`) — an in-memory tree of `FileNode` objects. Generated components are never written to disk. The `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) wraps this class in React state and handles incoming AI tool calls that mutate the FS.

### AI Integration & Tools
`src/app/api/chat/route.ts` is the streaming chat endpoint. It accepts serialized VFS state, reconstructs the `VirtualFileSystem`, and calls `streamText` with two AI tools:
- `str_replace_editor` — view/create/str_replace/insert operations on VFS files
- `file_manager` — rename/delete operations

The system prompt lives in `src/lib/prompts/generation.tsx`. Generated code must use `/App.jsx` as the entry point and `@/` as the import alias for local files.

### Live Preview Pipeline
`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders an `<iframe>`. On each VFS change, `jsx-transformer.ts` transpiles all `.jsx`/`.tsx` files in-browser using `@babel/standalone`, creates blob URLs for each module, and assembles an [import map](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap). Third-party npm packages are fetched from `esm.sh`. The iframe uses `srcdoc` to render the complete HTML.

### Auth & Persistence
`src/lib/auth.ts` (server-only) manages JWT sessions in HTTP-only cookies. Anonymous users can generate components without signing in — projects are only persisted to SQLite for authenticated users. The `Project` model stores messages and VFS data as JSON strings.

### Routing
- `/` — Redirects authenticated users to their most recent project; shows blank canvas for anonymous users
- `/[projectId]` — Main workspace with chat, code editor, and preview panels
- `/api/chat` — Streaming POST endpoint for AI generation

### Node Compatibility
`node-compat.cjs` is required by Next.js and loaded via `NODE_OPTIONS='--require ./node-compat.cjs'` in all npm scripts.
