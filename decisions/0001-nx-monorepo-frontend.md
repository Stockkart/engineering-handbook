# 0001. Nx monorepo for the frontend

## Status

Accepted (recorded retrospectively).
Recorded against inventory-api `ddfd9d79` and inventory-platform `780e504`.

## Context

- The frontend is one app built from 24 packages: domains, platform infrastructure, plugins and a design system.
- Reason not recorded for Nx and the pnpm workspace.
- Reason for dynamic plugin loading: "Each loader is dynamically imported so unused vertical UI stays out of the main bundle." (`plugins/registry/src/plugin-loaders.ts`).

## Decision

- One pnpm workspace: `apps/*`, `core/*`, `platform/*`, `plugins/*`, `ui-kit`.
- Nx 22.1.2 infers targets from four plugins: `@nx/js/typescript`, `@nx/react/router-plugin`, `@nx/eslint/plugin`, `@nx/vite/plugin`.
- Five layer tags are enforced by `@nx/enforce-module-boundaries`: `type:app`, `type:platform`, `type:core`, `type:plugin`, `type:ui-kit`.
- One app, `apps/inventory`, composes routes through `plugins/registry`.
- Frontend vertical plugins load by `verticalId` through dynamic import.
- Only `cafe` is registered as a frontend plugin.

## Consequences

- A new package needs the wiring that the package-tracking workflow checks: labels, labeler, CODEOWNERS, ESLint app ban, root tsconfig reference.
- `type:journey` is declared in the boundary rules. No project carries it.
- Backend verticals medical, grocery and sports have no frontend plugin. Their fields come from `platform/schema`.
- A vertical without a loader gets `null` from `loadVerticalPlugin` and no extra UI.

## Evidence (paths)

- inventory-platform: `pnpm-workspace.yaml`
- inventory-platform: `nx.json`
- inventory-platform: `package.json`
- inventory-platform: `eslint.config.mjs`
- inventory-platform: `apps/inventory/app/routes.tsx`
- inventory-platform: `plugins/registry/src/plugin-loaders.ts`
- inventory-platform: `platform/schema/README.md`
- inventory-platform: `.github/workflows/package-tracking.yml`
- inventory-platform: `.github/scripts/check-package-tracking.mjs`
- Rules: [frontend AGENTS.md](https://github.com/Stockkart/inventory-platform/blob/main/AGENTS.md) "Nx workspace structure and project boundaries", "New package checklist".
