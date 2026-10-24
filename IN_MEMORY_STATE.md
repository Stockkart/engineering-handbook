# In-memory state per backend instance

## Purpose

Production runs one backend instance (DigitalOcean App Platform, `instance_count` 1). Anything the backend keeps in memory, and anything that runs on every instance, behaves differently once a second instance runs. This page lists all of it, so scaling out is a checklist and not a surprise.

Recorded against inventory-api `5f7e67f6` (#273), plus `RequestReadMemoAspect` (branch `perf/request-shop-plan-usage`), `ShopReadCache` (branch `perf/shop-read-cache`) and `HeavyRequestLimiter` (branch `fix/heavy-request-limits`).

## Rule

- A change that adds a cache, a memo, a held connection or a scheduled job adds a row here in its handbook PR.
- Reviewer agents: cite this page when a backend PR adds such state without a row.

## Caches and held state

| State | Holds | Lifetime and bound | Kept correct by | With more than one instance | Needed before scaling out |
|---|---|---|---|---|---|
| `AuthCache.tokens` | access token → user copy (no password, no other sessions) | 60 s after write, 10 000 entries (`auth.cache.*`) | `AuthCacheEvictionListener`: any repository save or delete of a user or membership evicts that user; token expiry checked on every request | logout, deactivation, shop switch or role change on one instance reaches the others only when their entry expires (≤ 60 s) | cross-instance eviction, e.g. change streams on `users` and `user_shop_memberships` (latency tasks 9.0) |
| `AuthCache.shopAccess` | user + shop → may act in shop | same as above | same as above | adding or removing shop access reaches other instances in ≤ 60 s | same as above |
| `RequestReadMemoAspect` | shop and plan reads (`findById`, `findByPlanName`) for the current HTTP request | one request; dropped on any save, insert or delete on that repository | request attributes; each caller gets a copy | none: nothing outlives the request | nothing |
| `ShopReadCache` | `shopRepository.findById` results (pristine copies; callers get copies) | 60 s after write, 10 000 entries (`shop.cache.*`) | cleared on any save, insert or delete through `shopRepository` on this instance; a read that raced a write is not stored | a shop settings edit, plan renewal or approval on one instance reaches the others in ≤ 60 s (for example, a renewed plan can still look expired there for up to 60 s) | cross-instance eviction, e.g. a change stream on `shops`, or turn it off (`SHOP_CACHE_ENABLED=false`) |
| `SearchValueSuggester.cache` | per shop and field: distinct values for search suggestions | 60 s per entry, checked on read; no size cap, entries never removed | TTL only | suggestions can differ between instances for ≤ 60 s | nothing for correctness; a size cap as shops grow |
| `SchemaLoader.schemaCache`, `activeVersionCache` | vertical schemas from `vertical_schemas` | until restart; warmed and evicted only by `VerticalSchemaSeeder` at startup | schemas change only via seed files and a restart (backend AGENTS.md P9) | during a rolling deploy, instances can run different schema versions until all have restarted | nothing while schemas change only at deploy |
| `EventService.emittersByShop` | open SSE connections (`/api/v1/events/stream`) per shop | while each browser stays connected: a dropped connection is removed when a keep-alive write fails (2–3 × 25 s), else at the 30-minute emitter timeout; at most 20 per shop (the oldest is closed) | in-process only; gauge `sse_emitters_open` | an event raised on instance A reaches only browsers connected to A; others miss live notifications | publish events through a shared channel (e.g. a change stream on `events`) to every instance |
| `HeavyRequestLimiter` | per kind (export, analytics, OCR, import): permits in use and a waiting line of web requests | permits are held only while a heavy request runs; limit, line length and wait per kind (`heavy.<kind>.limit`, `.queue`, `.wait-ms`, `.background-wait-ms`); over the line or the wait → 503 with `Retry-After`; gauges `inventory_heavy_in_use`, `inventory_heavy_waiting`, counter `inventory_heavy_rejected` | in-process only; a permit is released when the request ends, including on error | each instance has its own permits, so N instances allow N × limit heavy requests at once against the same database | nothing for correctness; lower the per-instance limits, or move heavy work to the worker (scaling step 2) |
| `taskExecutor`, `ocrTaskExecutor` | queued async work (`@Async`, OCR parsing) | until done; in-memory queue | none | queued work is lost if that instance stops; each instance has its own pool and limits | nothing for correctness; size each instance's pool |

## Scheduled jobs (run on every instance)

| Job | Schedule | Claims work first | With more than one instance |
|---|---|---|---|
| `MessageQueueProcessor.processQueue` | `messaging.dispatch-interval-ms` | no | the same email can be sent twice (ADR 0008) |
| `EventService.processEventsScheduler` | `reminders.dispatch-interval-ms` | no | the same reminder can be raised twice |
| `EventService.heartbeat` | `sse.heartbeat-interval-ms` (25 s) | not needed | harmless: each instance pings only its own connections |
| `QRUploadService.cleanupExpiredTokens` | hourly | not needed | harmless: idempotent delete |
| `ExpiredTokenCleanupJob.run` | `auth.token-cleanup.cron` (03:30 Asia/Kolkata) | not needed | harmless: idempotent `$pull` |

## Before running a second instance

1. Cross-instance eviction for `AuthCache` and `ShopReadCache`.
2. Shared delivery for SSE events.
3. A claim step for `MessageQueueProcessor` and `EventService.processEventsScheduler`.

## Evidence (paths)

- inventory-api: `core/user/src/main/java/com/inventory/user/service/AuthCache.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/service/AuthCacheEvictionListener.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/service/ExpiredTokenCleanupJob.java`
- inventory-api: `app/src/main/java/com/inventory/app/memo/RequestReadMemoAspect.java`
- inventory-api: `app/src/main/java/com/inventory/app/memo/ShopReadCache.java`
- inventory-api: `core/product/src/main/java/com/inventory/product/search/SearchValueSuggester.java`
- inventory-api: `core/product/src/main/java/com/inventory/product/service/vertical/SchemaLoader.java`
- inventory-api: `core/product/src/main/java/com/inventory/product/migration/VerticalSchemaSeeder.java`
- inventory-api: `core/reminders/src/main/java/com/inventory/reminders/service/EventService.java`
- inventory-api: `core/notifications/src/main/java/com/inventory/notifications/service/MessageQueueProcessor.java`
- inventory-api: `core/product/src/main/java/com/inventory/product/service/QRUploadService.java`
- inventory-api: `app/src/main/java/com/inventory/app/config/AsyncExecutorConfiguration.java`
- inventory-api: `core/common/src/main/java/com/inventory/common/heavy/HeavyRequestLimiter.java`
- inventory-api: `app/src/main/java/com/inventory/app/heavy/HeavyRequestAspect.java`
- inventory-api: `app/src/main/resources/application.properties`
