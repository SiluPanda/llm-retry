# llm-retry -- Task Breakdown

This file tracks all implementation tasks derived from the SPEC.md. Tasks are grouped by phase following the implementation roadmap (Section 19) with additional granularity for scaffolding, testing, documentation, and publishing.

---

## Phase 0: Project Scaffolding and Setup

- [ ] **Install dev dependencies** -- Add `typescript`, `vitest`, `eslint`, `@types/node`, and `zod` (for tests) as devDependencies. Add `zod` as an optional peerDependency (`^3.22.0`). Add `ajv` as an optional peerDependency (`^8.12.0`). Update `package.json` with `peerDependencies` and `peerDependenciesMeta` sections. | Status: not_done

- [ ] **Configure ESLint** -- Add an ESLint config file suitable for TypeScript. Ensure `npm run lint` works against `src/`. | Status: not_done

- [ ] **Configure Vitest** -- Add a `vitest.config.ts` if needed (or verify the default config works). Ensure `npm run test` runs vitest against `src/__tests__/`. | Status: not_done

- [ ] **Create directory structure** -- Create all directories specified in the file structure (Section 18): `src/repair/`, `src/validation/`, `src/feedback/`, `src/escalation/`, `src/api-retry/`, `src/__tests__/` and its subdirectories (`repair/`, `validation/`, `feedback/`, `escalation/`, `api-retry/`, `providers/`, `fixtures/`). | Status: not_done

- [ ] **Create stub files** -- Create all source files listed in Section 18 as empty TypeScript files with placeholder exports, so that cross-module imports resolve during development. | Status: not_done

- [ ] **Verify build pipeline** -- Run `npm run build` (`tsc`) and confirm it compiles the stub files into `dist/` without errors. | Status: not_done

---

## Phase 1: Core Types (types.ts)

- [ ] **Define Message interface** -- `Message` with `role: 'system' | 'user' | 'assistant'` and `content: string`. | Status: not_done

- [ ] **Define RetryContext interface** -- `RetryContext` with `attempt`, `temperature?`, `model?`, `maxTokens?`, `escalated`, and `custom?` fields as specified in Section 10. | Status: not_done

- [ ] **Define CallLLMFunction type** -- `(messages: Message[], context?: RetryContext) => Promise<string>`. | Status: not_done

- [ ] **Define ValidationError interface** -- `path`, `message`, `code`, `expected?`, `received?` fields as specified in Section 7/10. | Status: not_done

- [ ] **Define ValidationResult type** -- Discriminated union: `{ success: true; data: T } | { success: false; errors: ValidationError[] }`. | Status: not_done

- [ ] **Define ValidatorFunction type** -- `(data: unknown) => ValidationResult<T>`. | Status: not_done

- [ ] **Define RepairLevel type** -- `'none' | 'minimal' | 'standard' | 'aggressive'`. | Status: not_done

- [ ] **Define RepairFunction type** -- `(text: string) => string`. | Status: not_done

- [ ] **Define RepairConfig interface** -- `level?`, `custom?`, `useNormalizer?`, `normalizerOptions?` fields. | Status: not_done

- [ ] **Define EscalationAction type** -- Discriminated union for `increase-temperature`, `switch-model`, `simplify-schema`, and `custom` actions with all their fields as specified in Section 10. | Status: not_done

- [ ] **Define EscalationState interface** -- `messages`, `context`, `lastErrors`, `lastOutput`, `attempt` fields. | Status: not_done

- [ ] **Define ApiRetryConfig interface** -- `enabled?`, `maxRetries?`, `initialDelayMs?`, `maxDelayMs?`, `multiplier?`, `jitter?`, `isRetriable?`, `respectRetryAfter?` fields as specified in Section 10. | Status: not_done

- [ ] **Define FeedbackOutputStrategy type** -- `'full' | 'truncated' | 'errors-only' | 'none'`. | Status: not_done

- [ ] **Define FeedbackConfig interface** -- `outputStrategy?`, `maxOutputLength?`, `includeSchema?`, `template?` fields. | Status: not_done

- [ ] **Define AttemptRecord interface** -- `attempt`, `rawOutput`, `repairedOutput?`, `repaired`, `repairs`, `parsed`, `parseError?`, `valid`, `validationErrors`, `escalated`, `escalationAction?`, `durationMs`, `tokenUsage?` fields as specified in Section 10. | Status: not_done

