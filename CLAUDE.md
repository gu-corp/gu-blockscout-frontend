# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## GU fork: how to support a new upstream version

This repo is a thin fork of [blockscout/frontend](https://github.com/blockscout/frontend).
The sections after this one are upstream guidance about the app itself. This
section is GU-specific and takes precedence where the two disagree.

### What we change vs upstream

Exactly four things, nothing else. Keep the diff this small.

| Change | File |
| --- | --- |
| `version` set to the version *before* the target (see step 4) | `package.json` |
| Footer hidden — early `return <></>` | `ui/snippets/footer/Footer.tsx` (newer: `src/shell/footer/Footer.tsx`) |
| Network logo replaced by a plain `Home` link | `ui/snippets/networkLogo/NetworkLogo.tsx` (newer: `src/slices/chain/logo/NetworkLogo.tsx`) |
| Upstream CI removed, GU publish workflow added | `.github/workflows/*`, `.husky/` |

Both component changes are written as an **early return followed by the
untouched upstream body**, with `// eslint-disable-next-line no-unreachable`.
Do not delete the upstream code — leaving it in place makes the next version
bump a trivial re-apply instead of a merge.

Upstream reorganises directories between minor versions (`ui/snippets/*` moved
to `src/shell/*` and `src/slices/*` around v2.9). Locate the files rather than
assuming a path:

```bash
find . -path ./node_modules -prune -o -name 'Footer.tsx' -print -o -name 'NetworkLogo.tsx' -print
```

The config key for the logo moved too (`config.UI.navigation.logo` →
`config.chain.logo`), but the GU override does not read it, so only the
surrounding code differs.

### Branch / tag layout

| Thing | Convention | Example |
| --- | --- | --- |
| Version branch | `feature/v<UPSTREAM_VERSION>` | `feature/v2.7.2` |
| Build tag | `v<UPSTREAM_VERSION>-gubuild.<N>` | `v2.7.2-gubuild.0` |
| Docker image | `gulabs/gu-blockscout-frontend:<BUILD_TAG>` | `gulabs/gu-blockscout-frontend:v2.7.2-gubuild.0` |

One branch per upstream version, each cut fresh from the upstream tag. Branches
are never merged into each other.

### Procedure

Replace `2.7.2` with the target version throughout.

**1. Fetch the upstream tag.** The `upstream` remote is usually not configured:

```bash
git remote add upstream https://github.com/blockscout/frontend.git   # once
git fetch upstream tag v2.7.2 --no-tags
```

**2. Cut the branch from the upstream tag** — never from a previous
`feature/*` branch:

```bash
git checkout -b feature/v2.7.2 v2.7.2
```

**3. Re-apply the four changes.** Copy the publish workflow forward from the
previous version branch:

```bash
git rm -r .husky .github/workflows
mkdir -p .github/workflows
git show feature/v2.9.4:.github/workflows/publish.yml > .github/workflows/publish.yml
```

`publish.yml` is version-independent — no edit needed.

**4. Set `package.json` `version` to one patch BELOW the target.**

This is the step that is easy to get wrong. The workflow runs
`npm version prerelease --preid gubuild`, which bumps the **patch** and appends
`-gubuild.0`. So to produce the tag `v2.7.2-gubuild.0`, `package.json` must say
`2.7.1`:

```
2.7.1  --prerelease-->  2.7.2-gubuild.0
2.9.3  --prerelease-->  2.9.4-gubuild.0
```

Upstream ships `package.json` with `"version": "1.0.0"` — it is a placeholder,
always overwrite it.

**5. Commit and push the branch:**

```bash
git push -u origin feature/v2.7.2
```

**6. Build.** Unlike the backend, this workflow is dispatched on the **branch**,
not on a tag — it commits the version bump and pushes the tag itself:

```bash
gh workflow run publish.yml --repo gu-corp/gu-blockscout-frontend \
  --ref feature/v2.7.2 -f bump=build -f publish-docker-image=true
```

`bump` options: `build` (prerelease, the normal case), `patch`, `minor`,
`major`. The workflow then: bumps the version → pushes the commit and tag to
the branch → creates a GitHub release → builds and pushes the multi-arch image
(`linux/amd64`, `linux/arm64/v8`) to Docker Hub. Expect roughly 20–40 minutes.

Afterwards `git pull` the branch to pick up the bump commit the workflow made.

**7. Verify:**

```bash
docker run --rm --entrypoint node gulabs/gu-blockscout-frontend:v2.7.2-gubuild.0 \
  -e "console.log(require('/app/package.json').version)"
```

(`--entrypoint` is required — the image's entrypoint is `./entrypoint.sh`,
which validates the runtime ENVs and would fail without a full config.)

Pair the image with a compatible backend build from `gu-blockscout` (see that
repo's `CLAUDE.md`); upstream releases backend and frontend in matching pairs.

### Required repository secrets

`DOCKERHUB_USERNAME`, `DOCKERHUB_ACCESS_TOKEN` — note the frontend uses
`_ACCESS_TOKEN` while the backend repo uses `_PASSWORD`.

### Known quirks

- **`GIT_TAG` build-arg is always empty.** The workflow sets it from
  `github.ref_type == 'tag'`, but we dispatch on a branch, so it resolves to
  `''`. It only feeds the frontend-version link in the footer, and the footer
  is hidden in GU builds, so this is cosmetic.
- **The repo default branch (`main-v1.29.2`) is stale.** It matters only
  because GitHub requires `publish.yml` to exist there for `workflow_dispatch`
  to be dispatchable at all.
- **`prepare: husky install` stays in `package.json`** even though `.husky/` is
  deleted. It is harmless: the Docker build never copies `.husky/`, and husky
  recreates the directory if it is missing.

---

## Commands

```bash
# Development
yarn dev                    # Start dev server
yarn build                  # Production build (Next.js only)
yarn build:next             # Full build with asset downloading and sprite building

# Linting
yarn lint:eslint            # ESLint check
yarn lint:eslint:fix        # ESLint auto-fix
yarn lint:tsc               # TypeScript type check
yarn lint:cspell            # Spell check
yarn lint:envs-validator:test  # Validate environment variable schemas

# Testing
yarn test:vitest            # Unit tests (Vitest, matches **/*.spec.ts(x))
yarn test:pw                # Playwright component/E2E tests (matches *.pw.tsx)
yarn test:pw:docker         # Playwright tests in Docker

# Assets
yarn svg:build-sprite       # Rebuild SVG sprite
yarn chakra:typegen         # Regenerate Chakra UI types
```

**Requirements:** Node >=22.14.0, npm >=10.9.0

## Architecture

**Stack:** Next.js 15 (Pages Router, not App Router), React 19, Chakra UI v3, React Query 5, Wagmi 2 / Viem 2, Valibot for schema validation, Vitest + Playwright for testing.

**Key directories:**
- `pages/` — Next.js page components (require default exports)
- `ui/` — React UI components organized by feature (~65 subdirectories)
- `lib/` — Business logic, API utilities, custom hooks, context providers
- `toolkit/` — Design system layer: `toolkit/chakra/` (custom Chakra components), `toolkit/theme/` (semantic color tokens), `toolkit/hooks/`
- `configs/app/` — Runtime app configuration (features, API endpoints, UI settings)
- `nextjs/` — Next.js config utilities: headers, rewrites, redirects, type-safe routes via `nextjs-routes`
- `mocks/` — Mock data for tests
- `deploy/tools/envs-validator/` — Environment variable validation schema and tests

**Data flow:** Pages use React Query for server state. Global UI state lives in React Context providers (`AppContextProvider`, `SettingsContextProvider`, etc.) initialized in `pages/_app.tsx`. WebSocket real-time data flows through `SocketProvider`.

**Routing:** Use `nextjs-routes` / `nextjs/routes` utilities for constructing links to application pages — never string concatenation. The full route list is in `nextjs/nextjs-routes.d.ts`.

## Design System Rules

These are enforced by ESLint and must be followed:

- **Always import from `toolkit/chakra/**`** before falling back to native Chakra UI. If a custom version exists in `toolkit/chakra/`, use it.
- **Never use hardcoded colors** (RGB, hex). Use semantic color tokens from `toolkit/theme/foundations/semanticTokens.ts` and `toolkit/theme/foundations/colors.ts` (e.g., `text.secondary`, `border.divider`, `icon.secondary`).
- **No custom `box-shadow`** — use design system shadow tokens.
- Don't override spacing on internal parts of compound components (e.g., don't add custom padding to `DialogHeader` inside a `Dialog`).
- Use `toolkit/chakra/link` instead of `next/link`.
- Use `lib/date/dayjs.ts` instead of importing `dayjs` directly.
- Date/time rendering must use the shared `Time` or `TimeWithTooltip` components.

## Global Type Declarations

- **Never use `(window as any)`** to access third-party globals. Instead, declare the property in `global.d.ts` inside the existing `declare global { interface Window { ... } }` block.
- Use `decs.d.ts` only for untyped third-party module declarations (`declare module 'foo'`).

## TypeScript Conventions

- Prefer `interface` over `type`. Use `interface extends` over `&` intersection (performance).
- No `enum` — use `as const` objects instead.
- Use top-level `import type { Foo }` not inline `import { type Foo }`.
- Default exports only when required by the framework (Next.js pages). All other exports are named.
- Declare return types on top-level module functions.
- `readonly` properties by default; omit only when genuinely mutable.
- Use `satisfies` for type validation instead of `as MyType[]` assertions.
- Outside generic functions, use `any` extremely sparingly; prefer `unknown` with proper narrowing.
- Extract magic numbers as `UPPER_SNAKE_CASE` constants above the component definition.
- Define empty array/object defaults as static constants outside components (not inline `?? []`).
- Wrap `.filter()`, `.map()`, `.reduce()` results in `useMemo` when passed as props or used as hook deps.
- Type parameters in generics are prefixed with `T` (e.g., `TKey`, `TValue`).

## Adding Environment Variables

When adding, renaming, or removing an environment variable, all of the following must be updated:

1. `docs/ENVS.md` — document name, type, whether required, default, and example
2. `configs/app/` — add to the appropriate section (`features/`, `ui.ts`, `api.ts`, etc.)
3. `deploy/tools/envs-validator/schema.ts` — add/update validation schema
4. `deploy/tools/envs-validator/test/.env.base` — add to test presets
5. `nextjs/csp/policies/` — update CSP if the variable references an external (non-asset) URL
6. `deploy/scripts/download_assets.sh` — add to `ASSETS_ENVS` if it's an asset URL
7. If it's a JSON config URL: add example to `deploy/tools/envs-validator/test/assets/configs/` and extend `envsWithJsonConfig` in `deploy/tools/envs-validator/index.ts`

## Testing

**Vitest** (unit): Files named `*.spec.ts` / `*.spec.tsx`. Run a single file with `yarn test:vitest path/to/file.spec.ts`.

**Playwright** (component/E2E): Files named `*.pw.tsx`. Three test projects run against each test: `default` (desktop Chrome 1200×750), `mobile` (iPhone 13 Pro), `dark-color-mode`. Tag tests with `@mobile` or `@dark-mode` to target specific projects; use `-@default` to exclude desktop.

- Use roles, test IDs, and text content as selectors — never CSS class selectors.
- Import mock values from existing files in `mocks/` rather than hardcoding them.
- Avoid the `testFn: TestFixture<...>` pattern unless sharing logic across multiple suites.
