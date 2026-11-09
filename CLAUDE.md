# CLAUDE.md - whisperly_lib

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

## Project Overview

`@sudobility/whisperly_lib` is the frontend business logic library for Whisperly. It provides Zustand stores for state management and manager hooks that sync TanStack Query data with local state. This library sits between the API client (`whisperly_client`) and the UI (`whisperly_app`).

**Platform**: Web + React Native (no DOM APIs). Tests use node environment.

## Package Manager

**Bun** (not npm/yarn): `bun install`, `bun run <script>`, `bun add <package>`

## Project Structure

```
src/
├── index.ts                          # Barrel export (all public API)
├── stores/
│   ├── index.ts                      # Stores barrel export
│   ├── projectStore.ts               # Projects state (flat array, selected ID)
│   ├── projectStore.test.ts
│   ├── dictionaryStore.ts            # Dictionary state (keyed by projectId)
│   ├── dictionaryStore.test.ts
│   ├── settingsStore.ts              # User settings state
│   ├── settingsStore.test.ts
│   ├── analyticsStore.ts             # Analytics state with filters
│   └── analyticsStore.test.ts
├── managers/
│   ├── index.ts                      # Managers barrel export
│   ├── useProjectManager.ts          # useProjectManager + useProjectDetail
│   ├── useDictionaryManager.ts       # Dictionary CRUD (entity+project scoped)
│   ├── useSettingsManager.ts         # Settings read/update (user-scoped)
│   ├── useAnalyticsManager.ts        # Analytics with date filters (entity-scoped)
│   ├── useTranslationManager.ts      # Translation mutation (public)
│   └── useLanguagesManager.ts        # Project + available languages (entity+project scoped)
├── hooks/
│   ├── index.ts
│   ├── useFirebaseAuth.ts            # Firebase auth state listener
│   └── useWhisperlyClient.ts         # Client instantiation helper (NOT re-exported from src/index.ts)
└── utils/
    ├── index.ts
    ├── resetAllStores.ts             # Reset all stores on logout
    └── formatStoreError.ts           # Error -> message string (prefixes WhisperlyApiError with [status])
```

Every `src/**/*.ts` with a sibling `.test.ts` is covered by Vitest (8 test files).

## Key Scripts

```bash
bun install          # Dependencies (node_modules is NOT checked in)
bun run build        # tsc -p tsconfig.build.json + scripts/fix-esm-imports.mjs
bun run build:watch  # Build in watch mode
bun run typecheck    # tsc --noEmit
bun run lint         # ESLint over src (lint:fix to autofix)
bun run test:run     # Run Vitest once (test:unit is an alias)
bun run format       # Prettier write (format:check to verify)
```

**Gotchas**
- `build` is two steps: `tsc` emits extensionless relative imports (`moduleResolution:
  "bundler"`), then `fix-esm-imports.mjs` rewrites them to `.js` so Node's ESM loader
  resolves them. Don't drop the second step.
- Vitest env: store/util tests run in `node`; hook/manager tests need a
  `// @vitest-environment jsdom` directive at the top of the file.
- Two ESLint config files exist (`eslint.config.js` and `eslint.config.mjs`). ESLint uses
  `eslint.config.js`; `eslint.config.mjs` is stale/unused.
- Published to GitHub Packages (`@sudobility` scope via `.npmrc`), not public npm.

## Architecture

```
UI Components (whisperly_app)
        ↓ uses
Manager Hooks (whisperly_lib)  ←→  TanStack Query (whisperly_client hooks)
        ↓ syncs to
Zustand Stores (whisperly_lib)
```

### Data Flow (every manager follows this)
1. Create `WhisperlyClient` internally via `useMemo` (stable ref keyed on baseUrl/networkClient)
2. Call TanStack Query hooks from `whisperly_client`
3. Sync query → store via `useEffect`s (data, loading, error) using `formatStoreError`
4. Wrap mutations in `useCallback`; combine `isLoading` from store + mutation pending states
5. Return store state + mutation actions. On logout, `resetAllStores()` clears all stores.

## Manager Signatures

**Every manager config is `{ baseUrl: string, networkClient: NetworkClient, ...scopeFields }`.**
`NetworkClient` comes from `@sudobility/types`; the consuming app builds it and is
responsible for injecting auth (e.g. a Firebase ID token) into requests. There is **no
`getIdToken` config field** — do not add one. `getIdToken` only exists as a *return value*
of `useFirebaseAuth`.