- [ ] **Define EventHooks interface** -- `onAttempt?`, `onRepair?`, `onValidationError?`, `onEscalation?`, `onSuccess?`, `onFailure?` hooks with exact signatures from Section 10. | Status: not_done

- [ ] **Define RetryOptions interface** -- `messages`, `maxRetries?`, `timeoutMs?`, `repair?`, `apiRetry?`, `escalation?`, `feedback?`, `hooks?`, `parser?`, `preserveHistory?`, `signal?`, `temperature?`, `model?` fields as specified in Section 10. | Status: not_done

- [ ] **Define RetryResult interface** -- `success`, `data`, `attempts`, `repaired`, `escalated`, `simplified`, `totalMs`, `history`, `errors`, `lastOutput`, `totalTokenUsage` fields as specified in Section 10. | Status: not_done

- [ ] **Define RetryLoopConfig interface** -- Extends `Omit<RetryOptions, 'messages'>` with `callLLM`, `schema`, `validationType?` fields. | Status: not_done

- [ ] **Define RetryLoop interface** -- `execute(messages: Message[]): Promise<RetryResult<T>>` and `execute(messages: Message[], signal: AbortSignal): Promise<RetryResult<T>>` overloads. | Status: not_done

- [ ] **Export all types from types.ts** -- Ensure all types/interfaces are exported for use by other modules and consumers. | Status: not_done

---

## Phase 1: Repair Pipeline

- [ ] **Implement fence stripping (fence-strip.ts)** -- Strip markdown code fences: backtick fences (` ```json ... ``` `), tilde fences, fences with language tags, nested fences, unclosed fences. Extract the inner content. | Status: not_done

- [ ] **Implement JSON extraction from prose (json-extract.ts)** -- Extract JSON objects/arrays from surrounding text using bracket-matching. Handle cases like `"Here is the result: {...}"`. | Status: not_done

- [ ] **Implement trailing comma removal (trailing-comma.ts)** -- Remove commas before `}` or `]`. Handle nested structures. Do not modify commas inside string values. | Status: not_done

- [ ] **Implement comment removal (comment-removal.ts)** -- Remove `//` line comments and `/* */` block comments from JSON. Do not modify comment-like sequences inside string values. | Status: not_done

- [ ] **Implement single-to-double quote conversion (single-quotes.ts)** -- Convert single-quoted strings to double-quoted strings using a state machine to avoid modifying strings that contain apostrophes. | Status: not_done

- [ ] **Implement unquoted key quoting (unquoted-keys.ts)** -- Detect bare identifier keys (e.g., `{name: "Alice"}`) and wrap them in double quotes. | Status: not_done

- [ ] **Implement JavaScript literal replacement (js-literals.ts)** -- Replace `undefined`, `NaN`, `Infinity` with `null`. | Status: not_done

- [ ] **Implement truncation completion (truncation.ts)** -- Detect truncated JSON (open brackets/strings without matching close), append the missing closing characters. | Status: not_done

- [ ] **Implement repair pipeline orchestrator (repair/index.ts)** -- Apply repair operations in the correct order (1-8) based on the configured `RepairLevel`. Support `none`, `minimal` (1-2), `standard` (1-5, default), `aggressive` (1-8). Run custom repair functions after built-in repairs. Record which repairs were applied. | Status: not_done

- [ ] **Implement llm-output-normalizer integration in repair pipeline** -- When `useNormalizer: true`, dynamically import `llm-output-normalizer` and delegate repair to its `extractJSON` function. Throw a clear error at configuration time if the package is not installed. | Status: not_done

- [ ] **Support repair shorthand configuration** -- Allow `repair: false` (no repair), `repair: 'aggressive'` (string level), and `repair: { level: 'standard', custom: [...] }` (full config object). Normalize all forms into a `RepairConfig`. | Status: not_done

---

## Phase 1: Zod Validation

- [ ] **Implement Zod validation (zod-validator.ts)** -- Call `schema.safeParse(parsed)` on the parsed value. On success, return `{ success: true, data }`. On failure, extract `ZodError.issues` into `ValidationError[]`. | Status: not_done

- [ ] **Implement Zod error extraction** -- Transform each `ZodIssue` into a `ValidationError`: build `path` string from the issue's `path` array (join with `.` for keys, `[n]` for array indices, prefix with `$`). Map `code`, `message`, `expected`, `received` fields. Handle all common issue types: `invalid_type`, `too_small`, `too_big`, `invalid_enum_value`, `custom`, `unrecognized_keys`. | Status: not_done

