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
- Validators are imperative `@Component` classes.
- One `@RestControllerAdvice`, `GlobalExceptionHandler extends ResponseEntityExceptionHandler`, turns exceptions into `ApiResponse<ApiError>`.
- The body has `success: false`, `error`, and `data.{message,status,errors}`.

## Consequences

- Statuses and the envelope are as in backend AGENTS.md E2 and E3.
- Any other exception becomes 500.
- Spring MVC framework errors return ProblemDetail, not the envelope (E8).
- `ErrorCode.code` is not serialized.
- Frontend `apiClient` reads `data.message` first, so backend messages reach the user.
- `apiClient` reads `errors` and `code` at the top level. The backend nests `errors` under `data` and sends no `code`.
- So `ApiError.errors` and `ApiError.code` are empty in the frontend today.
- Handlers log every handled exception at error level, including validation failures.

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
