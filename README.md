# @sudobility/whisperly_lib

Frontend business logic library for Whisperly. Provides Zustand stores for state
management and React manager hooks that sync [TanStack Query](https://tanstack.com/query)
data (from `@sudobility/whisperly_client`) into those stores. It sits between the API
client (`whisperly_client`) and the UI (`whisperly_app`), and is DOM-free so it runs on
both web and React Native.

- ESM-only (`"type": "module"`); ships `dist/index.js` + `dist/index.d.ts`.
- Package manager is **Bun**.
- License: BUSL-1.1.

## Installation

This package is published to **GitHub Packages** (not the public npm registry). The repo's
`.npmrc` already routes the `@sudobility` scope:

```
@sudobility:registry=https://npm.pkg.github.com
```

Consumers need a GitHub token with `read:packages` configured for that registry, then:

```bash
bun add @sudobility/whisperly_lib
```

### Peer dependencies

Install these in the consuming app (they are not bundled):

- `@sudobility/types` `^1.9.78`
- `@tanstack/react-query` `>=5.0.0`
- `react` `>=18.0.0`
- `zustand` `>=5.0.0`

`firebase` is required only if you use `useFirebaseAuth`.

## Usage

Every manager hook takes `baseUrl` plus a platform-agnostic `networkClient`
(`NetworkClient` from `@sudobility/types`) — the app constructs the `networkClient` and is
responsible for injecting auth (e.g. a Firebase ID token) into its requests. Managers
create a `WhisperlyClient` internally and keep it stable via `useMemo`.

```typescript
import { useProjectManager, useTranslationManager } from "@sudobility/whisperly_lib";

const { projects, isLoading, createProject } = useProjectManager({
  baseUrl,
  networkClient,
  entitySlug,
});

const { translate, isTranslating } = useTranslationManager({ baseUrl, networkClient });
const result = await translate({
  orgPath: "my-org",
  projectName: "my-project",
  request: { strings: ["Hello"], target_languages: ["es"] },
});
```

## API

### Manager Hooks

All managers take a config of `{ baseUrl, networkClient, ...scopeFields }`.

- `useProjectManager({ baseUrl, networkClient, entitySlug, autoFetch? })` — Project CRUD +
  API-key management (`createProject`, `updateProject`, `deleteProject`, `generateApiKey`,
  `deleteApiKey`, `selectProject`, `refetch`).
- `useProjectDetail({ baseUrl, networkClient, entitySlug, projectId })` — Single project
  detail.
- `useDictionaryManager({ baseUrl, networkClient, entitySlug, projectId })` — Dictionary
  CRUD, scoped per project.
- `useAnalyticsManager({ baseUrl, networkClient, entitySlug, startDate?, endDate?, projectId?, enabled? })`
  — Analytics with date/project filters.
- `useSettingsManager({ baseUrl, networkClient, userId })` — User settings read/update.
- `useLanguagesManager({ baseUrl, networkClient, entitySlug, projectId })` — Project +
  available languages (`projectLanguages` is a comma-separated string).
- `useTranslationManager({ baseUrl, networkClient, testMode? })` — Public translation
  mutation (no auth). Returns `{ translate, isTranslating, lastResponse, error, clearLastResponse }`.

### Zustand Stores

`useProjectStore`, `useDictionaryStore`, `useSettingsStore`, `useAnalyticsStore` — exported
with selectors (e.g. `selectProjects`, `selectAnalytics`). Call `resetAllStores()` to clear
all stores on logout.

### Hooks & Utils

- `useFirebaseAuth(auth)` — Subscribes to Firebase `onAuthStateChanged` for the given
  `Auth` instance; returns `{ user, loading, getIdToken }`.
- `resetAllStores()` — Resets every store (logout cleanup).
- `formatStoreError(error)` — Normalizes an error to a message string (prefixes
  `WhisperlyApiError` with its `[statusCode]`).

`WhisperlyClient`, `WhisperlyApiError`, and the common `@sudobility/whisperly_types` types
are re-exported from the package root for convenience.

## Development

```bash
bun install            # Install dependencies

bun run build          # tsc (tsconfig.build.json) + scripts/fix-esm-imports.mjs
bun run build:watch    # Build in watch mode
bun run typecheck      # tsc --noEmit
bun run test:run       # Run vitest once (alias: test:unit)
bun run lint           # ESLint over src
bun run lint:fix       # ESLint with --fix
bun run format         # Prettier write src
bun run format:check   # Prettier check
```

Tests use Vitest. Store/util tests run in the `node` environment; hook/manager tests opt
into `jsdom` via a `// @vitest-environment jsdom` directive at the top of the file.

The `fix-esm-imports.mjs` post-build step rewrites `tsc`'s extensionless relative imports
(emitted under `moduleResolution: "bundler"`) into fully-qualified `.js` specifiers so
Node's native ESM loader can resolve them.

## Related Packages

- `whisperly_types` — Shared type definitions
- `whisperly_client` — API client SDK + TanStack Query hooks
- `whisperly_api` — Backend API server
- `whisperly_app` — Web application consumer

## License

BUSL-1.1
