# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This is a **Turbo monorepo** using **Bun workspaces**. Workspaces are `apps/*` and `packages/*`, with one exclusion: `apps/raycast-extension` is deliberately excluded (`"!apps/raycast-extension"` in the root `package.json`) and manages its own dependencies/tooling independently.

### Applications (`apps/`)
- **`web/`** - Next.js 16 (React 19) consumer app. Deployed to Cloudflare Workers via OpenNext (`@opennextjs/cloudflare` + `wrangler.jsonc`)
- **`mcp/`** - Standalone Model Context Protocol server on Cloudflare Workers with a Durable Object (`SupermemoryMCP`). Hono + `@modelcontextprotocol/sdk`, API-key and OAuth auth (`@cloudflare/workers-oauth-provider`). Served at `mcp.supermemory.ai`
- **`docs/`** - Mintlify documentation site
- **`browser-extension/`** - Browser extension built with WXT (React, Tailwind 4)
- **`memory-graph-playground/`** - Next.js demo app for the `@supermemory/memory-graph` package
- **`raycast-extension/`** - Raycast extension (outside the workspace; uses npm + its own eslint config)

### Packages (`packages/`)
Published to npm (each has a `publish-*.yml` GitHub workflow):
- **`ai-sdk/`** - `@supermemory/ai-sdk`: Vercel AI SDK utilities for Supermemory
- **`tools/`** - `@supermemory/tools`: memory tools for AI SDK, OpenAI, Voltagent, Mastra
- **`memory-graph/`** - `@supermemory/memory-graph`: interactive graph visualization React component

Published to PyPI (each has a publish workflow):
- **`agent-framework-python/`**, **`openai-sdk-python/`**, **`pipecat-sdk-python/`**, **`cartesia-sdk-python/`**

Internal `@repo/*` packages (private, consumed directly as source — no build step):
- **`lib/`** - shared Better Auth client (`auth.ts`), typed API client for the backend (`api.ts`, built on `@better-fetch/fetch`), TanStack Query setup, PostHog/error-tracking helpers
- **`validation/`** - Zod schemas for backend API requests/responses
- **`ui/`** - shared UI components and global CSS
- **`hooks/`** - shared React hooks
- **`docs-test/`** - runner that executes/tests the code snippets in the docs (`bun run test`, `test:ts`, `test:py`, ...)

Other notable top-level items:
- **`skills/supermemory/`** - an agent skill (SKILL.md + references) describing the Supermemory platform APIs
- **`portless.json`** - local dev-domain proxy config (see Development Commands)

## The backend API is NOT in this repo

The `/v3/*` REST API (documents, search, connections, settings, analytics), the Better Auth **server**, the database, and the content-ingestion pipeline all live in a **separate codebase**, deployed at `https://api.supermemory.ai`. This repo contains only **clients** of that API:
- `apps/web` talks to it via `packages/lib/api.ts` (base URL from `NEXT_PUBLIC_BACKEND_URL`, default `https://api.supermemory.ai`)
- `apps/mcp` proxies to it via its `API_URL` var
- the browser/raycast extensions and playground call it directly

Consequently there is no Drizzle schema, no migrations, no Hyperdrive/KV/Workflows bindings, and no cron triggers anywhere in this repo. **Gotcha:** the root `package.json` still lists backend-flavored dependencies (`drizzle-orm`, `drizzle-kit`, `pg`, `postgres`, `hono-openapi`, `@hono/zod-validator`, various AI SDKs, ...) that no source file here imports — do not treat root dependencies as evidence of in-repo usage. The only in-repo Hono app is `apps/mcp`; the only Better Auth usage is client-side (`better-auth/react`, `better-auth/cookies`).

## Development Commands

**Package manager: Bun** (`packageManager: bun@1.3.6`, Node >= 20, lockfile `bun.lock`). Install with `bun install`.

### Root Level (Monorepo)
- `bun run dev` - `turbo run dev`; for web/mcp/docs/playground the `dev` script wraps `dev:app` with **portless**, which serves the apps on local dev domains (`app.dev.supermemory`, `mcp.dev.supermemory`, `docs.dev.supermemory`, `graph.dev.supermemory` — see `portless.json`). The browser extension's `dev` runs WXT directly on port 3001
- `bun run dev:local` - `turbo run dev:app`; plain localhost ports instead (web 3000, docs 3003, playground 3004, mcp 8788; the browser extension has no `dev:app` and is skipped)
- `bun run build` - Build all workspaces via Turbo
- `bun run check-types` - `turbo run check-types`; only `ai-sdk`, `memory-graph`, and `tools` define this script — the apps are **not** type-checked by it
- `bun run format-lint` - `bunx biome check --write` (format + lint the whole repo)
- `bun run sentry:sourcemaps` - upload sourcemaps to Sentry (also runs as root `postbuild` and web `postdeploy`)

