# 0005. Docker Compose per repository; MongoDB is external

## Status

Accepted (recorded retrospectively).
Recorded against inventory-api `ddfd9d79` and inventory-platform `780e504`.

## Context

- Backend and frontend live in separate repositories, each with its own compose file.
- Reason not recorded.

## Decision

- Each repository has its own `docker-compose.yml` with one active service.
- inventory-api: service `app` is built from `Dockerfile`. Services `image-preprocess` and `mongodb` are commented out.
- inventory-platform: service `inventory-platform` is built from `Dockerfile.dev`. It runs `nx dev inventory` with the source bind-mounted. The backend service is commented out.
- The two compose files share no network.
- MongoDB comes from `DB_URI`.
- Both READMEs document a non-Docker run: `mvn -pl app spring-boot:run` and `pnpm nx dev inventory`.
- Deploy artefacts are separate from compose.
- The backend deploys a Docker image built by `.github/workflows/_build-image.yml`.
- The frontend deploys through `scripts/deploy/vercel-deploy.sh`.

## Consequences

- Compose is not a full local stack.
- A developer supplies MongoDB and points the frontend at the backend.
- The frontend README says Docker runs "frontend + API + Mongo". Its compose file runs the frontend only.
- Backend compose is not verified in the agent workspace (backend AGENTS.md Commands #10).

## Evidence (paths)

- inventory-api: `docker-compose.yml`
- inventory-api: `Dockerfile`
- inventory-api: `README.md`
- inventory-api: `docs/deploying.md`
- inventory-api: `.github/workflows/_build-image.yml`
- inventory-platform: `docker-compose.yml`
- inventory-platform: `Dockerfile.dev`
- inventory-platform: `README.md`
- inventory-platform: `docs/deploying.md`
- inventory-platform: `scripts/deploy/vercel-deploy.sh`
- Rules: [backend AGENTS.md](https://github.com/Stockkart/inventory-api/blob/main/AGENTS.md) C2, Commands; [frontend AGENTS.md](https://github.com/Stockkart/inventory-platform/blob/main/AGENTS.md) "Commands".