- [ ] **Implement Zod detection** -- Detect whether the provided schema is a Zod schema (check for `safeParse` method). Throw a clear error if `zod` is not installed when a Zod schema is passed. | Status: not_done

---

## Phase 1: Error Feedback Generation

- [ ] **Implement default feedback template (feedback/template.ts)** -- Build the feedback message with three sections: error header, numbered error list (with path, message, expected, received), and optional previous output. Follow the template from Section 8. | Status: not_done

- [ ] **Implement feedback message generation (feedback/index.ts)** -- Orchestrate feedback generation: apply the output strategy (`full`, `truncated`, `errors-only`, `none`), call the template, optionally include schema description. | Status: not_done

- [ ] **Implement full output strategy** -- Include the entire previous output in the feedback message. | Status: not_done

- [ ] **Implement truncated output strategy** -- Include only the first N characters of the previous output (default 2000, configurable via `maxOutputLength`). | Status: not_done

- [ ] **Implement errors-only output strategy** -- Include only the fields that had errors, not the full output. | Status: not_done

- [ ] **Implement none output strategy** -- Do not include any previous output in the feedback. | Status: not_done

- [ ] **Implement custom feedback template support** -- Allow the caller to provide a custom `template` function `(errors, previousOutput, attempt) => string` that overrides the default template. | Status: not_done

- [ ] **Implement schema-to-description conversion (feedback/schema-description.ts)** -- Convert a Zod schema to a human-readable JSON Schema description for inclusion in feedback messages. Cover common Zod types: string, number, boolean, object, array, enum, optional, nullable, with refinement descriptions. | Status: not_done

- [ ] **Implement schema inclusion logic** -- Include schema description in the first retry's feedback (when `includeSchema` is true, default). Omit from subsequent retries to save tokens. | Status: not_done

---

## Phase 1: Core Retry Loop

- [ ] **Implement retry loop (retry-loop.ts)** -- Implement the core loop: call LLM, repair, parse, validate, format errors, append feedback, retry. Maintain state across iterations: conversation history, attempt count, cumulative timing. | Status: not_done

- [ ] **Implement maxRetries termination** -- Terminate the loop when `maxRetries` is reached. Total attempts = maxRetries + 1 (initial + retries). Default: 3 retries. | Status: not_done

- [ ] **Implement timeoutMs termination** -- Terminate the loop when the overall timeout is exceeded. Default: 120000ms (2 minutes). Return the best partial result on timeout. | Status: not_done

- [ ] **Implement conversation history management** -- On retry, append the LLM's failed response as an assistant message and the feedback as a user message to the messages array (when `preserveHistory` is true, default). When `preserveHistory` is false, only append the feedback user message. | Status: not_done

- [ ] **Implement parse step** -- Parse the repaired output using `JSON.parse` (default) or the caller-provided `parser` function. If parsing fails, treat the parse error as a validation error and proceed to feedback/retry. | Status: not_done

- [ ] **Implement AttemptRecord building** -- For each attempt, build an `AttemptRecord` with all fields: raw output, repaired output, repair status, repairs list, parse status, validation errors, escalation status, duration, token usage. | Status: not_done

- [ ] **Implement RetryResult construction** -- Build the final `RetryResult<T>` with `success`, `data`, `attempts`, `repaired`, `escalated`, `simplified`, `totalMs`, `history`, `errors`, `lastOutput`, `totalTokenUsage`. | Status: not_done

- [ ] **Implement RetryContext construction and passing** -- Create a `RetryContext` object for each attempt with `attempt`, `temperature`, `model`, `maxTokens`, `escalated`, `custom`. Pass it as the second argument to `callLLM`. | Status: not_done

- [ ] **Implement token usage aggregation** -- Read token usage from `context.custom.tokenUsage` after each `callLLM` call. Aggregate across all attempts into `RetryResult.totalTokenUsage`. | Status: not_done

---

## Phase 1: Public API (retryWithValidation)

- [ ] **Implement retryWithValidation function** -- Wire together: validate config, build initial RetryContext, invoke the retry loop, return RetryResult. Accept `callLLM`, `schema` (Zod/JSON Schema/custom), and `options`. | Status: not_done

