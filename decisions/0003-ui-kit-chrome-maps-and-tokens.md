# 0003. Styling only in ui-kit: chrome class maps and `--sk-*` tokens

## Status

Accepted (recorded retrospectively).
Recorded against inventory-api `ddfd9d79` and inventory-platform `780e504`.

## Context

- Domain, platform and plugin packages render UI but own no CSS.
- Reason not recorded.

## Decision

- CSS modules exist only in ui-kit. `cssModuleImportBan` in the root ESLint config blocks them elsewhere.
- Domain code styles through exported `as const` class maps ("chrome").
- The chrome maps are `surfaceChrome`, `shellChrome`, `journeyChrome`, `chartChrome`, `productChrome`, `registrationChrome`, `accountingChrome`, `searchChrome`.
- Theme values are 63 `--sk-*` custom properties in `ui-kit/src/theme/tokens.css`.
- Each token is set in two blocks: `:root` plus the light `data-theme`, and the dark `data-theme`.
- `ThemeProvider` switches theme by setting `data-theme` on `<html>`.
- Global CSS is only the token import and the reset in `apps/inventory/styles.css`.
- Native HTML tags are banned in domain code (`bannedNativeHtmlTags`). ui-kit primitives replace them.

## Consequences

- `SurfaceChrome.module.css` is about 3,200 lines and shared by many views.
- Editing a shared class changes every consumer.
- #222 restyled shared `profileHero*` classes through descendant selectors. #223 reverted it.
- A new style is a new key in a chrome map, so every style change touches ui-kit.

## Evidence (paths)

- inventory-platform: `ui-kit/src/patterns/SurfaceChrome.tsx`
- inventory-platform: `ui-kit/src/patterns/SurfaceChrome.module.css`
- inventory-platform: `ui-kit/src/patterns/index.ts`
- inventory-platform: `ui-kit/src/theme/tokens.css`
- inventory-platform: `ui-kit/src/theme/ThemeProvider.tsx`
- inventory-platform: `apps/inventory/styles.css`
- inventory-platform: `eslint.config.mjs`
- inventory-platform: `ui-kit/README.md`
- inventory-platform commits `1335f8c` (#222) and `d4ce2d0` (#223)
- Rules: [frontend AGENTS.md](https://github.com/Stockkart/inventory-platform/blob/main/AGENTS.md) "Styling", "Component conventions".
