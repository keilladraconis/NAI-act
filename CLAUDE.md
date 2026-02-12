# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NAI-act is a NovelAI Script module — a minimal reactive retained-UI framework (`defineComponent`, `mount`) for building interactive UIs on top of NovelAI's retained, imperative UI API. It runs inside NovelAI's scripting runtime, not as a standalone Node.js application.

## Package

- **npm name:** `nai-act`
- **Distribution:** Raw TypeScript source (no compilation)
- **Exports:** `./src/nai-act.ts`

## Build & Type Checking

There is no build step or bundler — the project uses `noEmit` TypeScript for type-checking only.

```bash
npm install              # Install devDependencies (typescript)
npm run typecheck        # Type-check the project (tsc --noEmit)
```

## Release Workflow

Publishing is automated via GitHub Actions (`.github/workflows/publish.yml`):
- **Trigger:** Creating a GitHub Release
- **Method:** OIDC trusted publishing (no NPM_TOKEN secret needed)
- **Steps:** checkout → install → type-check → dry-run pack → publish
- **First publish** must be done manually (`npm publish --access public`), then configure trusted publishing on npmjs.com

## Architecture

- **`src/nai-act.ts`** — The entire library. Exports `defineComponent`, `mount`, `mergeStyles`, and the `StoreLike`, `BindContext`, `Component`, `Style` types.
- **`external/script-types.d.ts`** — Ambient type declarations for the NovelAI Scripting API (`api.v1`, `UIPart`, UI component types, etc.). These types are globally available — no imports needed.

### Two-Phase Model

1. **Build** — `mount()` calls the component's `build()` method, which returns static UI structure while simultaneously binding dynamic behavior to state.
2. **Register** — The resulting UI parts are passed to NovelAI's `api.v1.ui.register()`.

Key concepts:
- UI structure is created once during build.
- State changes drive targeted updates via element IDs using `api.v1.ui.updateParts`.
- Behavior (subscriptions, effects) is bound during build and auto-cleaned up on unmount.
- `StoreLike<S>` interface defines the store contract — compatible with NAIStore or any store matching the interface.

### BindContext (passed to build)

- `getState()` — Read current state
- `dispatch(action)` — Dispatch actions
- `useSelector(selector, listener)` — Subscribe to state changes (auto-cleanup)
- `useEffect(predicate, effect)` — Subscribe to actions (auto-cleanup)
- `render(Component, props)` — Mount child components (auto-cleanup)

### Style Updates

NovelAI's `updateParts` does shallow merge. Use `mergeStyles(...styles)` to preserve base styles when applying dynamic changes. Components with a `styles` property get an auto-injected `this.style()` method.

## TypeScript Conventions

- Strict mode with `noImplicitAny`, `noUnusedLocals`, `noUnusedParameters`
- Target ES2023, module resolution `bundler`
- Avoid `any` type assertions
- Adhere to KISS principle