- [ ] **Implement schema type detection** -- Detect whether the schema is a Zod schema (has `safeParse`), a JSON Schema object (plain object with `type` property or `validationType: 'json-schema'`), or a custom validator function (is a function). Route to the correct validator. | Status: not_done

- [ ] **Export retryWithValidation from index.ts** -- Re-export the function and all public types from `src/index.ts`. | Status: not_done

---

## Phase 1: Tests -- Repair Operations

- [ ] **Test fence stripping** -- Test backtick fences, tilde fences, fences with language tags (` ```json `, ` ```typescript `), nested fences, unclosed fences, no fences (passthrough), fences with extra whitespace. File: `src/__tests__/repair/fence-strip.test.ts`. | Status: not_done

- [ ] **Test JSON extraction from prose** -- Test JSON object in prose, JSON array in prose, multiple JSON objects (extract first), no JSON (return original), JSON at start/end of string. File: `src/__tests__/repair/json-extract.test.ts`. | Status: not_done

- [ ] **Test trailing comma removal** -- Test object trailing comma, array trailing comma, nested trailing commas, comma inside string (no modification), multiple trailing commas. File: `src/__tests__/repair/trailing-comma.test.ts`. | Status: not_done

- [ ] **Test comment removal** -- Test `//` line comments, `/* */` block comments, comments inside strings (no modification), multiline block comments. File: `src/__tests__/repair/comment-removal.test.ts`. | Status: not_done

- [ ] **Test single-to-double quote conversion** -- Test simple single-quoted strings, apostrophes in text (should not be affected), mixed quotes, nested quotes. File: `src/__tests__/repair/single-quotes.test.ts`. | Status: not_done

- [ ] **Test truncation completion** -- Test missing closing brace, missing closing bracket, missing closing quote, multiple missing closings, already complete JSON. File: `src/__tests__/repair/truncation.test.ts`. | Status: not_done

- [ ] **Test custom repair functions** -- Test custom repair functions running after built-in repairs, multiple custom functions applied in order, custom functions with level `none`. File: `src/__tests__/repair/custom-repair.test.ts`. | Status: not_done

---

## Phase 1: Tests -- Validation

- [ ] **Test Zod validation** -- Test valid input passes, invalid type detected, missing required field detected, constraint violations (too_small, too_big), enum validation, nested object validation, array element validation. File: `src/__tests__/validation/zod-validation.test.ts`. | Status: not_done

- [ ] **Test validation error formatting** -- Test Zod error path construction (nested objects, arrays), expected/received extraction, all common issue codes, multiple errors combined. File: `src/__tests__/validation/error-formatting.test.ts`. | Status: not_done

---

## Phase 1: Tests -- Feedback

- [ ] **Test feedback message generation** -- Test single error, multiple errors, errors with and without expected/received values, default template output format. File: `src/__tests__/feedback/message-generation.test.ts`. | Status: not_done

- [ ] **Test output strategies** -- Test `full` (entire output included), `truncated` (first N chars), `errors-only` (only error fields), `none` (no output). File: `src/__tests__/feedback/output-strategies.test.ts`. | Status: not_done

- [ ] **Test schema inclusion in feedback** -- Test schema included on first retry, schema omitted on subsequent retries, `includeSchema: false` disables entirely. File: `src/__tests__/feedback/schema-inclusion.test.ts`. | Status: not_done

---

## Phase 1: Tests -- Retry Loop Integration

- [ ] **Test first attempt succeeds** -- Mock callLLM returns valid JSON on first call. No retry needed. Result has `success: true`, `attempts: 1`, `repaired: false`. File: `src/__tests__/retry-loop.test.ts`. | Status: not_done

- [ ] **Test repair fixes output on first attempt** -- Mock callLLM returns JSON wrapped in fences. Repair strips fences, validation passes. No retry needed. Result has `repaired: true`. | Status: not_done

- [ ] **Test retry with feedback succeeds on second attempt** -- Mock callLLM returns invalid JSON first, valid JSON second. Verify feedback message was appended. Result has `attempts: 2`. | Status: not_done

- [ ] **Test all retries exhausted** -- Mock callLLM always returns invalid output. Verify result has `success: false`, full `history` array, and `errors` from last attempt. | Status: not_done

- [ ] **Test timeout terminates loop** -- Set a short `timeoutMs`. Mock callLLM with a delay. Verify loop terminates and returns failure. | Status: not_done

- [ ] **Test preserveHistory: true** -- Verify assistant messages and user feedback are both appended to conversation. | Status: not_done

