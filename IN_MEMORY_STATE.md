# In-memory state per backend instance

## Purpose

Production runs one backend instance (DigitalOcean App Platform, `instance_count` 1). Anything the backend keeps in memory, and anything that runs on every instance, behaves differently once a second instance runs. This page lists all of it, so scaling out is a checklist and not a surprise.

Recorded against inventory-api `5f7e67f6` (#273), plus `RequestReadMemoAspect` (branch `perf/request-shop-plan-usage`), `ShopReadCache` (branch `perf/shop-read-cache`), `HeavyRequestLimiter` (#304), `MongoTx` (branch `feat/mongo-transactions`), the Attendance app rows (`AppRateLimiter`, `AppPunchHealthService`, `PairingExitJob`; HR-module spec, `core/hr`) and the Stockkart Team app rows (the three worker keys in `AppRateLimiter`, `WorkerExitJob`; worker-app spec, `core/hr`).

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
| `MongoTx.shopLocks` | one fair lock per shop: the shop's money flows (bill completion, refund, vendor invoice and return, credit charge and settlement) take turns, each in its own MongoDB transaction | a lock per shop seen since start, never removed (a few bytes each); a flow waits up to `mongo.transactions.shop-wait-ms` (15 s) for its turn, else 503 busy | in-process only; released when the flow ends, error or not | each instance has its own turns, so two instances can run one shop's flows at once; they then write-conflict and MongoDB aborts one, which `MongoTx` runs again (up to `mongo.transactions.max-attempts`): correct, but slower when one shop bills on both | nothing for correctness; under heavy per-shop load, route a shop to one instance or shorten the transactions |
| `taskExecutor`, `ocrTaskExecutor` | queued async work (`@Async`, OCR parsing) | until done; in-memory queue | none | queued work is lost if that instance stops; each instance has its own pool and limits | nothing for correctness; size each instance's pool |
| `AppRateLimiter.windows` | per key, the Attendance app's admitted request times and lock: `punch:<pairingId>` (20 punches per rolling 60 s, R9.9) and `code-fail:<installationId>` (5 refused pairing codes in 15 min → locked 15 min, R2.5) | an entry lives while its window or lock is live; idle keys are dropped by the one-minute `sweep`; bounded by the keys active in the last 15 min | in-process only; the per-code attempt count that invalidates a code (R9.10) is on the `hr_pairing_codes` document, not here | each instance has its own counters: a pairing can send N × 20 punches a minute and an installation can try N × 5 codes before the lock; nothing is stored or lost | nothing for correctness; a shared counter (Mongo or Redis) if the limits must hold across instances |
| `AppRateLimiter.windows`, worker keys | the same map, three more key families for the Stockkart Team app (`tryAcquire`): `lookup:<userId>` (10 shop lookups per rolling hour, `WorkerJoinService`), `join:<userId>` (5 join requests per rolling 24 h, `WorkerJoinService`), `signup-ip:<ip>` (20 `source = worker-app` sign-ups per rolling hour, `WorkerAppAuthPortImpl`, called by `AuthService.signup` before anything is read or written) | an entry lives while its window is live, so a `join:` key lives up to 24 h after the last admitted request; idle keys are dropped by the same `sweep`; bounded by the accounts and addresses active in the last 24 h | in-process only and lost on restart: after a deploy every account may look up, join and sign up afresh. The limit that survives a restart is the partial unique index `user_open_unique` on `hr_join_requests (userId)` where `status in [PENDING, APPROVING]`: whatever the counter says, an account can hold one open join request (`HR_WORKER_REQUEST_PENDING`). A refused call writes nothing and is not counted | each instance has its own counters: an account can make N × 10 lookups an hour and N × 5 join requests a day, and an address N × 20 sign-ups an hour; the one-open-request index holds across instances because it is in Mongo | nothing for correctness; a shared counter if the limits must hold across instances |
| `AppPunchHealthService.windows` | per shop, 60 one-minute buckets of Attendance app punch attempts and signature failures (R10.4), fed by `AttendanceAppAuthInterceptor` and `AppPunchService` | a shop's ring lives while it had an attempt in the last 60 min; idle shops are dropped at each `recalculate`; bounded by the shops active in the last hour | in-process only; the alerts themselves (`hr_app_alerts`, one open row per shop) are in Mongo; the window starts empty after a restart, so for 60 min after a deploy no alert can be raised (too few attempts) or cleared (the window is not full) | each instance sees only its share of a shop's punches, so a shop needs N × 20 attempts an hour before any instance can alert; an open alert is read before an insert, so two instances do not duplicate it | nothing for correctness; the buckets in a shared store (or the rate from the metrics backend) if the alert must see all punches |

## Scheduled jobs (run on every instance)

| Job | Schedule | Claims work first | With more than one instance |
|---|---|---|---|
| `MessageQueueProcessor.processQueue` | `messaging.dispatch-interval-ms` | no | the same email can be sent twice (ADR 0008) |
| `EventService.processEventsScheduler` | `reminders.dispatch-interval-ms` | no | the same reminder can be raised twice |
| `EventService.heartbeat` | `sse.heartbeat-interval-ms` (25 s) | not needed | harmless: each instance pings only its own connections |
| `QRUploadService.cleanupExpiredTokens` | hourly | not needed | harmless: idempotent delete |
| `ExpiredTokenCleanupJob.run` | `auth.token-cleanup.cron` (03:30 Asia/Kolkata) | not needed | harmless: idempotent `$pull` |
| `AppRateLimiter.sweep` | every 60 s (`fixedDelay`) | not needed | harmless: each instance drops idle keys from its own map (the Attendance app keys and the worker keys alike) |
| `AppPunchHealthService.recalculate` | every 5 min (`fixedDelay`) | no | each instance rates its own share of a shop's punches (see the cache row); an alert is raised once at most (the open row is checked before the insert) and clearing an already cleared row is skipped, so a second instance adds no duplicates |
| `PairingExitJob.run` | every 60 s (`hr.pairing-exit.interval-ms`, `fixedDelay`; off with `hr.pairing-exit.enabled=false`) | no | harmless: the schedule is the `scheduledUnpairAt` field on the `ACTIVE` pairing in Mongo, not in memory, so nothing is lost on a restart and the next tick catches up; each unpair is one conditional `ACTIVE → UNPAIRED` update (`EMPLOYEE_EXITED`, by `system`) that writes its audit row and counter only when it won, so two instances cannot unpair, audit or count the same phone twice; a failed unpair keeps its schedule and is retried next tick |
| `WorkerExitJob.run` | every 60 s (`hr.worker-exit.interval-ms`, `fixedDelayString`; off with `hr.worker-exit.enabled=false`) | no | harmless: the job holds no state. Its two reads are of Mongo: linked `EXITED` employees whose `exitDate <= today` (Asia/Kolkata) and whose `membershipEndedAt` is still null (the `exit_membership_pending` partial index), each handed to `WorkerMembershipEnder` (release the WORKER account's shop, end the membership) with a conditional write on `membershipEndedAt == null`; and `hr_join_requests` rows left `APPROVING` for over 5 minutes (a crashed approval transaction), each reset to `PENDING` by a conditional update and logged. Both writes win on one instance only, so two instances cannot end a membership or reset a request twice; nothing is lost on a restart and the next tick catches up; a row that fails is logged and read again next tick |

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
- inventory-api: `core/common/src/main/java/com/inventory/common/tx/MongoTx.java`
- inventory-api: `core/hr/src/main/java/com/inventory/hr/service/AppRateLimiter.java`
- inventory-api: `core/hr/src/main/java/com/inventory/hr/service/AppPunchHealthService.java`
- inventory-api: `core/hr/src/main/java/com/inventory/hr/service/PairingExitJob.java`
- inventory-api: `core/hr/src/main/java/com/inventory/hr/service/WorkerJoinService.java` (`lookup:` and `join:` keys)
- inventory-api: `core/hr/src/main/java/com/inventory/hr/service/api/WorkerAppAuthPortImpl.java` (`signup-ip:` key)
- inventory-api: `core/hr/src/main/java/com/inventory/hr/service/WorkerExitJob.java`
- inventory-api: `core/hr/src/main/java/com/inventory/hr/service/WorkerMembershipEnder.java`
- inventory-api: `core/hr/src/main/java/com/inventory/hr/migration/HrIndexMigration.java` (`user_open_unique`, `exit_membership_pending`)
- inventory-api: `app/src/main/java/com/inventory/app/heavy/HeavyRequestAspect.java`
- inventory-api: `app/src/main/resources/application.properties`
