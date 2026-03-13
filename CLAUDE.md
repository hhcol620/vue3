# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

Requires Node.js >=20.0.0 and pnpm (version specified in `package.json`'s `packageManager` field).

```bash
pnpm i
```

## Common Commands

```bash
# Development - watch mode for a package (default: vue, default format: global)
pnpm dev
pnpm dev runtime-core          # specific package (must use full name)
pnpm dev -f esm-bundler        # specific format
pnpm dev -i                    # inline all deps

# Build
pnpm build                     # all public packages
pnpm build runtime-core        # fuzzy match package name
pnpm build runtime --all       # all packages matching "runtime"
pnpm build runtime-core -f global          # specific format
pnpm build runtime-core -f esm-browser,cjs # multiple formats
pnpm build -s                  # with source maps

# Type checking
pnpm check

# Lint / Format
pnpm lint
pnpm format

# Testing
pnpm test                          # watch mode (all tests)
pnpm test run                      # run once
pnpm test runtime-core             # tests under a package
pnpm test <fileNamePattern>        # files matching pattern
pnpm test <fileNamePattern> -t 'test name'  # specific test
pnpm test-unit                     # unit tests only (against source)
pnpm test-e2e                      # e2e tests (against built files, in real browser)
pnpm test-coverage                 # unit tests with coverage report

# Type declaration tests
pnpm test-dts                  # build dts then run type tests
pnpm test-dts-only             # run type tests against already-built dts

# Dev tools
pnpm dev-sfc                   # SFC Playground at localhost
pnpm dev-compiler              # Template Explorer at http://localhost:3000
pnpm dev-esm                   # ESM build with all deps inlined (for linking into repros)
```

## Architecture

This is the Vue.js core monorepo. Public packages live in `packages/`, private utilities in `packages-private/`.

### Package Dependency Graph

```
vue ──────────────────────────┐
  └─► runtime-dom             │
        └─► runtime-core      │
              └─► reactivity  │
  └─► compiler-dom ◄──────────┘
        └─► compiler-core
              ▲
        compiler-sfc ──► compiler-dom
        compiler-ssr ──► compiler-core
```

### Key packages

- **`reactivity`** — standalone reactive system (`ref`, `reactive`, `computed`, `watch`, etc.)
- **`runtime-core`** — platform-agnostic renderer, component system, and public JS APIs
- **`runtime-dom`** — browser-specific runtime (DOM API, events, attributes, directives)
- **`runtime-test`** — lightweight renderer that outputs plain JS objects; used in unit tests
- **`compiler-core`** — platform-agnostic template compiler and AST transforms
- **`compiler-dom`** — browser-specific compiler plugins
- **`compiler-sfc`** — Single File Component (`*.vue`) compiler
- **`compiler-ssr`** — SSR-optimized code generation
- **`server-renderer`** — SSR runtime
- **`shared`** — internal utilities shared by compiler and runtime (must stay environment-agnostic)
- **`vue`** — public full build (runtime + compiler); entry point for most users
- **`vue-compat`** — Vue 2 migration build

### Cross-package import rules

- Always import from the package name (e.g. `@vue/runtime-core`), never via relative paths across package boundaries.
- **Compiler packages must not import from runtime packages, and vice versa.** Shared code belongs in `@vue/shared`.
- If package A has a non-type import from package B, B must be listed as a dependency in A's `package.json`.

### Dev-only code

Wrap dev-only branches in `__DEV__` checks so they are tree-shakable from production builds:

```ts
if (__DEV__) {
  warn(...)
}
```

Runtime code is more size-sensitive than compiler code. Avoid pulling compiler-only utilities (e.g. `isHTMLTag`, `isSVGTag` from `@vue/shared`) into runtime code.

### Tests

Unit tests are co-located with source in `__tests__/` directories inside each package. Prefer `@vue/runtime-test` for platform-agnostic assertions about vdom behavior. Use platform-specific runtimes only when testing platform-specific behavior.

Type tests live in `packages-private/dts-test/`.

### Build formats

Each package builds to multiple formats defined in its `package.json` `buildOptions.formats`:

- `global` — IIFE for direct browser `<script>` use
- `esm-bundler` — ESM for bundlers (externalizes deps)
- `esm-browser` — ESM for direct browser `<script type="module">` use
- `cjs` — CommonJS for Node.js

## Commit Convention

Commit messages must match:
```
/^(revert: )?(feat|fix|docs|dx|style|refactor|perf|test|workflow|build|ci|chore|types|wip)(\(.+\))?: .{1,50}/
```

Examples: `fix(runtime-core): handle edge case in patch`, `feat(compiler): add new transform`

Only `feat`, `fix`, and `perf` types appear in the changelog. Breaking changes must include `BREAKING CHANGE:` in the footer.

## Branch Strategy

- `main` — bug fixes, chores, non-API changes
- `minor` — new API surface features (submit PRs here for new APIs)

## Git Hooks

Pre-commit runs `pnpm lint-staged && pnpm check` (type check + format staged files). The commit-msg hook validates the commit message format.