- [ ] **Test preserveHistory: false** -- Verify only feedback user messages are appended, not assistant messages. | Status: not_done

- [ ] **Test custom parser** -- Provide a custom `parser` function. Verify it is called instead of `JSON.parse`. | Status: not_done

- [ ] **Test parse error treated as validation error** -- Mock callLLM returns unparseable output that repair cannot fix. Verify parse error flows into feedback message and retry. | Status: not_done

---

## Phase 1: Test Fixtures

- [ ] **Create test schemas (fixtures/schemas.ts)** -- Define reusable Zod schemas for tests: simple flat object, nested object, array of objects, schema with refinements, schema with enums, deeply nested schema. | Status: not_done

- [ ] **Create mock LLM responses (fixtures/mock-responses.ts)** -- Define reusable mock responses: valid JSON, JSON in fences, JSON in prose, invalid JSON, JSON with wrong types, JSON missing required fields, truncated JSON. | Status: not_done

- [ ] **Create mock API errors (fixtures/mock-errors.ts)** -- Define reusable mock errors: 429 rate limit, 500 server error, 502 bad gateway, 503 service unavailable, 529 overloaded, network error (ECONNREFUSED), timeout error, 401 auth error, 400 bad request. | Status: not_done

---

## Phase 2: API-Level Retry -- Backoff and Jitter

- [ ] **Implement exponential backoff calculation (api-retry/backoff.ts)** -- Calculate `baseDelay = min(initialDelayMs * multiplier^(attempt-1), maxDelayMs)`. | Status: not_done

- [ ] **Implement full jitter strategy (api-retry/jitter.ts)** -- `random(0, baseDelay)`. Default strategy. | Status: not_done

- [ ] **Implement equal jitter strategy** -- `baseDelay / 2 + random(0, baseDelay / 2)`. | Status: not_done

- [ ] **Implement decorrelated jitter strategy** -- `random(initialDelayMs, previousDelay * 3)`. | Status: not_done

- [ ] **Implement none jitter strategy** -- `baseDelay` with no randomization. | Status: not_done

---

## Phase 2: API-Level Retry -- Error Classification

- [ ] **Implement default isRetriable function (api-retry/retriable.ts)** -- Return true for status codes 429, 500, 502, 503, 504, 529. Return true for network errors: `ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`, DNS failures. Return false for 400, 401, 403, 404, content policy errors. | Status: not_done

- [ ] **Support custom isRetriable function** -- Allow caller to override the default error classification. | Status: not_done

---

## Phase 2: API-Level Retry -- Retry-After Handling

- [ ] **Implement Retry-After header parsing** -- Parse `Retry-After` header as seconds (numeric) or HTTP date. Use the larger of the Retry-After value and the calculated backoff delay. Controlled by `respectRetryAfter` option (default: true). | Status: not_done

---

## Phase 2: API-Level Retry -- Wrapper

- [ ] **Implement API-level retry wrapper (api-retry/index.ts)** -- Wrap the `callLLM` function in an inner retry loop. Catch errors, check `isRetriable`, calculate backoff with jitter, wait, retry up to `maxRetries` (default 3). Pass through non-retriable errors. | Status: not_done

- [ ] **Support apiRetry: false to disable** -- When `apiRetry` is false, do not wrap `callLLM` with the inner retry loop. | Status: not_done

- [ ] **Integrate API-level retry into the main retry loop** -- The outer loop calls the API-retry-wrapped `callLLM` on each attempt. API retries are transparent to the output-level retry loop. | Status: not_done

---

## Phase 2: Escalation -- Temperature

- [ ] **Implement temperature escalation (escalation/temperature.ts)** -- Increase `RetryContext.temperature` by the configured `delta` (default 0.2). Respect `maxTemperature` ceiling (default 1.0). | Status: not_done

---

## Phase 2: Escalation -- Model Switch

- [ ] **Implement model switch escalation (escalation/model-switch.ts)** -- Replace the active `callLLM` function with the one provided in the escalation action. Update `RetryContext.model` with the new model identifier. | Status: not_done

---

## Phase 2: Escalation -- Schema Simplification

- [ ] **Implement schema simplification (escalation/schema-simplify.ts)** -- Programmatically modify a Zod schema to reduce complexity. Support removing optional fields (`.optional()` fields removed from object shape), widening string constraints (remove `.email()`, `.url()`, `.uuid()`, `.regex()`), widening numeric constraints (remove `.min()`, `.max()`, `.int()`, `.positive()`), and removing custom refinements (`.refine()`, `.superRefine()`). Each operation individually configurable. | Status: not_done