### Web Application (`apps/web/`)
- `bun run dev` / `bun run dev:app` - Next.js dev server (portless-wrapped / port 3000)
- `bun run build` - `next build`
- `bun run lint` - `biome check --write` (Biome, not `next lint`)
- `bun run preview` / `deploy` / `upload` - OpenNext build + Cloudflare preview/deploy/upload
- `bun run cf-typegen` - generate Cloudflare binding types (`CloudflareEnv`)

### MCP Server (`apps/mcp/`)
- `bun run dev:app` - `vite build && wrangler dev` (port 8788)
- `bun run deploy` - `vite build && wrangler deploy --minify`
- `bun run test:e2e` - Vitest e2e tests
- `bun run cf-typegen` - generate `CloudflareBindings` types

### Published TS packages (`packages/ai-sdk`, `tools`, `memory-graph`)
- `bun run build` (tsdown / vite), `bun run test` (vitest), `bun run check-types` (`tsc --noEmit`)

## Architecture Overview

### Core Technology Stack
- **Language**: TypeScript throughout (plus Python for the PyPI SDK packages)
- **Package Manager**: Bun; **Monorepo**: Turbo; **Lint/Format**: Biome
- **Web**: Next.js 16 App Router, React 19, Tailwind CSS 4, Radix UI, TanStack Query/Form/Table, Zustand, Tiptap, Recharts, Motion
- **Deployment**: Cloudflare Workers — OpenNext for `web`, Hono + Durable Objects for `mcp`
- **Auth**: Better Auth (client only; server lives in the backend repo)
- **Monitoring**: Sentry (`@sentry/nextjs`) and PostHog

### Web Application (`apps/web`)
- App Router with route groups `app/(app)` and `app/(auth)`; local API routes exist only for `emails`, `og`, and `onboarding` (`app/api/`) — all product data comes from the backend API
- `middleware.ts` is the auth proxy: it gates routes on the Better Auth session cookie and whitelists public paths (login, MCP setup view, guest-mode integrations). CONTRIBUTING.md documents the localhost bypass you need to add for local development
- Data layer: `packages/lib/api.ts` (better-fetch client validated with `@repo/validation` Zod schemas) + TanStack Query (`packages/lib/queries.ts`); client state in `apps/web/stores` (Zustand)
- `wrangler.jsonc` bindings: static assets, a worker self-reference service binding, and an R2 bucket for the Next.js incremental cache — nothing else

### MCP Server (`apps/mcp`)
- Hono router in `src/index.ts`; per-user MCP sessions in the `SupermemoryMCP` Durable Object (`src/server.ts`)
- Auth via Supermemory API keys or OAuth (`src/auth.ts`); telemetry via PostHog
- Vite (`vite-plugin-singlefile`) builds the setup UI into a single HTML file bundled with the Worker

## Code Quality & Standards

### Linting & Formatting
- **Biome** for linting and formatting (tabs, config in `biome.json` at the root)
- Run `bun run format-lint` before committing

### TypeScript
- Strict configuration based on `@total-typescript/tsconfig`
- `bun run check-types` only covers the three published TS packages; type-check apps with `tsc --noEmit` (or `bun run compile` in `browser-extension`) if you touch them
- `cf-typegen` scripts (web, mcp) regenerate Cloudflare binding types after editing `wrangler.jsonc`

### CI (`.github/workflows/`)
- `ci.yml` (on PRs): Bun 1.3.6 install, `turbo run check-types` filtered to `@supermemory/ai-sdk` and `@supermemory/memory-graph`, then `biome ci --changed --since=origin/main`
- `claude.yml`, `claude-code-review.yml`, `claude-auto-fix-ci.yml`: Claude automation
- `publish-*.yml`: per-package npm/PyPI release workflows

## Environment Configuration
- `apps/web/.env.example` is the only env template: `NEXT_PUBLIC_BACKEND_URL` (defaults to `https://api.supermemory.ai`), `NEXT_PUBLIC_POSTHOG_KEY`, `EXA_API_KEY`, `XAI_API_KEY`
- `apps/mcp/wrangler.jsonc` vars: `API_URL` (backend), custom domain route `mcp.supermemory.ai`

## Gotchas
- `apps/raycast-extension` is not a Bun workspace — don't expect root `bun install`/turbo tasks to cover it
- Root dependencies ≠ in-repo usage (see "The backend API is NOT in this repo" above)
- Internal `@repo/*` packages are consumed as raw source (`"exports": { "./*": "./*" }`) — there is no build step for them, so type errors surface in the consuming app
- Version skew is normal across workspaces (e.g. different `ai`/`@ai-sdk/*` majors at root vs `apps/web`); check the nearest `package.json`, not the root
- `apps/mcp` has a stray `pnpm-lock.yaml`, but Bun is the package manager for the workspace
