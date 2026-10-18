# 0006. MongoDB only; schema evolves in code, indexes and seeds at startup

## Status

Accepted (recorded retrospectively).
Recorded against inventory-api `ddfd9d79` and inventory-platform `780e504`.

## Context

- The backend stores all data in one MongoDB database. There is no schema file.
- Reason not recorded for MongoDB and for startup runners.
- Reason for removing data backfills, from commit `ddfd9d79` (#266): they "still scan their collections on every boot: slower start …, a heap spike …, and a race if a second container ever starts alongside the first."
- Reason for keeping the three runners, same commit: the index runners "ensure indexes, not data"; `VerticalSchemaSeeder` "seeds schemas on every boot by design".

## Decision

- Spring Data MongoDB is the only store (`spring-boot-starter-data-mongodb`, property `spring.data.mongodb.uri`).
- There are 49 `@Document` classes.
- Repository packages are listed explicitly in `@EnableMongoRepositories` on `AppApplication`.
- `MongoConfig` stores `BigDecimal` as Decimal128.
- No migration framework: no Mongock, Flyway or Liquibase in any pom.
- Fields are added on the model. Old documents read them as `null`.
- Startup components run on `ApplicationReadyEvent`, ordered by `@Order`.
- `ProductBarcodeIndexMigration` (5) and `SearchIndexMigration` (6) call `ensureIndex`.
- `VerticalSchemaSeeder` (10) loads `seeds/*.json`. It overwrites only rows whose `createdBy` starts with `seed:`.
- Automatic index creation is not enabled.
- No `MongoTransactionManager` is declared, though `@Transactional` appears in 59 files.
- One manual script exists: `scripts/migrate-lot-id-to-stock-in.mongodb.js`.

## Consequences

- Data backfills are temporary code. Commit `ddfd9d79` removed six runners after they ran in production.
- All three remaining runners catch and log errors, so boot continues on failure.
- Multi-document writes are not atomic.
- Backend AGENTS.md P6 still lists the six removed runners.

## Evidence (paths)

- inventory-api: `core/pom.xml`
- inventory-api: `app/pom.xml`
- inventory-api: `app/src/main/resources/application.properties`
- inventory-api: `app/src/main/java/com/inventory/app/AppApplication.java`
- inventory-api: `app/src/main/java/com/inventory/app/config/MongoConfig.java`
- inventory-api: `core/product/src/main/java/com/inventory/product/migration/ProductBarcodeIndexMigration.java`
- inventory-api: `core/product/src/main/java/com/inventory/product/migration/SearchIndexMigration.java`
- inventory-api: `core/product/src/main/java/com/inventory/product/migration/VerticalSchemaSeeder.java`
- inventory-api: `core/product/src/main/resources/seeds/`
- inventory-api: `scripts/migrate-lot-id-to-stock-in.mongodb.js`
- inventory-api commit `ddfd9d79` (#266)
- Rules: [backend AGENTS.md](https://github.com/Stockkart/inventory-api/blob/main/AGENTS.md) S3, P1–P9.
