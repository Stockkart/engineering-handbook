# Architecture decision records

Settled architecture decisions. Designs follow them or propose a change.

| # | Title | Status |
|---|---|---|
| 0001 | [Nx monorepo for the frontend](0001-nx-monorepo-frontend.md) | Accepted (recorded retrospectively) |
| 0002 | [Spring Boot modular monolith with plugins per vertical](0002-modular-backend-with-vertical-plugins.md) | Accepted (recorded retrospectively) |
| 0003 | [Styling only in ui-kit: chrome class maps and `--sk-*` tokens](0003-ui-kit-chrome-maps-and-tokens.md) | Accepted (recorded retrospectively) |
| 0004 | [OCR behind a provider and a preprocessor interface](0004-ocr-provider-abstraction.md) | Accepted (recorded retrospectively) |
| 0005 | [Docker Compose per repository; MongoDB is external](0005-docker-compose-per-repository.md) | Accepted (recorded retrospectively) |
| 0006 | [MongoDB only; schema evolves in code, indexes and seeds at startup](0006-mongodb-and-startup-migrations.md) | Accepted (recorded retrospectively) |
| 0007 | [In-app opaque bearer tokens checked by an interceptor](0007-opaque-bearer-token-authentication.md) | Accepted (recorded retrospectively) |
| 0008 | [Outbound messages: MongoDB outbox polled on a fixed delay](0008-scheduled-outbox-message-dispatch.md) | Accepted (recorded retrospectively) |
| 0009 | [Errors as `core/common` exceptions mapped by one GlobalExceptionHandler](0009-exception-based-error-envelope.md) | Accepted (recorded retrospectively) |

## Using ADRs

- Design agents: follow every Accepted ADR.
- Design agents: cite each ADR a design touches in its `Conventions and decisions applied` section, as `decisions/NNNN`.
- Reviewer agents: when a design or change departs from an Accepted ADR, and its PR adds no superseding ADR, cite the ADR number in the finding.

## Proposing a new or superseding ADR

1. In the design, state the new or changed decision in `Conventions and decisions applied`. Name the ADR it replaces and why.
2. In the same PR, copy [0000-template.md](0000-template.md) to the next free number: `NNNN-kebab-title.md`.
3. Set Status `Accepted`. Merging the PR accepts it. An unmerged PR changes nothing.
4. To supersede, add `Supersedes` to the new ADR's Context.
5. Then set the old ADR's Status to `Superseded by NNNN`. Keep the old file.
6. Never renumber or delete an ADR.
7. Add or update the index row above in the same PR.
8. When the code lives in inventory-api or inventory-platform, the ADR goes in the handbook PR of the same task.

## Writing rules

- Keep each ADR under 60 lines.
- Back every claim with an Evidence path.
- Retrospective ADRs write "Reason not recorded" where the code gives no reason.
- Link AGENTS.md rules by ID or section name. Do not copy their text.
- No secrets and no environment variable values. Names are fine.
