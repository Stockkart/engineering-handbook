# 0004. OCR behind a provider and a preprocessor interface

## Status

Accepted (recorded retrospectively).
Recorded against inventory-api `ddfd9d79` and inventory-platform `780e504`.

## Context

- Invoice images are parsed into inventory line items by an external OCR or LLM service.
- Reason not recorded.

## Decision

- `OcrProvider` parses an invoice image into `ParsedInventoryItem`s for an `InvoicePricingLayout`.
- Three implementations: `AwsTextractOcrProvider`, `ChatGptOcrProvider` (OpenAI), `GeminiOcrProvider`.
- `OcrConfig` creates exactly one provider bean, selected by property `ocr.provider` with `@ConditionalOnProperty`.
- Selector values are `aws-textract`, `chatgpt`, `gemini`. `aws-textract` also matches when the property is missing.
- `ImagePreprocessor` has three implementations, selected by `ocr.preprocess.mode`.
- Mode `none` is `NoOpImagePreprocessor`. Mode `subprocess` is `SubprocessImagePreprocessor`, which runs `python -m image_preprocess`. Mode `http` is `HttpImagePreprocessor`.
- Only the ChatGPT and Gemini providers receive the preprocessor. Textract does not.
- Prompts per layout live in `core/ocr/.../prompt/`: wholesaler and retailer. Table-based providers ignore the layout.

## Consequences

- Boot fails when the chosen LLM provider has no API key.
- Boot fails when mode `http` has no preprocess URL.
- Adding a provider means a new implementation, a conditional bean and a key property.
- The preprocessing service's source is in neither repository.
- Its compose service `image-preprocess` is commented out.

## Evidence (paths)

- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/provider/OcrProvider.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/provider/impl/AwsTextractOcrProvider.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/provider/impl/ChatGptOcrProvider.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/provider/impl/GeminiOcrProvider.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/config/OcrConfig.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/preprocess/ImagePreprocessor.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/preprocess/NoOpImagePreprocessor.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/preprocess/SubprocessImagePreprocessor.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/preprocess/HttpImagePreprocessor.java`
- inventory-api: `core/ocr/src/main/java/com/inventory/ocr/prompt/InvoicePricingLayout.java`
- inventory-api: `app/src/main/resources/application.properties`
- inventory-api: `docker-compose.yml`
- Rules: [backend AGENTS.md](https://github.com/Stockkart/inventory-api/blob/main/AGENTS.md) C3 and the environment variable table.
