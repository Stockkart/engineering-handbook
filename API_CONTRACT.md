# API contract

## Purpose

- These are the rules the client repositories (inventory-platform, the web frontend; stockkart-team-app, the Stockkart Team app) and the backend (inventory-api) follow for HTTP JSON.
- Design agents use them to fill a design's `## Cross-repo contract` section (CH5). Reviewers cite them as "API_CONTRACT UV3".
- Per-repo rules live in [backend AGENTS.md](https://github.com/Stockkart/inventory-api/blob/main/AGENTS.md), [frontend AGENTS.md](https://github.com/Stockkart/inventory-platform/blob/main/AGENTS.md) and [app AGENTS.md](https://github.com/Stockkart/team-app/blob/main/AGENTS.md). This file cites them by rule ID and does not repeat them.
- Each rule is one bullet. `Ex:` names one endpoint that follows it, as `VERB path` (backend file) ↔ client call (frontend file).
- Paths start with the repository: `backend/` is inventory-api, `frontend/` is inventory-platform, `app/` is stockkart-team-app (the Stockkart Team app, its own repository since worker-app D17; formerly `frontend/apps/attendance`).
- Scope: 54 `@RestController` classes, 219 mappings. Facts as of backend `ddfd9d79` and frontend `780e504`, 2026-10-17.

## URL and verb conventions

Class paths and verbs are backend AGENTS.md N4 and N5. These rules add the client side.

- UV1. The class path is `/api/v1/<kebab-plural or mass noun>`. The client base URL (`VITE_API_URL`) already ends in `/api/v1`. Constants in `core/<d>/src/api/endpoints.ts` leave the prefix out. Ex: `GET /api/v1/vendors` (`backend/core/user/src/main/java/com/inventory/user/rest/controller/VendorController.java`) ↔ `VENDOR_ENDPOINTS.BASE` (`frontend/core/user/src/api/endpoints.ts`), `vendorsApi.list` (`frontend/core/user/src/api/vendors.api.ts`).
- UV2. The client verb equals the backend verb. An update uses the verb the resource already uses (N5). Ex: `PATCH /api/v1/customers/{customerId}` (`backend/core/user/src/main/java/com/inventory/user/rest/controller/CustomerController.java`) ↔ `customersApi.update` (`frontend/core/user/src/api/customers.api.ts`).
- UV3. An action is `POST` on a sub-path: `/{id}/<action>` or `/<action>`. Never put `create`, `update`, `upsert` or `delete` in a path. Ex: `POST /api/v1/vendor-purchase-invoices/preview-totals` (`backend/core/product/src/main/java/com/inventory/product/rest/controller/VendorPurchaseInvoiceController.java`) ↔ `inventoryApi.previewPurchaseTotals` (`frontend/core/product/src/api/inventory.api.ts`).
- UV4. Every client constant matches a backend mapping. A path value that is not an ObjectId goes through `encodeURIComponent`. Ex: `POST /api/v1/barcodes/{code}/attach` (`backend/core/product/src/main/java/com/inventory/product/rest/controller/BarcodeController.java`) ↔ `BARCODE_ENDPOINTS.ATTACH` (`frontend/core/product/src/api/endpoints.ts`), `barcodesApi.attach` (`frontend/core/product/src/api/barcodes.api.ts`).
- UV5. Query values are strings. `apiClient.get` takes `Record<string, string>`, so callers wrap numbers in `String()`. Ex: `GET /api/v1/reminders?page&size` (`backend/core/reminders/src/main/java/com/inventory/reminders/rest/controller/ReminderController.java`) ↔ `remindersApi.getAll` (`frontend/core/reminders/src/api/reminders.api.ts`).

## Status codes

- SC1. Every success is 200 with an `ApiResponse<T>` body, for every verb. Controllers call `ResponseEntity.ok` 216 times and never send 201 or 204. The client resolves any 2xx, and api methods return `response.data`. Ex: `POST /api/v1/vendors` (`VendorController.java`) ↔ `vendorsApi.create`.
- SC2. Error statuses come only from `ErrorCode` through `GlobalExceptionHandler` (backend AGENTS.md E3). The client reacts in `frontend/platform/api-client/src/lib/client.ts` as below. Ex: `GET /api/v1/vendors/{vendorId}` → 404 `ResourceNotFoundException` ↔ `vendorsApi.getById` throws `ApiError`.

| Status | Backend source | Client reaction |
|---|---|---|
| 400 | `ValidationException`, `@Valid`, malformed JSON | throws `ApiError` |
| 401 | missing, invalid or expired token; no shop context (backend AGENTS.md E7) | `onUnauthorized` (`frontend/platform/shell/src/lib/AuthInitializer.tsx`) clears the session and goes to `/login`. Skipped when `isPublicAuthRequest` matches the URL. |
| 402 | `PlanExpiryInterceptor` (`ErrorCode.PLAN_EXPIRED`) | `onPlanExpired` (`frontend/platform/shell/src/guards/DashboardRouteGuard.tsx`) goes to `/dashboard/plan-status` unless `isPlanExpiryAllowedPath` |
| 403 | `ErrorCode.ACCESS_DENIED` (`RbacService`) | throws `ApiError` |
| 404 | `ResourceNotFoundException` | throws `ApiError`. Some callers map it to `null`, e.g. `customersApi.searchByPhone` on `GET /api/v1/customers/search`. |
| 409, 422, 500 | `ResourceExistsException`, `GstConfigurationException`, any other exception | throws `ApiError` |
| 410, 429 | Attendance app and Stockkart Team app only: `HR_PAIRING_CODE_EXPIRED` (410); `HR_TOO_MANY_ATTEMPTS`, `HR_PAIRING_CODE_LOCKED` (429), see ER4 and ER5 | never reaches `apiClient`; the app's own `fetch` clients read `data.message` (AU7, AU9) |
| 0 | no response | throws `ApiError` "Network error. Please check your connection." |

- SC3. Use 401 only for "who are you". Deny access with 403 (`ErrorCode.ACCESS_DENIED`). Reason: any 401 logs the user out. Ex: `GET /api/v1/shops/{shopId}/rbac` → `RbacService.requireOwner` (`backend/core/user/src/main/java/com/inventory/user/service/RbacService.java`) ↔ `shopAccessApi.getAdmin` (`frontend/core/user/src/api/shop-access.api.ts`).
- SC4. An endpoint that must work while the plan is expired goes in `ALLOWED_PREFIXES` or `ALLOWED_PATTERNS` of `backend/app/src/main/java/com/inventory/app/interceptor/PlanExpiryInterceptor.java`. The frontend `PLAN_EXPIRY_ALLOWED_PATHS` (`frontend/platform/contracts/src/plan-guards.ts`) lists routes, not API paths. Ex: `GET /api/v1/plans/shop/status` (`backend/core/plan/src/main/java/com/inventory/plan/rest/controller/PlanController.java`) ↔ `PLAN_ENDPOINTS.SHOP_STATUS` (`frontend/core/plan/src/api/endpoints.ts`), `frontend/core/plan/src/api/plans.api.ts`.

Spring MVC framework errors return ProblemDetail (backend AGENTS.md E8). The client finds no message field there, so `ApiError.message` is the HTTP status text.

## Pagination, filtering and sorting

Counts are over the 24 endpoints that take `page`.

- PG1. `page` is 0-based (20 of 24). Ex: `GET /api/v1/inventory` (`backend/core/product/src/main/java/com/inventory/product/rest/controller/InventoryController.java`, default `0`) ↔ `inventoryApi.getAll(page = 0, size = 10)` (`frontend/core/product/src/api/inventory.api.ts`).
- PG2. The page size is `size` (18 of 24). `limit` only caps a list that has no pages: `GET /api/v1/barcodes`, `/inventory/search/values`, `/purchases/customer-product-history`. Ex: `GET /api/v1/reminders` ↔ `remindersApi.getAll(page, size)`.
- PG3. A paged response carries `page`, `size`, `totalItems` and `totalPages`. Each name is the majority (`totalItems` 16, `totalPages` 20 of 24). New endpoints nest them as `PageMeta` under the key `page`. The frontend type is `PageMeta` in `frontend/platform/contracts/src/notification.types.ts`. Ex: `GET /api/v1/inventory` → `InventoryListResponse.page` ↔ `PaginationInventoryResponse.page: PageMeta` (`frontend/core/product/src/model/types.ts`).
  - Backend copies: `backend/core/product/src/main/java/com/inventory/product/rest/dto/response/PageMeta.java` and `backend/core/reminders/src/main/java/com/inventory/reminders/rest/dto/response/PageMeta.java`. A module with no pom edge to product (backend AGENTS.md L1) copies the 4-field class.
  - Flat fields (accounting, credit, HR `JoinRequestsPageResponse {items, pendingCount, page, size, totalItems, totalPages}` on `GET /api/v1/hr/join-requests`) and the key `meta` (reminders) also exist. They follow PG3.
- PG4. The free-text filter is `q`. Ex: `GET /api/v1/vendors?q=` ↔ `vendorsApi.list({ q })`.
- PG5. A date range is `from` and `to`, ISO dates `yyyy-MM-dd` (`@DateTimeFormat(iso = DATE)`, 34 uses). A single day is `asOf`. A GST month is `period` as `YYYY-MM`. Ex: `GET /api/v1/accounting/reports/profit-and-loss` (`backend/core/accounting/src/main/java/com/inventory/accounting/rest/controller/AccountingController.java`) ↔ `accountingApi.profitAndLoss(from, to)` (`frontend/core/accounting/src/api/accounting.api.ts`).
- PG6. A multi-value filter is one comma-separated parameter. Ex: `txnTypes` on `GET /api/v1/mis/customer-money` (split in `backend/core/analytics/src/main/java/com/inventory/analytics/mis/service/CustomerMoneyMisService.java`) ↔ `join(',')` (`frontend/core/mis/src/api/mis.api.ts`).
- PG7. Sort is `sort=<field>:<asc|desc>` (1 of the 2 sorting endpoints). The server allow-lists the fields and sets the default. The body field `sort` of `POST /api/v1/inventory/search` uses the same format. Ex: `GET /api/v1/inventory/search` (`backend/core/product/src/main/java/com/inventory/product/utils/InventorySearchQueryParser.java`, `backend/core/product/src/main/java/com/inventory/product/search/SearchRequestValidator.java`) ↔ `queryParams.sort` in `inventoryApi.search`.

## Formats (dates, money, IDs, enums)

- FM1. Dates use Jackson defaults. `Instant` is an ISO-8601 UTC string. `LocalDate` is `yyyy-MM-dd`. Main code has no `spring.jackson.*` property, no `ObjectMapper` bean and no `@JsonFormat`. The frontend types dates as `string`. Ex: `ReminderResponse.reminderAt` (`Instant`) on `GET /api/v1/reminders` ↔ `Reminder.reminderAt: string` (`frontend/core/reminders/src/model/types.ts`).
- FM2. Money is `BigDecimal` in DTOs (578 fields) and a JSON number on the wire. The frontend types it as `number`. The server computes totals and tax; the client displays them (`backend/README.md`, "Previewing a bill before stock-in"). Ex: `POST /api/v1/vendor-purchase-invoices/preview-totals` → `backend/core/product/src/main/java/com/inventory/product/rest/dto/response/PurchaseTaxPreviewResponse.java` ↔ `PurchaseTaxPreviewResponse` (`frontend/core/product/src/model/types.ts`).
  - The GSTR-1 offline-return file (`Gstr1PortalReturnDto`) follows the GST portal's JSON schema by design.
- FM3. IDs are opaque strings. Generated document IDs are 24-hex Mongo ObjectIds. The interceptors match shop and plan IDs in paths with `[a-fA-F0-9]{24}`. The client never parses or builds an ID. Embedded line IDs are UUIDs (shop menu, stock-correction lines). Ex: `GET /api/v1/vendors/{vendorId}` ↔ `vendorsApi.getById`.
- FM4. An enum travels as the Java constant name (UPPER_SNAKE). The frontend mirrors it as a same-named string union with every value. Ex: `AccountType` (`backend/core/accounting/src/main/java/com/inventory/accounting/domain/model/AccountType.java`) on `GET /api/v1/accounting/accounts` ↔ `AccountType` (`frontend/core/accounting/src/model/types.ts`), `accountingApi.accounts`.
  - Lower-case wire names by design:
    - the 9 product search and label enums with `@JsonValue` (`backend/core/product/src/main/java/com/inventory/product/search/FilterOp.java` and its siblings) ↔ `FilterOp` (`frontend/core/product/src/model/search.types.ts`);
    - `ShopSchemaResponse.mode`, lower-cased in `backend/core/product/src/main/java/com/inventory/product/service/vertical/VerticalSchemaService.java`;
    - `MenuSellMode`, whose constants are lower-case.
- FM5. `ApiResponse`, `ApiError` and 7 DTOs in use omit null fields (`@JsonInclude(NON_NULL)`). Other DTOs send `null`. Type an optional field as `?: T | null` either way. The backend ignores unknown request fields (Spring Boot Jackson default). Ex: `taxTreatmentSource` in `PurchaseTaxPreviewResponse` ↔ `taxTreatmentSource?: … | null`.

## Error response shape

- Producer: `backend/core/common/src/main/java/com/inventory/common/advice/GlobalExceptionHandler.java`. It puts an `ApiError` (`backend/core/common/src/main/java/com/inventory/common/dto/response/ApiError.java`, built by `ApiErrorMapper.java`) into `data` of `ApiResponse.error(message)` (`backend/core/common/src/main/java/com/inventory/common/dto/response/ApiResponse.java`). Full JSON: backend AGENTS.md E2.
- Consumer: the response interceptor in `frontend/platform/api-client/src/lib/client.ts`. It throws `ApiError` (`frontend/platform/api-client/src/lib/ApiError.ts`). Callers test with `isApiError`.

| Wire field | Sent | Client reads it as |
|---|---|---|
| `data.message` | always | `ApiError.message`, first choice |
| `error` | always | `ApiError.message`, second choice |
| `data.status` | always | not read; `ApiError.status` is the HTTP status |
| `data.errors` | validation failures only | not read; the client reads a top-level `errors`, so `ApiError.errors` is always undefined |
| `code` | never (`ErrorCode.code` is not serialized) | `ApiError.code` is always undefined |
| ProblemDetail `detail` | Spring MVC errors (E8) | not read; the message falls back to the status text |

- ER1. Throw from the service or validator, and let `GlobalExceptionHandler` build the body. Never build `ApiResponse.error` in a controller. Ex: `POST /api/v1/vendors`: `VendorValidator` (`backend/core/user/src/main/java/com/inventory/user/validation/VendorValidator.java`) throws `ValidationException` → 400 ↔ `vendorsApi.create` rejects with `ApiError.message`.
- ER2. Show `ApiError.message`. For a `ValidationException` it already joins the messages with "; ". Do not depend on `ApiError.errors` or `ApiError.code` while `client.ts` reads them from the top level. Ex: `isApiError(err) && err.status === 400` for `GET /api/v1/inventory` (`frontend/core/product/src/api/inventory.api.ts`).
- ER3. Use `ApiError` and `isApiError` from `@inventory-platform/api-client`. The `ApiError` interface and `PaginatedResponse` in `frontend/platform/contracts/src/index.ts` are unused and match no endpoint. Ex: `frontend/core/product/src/api/inventory.api.ts` imports `isApiError` for `GET /api/v1/inventory`.
- ER4. HR `ErrorCode` entries are 8000–8099 (`backend/core/common/src/main/java/com/inventory/common/constants/ErrorCode.java`); 8022–8034 belong to the Attendance app and 8035–8047 to the Stockkart Team app (ER5). The app shows `data.message` word for word, so a message change is a contract change (CH3), and the ones a worker cannot fix end with "Ask your manager to mark your attendance." (HR R7.10). `HR_PAIRING_CODE_LOCKED` is the one message with a `%d` placeholder (minutes to wait). Ex: `POST /api/v1/hr/attendance-app/punches` (`backend/core/hr/src/main/java/com/inventory/hr/rest/controller/AttendanceAppController.java`) ↔ `app/src/api/appClient.ts`, texts mirrored in both protocol copies, `frontend/platform/attendance-protocol/src/messages.ts` and `app/packages/attendance-protocol/src/messages.ts` (parity-checked).

| Code | Name | Status | When |
|---|---|---|---|
| 8022 | `HR_APP_OFF` | 409 | Biometric app check-in is off for the shop |
| 8023 | `HR_APP_NOT_PAIRED` | 403 | pairing unknown, unpaired or replaced; the app wipes its storage |
| 8024 | `HR_APP_NOT_VERIFIED` | 422 | envelope or punch signature fails, or a resent punch id with different content |
| 8025 | `HR_APP_TOO_LATE` | 422 | offline punch older than 24 h |
| 8026 | `HR_APP_CLOCK_WRONG` | 422 | offline punch more than 5 min ahead of server time |
| 8027 | `HR_FAKE_LOCATION` | 422 | mock location under Location check **Block** |
| 8028 | `HR_TOO_MANY_ATTEMPTS` | 429 | over 20 punches per pairing per 60 s, or the 11th attempt on one code |
| 8029 | `HR_PAIRING_CODE_EXPIRED` | 410 | code older than 15 min |
| 8030 | `HR_PAIRING_CODE_USED` | 409 | code already completed a pairing |
| 8031 | `HR_PAIRING_CODE_CANCELLED` | 409 | code replaced by a newer one, or abandoned |
| 8032 | `HR_PAIRING_CODE_WRONG` | 404 | no code matches |
| 8033 | `HR_PAIRING_EMPLOYEE_EXITED` | 422 | the code's employee is `EXITED` |
| 8034 | `HR_PAIRING_CODE_LOCKED` | 429 | 5 refused codes from one installation in 15 min; locked 15 min |

- ER5. 8035–8047 belong to the Stockkart Team app (spec `worker-app`). The app shows them word for word on the worker paths of AU9, the manager inbox shows 8037–8040 word for word in its drawer, and the row stays Pending on a refused decision. `HR_MATCH_LINKED_ELSEWHERE` and `HR_MATCH_EXITED` carry a `%s` placeholder (the employee code). `HR_TOO_MANY_ATTEMPTS` (8028) keeps its code but has a second text on the worker paths: "Too many attempts, try again later" (`ErrorCode.HR_TOO_MANY_ATTEMPTS_WORKER_MESSAGE`, passed as the exception message); the punch and pairing paths keep "Too many attempts, wait a minute". 8042, 8043 and 8046 are unused: they belonged to the removed legacy pairing flows (worker-app D11) and the gaps are kept so the other codes do not move. The exact messages are in the backend README, "Stockkart Team (worker-app)". Ex: `POST /api/v1/hr/worker/join-requests` (`backend/core/hr/src/main/java/com/inventory/hr/rest/controller/WorkerController.java`) ↔ `app/src/api/accountClient.ts`, texts mirrored in both protocol copies, `frontend/platform/attendance-protocol/src/worker-messages.ts` and `app/packages/attendance-protocol/src/worker-messages.ts` (parity-checked).

| Code | Name | Status | When |
|---|---|---|---|
| 8035 | `HR_WORKER_HAS_SHOP` | 409 | join request from an account that already has a shop or a linked `ACTIVE` employee |
| 8036 | `HR_WORKER_REQUEST_PENDING` | 409 | a second join request while one is open (also enforced by the partial unique index `user_open_unique` on `hr_join_requests`) |
| 8037 | `HR_WORKER_LOGIN_IN_OTHER_SHOP` | 409 | approval that would give the account a second shop (`WorkerAccountService.claimShop` lost); the transaction rolls back |
| 8038 | `HR_REQUEST_ALREADY_HANDLED` | 409 | approve, reject or withdraw on a request that is no longer Pending |
| 8039 | `HR_MATCH_LINKED_ELSEWHERE` | 409 | "Link to existing" on an employee linked to another login; `%s` = employee code |
| 8040 | `HR_MATCH_EXITED` | 422 | "Link to existing" on an `EXITED` employee; `%s` = employee code |
| 8041 | `HR_PAIRING_WRONG_EMPLOYEE` | 403 | Bearer `pairings/claim` or `complete` with a code issued for another employee; the code stays valid, one attempt is counted on it |
| 8044 | `HR_TWO_CREDENTIALS` | 400 | a worker-path request carrying `X-Pairing-Id` or `X-Request-Signature`, or a device-path request carrying `Authorization` (AU9) |
| 8045 | `HR_WORKER_NOT_ACTIVE` | 409 | worker pairing call from an account with no linked `ACTIVE` employee |
| 8047 | `HR_WORKER_ONLY` | 403 | staff account with no linked employee on any worker path but `GET /api/v1/hr/worker/me` |
| 8028 | `HR_TOO_MANY_ATTEMPTS` | 429 | worker paths: the 11th shop lookup per account per hour, the 6th join request per account per 24 h, the 21st `source = worker-app` sign-up per IP per hour; text "Too many attempts, try again later" |
| 4004 | `OAUTH_ACCOUNT_NOT_FOUND` | 401 | user range, not HR: `POST /api/v1/auth/login` with a Google ID token whose e-mail has no account (worker-app DD20). Text "No account found for this Google account. Please sign up first.", unchanged from before the code existed, so the web is unaffected; the Stockkart Team app infers the code from the exact text (`worker-messages.ts`) and opens its phone step in sign-up mode with the same token. A wrong password and a token for a foreign `aud` stay `INVALID_CREDENTIALS` |

Blob downloads get the error body as a `Blob` and skip this mapping (the download helpers in frontend AGENTS.md "State and data fetching").

## Authentication and authorization between frontend and backend

```mermaid
sequenceDiagram
  participant P as page / api method
  participant C as apiClient
  participant A as AuthenticationInterceptor
  participant R as RbacModuleInterceptor
  participant X as PlanExpiryInterceptor
  participant S as Controller / Service
  participant H as GlobalExceptionHandler
  P->>C: call
  C->>A: Authorization: Bearer, X-Shop-Id
  A->>R: sets userId, shopId (else 401)
  R->>X: module allowed (else 403)
  X->>S: plan active or allow-listed (else 402)
  S-->>C: 200 ApiResponse
  S->>H: exception
  H-->>C: status + error envelope
  C-->>P: data, or 401 onUnauthorized, 402 onPlanExpired, else ApiError
```

- AU1. `POST /api/v1/auth/login` and `/signup` return `accessToken`. It is an opaque UUID that lives 7 days, stored in `UserAccount.tokens` (`backend/core/user/src/main/java/com/inventory/user/mapper/UserMapper.java`). The client stores it in localStorage `auth_token` and sends `Authorization: Bearer`. A `refreshToken` is returned, but no refresh endpoint exists. Ex: `backend/core/user/src/main/java/com/inventory/user/rest/controller/AuthController.java` ↔ `authApi.login` (`frontend/platform/session/src/api/auth.api.ts`).
  - Both take two optional fields since the Stockkart Team app (CH2): `source` (`"web"` default, `"worker-app"`; tags `inventory_user_auth_total{source}` and the login or welcome e-mail) and, on `/signup`, `role: "WORKER"` (AU8). A WORKER sign-up requires `phone` on the password and the Google path alike; a `source = worker-app` sign-up is admitted by the per-IP limit first (AU9). The web never sends either field. `/login` with a Google token whose e-mail has no account answers 401 `OAUTH_ACCOUNT_NOT_FOUND` (4004, ER5 table) with the same text as before. Ex: `AuthController.java` ↔ `AccountClient.createAccount` (`app/src/api/accountClient.ts`).
  - Google ID tokens are accepted only when one `aud` value is in `google.oauth.client-ids` (env `GOOGLE_OAUTH_CLIENT_IDS`: the web client id and the Stockkart Team app's Android client id). The rule fails closed: an empty list refuses every Google token with `INVALID_CREDENTIALS` and logs a warning at start-up (`backend/core/user/src/main/java/com/inventory/user/externalservice/GoogleTokenVerifier.java`).
- AU2. Every `apiClient` call sends `X-Shop-Id` from localStorage `x_shop_id`, which `useAuthStore` sets. The backend uses it if the user is a member. Otherwise it silently uses the active shop (backend AGENTS.md L6). Ex: `POST /api/v1/users/me/active-shop` (`backend/core/user/src/main/java/com/inventory/user/rest/controller/UserShopController.java`) ↔ `useAuthStore.switchActiveShop` (`frontend/platform/session/src/lib/useAuthStore.ts`).
- AU3. Register a public path in both backend lists (backend AGENTS.md X3). Add a public auth route to `isPublicAuthRequest` in `client.ts` too, so a 401 there does not log the user out. Ex: `POST /api/v1/auth/forgot-password` (`AuthController.java`) ↔ `AUTH_ENDPOINTS.FORGOT_PASSWORD` (`frontend/platform/session/src/api/endpoints.ts`).
  - The Attendance app device paths under `/api/v1/hr/attendance-app/` are public to the Bearer chain as **patterns, not a prefix**: `AuthenticationInterceptor.PUBLIC_ENDPOINT_PATTERNS` and `CorsConfig.bearerChainExclusions()` (from `AttendanceAppAuthInterceptor.DEVICE_PATH_PATTERNS`: `pairings/me/**`, `me`, `punches`, `consent`, `consent/**`). `GET /api/v1/hr/attendance-app/health` is not in them and stays on Bearer + `VIEW_ATTENDANCE`; `pairings/claim`, `/complete`, `/abandon` are not in them either, so with no handler left (AU9) they fall to the Bearer chain as unknown `/api` routes. A new device path is added to all three lists in one PR. No `isPublicAuthRequest` entry: the web client never calls these paths.
- AU7. Exception to AU1 and AU2: the device paths carry no `Authorization` and no `X-Shop-Id` (AU9). On the device paths of AU3, `AttendanceAppAuthInterceptor` (`backend/core/hr/src/main/java/com/inventory/hr/service/AttendanceAppAuthInterceptor.java`, registered in `CorsConfig` on exactly `DEVICE_PATH_PATTERNS`) replaces the Bearer, RBAC and plan-expiry interceptors (the paths are excluded from all three; plan expiry does not apply to them, HR DD12). Ex: `POST /api/v1/hr/attendance-app/punches` (`backend/core/hr/src/main/java/com/inventory/hr/rest/controller/AttendanceAppController.java`) ↔ `app/src/api/appClient.ts`.
  - Every device request except `GET .../consent/latest` carries a signed envelope: headers `X-Pairing-Id` (24-hex), `X-Request-Timestamp` (epoch seconds, within ±10 min of server time), `X-Request-Nonce` (UUID, single use per pairing for 15 min, `hr_app_nonces`) and `X-Request-Signature` (Base64 DER ECDSA P-256/SHA-256 over the `SKAT-REQ-1` lines of `backend/core/hr/src/main/java/com/inventory/hr/utils/AppSignatures.java` ↔ `frontend/platform/attendance-protocol/src/canonical.ts`: pairing id, method, path, raw query, timestamp, nonce, SHA-256 of the exact body bytes), made with the pairing's hardware-bound, non-exportable device key. The server holds public keys only; there is no shop-wide or app-wide secret (HR R9.8).
  - Tenant and identity come from the pairing: the interceptor sets the request attributes `pairingId`, `shopId`, `employeeId` (same names as the Bearer chain, so `HrRequestUtils.requireShopId` works) and the controller never reads them from the body (B11). Every response carries `X-Server-Time` (epoch seconds).
  - Failure statuses: 403 `HR_APP_NOT_PAIRED` when the pairing is unknown or `UNPAIRED` (the app wipes its storage), else 422 `HR_APP_NOT_VERIFIED`. Not 401: there is no session to end. Both count as `inventory_hr_app_punches_total{result="signature_failed"}`. A device request that also carries an `Authorization` header is 400 `HR_TWO_CREDENTIALS` before any handler and counts as `signature_failed` too (the device half of AU9).
  - Punches carry a second signature inside the body (`SKAT-PUNCH-1`, made with the biometric-gated punch key); that one is verified in `AppPunchService`, not in the interceptor.
- AU8. `WORKER` is the sixth `UserRole` (`backend/core/user/src/main/java/com/inventory/user/domain/model/UserRole.java`): the login of a *Stockkart Team* account. It is set only by `POST /api/v1/auth/signup` with `role: "WORKER"`; add-user, update-role, invitations and staff join requests refuse it ("WORKER is set only by approving a worker's join request"). The account's one shop is `UserAccount.shopId`, written by HR when a manager approves the worker's join request (`WorkerAccountService.claimShop`, one conditional update) and cleared on exit or "Leave this shop"; a `WORKER` membership row (`relationship = INVITED`) is written beside it so every existing membership check sees the worker. `RbacService` resolves a WORKER membership to every module off, HR included, and no team flag but `viewMyInvitations`, so `/api/v1/shops/me/access` answers `hr.permissions = []` and the `hr.employeeId` when linked. Workers are not plan seats (`staffCountForShop`). Ex: `GET /api/v1/shops/{shopId}/users` rows carry `email` and, for WORKER rows, `linkedEmployeeCode` (`UserDto`, `backend/core/user/src/main/java/com/inventory/user/rest/dto/response/UserDto.java`) ↔ the Users page "Workers" group (`frontend/core/user`).

| Caller | Web (`apps/inventory`) | Stockkart Team app (`stockkart-team-app`, Bearer) | Stockkart Team app (device envelope) | `/api/v1/hr/join-requests/**` | other `/api/v1/hr/**` and shop modules |
|---|---|---|---|---|---|
| WORKER | My HR (`/dashboard/hr/me`) only; the login redirect sends the role there, or to "not part of a shop yet" when `/shops/me/access` has no `employeeId` | `/api/v1/hr/worker/**`, `/api/v1/users/me/profile`, `/api/v1/users/me/photo`, `/api/v1/users/{userId}/photo`, `/api/v1/auth/*` | `/api/v1/hr/attendance-app/*` once paired, as before | 403 (module `hr` off) | 403 (`RbacModuleInterceptor`) |
| Staff with a linked employee | as today | all of the above (`GET /api/v1/hr/worker/me` answers `ACTIVE`) | as before | per HR permission | per role and HR permission |
| Staff without a linked employee | as today | `GET /api/v1/hr/worker/me` → `STAFF_NO_EMPLOYEE`; every other worker path 403 `HR_WORKER_ONLY` | as before | per HR permission | per role and HR permission |

- AU9. One request, one credential. The Stockkart Team app has two clients that never share headers: `AccountClient` sends `Authorization: Bearer` and never an envelope header; `AppClient` sends the envelope of AU7 and never `Authorization`. The server refuses a mix on both chains with 400 `HR_TWO_CREDENTIALS` before any handler runs (ER5): `WorkerCredentialInterceptor` (`backend/core/hr/src/main/java/com/inventory/hr/service/WorkerCredentialInterceptor.java`, registered in `CorsConfig` on `/api/v1/hr/worker/**` ahead of the Bearer chain) refuses a request carrying `X-Pairing-Id` or `X-Request-Signature` before the token is read; `AttendanceAppAuthInterceptor` refuses a device request carrying `Authorization` (AU7). A login and a pairing meet on the server exactly once: when a pairing code is redeemed on `POST /api/v1/hr/worker/pairings/claim`, `/complete`, `/abandon` with the Bearer token, where the server checks that the code's employee is the caller's linked `ACTIVE` employee (ER5, 8041 and 8045). A pairing code is redeemed only on `/hr/worker/pairings/*` with the Bearer token; a punch never needs an account session (R9.10). Ex: `POST /api/v1/hr/worker/pairings/claim` (`WorkerController.java`) ↔ `WorkerPairingClient` (`app/src/pairing/WorkerPairingClient.ts`, over `AccountClient`).
  - `/api/v1/hr/worker/**` (`WorkerController`): Bearer; `RbacModuleInterceptor.SKIP_PATHS` has `/api/v1/hr/worker(/.*)?` next to `/api/v1/hr/me(/.*)?`, so a WORKER membership (module `hr` off) reaches it; `PlanExpiryInterceptor.ALLOWED_PREFIXES` has `/api/v1/hr/worker/`, so Home, state, history and profile keep working while the shop's plan is expired (SC4). Every handler derives the employee from the token's `(shopId, userId)` and never reads an employee id from the query or body (B11); any role may call `GET /me`, a staff account without a linked employee gets `HR_WORKER_ONLY` everywhere else. Rate limits (`AppRateLimiter`, in memory per instance, IN_MEMORY_STATE): `lookup:<userId>` 10 per hour on `GET /shops?ownerEmail=`, `join:<userId>` 5 per 24 h on `POST /join-requests`, `signup-ip:<ip>` 20 per hour on `POST /api/v1/auth/signup` with `source = worker-app`; a refusal is 429 `HR_TOO_MANY_ATTEMPTS` with the worker text and reads or writes nothing.
  - `/api/v1/hr/join-requests/**` (`JoinRequestInboxController`): Bearer, RBAC module `hr` on (**not** in `SKIP_PATHS`), `HrPermissionGuard.require(ADD_EDIT_EMPLOYEES)` in every handler, plan expiry applies like every staff screen. A WORKER token is 403 here. `POST /api/v1/hr/employees/{id}/request-profile` ("Request profile details") sits on `EmployeeController` behind the same permission.
  - `PUT /api/v1/users/me/profile`, `PUT`/`DELETE /api/v1/users/me/photo`, `GET /api/v1/users/{userId}/photo` (`WorkerAccountController`, `backend/core/user/src/main/java/com/inventory/user/rest/controller/WorkerAccountController.java`): Bearer; the first three act on the token's user only; the photo is read by the owner, an approver of a shop where the owner has a Pending or recently Rejected request, or an active co-member, 403 before 404; never a public URL. `PUT /users/me/profile` takes `{name?, phone}`: `phone` required, `name` absent or blank = unchanged, for any role and any shop state (the app's "Add your phone number" step sends the phone alone, worker-app DD20). No RBAC module rule matches `/api/v1/users`; these paths are not in `ALLOWED_PREFIXES`, so they follow the plan check once the account has a shop.
  - There is no unsigned pairing handshake: the HR-module `POST /api/v1/hr/attendance-app/pairings/claim`, `/complete` and `/abandon` handlers are removed (worker-app D11, DD4). Those paths are not device paths (AU3), so they are ordinary unknown `/api` routes (401 without a token, 404 with one) and read or count nothing. Punches never depend on a session (R9.10).
  - Sessions are the 7-day tokens of AU1 with no refresh. The app treats a 401 on a Bearer call as "sign in again" and keeps the pairing, the offline queue and the last state; a session ending can never stop a waiting punch, because the device paths never see the token.
- AU4. The backend enforces RBAC (backend AGENTS.md L7; `RbacModuleInterceptor`, `RbacService` → 403). The frontend only hides UI. Module keys are the `RbacService.MODULE_*` strings. A new module key or capability changes both sides in one design. Ex: `GET /api/v1/shops/me/access` → `backend/core/user/src/main/java/com/inventory/user/rest/dto/response/ShopAccessResponse.java` ↔ `ShopAccess` (`frontend/platform/access/src/shop-access.ts`); `GET /api/v1/shops/me/capabilities` → `backend/pluginengine/src/main/java/com/inventory/pluginengine/capabilities/ShopUiCapabilities.java` ↔ `frontend/platform/access/src/shop-capabilities.ts`.
- AU5. Tenant scope is the header shop (AU2). A `{shopId}` path is only for cross-shop team, RBAC or plan actions. The server checks membership or ownership of that shop before use. Ex: `GET /api/v1/shops/{shopId}/rbac` → `RbacService.requireOwner` ↔ `shopAccessApi.getAdmin`.
- AU6. Some calls bypass the `apiClient` interceptors. Ex: `GET /api/v1/events/stream` (`backend/core/reminders/src/main/java/com/inventory/reminders/rest/controller/EventController.java`) ↔ `apiClient.createSseConnection` (`frontend/platform/shell/src/api/events.api.ts`).
  - SSE sends only `Authorization`.
  - Blob downloads (`frontend/core/*/src/api/download.ts`) send both headers but get no 401 or 402 handling.
  - `/m/**` takes an upload token as a query parameter and allows any origin (`backend/app/src/main/java/com/inventory/app/config/CorsConfig.java`).
  - CORS on `/api/**` allows the origin in property `client.url` (env `CLIENT_URL`) and any header.

## Changing a contract

```mermaid
flowchart LR
  A[Contract change] --> B{Additive? CH2}
  B -- yes --> C[Backend PR] --> D[Backend release green] --> E[Frontend PR]
  B -- no, CH3 --> F[Expand: backend adds new beside old] --> G[Backend release green]
  G --> H[Switch: frontend uses new] --> I[Frontend release green]
  I --> J[Contract: backend PR removes old]
```

- CH1. Backend first. Merge the backend PR, wait until its release run is green, then merge the frontend PR (`backend/docs/deploying.md`, `frontend/docs/deploying.md`). A merge to `main` ships to production; the normal release has no approval step.
- CH2. So every backend change must also serve the frontend that is live. These changes are additive and safe: a new endpoint, a new optional request field, a new response field (FM5). Ex: the Attendance app added to existing HR endpoints only optional request fields (`reason`, `reasonText` on `PUT /api/v1/hr/attendance/days` and `POST /api/v1/hr/attendance/mark-unmarked-present`, required by the server only when the shop's switch is on and the employee has a paired phone; `paired` on `GET /api/v1/hr/employees`; `attendanceCapture.biometricApp` on `PUT /api/v1/hr/settings`), new response fields (`pairedEmployeeCount`, `appPaired`, `punchSource`, `markReasonRequired`, `noPunchYet`, `deviceModel`) and new enum values no live client switches on (`BIOMETRIC_APP` in `AttendanceSource` and `PunchSource`; `FAKE_LOCATION`, `RECORDED_OFFLINE` in `PunchFlag`), so the backend shipped first and the web `core/hr` PR followed (`backend/README.md`, HR tables; `frontend/core/hr/src/model/attendance.types.ts`).
  - The Stockkart Team app did the same: optional request fields `role: "WORKER"` and `source` on `POST /api/v1/auth/signup`, `source` on `POST /api/v1/auth/login` (AU1); response fields `email` and `linkedEmployeeCode` on `GET /api/v1/shops/{shopId}/users` rows, `profileRequestedAt` on `EmployeeResponse`, `joinRequestId` on `HrNotificationResponse`; new enum values no live client switches on (`UserRole.WORKER`; `HrNotificationType` `JOIN_REQUEST_RECEIVED`, `JOIN_REQUEST_APPROVED`, `JOIN_REQUEST_REJECTED`, `PROFILE_REQUESTED`; `HrAuditAction` `JOIN_REQUEST_APPROVED`, `JOIN_REQUEST_REJECTED`, `PAIRING_CODE_REFUSED`, `PROFILE_REQUESTED`); and `ErrorCode` 8035–8047 (ER5). Its one removal, the unsigned `POST /api/v1/hr/attendance-app/pairings/claim`, `/complete`, `/abandon` handlers (AU9), needed no CH3 steps: no client was live on them (worker-app D11). Merge order: backend (indexes, endpoints, interceptor rules) → release → web `core/hr` + web protocol copy (`frontend/platform/attendance-protocol`) + `shell` → release → app repository (`app/packages/attendance-protocol`, the same protocol change, `scripts/protocol-parity.mjs` green against the merged platform `main`) + app → build (`eas build`), which talks only to live endpoints.
- CH3. These changes are breaking: renaming or removing a field; changing a type, format, nullability, path, verb, status or page base; a new required request field; a new enum value the client switches on; moving an endpoint behind RBAC or plan expiry. Ship a breaking change as expand → switch → contract, in three PRs (diagram).
- CH4. The frontend PR changes three things together: the type in `core/<d>/src/model/` (or `platform/contracts` for shared types), the constant in `endpoints.ts`, and the method in `<name>Api` (frontend AGENTS.md "State and data fetching"). Name the TS type after the backend DTO, as `PurchaseTaxPreviewResponse` does. The backend PR updates the README endpoint tables (backend AGENTS.md X4) and the access lists (A9).
- CH5. A multi-repo design's `## Cross-repo contract` section has one block per endpoint: a 2-column table (Item | Value) with these rows.
  - Endpoint: verb and full path.
  - Change class: new, additive, or breaking with the CH3 steps.
  - Request: query parameters and body fields, typed per FM1–FM5.
  - Response: `ApiResponse<T>` and the fields of `T`.
  - Errors: status → cause.
  - Access: public, RBAC module, plan-expiry allow-list.
  - Backend files: controller, DTOs.
  - Frontend files: constant, api method, type.
  - Merge order (CH1).
  - Deviations: rule ID and reason.
- CH6. New code follows these rules. Do not add rows to the table below. When a fix lands, remove its row in a handbook PR.

## Non-compliant endpoints today

Rows come from the commands under the table or from the one-off scan noted after them. The table is complete for those checks only. FM4 compares same-named unions only; inline or renamed unions are not checked.

| Endpoint | Rule broken | Path |
|---|---|---|
| `POST /api/v1/session/create-upload-token`, `GET /api/v1/upload/status`, `GET /api/v1/upload/parsed-items`, `/m/upload/**` | UV1 no class path, singular nouns; UV3 verb in path | `backend/core/product/src/main/java/com/inventory/product/rest/controller/QRUploadController.java` |
| `/api/v1/shops/{shopId}/users/**` | UV1 bare `@RequestMapping` | `backend/core/user/src/main/java/com/inventory/user/rest/controller/ShopUserController.java` |
| `/api/v1/cart/**`, `/api/v1/estimates/**`, `/api/v1/purchases/**` | UV1 class path is bare `/api/v1` | `backend/core/product/src/main/java/com/inventory/product/rest/controller/CheckoutController.java` |
| `/api/v1/shops/{shopId}/invitations`, `/api/v1/invitations/{invitationId}/accept`, `/api/v1/users/invitations`, `/api/v1/shops/{shopId}/users/all` | UV1 bare `/api/v1` | `backend/core/user/src/main/java/com/inventory/user/rest/controller/InvitationController.java` |
| `/api/v1/shops/by-owner-email`, `/api/v1/shops/join-request`, `/api/v1/shops/join-requests/**` | UV1 bare `/api/v1` | `backend/core/user/src/main/java/com/inventory/user/rest/controller/JoinRequestController.java` |
| `/api/v1/refund/**` | UV1 singular noun | `backend/core/product/src/main/java/com/inventory/product/rest/controller/RefundController.java` |
| `POST /api/v1/cart/upsert` | UV3 | `backend/core/product/src/main/java/com/inventory/product/rest/controller/CheckoutController.java` |
| `POST /api/v1/pricing/bulk-update` | UV3 | `backend/core/pricing/src/main/java/com/inventory/pricing/rest/controller/PricingController.java` |
| `GET`, `PATCH /api/v1/shops/{shopId}` | UV4 no backend mapping; the shop edit modal calls `shopsApi.getShop` on open and `updateShop` on save | `frontend/core/user/src/api/shops.api.ts`, `frontend/platform/shell/src/lib/DashboardLayout.tsx` |
| `POST /api/v1/checkout`, `POST /api/v1/auth/refresh`, `POST /api/v1/auth/accept-invite` | UV4 no backend mapping; `checkoutApi.create`, `refreshToken`, `acceptInvite` have no caller | `frontend/core/product/src/api/checkout.api.ts`, `frontend/platform/session/src/api/auth.api.ts` |
| `POST /api/v1/inventory/bulk` | ER1 returns `ApiResponse.error` 400 with no `data` | `backend/core/product/src/main/java/com/inventory/product/rest/controller/InventoryController.java` |
| `POST /m/upload` | ER1 builds its own 400 and 500; the 500 body carries the exception message | `backend/core/product/src/main/java/com/inventory/product/rest/controller/QRUploadController.java` |
| `POST`, `GET /api/v1/shops/{shopId}/invitations`; `GET /api/v1/shops/{shopId}/users/all` | SC3 401 on denied access | `backend/core/user/src/main/java/com/inventory/user/rest/controller/InvitationController.java` |
| `GET`, `PATCH`, `DELETE /api/v1/shops/{shopId}/users/**` | SC3 401 on denied access | `backend/core/user/src/main/java/com/inventory/user/rest/controller/ShopUserController.java` |
| `PUT /api/v1/shops/active-shop/card-layouts/{surfaceId}`, `PUT /api/v1/shops/active-shop/barcode-label-layout` | ER2 the client reads top-level `errors`; the backend sends `data.errors` | `frontend/platform/api-client/src/lib/client.ts`, `frontend/core/product/src/ui/cardLayout/ProductCardLayoutSection.tsx`, `frontend/core/product/src/ui/labelLayout/BarcodeLabelLayoutSection.tsx` |
| `GET /api/v1/purchases`, `/purchases/search`, `/refund`, `/vendor-purchase-returns` | PG1 1-based; PG2 `limit`; PG3 `limit`/`total` | `backend/core/product/src/main/java/com/inventory/product/service/CheckoutService.java`, `backend/core/product/src/main/java/com/inventory/product/service/RefundService.java`, `backend/core/product/src/main/java/com/inventory/product/service/VendorPurchaseReturnService.java`, `backend/core/product/src/main/java/com/inventory/product/rest/dto/response/PurchaseListResponse.java`, `backend/core/product/src/main/java/com/inventory/product/rest/dto/response/RefundListResponse.java`, `backend/core/product/src/main/java/com/inventory/product/rest/dto/response/VendorPurchaseReturnListResponse.java` |
| `GET /api/v1/customers`, `/vendors` | PG2 `limit`; PG3 `limit`/`total` | `backend/core/user/src/main/java/com/inventory/user/rest/dto/response/CustomerListResponse.java`, `backend/core/user/src/main/java/com/inventory/user/rest/dto/response/VendorListResponse.java` |
| `GET /api/v1/estimates`, `/stock-entry-estimates` | PG3 `total` instead of `totalItems` | `backend/core/product/src/main/java/com/inventory/product/rest/dto/response/EstimateListResponse.java`, `backend/core/product/src/main/java/com/inventory/product/rest/dto/response/StockEntryEstimateListResponse.java` |
| `GET /api/v1/mis/customer-money`, `/sales`, `/stock`, `/vendor-money` | PG3 no `totalPages` | `backend/core/analytics/src/main/java/com/inventory/analytics/mis/rest/dto/Mis*ReportResponse.java` |
| `GET /api/v1/inventory/lots` | PG4 `search` instead of `q` | `backend/core/product/src/main/java/com/inventory/product/rest/controller/InventoryController.java` |
| `GET /api/v1/analytics/customers`, `/profit`, `/sales`, `/vendors` | PG5 `startDate`/`endDate` as `Instant` | `Customer`, `Profit`, `Sales`, `VendorAnalyticsController.java` in `backend/core/analytics/src/main/java/com/inventory/analytics/rest/controller/` |
| `GET /api/v1/purchases` | PG7 `order` instead of `sort` | `backend/core/product/src/main/java/com/inventory/product/rest/controller/CheckoutController.java` |
| `GET /api/v1/taxation/gstr3b` | FM2 money as `Double` (40 fields) | `backend/core/taxation/src/main/java/com/inventory/taxation/rest/dto/Gstr3bReportResponse.java` |
| `GET /api/v1/reminders` (`status`) | FM4 union lacks `SENT`, `CANCELLED`, `SNOOZED` | `frontend/core/reminders/src/model/types.ts`, `frontend/platform/contracts/src/notification.types.ts` |
| `GET /api/v1/accounting/journal-entries` (`sourceType`) | FM4 union lacks `VENDOR_CREDIT_CHARGE`, `CUSTOMER_CREDIT_CHARGE` | `frontend/core/accounting/src/model/types.ts` |
| `GET /api/v1/shops/{shopId}/users/all` (`role`) | FM4 `UserRole` lacks `OWNER`; `ShopUsersList.tsx` casts to it | `frontend/core/user/src/model/party.types.ts` |
| `POST /api/v1/shops/admin/shops/{shopId}/approve` | AU5 no admin or owner check in the controller, service or validator | `backend/core/product/src/main/java/com/inventory/product/rest/controller/ShopController.java`, `backend/core/product/src/main/java/com/inventory/product/service/ShopService.java` |
| `GET /api/v1/plans/shop/{shopId}/suggested`, `POST /api/v1/plans/shop/{shopId}/assign` | AU5 no membership check on the path shop; RBAC checks the header shop | `backend/core/plan/src/main/java/com/inventory/plan/rest/controller/PlanController.java` |
| `/api/v1/auth/accept-invite`, `/api/product/get-plugin`, `/api/product/` | AU3 public-list entries with no controller | `backend/app/src/main/java/com/inventory/app/interceptor/AuthenticationInterceptor.java`, `backend/app/src/main/java/com/inventory/app/config/CorsConfig.java` |
| `GET /api/v1/events/stream` | AU2 no `X-Shop-Id`; the backend uses the active shop | `frontend/platform/api-client/src/lib/client.ts` |

Excluded after triage: non-JSON returns (files, text, SSE; backend AGENTS.md E1), `Gstr1PortalReturnDto` (portal schema), `SchemaDisplayMode` (lower-cased on the wire), the unused `ACCOUNTING_ENDPOINTS.BASE`, `ANALYTICS_ENDPOINTS.BASE` and `CREDIT_ENDPOINTS.BASE`, and the `{shopId}` endpoints that do check (invitations, `users/**`, `rbac/**`).

Regenerate from the workspace root (one directory holding `backend/` and `frontend/`):

```sh
grep -L '@RequestMapping("/api/v1/[a-z]' $(grep -rlw '@RestController' --include=*.java backend)  # UV1
grep -rhoE '@RequestMapping\("/api/v1/[a-z-]+' --include=*Controller.java backend | sort -u  # UV1 nouns
grep -rnoE 'Mapping\((value *= *)?"[^"]*(create|update|upsert|delete)[^"]*"' --include=*Controller.java backend  # UV3
grep -rn 'ApiResponse.error\|ResponseEntity.status\|badRequest()' --include=*Controller.java backend  # ER1
grep -rn -A2 '!.*has\(Owner\)\?Access(' --include=*Controller.java backend | grep AuthenticationException  # SC3
grep -rl 'page - 1' backend/core  # PG1
grep -rn 'Integer limit' --include=*Controller.java backend  # PG2, minus the 3 capped lists
grep -rlE 'private (double|Double) ' --include=*.java backend/core | grep /dto/  # FM2
grep -rl '@JsonInclude' --include=*.java backend/core | grep -v /test/  # FM5
```

The other rows (UV4, PG3–PG7, FM4, AU2, AU3, AU5, ER2) come from a one-off cross-repo scan. UV4 diffs `*_ENDPOINTS` constants against backend mappings. FM4 diffs same-named TS unions against Java enums. Where these checks live is open for `decisions/`.
