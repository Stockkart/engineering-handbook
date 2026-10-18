# 0009. Errors as `core/common` exceptions mapped by one GlobalExceptionHandler

## Status

Accepted (recorded retrospectively).
Recorded against inventory-api `ddfd9d79` and inventory-platform `780e504`.

## Context

- Every module reports user and system errors to the same frontend client.
- Reason not recorded.

## Decision

- Services and validators throw unchecked exceptions from `core/common/.../exception`.
- All of them extend `BaseException`, which carries an `ErrorCode` with an `HttpStatus`.
- `ValidationException` dominates: 553 throws in 75 main-source files.
- One `@RestControllerAdvice`, `GlobalExceptionHandler extends ResponseEntityExceptionHandler`, turns exceptions into `ApiResponse<ApiError>`.
- Envelope shape, status mapping, validators and framework errors: backend AGENTS.md E2, E3, E6, E8.

## Consequences

- Frontend `apiClient` reads `data.message` first, so backend messages reach the user.
- `apiClient` reads `errors` and `code` at the top level. The backend nests `errors` under `data` and sends no `code`.
- So `ApiError.errors` and `ApiError.code` are empty in the frontend today.
- Every handler that returns an envelope logs at error level, including validation failures.

## Evidence (paths)

- inventory-api: `core/common/src/main/java/com/inventory/common/advice/GlobalExceptionHandler.java`
- inventory-api: `core/common/src/main/java/com/inventory/common/exception/BaseException.java`
- inventory-api: `core/common/src/main/java/com/inventory/common/exception/ValidationException.java`
- inventory-api: `core/common/src/main/java/com/inventory/common/constants/ErrorCode.java`
- inventory-api: `core/common/src/main/java/com/inventory/common/dto/response/ApiResponse.java`
- inventory-api: `core/user/src/main/java/com/inventory/user/validation/VendorValidator.java`
- inventory-platform: `platform/api-client/src/lib/client.ts`
- inventory-platform: `platform/api-client/src/lib/ApiError.ts`
- inventory-platform: `platform/contracts/src/index.ts`
- Rules: [backend AGENTS.md](https://github.com/Stockkart/inventory-api/blob/main/AGENTS.md) E1–E8, X7.