- [ ] **Set simplified flag in RetryResult** -- When schema simplification is applied, set `result.simplified: true` and include information about which constraints were relaxed. | Status: not_done

---

## Phase 2: Escalation -- Custom

- [ ] **Implement custom escalation (escalation/index.ts)** -- Invoke the caller-provided `fn(state: EscalationState)` function. Support async escalation functions. Pass `messages`, `context`, `lastErrors`, `lastOutput`, `attempt` in the state object. | Status: not_done

---

## Phase 2: Escalation -- Schedule

- [ ] **Implement escalation schedule evaluation (escalation/index.ts)** -- Evaluate the ordered list of escalation actions. Check if the current attempt count matches an action's `afterAttempt` trigger. Dispatch the appropriate action. Support multiple actions at different attempt counts. | Status: not_done

- [ ] **Record escalation in AttemptRecord** -- Set `escalated: true` and `escalationAction` description on the AttemptRecord when escalation occurs. | Status: not_done

---

## Phase 2: Tests -- API-Level Retry

- [ ] **Test exponential backoff calculation** -- Test backoff values for multiple attempts, verify they respect `maxDelayMs`. File: `src/__tests__/api-retry/backoff.test.ts`. | Status: not_done

- [ ] **Test all jitter strategies** -- Test full, equal, decorrelated, none. Verify value ranges are correct for each. File: `src/__tests__/api-retry/jitter.test.ts`. | Status: not_done

- [ ] **Test retriable error classification** -- Test all status codes (429, 500, 502, 503, 504, 529 = retriable; 400, 401, 403, 404 = not). Test network errors. Test custom isRetriable. File: `src/__tests__/api-retry/retriable-errors.test.ts`. | Status: not_done

- [ ] **Test Retry-After header handling** -- Test numeric seconds, HTTP date, Retry-After larger than backoff, Retry-After smaller than backoff, missing header. File: `src/__tests__/api-retry/retry-after.test.ts`. | Status: not_done

- [ ] **Test API-level retry wrapper end-to-end** -- Mock callLLM that throws retriable errors then succeeds. Verify correct number of retries, delays, and final success. | Status: not_done

---

## Phase 2: Tests -- Escalation

- [ ] **Test temperature escalation** -- Verify context temperature increases by delta, respects maxTemperature ceiling. File: `src/__tests__/escalation/temperature.test.ts`. | Status: not_done

- [ ] **Test model switch escalation** -- Verify the new callLLM function is invoked after switch. Verify context.model is updated. File: `src/__tests__/escalation/model-switch.test.ts`. | Status: not_done

- [ ] **Test schema simplification** -- Verify optional fields removed, string formats stripped, numeric constraints removed, refinements removed. Test each operation individually and in combination. File: `src/__tests__/escalation/schema-simplify.test.ts`. | Status: not_done

- [ ] **Test custom escalation** -- Verify custom function receives correct state, can modify messages and context. File: `src/__tests__/escalation/custom-escalation.test.ts`. | Status: not_done

- [ ] **Test escalation schedule ordering** -- Test multiple actions at different afterAttempt values. Verify they trigger in order at the correct attempts. File: `src/__tests__/escalation/schedule.test.ts`. | Status: not_done

---

## Phase 3: JSON Schema Validation

- [ ] **Implement JSON Schema validation (validation/json-schema-validator.ts)** -- Dynamically import `ajv`. Create an Ajv instance and validate the parsed data. Extract errors from `ajv.errors` and transform into `ValidationError[]` format. Convert `instancePath` (e.g., `/users/0/email`) to `$`-prefixed dot notation (`$.users[0].email`). | Status: not_done

- [ ] **Handle missing ajv dependency** -- If `ajv` is not installed and JSON Schema validation is requested, throw a clear error at invocation time instructing the caller to install `ajv`. | Status: not_done

- [ ] **Test JSON Schema validation** -- Test valid input, invalid types, missing required fields, pattern constraints, ajv error path conversion. File: `src/__tests__/validation/json-schema-validation.test.ts`. | Status: not_done

---

## Phase 3: Custom Validator