```typescript
// Entity-scoped
useProjectManager({ baseUrl, networkClient, entitySlug, autoFetch? })
//   -> projects, selectedProjectId, selectedProject, isLoading, error,
//      createProject, updateProject, deleteProject, generateApiKey, deleteApiKey,
//      selectProject, refetch, isCreating, isUpdating, isDeleting
useProjectDetail({ baseUrl, networkClient, entitySlug, projectId })
//   -> project, isLoading, error, refetch
useDictionaryManager({ baseUrl, networkClient, entitySlug, projectId })
//   -> dictionaries, selectedDictionaryId, selectedDictionary, isLoading, error,
//      createDictionary, updateDictionary, deleteDictionary, selectDictionary,
//      isCreating, isUpdating, isDeleting   (NOTE: no refetch)
useAnalyticsManager({ baseUrl, networkClient, entitySlug, startDate?, endDate?, projectId?, enabled? })
//   -> analytics, aggregate, byProject, byDate, dateRange, filterProjectId,
//      isLoading, error, setDateRange, setFilterProjectId, refetch
useLanguagesManager({ baseUrl, networkClient, entitySlug, projectId })
//   -> projectLanguages (comma-separated string|null), availableLanguages,
//      isLoading, error, updateLanguages, isUpdating, refetch

// User-scoped
useSettingsManager({ baseUrl, networkClient, userId })
//   -> settings, organizationName, organizationPath, isLoading, error,
//      updateSettings, refetch, isUpdating

// Public (no auth)
useTranslationManager({ baseUrl, networkClient, testMode? })
//   -> translate, isTranslating, lastResponse, error, clearLastResponse
// TranslateParams: { orgPath, projectName, request: TranslationRequest, apiKey? }
```

## Store Pattern

All 4 stores are Zustand stores with `isLoading`, `error`, domain-specific state and
setters, plus a `reset()`. Setters are domain-named (e.g. `setProjects`/`addProject`/
`updateProject`/`removeProject`, `setAnalytics`, `setSettings`), **not** a generic `setData`.
Each store exports internal `selectIsLoading`/`selectError` selectors that `stores/index.ts`
re-exports under domain-prefixed aliases (e.g. `selectProjectIsLoading`).

### Exported Selectors

```typescript
// Project store
selectProjects, selectSelectedProjectId, selectSelectedProject, selectProjectIsLoading, selectProjectError

// Dictionary store (special: keyed by projectId)
selectDictionariesForProject(projectId)  // returns a selector function
selectSelectedDictionaryId, selectDictionaryIsLoading, selectDictionaryError

// Settings store
selectSettings, selectOrganizationName, selectOrganizationPath, selectSettingsIsLoading, selectSettingsError

// Analytics store
selectAnalytics, selectAggregate, selectByProject, selectByDate, selectDateRange, selectFilterProjectId, selectAnalyticsIsLoading, selectAnalyticsError
```

### Dictionary Store (Special Case)

Dictionaries are keyed by `projectId` to prevent cross-project contamination:

```typescript
// State shape
dictionaries: Record<string, DictionarySearchResponse[]>  // projectId → dictionaries

// Usage with selector factory
const dictionaries = useDictionaryStore(selectDictionariesForProject('proj-123'));
```

## Auth Hook

```typescript
const { user, loading, getIdToken } = useFirebaseAuth(auth); // takes a Firebase Auth instance
// user: Firebase User | null
// getIdToken: () => Promise<string | undefined>
```

`firebase` is a devDependency (used only by this hook), not a peer dependency — the consumer
that calls `useFirebaseAuth` supplies the `Auth` instance and has `firebase` installed.

## Adding New Features

- **New store**: follow the existing pattern (state + domain setters + selectors), export
  from `stores/index.ts`, add `useNewStore.getState().reset()` to `utils/resetAllStores.ts`,
  and add a `.test.ts`.
- **New manager**: define `Config`/`Result` interfaces, use `useMemo` for the client +
  `useCallback` for actions, sync query→store via `useEffect`, export from
  `managers/index.ts` **and** `src/index.ts` (the root barrel is the public API).

## Dependencies

- `@sudobility/whisperly_client` — API client and hooks
- `@sudobility/whisperly_types` — shared types (re-exported for convenience)
- Peer deps: `@sudobility/types`, `@tanstack/react-query` (>=5), `react` (>=18), `zustand` (>=5)
- `firebase` — devDependency only (for `useFirebaseAuth`)

## Build Output

ESM-only: `dist/index.js` (module) + `dist/index.d.ts` (types). `package.json` `files` ships `dist/**/*`.

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
