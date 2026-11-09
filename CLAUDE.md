# Genuivo Client

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

API client SDK for Genuivo with TanStack Query hooks.

**npm**: `@sudobility/genuivo_client` (public, BUSL-1.1)

## Tech Stack

- **Language**: TypeScript (strict mode)
- **Runtime**: Bun
- **Package Manager**: Bun (do not use npm/yarn/pnpm for installing dependencies)
- **Build**: TypeScript compiler (ESM)
- **Test**: Vitest
- **Data Fetching**: TanStack Query 5

## Project Structure

```
src/
├── index.ts                          # Main barrel exports (no subpath exports)
├── types.ts                          # QUERY_KEYS, FirebaseIdToken, DEFAULT_STALE/GC_TIME
├── types.test.ts
├── network/
│   ├── index.ts
│   ├── StarterClient.ts              # HTTP client class (DI-based)
│   ├── StarterClient.test.ts
│   └── client-singleton.ts           # initialize/get/reset StarterClient singleton
├── hooks/
│   ├── index.ts                      # Hook exports
│   ├── useChat.ts                    # Mutation hook for AI chat endpoint
│   ├── useHistories.ts               # Query + mutation hook for user histories
│   └── useHistoriesTotal.ts          # Query hook for global total (public)
└── utils/
    ├── index.ts
    ├── starter-helpers.ts            # createAuthHeaders, createHeaders, buildUrl, handleApiError
    └── starter-helpers.test.ts
```

## Commands

```bash
bun install            # Install deps (requires GitHub Packages read:packages auth)
bun run build          # tsc (tsconfig.build.json) + scripts/fix-esm-imports.mjs
bun run clean          # Remove dist/
bun run test:unit      # Run Vitest once (colocated *.test.ts). test:watch for watch
bun run typecheck      # tsc --noEmit
bun run lint           # ESLint over src/ (lint:fix to autofix)
bun run format         # Prettier write (format:check to verify)
bun run verify         # typecheck + test:unit + build (run before commit)
bun run prepublishOnly # Clean + build (runs on publish)
```

Use Bun only — do not use npm/yarn/pnpm. The configured test runner is Vitest
(`bun run test:unit`); `bun test` happens to pass but is not the project runner.

## Key Concepts

### StarterClient (class name, despite the `genuivo_client` package)

HTTP client class constructed with `{ baseUrl, networkClient }`. Uses dependency injection via the `NetworkClient` interface from `@sudobility/types` — no direct fetch calls. Methods: `getUser`, `getHistories`, `createHistory`, `updateHistory`, `deleteHistory`, `chat`, `getHistoriesTotal`. Each returns `BaseResponse<T>` and runs a defensive `validateResponse` shape check.

### Client singleton

`initializeStarterClient({ baseUrl, networkClient })` must be called once at app startup; hooks call `getStarterClient()` internally and throw if it is not initialized. `resetStarterClient()` is for tests.

### Hooks

- `useHistories(userId, token, options?)` — fetches user history list and provides `createHistory`, `updateHistory`, `deleteHistory` mutations with automatic query invalidation; disabled unless both `userId` and `token` are truthy
- `useHistoriesTotal(options?)` — fetches global total (public endpoint)
- `useChat(userId, token)` — mutation hook; `chat(request)` posts to the AI endpoint; `token` may be null (unauthenticated)

### QUERY_KEYS

Type-safe cache key factory for TanStack Query. Used internally by hooks and available for manual invalidation.

### Cache Settings

- `staleTime`: 5 minutes
- `gcTime`: 30 minutes

## Peer Dependencies

- `react` (>=18)
- `@tanstack/react-query` (>=5)
- `@sudobility/types` — NetworkClient interface, BaseResponse

## Related Projects

- **genuivo_types** — Shared type definitions; this project imports all API types (`History`, request/response types, `BaseResponse`)
- **genuivo_api** — Backend server that this client SDK communicates with over HTTP
- **genuivo_lib** — Business logic library that consumes this client's hooks and `GenuivoClient` class
- **genuivo_app** — Web frontend that uses this client transitively via genuivo_lib
- **genuivo_app_rn** — React Native app that uses this client via file: links

Dependency injection is central: `NetworkClient` interface is provided by the consumer, allowing different fetch implementations per platform (web vs React Native).

## Coding Patterns

- `QUERY_KEYS` factory in `src/types.ts` provides type-safe cache keys for TanStack Query -- always use it for query keys (note: emitted keys are namespaced `'starter'`)
- `StarterClient` class accepts `{ baseUrl, networkClient }` via constructor -- never use `fetch` directly inside this package
- Hooks wrap TanStack Query and resolve `StarterClient` via the `getStarterClient()` singleton -- not via a passed-in client/config
- `useHistories` combines query and mutations in a single hook — mutations automatically invalidate related queries after success
- Default `staleTime` is 5 minutes and `gcTime` is 30 minutes -- respect these defaults unless there is a specific reason to override
- Utility functions in `src/utils/starter-helpers.ts` handle auth headers (`createAuthHeaders` / `createHeaders`), URL construction (`buildUrl`), and API error handling (`handleApiError`)
- `FirebaseIdToken` must be passed to all protected endpoint calls for authentication

## Gotchas

- `NetworkClient` is dependency-injected -- never import or use `fetch` directly; all HTTP calls go through the injected `networkClient`
- Mutations auto-invalidate related queries -- adding a new mutation must include proper `onSuccess` invalidation to keep caches consistent
- `FirebaseIdToken` is required for all authenticated endpoints; omitting it will result in 401/403 errors from the API
- The `QUERY_KEYS` factory must be kept in sync with API route changes -- if a route path changes, update the corresponding key
- This is a published npm package (`@sudobility/genuivo_client`) -- breaking changes require version bumps and coordination with consumers
- Naming is inconsistent on purpose-for-now: package is `genuivo_client` but the class/singleton/helpers keep the `Starter*` / `starter-` prefix. Do not "fix" these in isolation — consumers import these exact names
- Hooks throw `StarterClient not initialized` if `initializeStarterClient()` was not called at startup
- Build emits extensionless ESM imports; `scripts/fix-esm-imports.mjs` (run by `bun run build`) rewrites them to `.js`. Don't bypass it or Node ESM consumers break
- `@sudobility` scope resolves to GitHub Packages (`.npmrc`); installs need a `read:packages` token in `~/.npmrc`

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