- [ ] **Implement custom validator dispatch (validation/index.ts)** -- Implement the validation dispatcher that routes to Zod, JSON Schema (ajv), or custom validator based on the schema type or `validationType` option. | Status: not_done

- [ ] **Test custom validator** -- Test custom validator that returns success, custom validator that returns errors, custom validator with cross-field validation. File: `src/__tests__/validation/custom-validator.test.ts`. | Status: not_done

---

## Phase 3: createRetryLoop Factory

- [ ] **Implement createRetryLoop factory (factory.ts)** -- Accept `RetryLoopConfig`, return a `RetryLoop<T>` instance with an `execute(messages, signal?)` method. The factory pre-configures all options so `execute` only needs the messages. | Status: not_done

- [ ] **Test createRetryLoop** -- Test factory creates a reusable instance, test calling `execute` multiple times with different messages, test AbortSignal overload. | Status: not_done

---

## Phase 3: Advanced Repair Operations

- [ ] **Test unquoted key quoting** -- Test bare identifier keys, numeric keys, keys with special characters, mixed quoted/unquoted. File: included in repair tests. | Status: not_done

- [ ] **Test JavaScript literal replacement** -- Test `undefined`, `NaN`, `Infinity` replaced with `null`, values inside strings not affected. File: included in repair tests. | Status: not_done

---

## Phase 3: llm-output-normalizer Integration

- [ ] **Test llm-output-normalizer integration** -- Test that `useNormalizer: true` delegates to `extractJSON`, test error when package is not installed, test `normalizerOptions` passthrough. | Status: not_done

---

## Phase 3: Event Hooks

- [ ] **Implement onAttempt hook** -- Fire before each LLM call with `(attempt, messages)`. Support async hooks. | Status: not_done

- [ ] **Implement onRepair hook** -- Fire when repair changes the output, with `(attempt, original, repaired, repairs)`. | Status: not_done

- [ ] **Implement onValidationError hook** -- Fire when validation fails, with `(attempt, errors, rawOutput)`. | Status: not_done

- [ ] **Implement onEscalation hook** -- Fire when an escalation action triggers, with `(attempt, action)`. | Status: not_done

- [ ] **Implement onSuccess hook** -- Fire when validation succeeds, with `(attempt, data)`. | Status: not_done

- [ ] **Implement onFailure hook** -- Fire when all retries are exhausted, with `(attempts)`. | Status: not_done

