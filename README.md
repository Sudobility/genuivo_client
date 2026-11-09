# @sudobility/genuivo_client

API client SDK for the Genuivo backend, with TanStack Query hooks for React.

- **npm**: `@sudobility/genuivo_client` (public, BUSL-1.1)
- Dependency-injected HTTP (no direct `fetch`), works on web and React Native.

## Installation

The `@sudobility` scope is hosted on GitHub Packages (see `.npmrc`), so install
requires an authenticated GitHub token with `read:packages` in your personal
`~/.npmrc`.

```bash
bun add @sudobility/genuivo_client
```

Peer dependencies (provided by the consumer):

```bash
bun add react @tanstack/react-query @sudobility/types @sudobility/genui_types
```

- `@sudobility/types` — provides the `NetworkClient` interface and `BaseResponse`.
- `@sudobility/genui_types` — GenUI renderable types used by the chat endpoint.

## Usage

Everything is exported from the package root (there are no subpath exports).

```ts
import {
  StarterClient,
  initializeStarterClient,
  useHistories,
  useHistoriesTotal,
  useChat,
  QUERY_KEYS,
} from "@sudobility/genuivo_client";

// 1. Initialize the DI singleton ONCE at app startup.
//    `networkClient` is your platform's NetworkClient implementation.
initializeStarterClient({
  baseUrl: "https://api.example.com",
  networkClient,
});

// 2. Use the hooks in React components. They read the singleton internally.
const {
  histories,
  createHistory,
  updateHistory,
  deleteHistory,
  isLoading,
} = useHistories(userId, idToken);

const { total } = useHistoriesTotal(); // public endpoint, no auth
const { chat, isLoading: chatLoading } = useChat(userId, idToken);
```

You can also use `StarterClient` directly without hooks:

```ts
const client = new StarterClient({ baseUrl, networkClient });
const res = await client.getHistories(userId, idToken);
```

## API

### `StarterClient`

HTTP client class constructed with `{ baseUrl, networkClient }`. All calls go
through the injected `NetworkClient` — this package never calls `fetch` directly.
Every method returns a `BaseResponse<T>` and validates the response shape.

> Note: the npm package is `genuivo_client`, but the class is still named
> `StarterClient` (and helpers/singletons use the `Starter*` / `starter-` prefix).

Methods:

- `getUser(userId, token, options?)` — fetch a user profile (auth)
- `getHistories(userId, token, options?)` — list a user's histories (auth)
- `createHistory(userId, data, token, options?)` — create a history (auth)
- `updateHistory(userId, historyId, data, token, options?)` — update (auth)
- `deleteHistory(userId, historyId, token, options?)` — delete (auth)
- `chat(data, token?, options?)` — AI chat; token optional (enables web search)
- `getHistoriesTotal()` — global total (public, no auth)

`options.timeout` (ms) is accepted on all authed methods.

### Client singleton

- `initializeStarterClient({ baseUrl, networkClient })` — call once at startup
- `getStarterClient()` — returns the instance (throws if not initialized)
- `resetStarterClient()` — clears the instance (testing only)

The hooks require the singleton to be initialized.

### Hooks

- `useHistories(userId, token, options?)` — fetches the user's history list and
  provides `createHistory`, `updateHistory`, `deleteHistory` mutations with
  automatic cache invalidation. Returns loading/error/pending flags and
  `update()` (refetch). Query is disabled unless both `userId` and `token` are set.
- `useHistoriesTotal(options?)` — fetches the global total (public endpoint).
- `useChat(userId, token)` — mutation hook; `chat(request: string)` sends a
  message to the AI endpoint. `token` may be `null` (unauthenticated chat).

All hooks accept `options.enabled` where applicable and use the shared cache
defaults (`staleTime` 5 min, `gcTime` 30 min).

### Utilities

- `createAuthHeaders(token)` — JSON headers + `Authorization: Bearer <token>`
- `createHeaders()` — JSON headers only (for public endpoints)
- `buildUrl(base, path)` — URL construction (strips trailing slash from base)
- `handleApiError(response, operation)` — builds a descriptive `Error`

### Constants & types

- `QUERY_KEYS` — type-safe TanStack Query cache key factory
  (`histories(userId)`, `historiesTotal()`, `user(userId)`)
- `DEFAULT_STALE_TIME` (5 min), `DEFAULT_GC_TIME` (30 min)
- `FirebaseIdToken` — branded `string` alias for Firebase ID tokens

## Development

```bash
bun install            # Install deps (requires GitHub Packages auth)
bun run build          # Compile ESM to dist/ + fix-up extensionless imports
bun run build:watch    # tsc --watch
bun run clean          # Remove dist/
bun run typecheck      # tsc --noEmit
bun run lint           # ESLint over src/
bun run lint:fix       # ESLint --fix
bun run format         # Prettier write
bun run format:check   # Prettier check
bun run test:unit      # Run Vitest once (colocated *.test.ts files)
bun run test:watch     # Vitest watch mode
bun run verify         # typecheck + test:unit + build (run before commit)
```

Use **Bun** as the package manager (not npm/yarn/pnpm). The build runs
`scripts/fix-esm-imports.mjs` after `tsc` to rewrite extensionless relative
imports to explicit `.js` paths so Node's native ESM loader can resolve them.

## Related Packages

- **genuivo_types** — shared type definitions (`History`, request/response types,
  `BaseResponse`, GenUI types); imported for all API types
- **genuivo_api** — backend server this client communicates with over HTTP
- **genuivo_lib** — business logic library wrapping this client's hooks/class
- **genuivo_app** — web frontend (consumes transitively via genuivo_lib)
- **genuivo_app_rn** — React Native app (consumes via genuivo_lib)

Dependency injection is central: the `NetworkClient` interface is supplied by the
consumer, allowing a different fetch implementation per platform.

## License

BUSL-1.1 © Sudobility
