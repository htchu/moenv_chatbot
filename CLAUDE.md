# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Next.js 16 AI chatbot application (v3.1.0) built with the Vercel AI SDK and Claude Sonnet. Features streaming chat, resumable streams via Redis, artifact creation/editing (text, code, sheets), guest/credentials auth, and document collaboration tools.

## Commands

- **Dev server**: `pnpm dev` (Next.js with Turbopack)
- **Build**: `pnpm build` (runs DB migration then Next.js build)
- **Lint**: `pnpm lint` (Biome via ultracite)
- **Format**: `pnpm format` (Biome auto-fix via ultracite)
- **Test all**: `pnpm test` (Playwright e2e + route tests, sets PLAYWRIGHT=True)
- **Test single file**: `pnpm exec playwright test tests/e2e/chat.test.ts`
- **Test single test**: `pnpm exec playwright test -g "test name"`
- **DB migrate**: `pnpm db:migrate`
- **DB generate**: `pnpm db:generate`
- **DB studio**: `pnpm db:studio`

## Architecture

### AI Pipeline

- **Models**: Configured in `lib/ai/models.ts` — uses `claude-sonnet-4-20250514` for chat, titles, and artifacts. Test environment uses `MockLanguageModelV2`.
- **Providers**: `lib/ai/providers.ts` — wraps `@ai-sdk/anthropic` via `customProvider`. Test env swaps in mock models from `lib/ai/models.mock.ts`.
- **Prompts**: `lib/ai/prompts.ts` — system prompts with geolocation hints. Reasoning model uses a different prompt (no artifacts).
- **Tools**: `lib/ai/tools/` — `create-document`, `update-document`, `get-weather`, `request-suggestions`. Tools stream artifact content in real-time.
- **Entitlements**: `lib/ai/entitlements.ts` — rate limiting and feature access by user type.

### Streaming & Data Flow

- Chat endpoint (`app/(chat)/api/chat/route.ts`) uses AI SDK `streamText` with `UIMessageStream`.
- Resumable streams backed by Redis (`resumable-stream` package).
- Client-side: `DataStreamProvider` + `DataStreamHandler` manage stream state and custom data types (code deltas, text deltas, etc.).
- Token usage tracked via `tokenlens` and stored in `Chat.lastContext`.

### Database

- **ORM**: Drizzle with PostgreSQL (`postgres` driver)
- **Schema**: `lib/db/schema.ts` — tables: User, Chat, Message_v2 (current), Message (deprecated), Vote_v2, Document, Suggestion, Stream
- **Migrations**: `lib/db/migrations/`
- **Queries**: `lib/db/queries.ts`

### Authentication

- NextAuth 5 beta with two credential providers: regular users and guest auto-creation.
- `auth.ts` + `auth.config.ts` handle session/user logic.
- `proxy.ts` middleware handles guest token validation, `/ping` health check, and auth redirects.
- User types: "guest" vs "regular" (affects entitlements).

### Artifact System

- `lib/artifacts/server.ts` defines `DocumentHandler` interface with `onCreateDocument`/`onUpdateDocument` callbacks.
- Artifact kinds: text (`artifacts/text/`), code (`artifacts/code/`), sheet (`artifacts/sheet/`).
- Code artifacts execute Python client-side via Pyodide.
- Text editing uses ProseMirror; code editing uses CodeMirror; sheets use react-data-grid.

### Error Handling

- `lib/errors.ts` — typed `ChatSDKError` system with `ErrorCode` format `${ErrorType}:${Surface}`.
- Errors have visibility levels controlling whether they're logged or returned to the client.

## Key Conventions

- **Package manager**: pnpm 9.12.3
- **Path alias**: `@/*` maps to project root
- **UI components**: shadcn/ui in `components/ui/` (excluded from linting)
- **Linting**: Biome via ultracite — no `any` types, no enums, no namespaces, no console usage
- **Testing**: Playwright with page object models in `tests/pages/`, mock AI responses in `tests/prompts/`. Two test projects: "e2e" and "routes".
- **Environment variables**: `AUTH_SECRET`, `ANTHROPIC_API_KEY`, `POSTGRES_URL`, `REDIS_URL`, `BLOB_READ_WRITE_TOKEN` (see `.env.example`)