- [ ] **Test all event hooks** -- Verify each hook fires at the correct point with the correct arguments. Test async hooks. Test hooks with no-op (undefined hooks don't throw). | Status: not_done

---

## Phase 3: Tests -- Provider Adapters

- [ ] **Test OpenAI adapter pattern** -- Mock OpenAI-style response. Verify callLLM returns text content. File: `src/__tests__/providers/openai.test.ts`. | Status: not_done

- [ ] **Test Anthropic adapter pattern** -- Mock Anthropic-style response (content blocks). Verify callLLM extracts text. File: `src/__tests__/providers/anthropic.test.ts`. | Status: not_done

- [ ] **Test Google Gemini adapter pattern** -- Mock Gemini-style response. Verify callLLM returns text. File: `src/__tests__/providers/gemini.test.ts`. | Status: not_done

- [ ] **Test Ollama adapter pattern** -- Mock Ollama-style response. Verify callLLM returns message content. File: `src/__tests__/providers/ollama.test.ts`. | Status: not_done

---

## Phase 4: Configuration Validation

- [ ] **Validate maxRetries** -- Must be a non-negative integer. Throw `TypeError` with clear message for negative or non-integer values. | Status: not_done

- [ ] **Validate timeoutMs** -- Must be a positive number or `Infinity`. Throw `TypeError` for zero or negative values. | Status: not_done

- [ ] **Validate escalation afterAttempt ranges** -- Each `escalation[n].afterAttempt` must be between 1 and `maxRetries` inclusive. Throw `RangeError` if out of range. | Status: not_done

- [ ] **Validate apiRetry.initialDelayMs** -- Must be positive. Throw `TypeError` for zero or negative. | Status: not_done

- [ ] **Validate apiRetry.multiplier** -- Must be >= 1. Throw `TypeError` for values below 1. | Status: not_done

- [ ] **Validate all config at invocation time** -- All configuration validation happens eagerly when `retryWithValidation` or `createRetryLoop` is called, not lazily during the loop. | Status: not_done

- [ ] **Test configuration validation** -- Test all validation error messages match spec. Test valid configs pass. Test edge cases (0, negative, Infinity, NaN). | Status: not_done

---

## Phase 4: AbortSignal Support

- [ ] **Implement AbortSignal cancellation** -- Check the `signal.aborted` flag at every async boundary: before each LLM call, before each backoff delay, between retry attempts. Throw `AbortError` or return a failure result when aborted. | Status: not_done

- [ ] **Wire AbortSignal through API-level retry** -- The inner retry loop also respects the abort signal during backoff waits. | Status: not_done

- [ ] **Test AbortSignal cancellation** -- Test aborting before any call, aborting mid-retry, aborting during backoff delay. Verify the result reflects the cancellation. | Status: not_done

---

## Phase 4: Token Usage Tracking

- [ ] **Implement token usage extraction from RetryContext** -- After each `callLLM` call, read `context.custom.tokenUsage` and populate the `AttemptRecord.tokenUsage` field. | Status: not_done

- [ ] **Implement token usage aggregation** -- Sum all `AttemptRecord.tokenUsage` values into `RetryResult.totalTokenUsage`. Handle missing/undefined token usage gracefully (default to 0). | Status: not_done

- [ ] **Test token usage tracking** -- Test that token usage from callLLM flows into AttemptRecord and aggregates correctly in RetryResult. | Status: not_done

---

## Phase 4: Edge Case Tests

- [ ] **Test empty string output** -- callLLM returns `""`. Verify parse error and retry. | Status: not_done

- [ ] **Test whitespace-only output** -- callLLM returns `"   \n  "`. Verify parse error and retry. | Status: not_done

- [ ] **Test valid JSON but wrong shape** -- callLLM returns `"hello"` (valid JSON string, not an object). Verify schema validation catches it. | Status: not_done

- [ ] **Test JSON array when object expected** -- callLLM returns `[1,2,3]`. Verify schema validation catches it. | Status: not_done

- [ ] **Test extremely large output** -- callLLM returns 100KB+ JSON. Verify repair and validation handle it without errors or excessive memory. | Status: not_done

- [ ] **Test output with Unicode characters** -- callLLM returns JSON containing emoji, CJK characters, RTL text. Verify repair and parse handle it correctly. | Status: not_done

- [ ] **Test output with control characters** -- callLLM returns JSON containing tabs, null bytes, etc. Verify handling. | Status: not_done

- [ ] **Test nested retry: API retry inside output retry** -- Mock callLLM that throws 429 once then returns invalid JSON, then returns valid JSON on retry. Verify both inner and outer loops work together. | Status: not_done

---

## Phase 4: Performance

- [ ] **Benchmark repair pipeline** -- Profile the repair pipeline with typical (1-10KB) and large (100KB+) outputs. Ensure sub-millisecond for typical outputs. | Status: not_done

- [ ] **Benchmark full retry loop overhead** -- Measure the overhead added by llm-retry on a successful first attempt (repair + parse + validate). Target: under 1ms. | Status: not_done

---

## Documentation

- [ ] **Write README.md** -- Comprehensive README with: overview, installation instructions, quick start example, configuration reference (all options with defaults), provider integration examples (OpenAI, Anthropic, Gemini, Ollama, Vercel AI), repair levels reference, escalation strategies reference, event hooks reference, API reference (retryWithValidation, createRetryLoop, all types), troubleshooting guide. | Status: not_done

- [ ] **Add JSDoc comments to all public APIs** -- Add JSDoc to `retryWithValidation`, `createRetryLoop`, all exported types and interfaces. Include parameter descriptions, return types, and usage examples. | Status: not_done

---

## CI/CD and Publishing

- [ ] **Verify full test suite passes** -- Run `npm run test` and confirm all tests pass. | Status: not_done

- [ ] **Verify lint passes** -- Run `npm run lint` and confirm no lint errors. | Status: not_done

- [ ] **Verify build succeeds** -- Run `npm run build` and confirm TypeScript compiles with no errors and `dist/` is populated correctly. | Status: not_done

- [ ] **Bump version to target release** -- Update `package.json` version per semver: `0.1.0` for Phase 1, `0.2.0` for Phase 2, `0.3.0` for Phase 3, `1.0.0` for Phase 4. | Status: not_done

- [ ] **Publish to npm** -- Run `npm publish` from master after PR merge (follow monorepo workflow in CLAUDE.md). Verify `prepublishOnly` runs build. | Status: not_done
