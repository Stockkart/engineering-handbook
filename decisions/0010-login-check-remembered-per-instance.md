# 0010. Opaque bearer tokens; the login check is remembered per instance for 60 s

## Status

Accepted.
Recorded against inventory-api `5f7e67f6` (#273) and inventory-platform `59d9b36`.

## Context

- Supersedes: [0007](0007-opaque-bearer-token-authentication.md). Token model unchanged; the check no longer reads MongoDB on every request, and tokens no longer pile up.
- Reason: every request looked the user up by token with no usable index (`accessToken_1` was on a field no document has) and checked shop membership: 3 MongoDB trips before any work. Production users held 178 tokens, 155 expired (#273).

## Decision

- Login and signup issue random UUID access and refresh tokens, stored in `UserAccount.tokens` with a device id and a 7-day expiry (`UserMapper`). No JWT.
- `UserAccount.addToken` drops the user's expired tokens and keeps at most 10 live sessions; the oldest is logged out.
- `ExpiredTokenCleanupJob` removes expired tokens from all users nightly (`auth.token-cleanup.*`).
- Index `tokens_accessToken` `{tokens.accessToken: 1}` serves the token lookup.
- `AuthenticationInterceptor` requires `Authorization: Bearer`, then `TokenValidationService` and `UserShopMembershipService.hasAccess` consult `AuthCache` first.
- `AuthCache`: token → user copy and user + shop → access, 60 s after write, 10 000 entries each (`auth.cache.*`, off with `AUTH_CACHE_ENABLED=false`).
- `AuthCacheEvictionListener` evicts a user on any repository save or delete of a user or a membership.
- Token expiry and `active` are checked on every request, cached or not.
- Request attributes, shop selection, public paths and the frontend side are as in 0007 (backend AGENTS.md L6, L7, X3).

## Consequences

- A cached request makes no MongoDB trip for auth; `/auth/me` went from 6 to 3 trips.
- Logout, deactivation, shop switch and role changes apply at once on the instance that made them.
- On other instances they apply within 60 s; see [IN_MEMORY_STATE.md](../IN_MEMORY_STATE.md) before running a second instance.
- Writes outside the repositories (scripts, the database directly) apply within 60 s.
- Refresh tokens are stored, but no endpoint reads them. An expired session means a new login.
- The 11th concurrent session logs out the oldest one.

## Evidence (paths)

- inventory-api: `core/user/src/main/java/com/inventory/user/service/AuthCache.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/service/AuthCacheEvictionListener.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/service/TokenValidationService.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/service/UserShopMembershipService.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/service/ExpiredTokenCleanupJob.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/domain/model/UserAccount.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/mapper/UserMapper.java`
- inventory-api: `app/src/main/java/com/inventory/app/interceptor/AuthenticationInterceptor.java`
- inventory-api: `app/src/main/resources/application.properties`
- Rules: [backend AGENTS.md](https://github.com/Stockkart/inventory-api/blob/main/AGENTS.md) L6, L7, E7, X3, P4.
