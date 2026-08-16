# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **Expo** monorepo: the Expo SDK (native modules), the Expo CLI, Expo Router, config-plugins,
Expo Go, the documentation site (docs.expo.dev), templates for `create-expo-app`, and the internal tooling
that builds/tests/releases all of it. It's a Yarn (v1, workspaces) monorepo spanning TypeScript/JavaScript,
Swift/Objective-C (iOS), Kotlin/Java (Android), and Next.js (docs).

Nested `CLAUDE.md` files exist for some packages with their own conventions — check for one in the package
you're editing before assuming this file covers it (e.g. `packages/expo-router/CLAUDE.md`,
`packages/@expo/cli/CLAUDE.md`).

## Repository layout

- `packages/` — All Expo SDK modules (`expo-camera`, `expo-router`, `expo`, etc.) and the `@expo/*` scoped
  tooling packages (`@expo/cli`, `@expo/config`, `@expo/metro-config`, ...). This is where you'll spend most
  of your time editing library code.
- `apps/` — Example/dev apps used to exercise packages during development, and Expo Go itself:
  - `apps/bare-expo` — bare React Native app linking every package in `packages/`; the primary dev harness.
  - `apps/test-suite` — Jasmine-style E2E tests, run inside `bare-expo` (and loadable directly via Expo Go).
  - `apps/native-component-list` — manual/visual smoke-test app for UI components.
  - `apps/expo-go` — source for the Expo Go client app (`ios/Exponent.xcworkspace` is the Xcode entry point;
    don't open `Exponent.xcodeproj` directly, it skips CocoaPods deps).
  - `apps/router-e2e` — Expo Router E2E fixtures/tests, driven from `packages/@expo/cli`.
- `docs/` — Next.js source for docs.expo.dev (separate Yarn project, own `package.json`/Node version).
- `react-native-lab/react-native` — Expo's fork of react-native, used to build Expo Go. Kept as close to
  upstream as possible; changes here are cherry-picks/minimal diffs.
- `templates/` — project templates behind `npx create-expo-app`.
- `template-files/` — templates for files requiring private keys/config, populated via `template-files/keys.json`.
- `tools/` — `expotools` (`et`/`expotools` CLI): CI helpers, release tooling, doc generation, local test runners.
- `guides/` — engineering guides (style guide, module infra, releasing, code review). Read `guides/README.md`
  first; `guides/Expo JavaScript Style Guide.md` and `guides/Creating Unimodules.md` matter most for SDK work.
- `fastlane/`, `scripts/` — native release/build automation.

## Common commands

Run from the repo root unless noted. Most package-level commands come from the shared `expo-module-scripts`
package (invoked as `expo-module` under the hood), so packages are largely consistent.

```bash
# Install (workspace-wide)
yarn install

# Root lint (ESLint over the whole repo)
yarn lint

# Native setup (submodules, Android NDK, etc.) — needed once for native/Android work
npm run setup:native
npm run setup:docs   # only if working on docs/
```

Inside an individual SDK package (e.g. `packages/expo-constants`):

```bash
yarn build            # tsc/swc build in watch mode (skip if package has no build script)
yarn test             # jest via expo-module-scripts preset, watch mode by default
yarn test <file>      # run a single test file, e.g. yarn test src/__tests__/Constants-test.ts
CI=1 yarn test        # non-watch, single run (what CI does)
yarn lint             # expo-module eslint .
yarn lint --fix
yarn clean            # remove build/
```

Platform-specific test files are picked up automatically: `*.test.ios.ts`, `*.test.android.ts`,
`*.test.web.ts`, `*.test.native.ts` (iOS+Android), `*.test.node.ts`.

E2E / native verification loop (see `CONTRIBUTING.md` for full setup):

```bash
cd apps/bare-expo
yarn ios              # or: yarn android
yarn test:ios         # or: yarn test:android — runs apps/test-suite E2E tests
```

`expotools` (the `et` command) does most of the "meta" work — docs generation, changelog checks, running CI
steps locally:

```bash
et --help
et generate-docs-api-data -p <package-name>            # regen API docs for a package (unversioned)
et generate-docs-api-data -p <package-name> -s <sdk>    # for a specific SDK version
```

Docs site (separate workspace, needs the Node version pinned in `docs/package.json`'s `volta` field):

```bash
cd docs
yarn
yarn dev   # localhost:3002
```

There is no root-level `tsc`/`test` — typecheck and test from inside the individual package
(`yarn tsc` at the root deliberately errors and tells you this).

## Architecture notes

### SDK packages (`packages/expo-*`)

Each native module package follows the same shape, generated via `expo generate-module` and standardized by
`expo-module-scripts`:

- `src/` — TypeScript, compiled to `build/` (build output **is committed to git** — contributors don't need
  to rebuild every dependency locally after `git pull`).
- `ios/`, `android/` — native implementation (Swift or Kotlin/Java), registered as an Expo Module.
- `expo-module.config.json` — declares which native module/service classes to autolink per platform
  (`apple`/`android`/`web`), consumed by `expo-modules-autolinking`.
- `src/__tests__/` — Jest unit tests using the `expo-module-scripts` jest preset (`jest-expo` mocks native
  modules — see `packages/jest-expo/src/preset/expoModules.js`; new bridged native functions must be added
  there, or use `guides/Generating Jest Mocks.md`).

Native modules are built on the **Expo Modules API** (`expo-modules-core`): native code exposes functions/
views/constants via a `Module` definition (Swift `ExpoModulesCore.Module`, Kotlin equivalent), and JS calls
into it through the shared native bridge — this is what `expo-module.config.json` wires up per package.

### `expo` package

The `expo` package (`packages/expo`) is the umbrella package apps depend on: it re-exports SDK APIs, and also
ships the `expo` CLI binaries (`bin/cli` → `@expo/cli`, `bin/autolinking`, `bin/fingerprint`), config-plugins,
Metro config, and the winter runtime shims. It's the integration point, not where most feature work happens —
that's usually in the individual `expo-*` package or in `@expo/cli`.

### `@expo/cli` (`packages/@expo/cli`)

The `expo` CLI. Commands live under `src/<command>/` (`start/`, `run/`, `export/`, `prebuild/`, `install/`,
`config/`, etc.), registered from `bin/cli.ts`. `start/server/metro/` is the Metro dev-server integration
(multi-platform bundling, custom resolvers); `start/server/middleware/` is the HTTP layer (manifest serving,
Expo Go support, DOM components, dev tools). Built with `swc` via a custom `taskr` `taskfile.js`, not
`expo-module-scripts`. See `packages/@expo/cli/CLAUDE.md` and `packages/@expo/cli/docs/testing.md` for its
own test/E2E setup and Windows path-handling caveats.

### `expo-router` (`packages/expo-router`)

File-based router built on React Navigation. See `packages/expo-router/CLAUDE.md` for its detailed structure,
routing pipeline (Metro `require.context()` → `getRoutes()` → React Navigation config → linking config), and
testing patterns (`renderRouter` from `testing-library`).

### Config plugins & prebuild

`packages/@expo/config-plugins` + `packages/@expo/prebuild-config` implement the "continuous native
generation" system: plugins mutate native project files (`Info.plist`, `AndroidManifest.xml`, Xcode/Gradle
projects) declaratively from `app.json`/`app.config.js`, invoked by `expo prebuild`.

### react-native-lab

`react-native-lab/react-native` is a git submodule fork of React Native used specifically for building Expo
Go. Keep divergence from upstream minimal (cherry-pick where possible); it's not where general React Native
app code lives.

## Conventions

- **Commit messages**: `[platform][api] Title`, e.g. `[ios][video] Fixed black screen bug on older devices`.
- **Changelogs**: user-facing changes need an entry in the changed package's own `CHANGELOG.md` (or the root
  `CHANGELOG.md` if the change isn't scoped to one package). See `guides/contributing/Updating Changelogs.md`.
- **Style**: `guides/Expo JavaScript Style Guide.md` (also covers TypeScript); Prettier config is
  `.prettierrc` (single quotes, 100 print width, trailing commas); ESLint config is `universe/native` +
  `universe/node` + `universe/web` (`eslint-config-universe`).
- **Docs versioning**: when editing current-SDK docs under `docs/pages/versions/vXX.0.0/`, mirror the change
  into `docs/pages/versions/unversioned/` too — the unversioned copy is what becomes the next SDK's docs.
- **Platform-specific source files**: `.ios.ts(x)`, `.android.ts(x)`, `.web.ts(x)`, `.native.ts(x)` (iOS +
  Android) suffixes are resolved automatically by Metro/jest-expo — this pattern is used throughout SDK
  packages, `@expo/cli`, and `expo-router`.
- Before submitting a PR touching `packages/`: `yarn build` the package, `yarn lint --fix`, `yarn test`, and
  remove stray `console.log`s — see `CONTRIBUTING.md` for the full pre-submission checklist.
