# llm-retry -- Specification

## 1. Overview

`llm-retry` is a provider-agnostic retry orchestrator for LLM output parsing, validation, and error-feedback re-prompting. It accepts any async function that calls an LLM and returns a string, pairs it with a schema or validator, and runs a multi-stage retry loop: call the LLM, attempt local repair on the raw output, validate against the schema, and if validation fails, format the errors into a feedback message, append it to the conversation, and re-call the LLM. The loop continues until the output validates, a configurable maximum number of retries is exhausted, or an escalation strategy (switch to a stronger model, increase temperature, simplify the schema) intervenes. The result is either a fully validated, typed object or a detailed failure report containing every attempt, every validation error, and the final raw output.

The gap this package fills is specific and well-defined. Generic retry libraries like `p-retry`, `async-retry`, and `exponential-backoff` handle transient failures -- network errors, rate limits, server errors -- by re-executing the same function after a delay. They know nothing about LLM output. When an LLM returns malformed JSON or an object that fails schema validation, re-executing the identical API call with the same prompt will often produce the same malformed output, because the failure is not transient -- it is a structural problem with the model's response. Fixing this requires a fundamentally different retry strategy: the validation errors must be fed back to the LLM as part of the next prompt, giving the model specific information about what was wrong and how to fix it. No generic retry library does this because it requires understanding the structure of an LLM conversation (messages array), the distinction between API errors (retriable with backoff) and output errors (retriable with feedback), and the multi-stage pipeline of repair, parse, validate, and re-prompt.

Existing tools that do address LLM output retries are tightly coupled to specific frameworks or providers. `instructor-js` (`@instructor-ai/instructor`) provides structured extraction with Zod validation and automatic retries, but it is built on top of the OpenAI SDK and its function-calling modes (TOOLS, FUNCTIONS, JSON, MD_JSON). Using it with Anthropic, Google Gemini, or a local model behind Ollama requires the `llm-polyglot` adapter, and the retry logic is interleaved with the SDK integration rather than being a separable concern. LangChain's `RetryWithErrorOutputParser` wraps a parser and re-calls the LLM with the original prompt, the failed output, and the error message -- but it is part of the LangChain framework, depends on LangChain's chain abstraction, and cannot be used standalone. LangChain's `OutputFixingParser` similarly requires a LangChain LLM instance. The Vercel AI SDK's `generateObject` function retries automatically when the model outputs invalid JSON, but it is part of a full-stack framework with opinions about routing, middleware, and UI integration -- extracting just the retry logic is not possible. The `@strands-agents/sdk` provides retry-on-validation-error for agent outputs, but it is an agent framework, not a retry primitive.

`llm-retry` provides the retry orchestration layer as a standalone, focused package. It accepts any `(messages: Message[]) => Promise<string>` function -- the caller wraps their provider SDK call in this function, and `llm-retry` handles the rest. The package does not import any LLM provider SDK. It does not make HTTP requests. It does not manage API keys. It operates entirely on the conversation-level abstraction: messages in, string out, validate, optionally repair, format errors, append feedback, retry. This makes it usable with OpenAI, Anthropic, Google, Mistral, Ollama, llama.cpp, vLLM, or any custom inference server -- the caller provides the LLM call function, and `llm-retry` provides the retry loop.

The package composes with other packages in this monorepo. `llm-output-normalizer` handles the repair stage -- stripping fences, fixing malformed JSON, extracting JSON from prose. `stream-validate` handles streaming validation with progressive field emission. `tool-call-retry` handles retry logic for tool execution (a different concern -- retrying the execution of a tool that failed, not retrying the LLM that generated the tool call). `llm-retry` sits between the LLM call and the application's consumption of the result, orchestrating the parse-validate-feedback loop that none of these other packages provide.

---

## 2. Goals and Non-Goals

### Goals

- Provide a `retryWithValidation<T>(callLLM, schema, options?)` function that calls an LLM, validates the output against a Zod or JSON Schema validator, and retries with error feedback until the output is valid or retries are exhausted.
- Provide a `createRetryLoop<T>(config)` factory that returns a reusable, preconfigured `RetryLoop<T>` instance for repeated use with different message inputs.
- Separate API-level retry (429 rate limits, 5xx server errors, timeouts, network failures) from output-level retry (malformed JSON, schema validation failures, incomplete responses). API-level retry uses exponential backoff with jitter. Output-level retry uses error-feedback re-prompting. The two compose as nested loops: API retry is the inner loop around each individual LLM call; output retry is the outer loop around the parse-validate cycle.
- Attempt local repair before retrying the LLM. Repair is free and instant; retrying costs money and time. Strip markdown fences, fix trailing commas, extract JSON from prose, close truncated brackets -- all before deciding whether to retry.
- Format validation errors into LLM-consumable feedback messages that include the JSON path of the error, what was expected, what was received, and optionally the original output, so the LLM has maximum context for correction.
- Support configurable escalation strategies: increase temperature, switch to a stronger model, simplify the schema, reduce prompt complexity, or invoke a custom escalation function -- triggered after a configurable number of failed retries.
- Provide a rich result object (`RetryResult<T>`) that includes the validated data, the number of attempts, whether repair was applied, whether escalation occurred, the full history of attempts with their errors, and cost/latency metadata.
- Provide event hooks (`onAttempt`, `onRepair`, `onValidationError`, `onEscalation`, `onSuccess`, `onFailure`) for observability, logging, and custom logic injection.
- Be provider-agnostic. Accept any `(messages) => Promise<string>` function. No dependency on any LLM provider SDK.
- Keep runtime dependencies minimal. Depend on `zod` as a peer dependency for schema validation. Use no other runtime dependencies.

### Non-Goals

- **Not an LLM API client.** This package does not make HTTP requests, manage API keys, handle streaming, or parse provider-specific response envelopes. The caller provides a function that calls the LLM and returns the raw text output. Use the OpenAI SDK, Anthropic SDK, Google AI SDK, or `fetch` to make the actual API call.
- **Not a structured output enforcer.** This package does not constrain LLM generation at the token level. OpenAI's structured output mode, Anthropic's tool use, and constrained decoding libraries prevent malformed output at generation time. `llm-retry` validates and retries after generation -- it is the fallback for when generation-side constraints are unavailable, insufficient, or produce validation errors at the application-schema level.
- **Not a JSON repair library.** While this package applies local repair as one stage of the retry pipeline, the repair logic is intentionally lightweight and delegates to `llm-output-normalizer` for comprehensive repair. `llm-retry` does not aim to handle every JSON malformation -- it handles the common ones (fence stripping, trailing commas, truncation completion) and retries the LLM for the rest.
- **Not a streaming validator.** This package validates complete LLM responses, not partial streaming output. For progressive validation of streaming responses, use `stream-validate`. The two packages are complementary: `stream-validate` for streaming UIs, `llm-retry` for batch/request-response validation with retries.
- **Not a prompt engineering tool.** This package does not modify the original user prompt to improve output quality. It appends error feedback messages to the conversation when retrying, but it does not rewrite, optimize, or lint the initial prompt. Use `prompt-lint` or `prompt-optimize` for prompt-side improvements.
- **Not a general-purpose retry library.** This package does not replace `p-retry` or `async-retry` for generic async operation retries. Its retry logic is specific to the LLM output validation loop. For retrying arbitrary async functions with backoff, use a generic retry library. For retrying LLM output with validation feedback, use `llm-retry`.
- **Not a cost optimization tool.** While the package tracks token usage and cost metadata per attempt, it does not implement cost budgets, billing alerts, or automatic cost-based termination. Cost tracking is informational, exposed through the result object and event hooks.

---

## 3. Target Users and Use Cases

### AI Application Developers Extracting Structured Data

Developers who call an LLM API and expect a JSON object conforming to a specific schema -- a user profile, a product listing, an analysis report, a function call argument set. The LLM may return the JSON wrapped in prose, with trailing commas, with an extra field, or missing a required field. These developers currently write ad-hoc retry loops with `try { JSON.parse(...) } catch { retry() }` that do not feed errors back to the model. `llm-retry` replaces this boilerplate with a single function call that handles repair, validation, feedback, and retry. A typical integration is: `const user = await retryWithValidation(callOpenAI, UserSchema)`.

### Agent Framework Authors

Teams building agent systems where the LLM generates structured action objects (tool calls, routing decisions, state transitions) that must conform to strict schemas. A malformed action object can crash the agent loop or cause undefined behavior. `llm-retry` provides a validated extraction primitive that agent frameworks can use internally, ensuring every LLM-generated action is schema-valid before execution. The escalation feature is particularly valuable here: if a lightweight model (GPT-4o-mini, Haiku) repeatedly fails to produce valid output for a complex schema, the framework can automatically escalate to a stronger model (GPT-4o, Sonnet) for that specific extraction.

### Data Pipeline Engineers

Teams running LLMs over large document corpora to extract structured data -- entities, relationships, metadata, classifications. These pipelines process thousands of documents and need high extraction reliability. A 95% success rate on first attempt means 50 failures per 1000 documents. `llm-retry` with repair and feedback re-prompting can push that to 99.5%+ without manual intervention. The cost tracking metadata lets pipeline operators monitor retry costs and identify documents that consistently cause extraction failures.

### Backend Service Developers with SLAs

Developers building API endpoints that call LLMs and return structured responses to clients. These services have latency and reliability SLAs. `llm-retry` provides configurable timeouts, maximum retry counts, and escalation strategies that let the service balance reliability against latency. The API-level retry with backoff handles transient provider outages. The output-level retry with feedback handles structural output failures. Together, they maximize the probability of returning a valid response within the SLA window.

### Developers Using Local or Open-Source Models

Developers using Ollama, vLLM, llama.cpp, or other inference servers with open-source models (Llama, Mistral, Phi, Qwen). These models are more prone to output formatting errors than commercial APIs -- they may omit closing brackets, mix JSON with commentary, use single quotes, or ignore schema instructions entirely. `llm-retry` is especially valuable here because the repair stage catches common formatting issues for free, and the feedback re-prompting gives the model a second chance with explicit error guidance.

### Multi-Provider Applications with Fallback

Applications that use multiple LLM providers and need to fall back gracefully. The escalation strategy in `llm-retry` supports switching providers mid-retry: try Claude Haiku first (fast, cheap), escalate to Claude Sonnet if Haiku fails twice, escalate to GPT-4o if Sonnet also fails. This provider-level fallback is configured declaratively through the escalation schedule, not through manual if/else chains.

---

## 4. Core Concepts

### Retry Loop

The retry loop is the central abstraction in `llm-retry`. It is a stateful loop that repeatedly calls an LLM, processes the output through a multi-stage pipeline (repair, parse, validate), and either returns a validated result or retries with error feedback. The loop maintains state across iterations: the conversation history (with appended feedback messages), the attempt count, the cumulative latency and cost, and the escalation state. The loop terminates when validation succeeds, the maximum retry count is reached, or an abort signal is received.

### Attempt

An attempt is a single iteration of the retry loop. Each attempt consists of: one LLM call (which may itself involve API-level retries for transient errors), one repair pass, one parse operation, and one validation check. The attempt produces either a validated result or a set of validation errors. If validation fails, the errors are formatted and appended to the conversation for the next attempt. Each attempt is recorded in the result's `attempts` array with its raw output, repair status, validation errors, latency, and token usage.

### Local Repair

Local repair is the first line of defense before retrying the LLM. It applies deterministic, zero-cost transformations to the raw output to fix common formatting issues. Repair operations include: stripping markdown code fences, extracting JSON from surrounding prose, fixing trailing commas, converting single quotes to double quotes, closing truncated brackets and strings, and removing JavaScript-style comments. Repair is always attempted before validation. If repair produces output that passes validation, the LLM is not re-called -- saving time and money. The repair stage is conceptually the same pipeline that `llm-output-normalizer` provides, and can optionally delegate to it.

### Validation

Validation checks whether the repaired, parsed output conforms to the expected schema. The primary validation mechanism is Zod's `safeParse`, which returns detailed error information including the path to each invalid field, the expected type, the received value, and a human-readable error message. Alternatively, the caller can provide a JSON Schema (validated via `ajv`) or a custom validator function `(data: unknown) => ValidationResult`. Validation is the gatekeeper: if it passes, the loop succeeds; if it fails, the errors drive the feedback for the next attempt.

### Error Feedback

Error feedback is the mechanism that distinguishes `llm-retry` from generic retry libraries. When validation fails, the validation errors are formatted into a user message and appended to the conversation history. This message tells the LLM: "Your previous response had the following errors: [list of errors with paths and details]. Here is your original output: [output]. Please fix these errors and return a valid response." The LLM then generates a new response with the benefit of knowing exactly what went wrong. This is fundamentally more effective than blindly retrying with the same prompt, because the LLM receives targeted correction information.

### Escalation

Escalation is a configurable strategy for when repeated retries with the same configuration are not working. If the LLM fails validation N times in a row, something about the current configuration is insufficient -- the model may be too weak for the schema complexity, the temperature may be too low (causing the same error repeatedly), or the schema may be too demanding. Escalation changes the configuration: increase temperature to break out of a repetitive failure mode, switch to a stronger model that is more likely to produce valid output, simplify the schema by removing optional fields or loosening constraints, or invoke a custom function that modifies any aspect of the retry configuration. Escalation is configured as a schedule: "after 2 retries, increase temperature to 0.7; after 4 retries, switch to gpt-4o."

### API-Level Retry vs Output-Level Retry

These are two distinct retry mechanisms that compose as nested loops. API-level retry handles transient infrastructure failures: HTTP 429 (rate limit), HTTP 5xx (server error), network timeouts, connection resets. These are retried with exponential backoff and jitter, respecting `Retry-After` headers when present. The LLM function provided by the caller may already handle these (the OpenAI SDK has built-in retry logic), or `llm-retry` can wrap the function with its own API-level retry. Output-level retry handles structural output failures: malformed JSON, schema validation errors, incomplete responses, refusals. These are retried with error feedback re-prompting -- a fundamentally different strategy that modifies the conversation. API-level retry is the inner loop (retry the same API call), output-level retry is the outer loop (retry with modified messages).

---

## 5. Retry Loop Architecture

### Pipeline Diagram

```
                                 ┌─────────────────────────┐
                                 │     Caller provides:     │
                                 │  - callLLM function      │
                                 │  - schema (Zod/JSON)     │
                                 │  - messages               │
                                 │  - options                │
                                 └────────────┬──────────────┘
                                              │
                                              ▼
                              ┌───────────────────────────────┐
                              │  Step 1: Call LLM              │
                              │  callLLM(messages) → string    │
                              │                                │
                              │  (API-level retry with backoff │
                              │   for 429/5xx/timeout/network) │
                              └───────────────┬───────────────┘
                                              │ raw output
                                              ▼
                              ┌───────────────────────────────┐
                              │  Step 2: Local Repair          │
                              │  - Strip markdown fences       │
                              │  - Extract JSON from prose     │
                              │  - Fix trailing commas         │
                              │  - Close truncated brackets    │
                              │  - Custom repair functions     │
                              └───────────────┬───────────────┘
                                              │ repaired output
                                              ▼
                              ┌───────────────────────────────┐
                              │  Step 3: Parse                 │
                              │  JSON.parse(repaired)          │
                              │  or custom parser function     │
                              └───────────────┬───────────────┘
                                              │ parsed value
                                              ▼
                              ┌───────────────────────────────┐
                              │  Step 4: Validate              │
                              │  schema.safeParse(parsed)      │
                              │  or custom validator           │
                              └───────────────┬───────────────┘
                                              │
                                   ┌──────────┴──────────┐
                                   │                     │
                              valid ▼                     ▼ invalid
                    ┌──────────────────┐    ┌──────────────────────────┐
                    │  Return          │    │  Step 5: Format Errors   │
                    │  RetryResult<T>  │    │  - Zod error paths       │
                    │  with data: T    │    │  - Expected vs received  │
                    │                  │    │  - Human-readable msg    │
                    └──────────────────┘    └────────────┬─────────────┘
                                                        │
                                                        ▼
                                           ┌────────────────────────┐
                                           │  Retries remaining?    │
                                           └─────┬──────────┬──────┘
                                                 │          │
                                            yes  ▼          ▼ no
                              ┌──────────────────────┐  ┌───────────────────┐
                              │  Step 6: Escalate?   │  │  Return            │
                              │  Check escalation    │  │  RetryResult<T>    │
                              │  schedule. Apply     │  │  with data: null   │
                              │  changes if due.     │  │  and error details │
                              └──────────┬───────────┘  └───────────────────┘
                                         │
                                         ▼
                              ┌───────────────────────────────┐
                              │  Step 7: Append Feedback       │
                              │  Add user message with:        │
                              │  - Validation errors           │
                              │  - Original output (optional)  │
                              │  - Correction instructions     │
                              └───────────────┬───────────────┘
                                              │
                                              │ Go to Step 1
                                              └──────────────────────┘
```

### Step 1: Call LLM

The retry loop invokes the caller-provided `callLLM` function with the current messages array. On the first attempt, this is the original messages. On subsequent attempts, the messages include appended feedback from previous validation failures.

The `callLLM` function is typed as `(messages: Message[]) => Promise<string>`. It is the caller's responsibility to translate this into the appropriate provider SDK call. The function must return the raw text content of the LLM response -- not the provider-specific response envelope.

If API-level retry is enabled (the default), the `callLLM` invocation is wrapped in an inner retry loop that catches retriable errors (rate limits, server errors, timeouts, network errors) and retries with exponential backoff. This inner loop is transparent to the outer output-validation loop.

### Step 2: Local Repair

The raw output from the LLM is passed through a configurable repair pipeline. The default repair operations, applied in order:

1. **Markdown fence stripping**: If the output is wrapped in a code fence (` ```json ... ``` ` or ` ```\n...\n``` `), the fence markers are removed and the inner content is extracted.
2. **JSON extraction from prose**: If the output contains JSON embedded in surrounding text ("Here is the result: {...}"), the JSON is extracted using bracket-matching.
3. **Trailing comma removal**: Commas before `}` or `]` are removed.
4. **Single-to-double quote conversion**: Single-quoted strings are converted to double-quoted strings (with a state machine to avoid modifying strings that contain apostrophes).
5. **Truncation completion**: If the JSON is truncated (open brackets without matching close brackets), the missing closing characters are appended.

Each repair operation is individually toggleable. The caller can disable all repair (`repair: false`), use the default set, or provide custom repair functions that run in addition to or instead of the defaults. The repair stage records what it did in the attempt metadata (`repaired: true`, `repairs: ['fence-strip', 'trailing-comma']`).

If the `llm-output-normalizer` package is installed, `llm-retry` can optionally delegate repair to it for the full nine-step normalization pipeline. This is configured via the `useNormalizer: true` option.

### Step 3: Parse

The repaired output is parsed into a JavaScript value. The default parser is `JSON.parse`. The caller can provide a custom parser function `(text: string) => unknown` for non-JSON formats (YAML, CSV, custom DSLs). If parsing fails (the output is not valid JSON even after repair), the parse error is treated as a validation error and the loop proceeds to Step 5 (format errors) with a message indicating the output could not be parsed.

### Step 4: Validate

The parsed value is validated against the schema. Three validation mechanisms are supported:

- **Zod schema**: The primary mechanism. `schema.safeParse(parsed)` is called. If it succeeds, the typed result `T` is extracted. If it fails, the `ZodError` is captured with full issue details (path, code, message, expected, received).
- **JSON Schema**: If a JSON Schema object is provided instead of a Zod schema, validation is performed using `ajv` (the caller must have `ajv` installed). Errors are extracted from `ajv`'s error array.
- **Custom validator**: A function `(data: unknown) => { success: true, data: T } | { success: false, errors: ValidationError[] }`. This supports arbitrary validation logic beyond what schemas can express.

If validation succeeds, the loop terminates and returns the validated data in a `RetryResult<T>`.

### Step 5: Format Errors

When validation fails, the errors are formatted into a structured feedback message suitable for inclusion in an LLM conversation. The formatting transforms machine-readable error objects into human-readable text that gives the LLM actionable information.

For each validation error, the formatted message includes:
- The JSON path where the error occurred (e.g., `$.users[0].email`)
- What was expected (e.g., "a valid email address")
- What was received (e.g., `"not-an-email"`)
- The validation rule that failed (e.g., "string must match email format")

The complete feedback message follows a template (customizable by the caller):

```
Your previous response did not pass validation. The following errors were found:

1. At path "$.users[0].email": Expected a valid email address, received "not-an-email".
2. At path "$.age": Expected a number, received a string ("thirty").

Please fix these errors and return a valid JSON object matching the schema.
```

### Step 6: Escalation Check

Before retrying, the loop checks whether the current attempt count triggers an escalation action according to the configured escalation schedule. If an escalation is due, the configured changes are applied -- the `callLLM` function may be replaced (switching models), options may be modified (increasing temperature), or the schema may be simplified. Escalation actions are recorded in the attempt metadata.

### Step 7: Append Feedback and Retry

The formatted error message is appended to the messages array as a new user message. If the configuration includes the original output in feedback (the default), the LLM's failed response is also included so it can see exactly what it produced. The loop then returns to Step 1 with the updated messages.

---

## 6. Repair Strategies

### Purpose

Local repair is the first line of defense against output formatting issues. Repair is applied before validation, and before any decision to retry. The rationale is simple: repair is free (no API call, no latency, no cost) and handles the most common LLM output issues. If repair produces valid output, the retry loop succeeds on the first attempt without ever re-calling the LLM.

### Built-in Repair Operations

| Order | Operation | What It Fixes | Example |
|---|---|---|---|
| 1 | Markdown fence stripping | Output wrapped in `` ```json ... ``` `` | `` ```json\n{"name":"Alice"}\n``` `` becomes `{"name":"Alice"}` |
| 2 | JSON extraction from prose | JSON embedded in surrounding text | `Here is the result: {"name":"Alice"}` becomes `{"name":"Alice"}` |
| 3 | Trailing comma removal | Commas before `}` or `]` | `{"a":1,"b":2,}` becomes `{"a":1,"b":2}` |
| 4 | Comment removal | `//` and `/* */` comments inside JSON | `{"a":1 // comment}` becomes `{"a":1}` |
| 5 | Single-to-double quote conversion | Single-quoted strings | `{'name':'Alice'}` becomes `{"name":"Alice"}` |
| 6 | Unquoted key quoting | Bare identifier keys | `{name:"Alice"}` becomes `{"name":"Alice"}` |
| 7 | JavaScript literal replacement | `undefined`, `NaN`, `Infinity` | `{"val":undefined}` becomes `{"val":null}` |
| 8 | Truncation completion | Missing closing brackets at end of input | `{"name":"Ali` becomes `{"name":"Ali"}` |

### Repair Levels

| Level | Operations Included | Use Case |
|---|---|---|
| `none` | No repair. Raw output is parsed directly. | When the caller handles repair externally or uses `llm-output-normalizer`. |
| `minimal` | Fence stripping and JSON extraction only (operations 1-2). | When the LLM produces valid JSON but wraps it in prose or fences. |
| `standard` (default) | Operations 1-5. The common set that fixes the vast majority of issues. | General-purpose use. |
| `aggressive` | All operations 1-8. Includes unquoted key quoting, JS literal replacement, and truncation completion. | Local/open-source models that produce heavily malformed output. |

### Custom Repair Functions

The caller can provide additional repair functions that run after the built-in operations:

```typescript
const result = await retryWithValidation(callLLM, UserSchema, {
  repair: {
    level: 'standard',
    custom: [
      (text) => text.replace(/True/g, 'true').replace(/False/g, 'false'),  // Python booleans
      (text) => text.replace(/None/g, 'null'),  // Python None
    ],
  },
});
```

Custom repair functions receive the output string after built-in repairs and return the modified string. They are applied in array order.

### Integration with llm-output-normalizer

When the `useNormalizer` option is set to `true` and `llm-output-normalizer` is installed, the repair stage delegates to `llm-output-normalizer`'s `extractJSON` function, which applies the full nine-step normalization pipeline (Unicode normalization, thinking block removal, XML artifact unwrapping, preamble stripping, postamble stripping, markdown fence extraction, JSON extraction, JSON repair, whitespace cleanup). This provides the most comprehensive repair coverage but adds a dependency:

```typescript
const result = await retryWithValidation(callLLM, UserSchema, {
  repair: { useNormalizer: true },
});
```

If `llm-output-normalizer` is not installed and `useNormalizer` is `true`, the package throws a clear error message at configuration time (not at runtime during a retry) instructing the caller to install the dependency.

---

## 7. Validation

### Zod Schema Validation

The primary validation mechanism. When a Zod schema is provided, `llm-retry` calls `schema.safeParse(parsed)` and processes the result:

```typescript
const result = schema.safeParse(parsed);
if (result.success) {
  return { success: true, data: result.data };
} else {
  return { success: false, errors: formatZodErrors(result.error) };
}
```

Zod validation runs the full pipeline including refinements (`.refine()`), transforms (`.transform()`), and pipes (`.pipe()`). This means `z.string().email()` validates email format, `z.number().int().positive()` validates integer and positivity constraints, and custom refinements like `.refine(user => user.age >= 18, { message: "Must be at least 18" })` are enforced.

### Zod Error Extraction

Each `ZodIssue` in the `ZodError.issues` array is transformed into a structured `ValidationError` object:

```typescript
interface ValidationError {
  /** JSON path to the invalid field, e.g., "$.users[0].email" */
  path: string;

  /** Human-readable error message */
  message: string;

  /** The Zod issue code (e.g., "invalid_type", "too_small", "custom") */
  code: string;

  /** What was expected (e.g., "string", "number", "email") */
  expected?: string;

  /** What was received (e.g., "undefined", "42", "not-an-email") */
  received?: string;
}
```

The `path` field is constructed from the Zod issue's `path` array by joining elements with `.` for object keys and `[n]` for array indices, prefixed with `$`. For example, `['users', 0, 'email']` becomes `$.users[0].email`.

### JSON Schema Validation

When a JSON Schema object is provided instead of a Zod schema, `llm-retry` uses `ajv` for validation. The caller must have `ajv` installed as a peer dependency. `llm-retry` does not bundle `ajv` -- it dynamically imports it at runtime and throws a clear error if it is not available.

```typescript
const result = await retryWithValidation(callLLM, {
  type: 'object',
  properties: {
    name: { type: 'string' },
    age: { type: 'integer', minimum: 0 },
  },
  required: ['name', 'age'],
}, { validationType: 'json-schema' });
```

Ajv validation errors are extracted from `ajv.errors` and transformed into the same `ValidationError` format. Ajv's `instancePath` (e.g., `/users/0/email`) is converted to the `$`-prefixed dot notation used by the feedback formatter.

### Custom Validator

For validation logic that cannot be expressed as a schema -- business rules, cross-field constraints, external lookups -- the caller provides a custom validator function:

```typescript
const result = await retryWithValidation(callLLM, {
  validate: (data: unknown): ValidationResult<User> => {
    if (!data || typeof data !== 'object') {
      return { success: false, errors: [{ path: '$', message: 'Expected an object', code: 'invalid_type' }] };
    }
    const user = data as User;
    const errors: ValidationError[] = [];
    if (user.startDate > user.endDate) {
      errors.push({ path: '$.endDate', message: 'End date must be after start date', code: 'custom' });
    }
    return errors.length === 0 ? { success: true, data: user } : { success: false, errors };
  },
});
```

### Error Formatting for LLM Feedback

The critical distinction between `llm-retry` and generic retry libraries is that validation errors are not just logged -- they are formatted into messages that the LLM can understand and act on. The formatting strategy is designed to maximize the LLM's ability to correct its output.

**What the LLM receives on retry:**

```
Your previous response did not match the required schema. Here are the specific errors:

1. At "$.email" — Expected a valid email address, but received "alice at example dot com".
   Fix: The email field must be a standard email format like "user@domain.com".

2. At "$.age" — Expected a number, but received the string "twenty-five".
   Fix: The age field must be a numeric value like 25, not a word.

3. At "$.address.zip" — Expected a string matching pattern /^\d{5}$/, but received "1234".
   Fix: The zip code must be exactly 5 digits, like "12345".

Your original response was:
{"name": "Alice", "email": "alice at example dot com", "age": "twenty-five", "address": {"zip": "1234"}}

Please return a corrected JSON object with all errors fixed.
```

This format gives the LLM: the exact field that was wrong (path), what was expected (constraint), what it actually produced (received value), a hint on how to fix it, and the full original output for context. Each piece of information increases the probability that the retry succeeds.

---

## 8. Error Feedback

### Feedback Message Structure

The feedback message appended to the conversation on retry consists of three sections:

1. **Error header**: A brief statement that the previous response failed validation. This sets the context for the LLM.
2. **Error list**: A numbered list of validation errors, each with path, expected value, received value, and a correction hint.
3. **Original output** (optional, enabled by default): The LLM's previous response, so it can see exactly what it produced and make targeted corrections rather than generating from scratch.

### Feedback Message Template

The default template is:

```
Your previous response did not pass validation. Please fix the following errors and return a valid response:

{{#each errors}}
{{index}}. At "{{path}}": {{message}}{{#if expected}} (expected {{expected}}, received {{received}}){{/if}}
{{/each}}

{{#if includeOutput}}
Your previous response was:
{{previousOutput}}
{{/if}}

Please return a corrected response that addresses all of the above errors.
```

The template is customizable via the `feedbackTemplate` option. The caller can provide a function `(errors: ValidationError[], previousOutput: string) => string` that generates the feedback message using any format.

### Partial Output Inclusion

By default, the full previous output is included in the feedback message. For very large outputs (10KB+), this can consume a significant portion of the context window. The `feedbackOutputStrategy` option controls this:

| Strategy | Behavior | Use Case |
|---|---|---|
| `full` (default) | Include the entire previous output | Small to medium outputs. Maximum correction context. |
| `truncated` | Include the first N characters (configurable, default 2000) | Large outputs where the full text would waste context. |
| `errors-only` | Include only the fields that had errors, not the full output | Very large outputs. Minimal context consumption. |
| `none` | Do not include the previous output | When the schema is simple enough that error descriptions alone are sufficient. |

### Feedback Message Role

The feedback message is appended as a `user` role message. This is consistent with the conversational structure: the user is telling the assistant that its response was incorrect and needs to be fixed. The assistant role message containing the failed output is preserved in the conversation history (if `preserveHistory` is enabled), so the LLM sees the full sequence: original prompt, its failed response, the user's correction request.

### Schema Inclusion in Feedback

On the first retry, the feedback message optionally includes a reminder of the expected schema (in JSON Schema format or a human-readable description). This is controlled by the `includeSchemaInFeedback` option (default: `true` for the first retry, `false` for subsequent retries). Including the schema on the first retry reinforces the expected structure. Omitting it on subsequent retries avoids wasting context tokens with repetitive information.

When a Zod schema is provided, it is converted to JSON Schema format for inclusion in the feedback message using a built-in lightweight converter (covering the most common Zod types: string, number, boolean, object, array, enum, optional, nullable, with refinement descriptions). For comprehensive Zod-to-JSON-Schema conversion, the caller can install `zod-to-json-schema` and configure `llm-retry` to use it.

---

## 9. Escalation Strategies

### Purpose

Escalation addresses the scenario where repeated retries with the same configuration are not converging on valid output. If the LLM produces the same category of error three times in a row, the fourth retry with the same model, temperature, and prompt is unlikely to succeed. Escalation changes the parameters to break out of the failure pattern.

### Built-in Escalation Actions

| Action | What It Does | When To Use |
|---|---|---|
| `increase-temperature` | Increases the temperature parameter by a configurable delta | When the LLM is stuck in a repetitive failure (producing the same invalid output). Higher temperature introduces more randomness, potentially producing a valid alternative. |
| `switch-model` | Replaces the `callLLM` function with one that uses a different (typically stronger) model | When the current model lacks the capability to produce valid output for the given schema complexity. |
| `simplify-schema` | Removes optional fields, loosens constraints (e.g., removes regex patterns, widens numeric ranges), or removes refinements | When the schema is too demanding for the model. Produces a partial but valid result. |
| `custom` | Invokes a caller-provided function that can modify any aspect of the retry configuration | For domain-specific escalation logic. |

### Escalation Schedule

Escalation is configured as an ordered list of actions triggered at specific retry counts:

```typescript
const result = await retryWithValidation(callLLM, UserSchema, {
  maxRetries: 6,
  escalation: [
    { afterAttempt: 2, action: 'increase-temperature', delta: 0.2 },
    { afterAttempt: 4, action: 'switch-model', callLLM: callGPT4o },
    { afterAttempt: 5, action: 'simplify-schema', removeOptional: true },
  ],
});
```

In this example: attempts 1-2 use the original configuration. If attempt 2 fails, the temperature is increased by 0.2 for attempt 3. If attempt 4 fails, the LLM function is switched to `callGPT4o` for attempt 5. If attempt 5 fails, optional fields are removed from the schema for attempt 6.

### Temperature Escalation

The `increase-temperature` action modifies the temperature parameter passed to the LLM. Since `llm-retry` does not call the LLM directly (it calls the caller's `callLLM` function), temperature escalation works through a mechanism that lets the caller's function read the current temperature:

```typescript
const callLLM = (messages: Message[], context?: RetryContext) => {
  return openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages,
    temperature: context?.temperature ?? 0.0,
  }).then(r => r.choices[0].message.content ?? '');
};
```

The `RetryContext` object is passed as an optional second argument to `callLLM` on each attempt. It contains the current temperature, the attempt number, the model identifier (if escalation has switched models), and any custom escalation state. The caller's function reads these values and applies them to the provider SDK call.

### Model Escalation

The `switch-model` action replaces the `callLLM` function entirely. The caller provides alternative `callLLM` functions for each escalation tier:

```typescript
escalation: [
  {
    afterAttempt: 2,
    action: 'switch-model',
    callLLM: (messages, ctx) =>
      anthropic.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 4096,
        messages: messages.map(m => ({ role: m.role, content: m.content })),
      }).then(r => r.content[0].type === 'text' ? r.content[0].text : ''),
  },
]
```

### Schema Simplification

The `simplify-schema` action programmatically modifies the Zod schema to reduce its complexity. Simplification operations (applied in order, each configurable):

1. **Remove optional fields**: Fields marked with `.optional()` are removed from the schema. The validated result will not contain these fields, but the required fields will be correct.
2. **Widen string constraints**: Remove `.email()`, `.url()`, `.uuid()`, `.regex()` refinements, keeping only `.string()`. The value will be a string but may not match the specific format.
3. **Widen numeric constraints**: Remove `.min()`, `.max()`, `.int()`, `.positive()` refinements, keeping only `.number()`.
4. **Remove custom refinements**: Remove `.refine()` and `.superRefine()` calls.

Schema simplification is a last resort -- it produces a result that partially conforms to the original schema. The `RetryResult` includes a `simplified: true` flag when simplification was applied, and the caller can inspect `result.simplifiedSchema` to see which constraints were relaxed.

### Custom Escalation

The `custom` action invokes a function that receives the current retry state and returns modifications:

```typescript
escalation: [
  {
    afterAttempt: 3,
    action: 'custom',
    fn: (state) => {
      // Add a "please be more careful" system message
      state.messages.unshift({
        role: 'system',
        content: 'You MUST return valid JSON matching the schema exactly. Do not include any text outside the JSON object.',
      });
      // Increase max_tokens
      state.context.maxTokens = 8192;
    },
  },
]
```

---

## 10. API Surface

### Installation

```bash
npm install llm-retry
```

### Peer Dependency

```json
{
  "peerDependencies": {
    "zod": "^3.22.0"
  },
  "peerDependenciesMeta": {
    "zod": { "optional": true }
  }
}
```

Zod is optional because the caller may use JSON Schema or a custom validator instead. If a Zod schema is passed and `zod` is not installed, the package throws a clear error at invocation time.

### Primary Function: `retryWithValidation`

```typescript
import { retryWithValidation } from 'llm-retry';
import { z } from 'zod';

const UserSchema = z.object({
  name: z.string(),
  age: z.number().int().positive(),
  email: z.string().email(),
});

type User = z.infer<typeof UserSchema>;

const result = await retryWithValidation<User>(
  // The LLM call function: messages in, string out
  async (messages) => {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages,
    });
    return response.choices[0].message.content ?? '';
  },
  // The schema
  UserSchema,
  // Options
  {
    messages: [
      { role: 'system', content: 'Return a JSON object with user data.' },
      { role: 'user', content: 'Create a profile for Alice, age 30, email alice@example.com.' },
    ],
    maxRetries: 3,
  },
);

if (result.success) {
  console.log(result.data); // User — fully typed and validated
} else {
  console.error(`Failed after ${result.attempts} attempts.`);
  console.error(result.errors);
}
```

### Factory Function: `createRetryLoop`

```typescript
import { createRetryLoop } from 'llm-retry';
import { z } from 'zod';

const UserSchema = z.object({
  name: z.string(),
  age: z.number().int().positive(),
  email: z.string().email(),
});

const retryLoop = createRetryLoop({
  callLLM: async (messages, context) => {
    const response = await openai.chat.completions.create({
      model: context?.model ?? 'gpt-4o-mini',
      messages,
      temperature: context?.temperature ?? 0.0,
    });
    return response.choices[0].message.content ?? '';
  },
  schema: UserSchema,
  maxRetries: 3,
  repair: { level: 'standard' },
  escalation: [
    { afterAttempt: 2, action: 'increase-temperature', delta: 0.3 },
  ],
});

// Use the configured loop for multiple requests
const result1 = await retryLoop.execute([
  { role: 'user', content: 'Create a profile for Alice.' },
]);

const result2 = await retryLoop.execute([
  { role: 'user', content: 'Create a profile for Bob.' },
]);
```

### Type Definitions

```typescript
// ── Message Types ───────────────────────────────────────────────────

/** A message in the LLM conversation. */
interface Message {
  role: 'system' | 'user' | 'assistant';
  content: string;
}

// ── LLM Call Function ───────────────────────────────────────────────

/** Context passed to the callLLM function on each attempt. */
interface RetryContext {
  /** Current attempt number (1-based). */
  attempt: number;

  /** Current temperature (may be modified by escalation). */
  temperature?: number;

  /** Current model identifier (may be modified by escalation). */
  model?: string;

  /** Maximum tokens for the response. */
  maxTokens?: number;

  /** Whether this attempt is after an escalation action. */
  escalated: boolean;

  /** Custom state set by escalation functions. */
  custom?: Record<string, unknown>;
}

/**
 * Function that calls an LLM and returns the raw text response.
 * The caller wraps their provider SDK call in this function.
 */
type CallLLMFunction = (
  messages: Message[],
  context?: RetryContext,
) => Promise<string>;

// ── Validation ──────────────────────────────────────────────────────

/** A single validation error with path and details. */
interface ValidationError {
  /** JSON path to the invalid field (e.g., "$.users[0].email"). */
  path: string;

  /** Human-readable error message. */
  message: string;

  /** Error code (e.g., "invalid_type", "too_small", "custom"). */
  code: string;

  /** What was expected (e.g., "string", "number >= 0"). */
  expected?: string;

  /** What was actually received (e.g., "undefined", "\"hello\""). */
  received?: string;
}

/** Result of a custom validator function. */
type ValidationResult<T> =
  | { success: true; data: T }
  | { success: false; errors: ValidationError[] };

/** Custom validator function type. */
type ValidatorFunction<T> = (data: unknown) => ValidationResult<T>;

// ── Repair ──────────────────────────────────────────────────────────

/** Repair level controlling which built-in repair operations are applied. */
type RepairLevel = 'none' | 'minimal' | 'standard' | 'aggressive';

/** Custom repair function that transforms the raw output string. */
type RepairFunction = (text: string) => string;

/** Repair configuration. */
interface RepairConfig {
  /** Preset repair level. Default: 'standard'. */
  level?: RepairLevel;

  /** Custom repair functions applied after built-in repairs. */
  custom?: RepairFunction[];

  /**
   * Delegate repair to llm-output-normalizer if installed.
   * Default: false.
   */
  useNormalizer?: boolean;

  /**
   * Options passed to llm-output-normalizer when useNormalizer is true.
   * Accepts the same options as llm-output-normalizer's extractJSON.
   */
  normalizerOptions?: Record<string, unknown>;
}

// ── Escalation ──────────────────────────────────────────────────────

/** An escalation action triggered after a specific number of failed attempts. */
type EscalationAction =
  | {
      afterAttempt: number;
      action: 'increase-temperature';
      /** Amount to increase temperature by. Default: 0.2. */
      delta?: number;
      /** Maximum temperature ceiling. Default: 1.0. */
      maxTemperature?: number;
    }
  | {
      afterAttempt: number;
      action: 'switch-model';
      /** Replacement callLLM function that uses a different model. */
      callLLM: CallLLMFunction;
      /** Identifier for the new model (for metadata). */
      model?: string;
    }
  | {
      afterAttempt: number;
      action: 'simplify-schema';
      /** Remove fields marked as optional. Default: true. */
      removeOptional?: boolean;
      /** Remove string format refinements. Default: true. */
      removeStringFormats?: boolean;
      /** Remove numeric range constraints. Default: false. */
      removeNumericConstraints?: boolean;
      /** Remove custom refinements. Default: true. */
      removeRefinements?: boolean;
    }
  | {
      afterAttempt: number;
      action: 'custom';
      /** Custom escalation function. */
      fn: (state: EscalationState) => void | Promise<void>;
    };

/** State available to custom escalation functions. */
interface EscalationState {
  /** The current messages array (mutable). */
  messages: Message[];

  /** The current retry context (mutable). */
  context: RetryContext;

  /** The validation errors from the most recent attempt. */
  lastErrors: ValidationError[];

  /** The raw output from the most recent attempt. */
  lastOutput: string;

  /** The current attempt count. */
  attempt: number;
}

// ── API-Level Retry ─────────────────────────────────────────────────

/** Configuration for API-level retry (transient errors). */
interface ApiRetryConfig {
  /** Whether to enable API-level retry. Default: true. */
  enabled?: boolean;

  /** Maximum number of API-level retries per LLM call. Default: 3. */
  maxRetries?: number;

  /** Initial delay in milliseconds before the first retry. Default: 1000. */
  initialDelayMs?: number;

  /** Maximum delay in milliseconds between retries. Default: 60000. */
  maxDelayMs?: number;

  /** Exponential backoff multiplier. Default: 2. */
  multiplier?: number;

  /**
   * Jitter strategy.
   * - 'full': Random delay between 0 and the calculated backoff.
   * - 'equal': Half the backoff plus a random value up to the other half.
   * - 'decorrelated': Each delay is random between initialDelay and 3x the previous delay.
   * Default: 'full'.
   */
  jitter?: 'full' | 'equal' | 'decorrelated' | 'none';

  /**
   * Function that determines whether an error is retriable at the API level.
   * Default: Returns true for errors with status 429, 500, 502, 503, 529 or network errors.
   */
  isRetriable?: (error: Error) => boolean;

  /**
   * Whether to respect the Retry-After header on 429 responses.
   * Default: true.
   */
  respectRetryAfter?: boolean;
}

// ── Feedback ────────────────────────────────────────────────────────

/** How much of the previous output to include in feedback. */
type FeedbackOutputStrategy = 'full' | 'truncated' | 'errors-only' | 'none';

/** Feedback configuration. */
interface FeedbackConfig {
  /** How much of the previous output to include. Default: 'full'. */
  outputStrategy?: FeedbackOutputStrategy;

  /** Maximum characters of output to include when strategy is 'truncated'. Default: 2000. */
  maxOutputLength?: number;

  /** Whether to include the schema description in the first retry. Default: true. */
  includeSchema?: boolean;

  /** Custom feedback message generator. Overrides the default template. */
  template?: (errors: ValidationError[], previousOutput: string, attempt: number) => string;
}

// ── Event Hooks ─────────────────────────────────────────────────────

/** Metadata about a single attempt. */
interface AttemptRecord {
  /** Attempt number (1-based). */
  attempt: number;

  /** Raw output from the LLM. */
  rawOutput: string;

  /** Output after repair (if repair was applied). */
  repairedOutput?: string;

  /** Whether repair was applied and changed the output. */
  repaired: boolean;

  /** List of repair operations that were applied. */
  repairs: string[];

  /** Whether parsing succeeded. */
  parsed: boolean;

  /** Parse error, if parsing failed. */
  parseError?: string;

  /** Whether validation succeeded. */
  valid: boolean;

  /** Validation errors, if validation failed. */
  validationErrors: ValidationError[];

  /** Whether an escalation action was triggered on this attempt. */
  escalated: boolean;

  /** Description of the escalation action, if any. */
  escalationAction?: string;

  /** Duration of this attempt in milliseconds. */
  durationMs: number;

  /** Estimated token usage for this attempt (if reported by callLLM). */
  tokenUsage?: {
    promptTokens?: number;
    completionTokens?: number;
    totalTokens?: number;
  };
}

/** Event hooks for observability and custom logic injection. */
interface EventHooks {
  /** Called before each LLM call. */
  onAttempt?: (attempt: number, messages: Message[]) => void | Promise<void>;

  /** Called when repair is applied and changes the output. */
  onRepair?: (attempt: number, original: string, repaired: string, repairs: string[]) => void;

  /** Called when validation fails. */
  onValidationError?: (attempt: number, errors: ValidationError[], rawOutput: string) => void;

  /** Called when an escalation action is triggered. */
  onEscalation?: (attempt: number, action: string) => void;

  /** Called when validation succeeds. */
  onSuccess?: (attempt: number, data: unknown) => void;

  /** Called when all retries are exhausted without success. */
  onFailure?: (attempts: AttemptRecord[]) => void;
}

// ── Options ─────────────────────────────────────────────────────────

/** Options for retryWithValidation. */
interface RetryOptions {
  /** Initial messages for the LLM conversation. Required. */
  messages: Message[];

  /** Maximum number of output-level retries. Default: 3. */
  maxRetries?: number;

  /** Overall timeout in milliseconds for the entire retry loop. Default: 120000 (2 minutes). */
  timeoutMs?: number;

  /** Repair configuration. Default: { level: 'standard' }. */
  repair?: RepairConfig | RepairLevel | false;

  /** API-level retry configuration. Default: { enabled: true }. */
  apiRetry?: ApiRetryConfig | false;

  /** Escalation schedule. Default: [] (no escalation). */
  escalation?: EscalationAction[];

  /** Feedback configuration. Default: {}. */
  feedback?: FeedbackConfig;

  /** Event hooks. Default: {}. */
  hooks?: EventHooks;

  /**
   * Custom parser function. Default: JSON.parse.
   * Use this for non-JSON output formats.
   */
  parser?: (text: string) => unknown;

  /**
   * Whether to preserve the full conversation history across retries.
   * If true, each retry appends to the existing messages array (assistant response + user feedback).
   * If false, only the feedback message is appended (not the assistant's failed response).
   * Default: true.
   */
  preserveHistory?: boolean;

  /** AbortSignal for external cancellation. */
  signal?: AbortSignal;

  /** Initial temperature. Default: undefined (not set). */
  temperature?: number;

  /** Initial model identifier (for metadata tracking). Default: undefined. */
  model?: string;
}

// ── Result ──────────────────────────────────────────────────────────

/** Result of a retry loop execution. */
interface RetryResult<T> {
  /** Whether the retry loop produced a valid result. */
  success: boolean;

  /** The validated data. Present when success is true. */
  data: T | null;

  /** Total number of attempts made (including the first). */
  attempts: number;

  /** Whether local repair was applied during any attempt. */
  repaired: boolean;

  /** Whether escalation was triggered during any attempt. */
  escalated: boolean;

  /** Whether schema simplification was applied. */
  simplified: boolean;

  /** Total duration of the retry loop in milliseconds. */
  totalMs: number;

  /** Detailed record of each attempt. */
  history: AttemptRecord[];

  /** The validation errors from the last attempt (if failed). */
  errors: ValidationError[];

  /** The raw output from the last attempt (if failed). */
  lastOutput: string | null;

  /** Aggregate token usage across all attempts. */
  totalTokenUsage: {
    promptTokens: number;
    completionTokens: number;
    totalTokens: number;
  };
}

// ── Retry Loop Instance ─────────────────────────────────────────────

/** Configuration for createRetryLoop. */
interface RetryLoopConfig extends Omit<RetryOptions, 'messages'> {
  /** The LLM call function. Required. */
  callLLM: CallLLMFunction;

  /** The validation schema (Zod, JSON Schema, or custom validator). Required. */
  schema: import('zod').ZodType<any> | object | ValidatorFunction<any>;

  /** If schema is a JSON Schema object, set this to 'json-schema'. Default: 'zod'. */
  validationType?: 'zod' | 'json-schema' | 'custom';
}

/** A preconfigured, reusable retry loop instance. */
interface RetryLoop<T> {
  /** Execute the retry loop with the given messages. */
  execute(messages: Message[]): Promise<RetryResult<T>>;

  /** Execute with an AbortSignal for cancellation. */
  execute(messages: Message[], signal: AbortSignal): Promise<RetryResult<T>>;
}
```

### Function Signatures

```typescript
/**
 * Call an LLM, validate the output against a schema, and retry with
 * error feedback until the output is valid or retries are exhausted.
 *
 * @param callLLM - Function that calls the LLM. Receives messages, returns raw text.
 * @param schema - Zod schema, JSON Schema object, or custom validator function.
 * @param options - Configuration options.
 * @returns A RetryResult containing the validated data or failure details.
 */
function retryWithValidation<T>(
  callLLM: CallLLMFunction,
  schema: import('zod').ZodType<T> | object | ValidatorFunction<T>,
  options: RetryOptions,
): Promise<RetryResult<T>>;

/**
 * Create a preconfigured retry loop for repeated use.
 *
 * @param config - Full retry loop configuration.
 * @returns A RetryLoop instance with an execute method.
 */
function createRetryLoop<T>(config: RetryLoopConfig): RetryLoop<T>;
```

---

## 11. Provider Integration

### Generic Contract

`llm-retry` accepts any function matching `(messages: Message[], context?: RetryContext) => Promise<string>`. This is the only integration point. The caller wraps their provider SDK call in this function. The function must return the raw text content of the LLM response.

### OpenAI Adapter Pattern

```typescript
import OpenAI from 'openai';
import { retryWithValidation } from 'llm-retry';

const openai = new OpenAI();

const callOpenAI: CallLLMFunction = async (messages, context) => {
  const response = await openai.chat.completions.create({
    model: context?.model ?? 'gpt-4o-mini',
    messages: messages.map(m => ({
      role: m.role as 'system' | 'user' | 'assistant',
      content: m.content,
    })),
    temperature: context?.temperature ?? 0.0,
    response_format: { type: 'json_object' },
  });
  return response.choices[0].message.content ?? '';
};

const result = await retryWithValidation(callOpenAI, UserSchema, {
  messages: [{ role: 'user', content: 'Generate a user profile for Alice.' }],
});
```

Note: When using OpenAI's `json_object` response format, the output is guaranteed to be valid JSON, so repair is less likely to be needed. However, the output may still fail Zod schema validation (wrong types, missing required fields, constraint violations), which is where `llm-retry`'s feedback re-prompting provides value.

### Anthropic Adapter Pattern

```typescript
import Anthropic from '@anthropic-ai/sdk';
import { retryWithValidation } from 'llm-retry';

const anthropic = new Anthropic();

const callAnthropic: CallLLMFunction = async (messages, context) => {
  const systemMessages = messages.filter(m => m.role === 'system');
  const nonSystemMessages = messages.filter(m => m.role !== 'system');

  const response = await anthropic.messages.create({
    model: context?.model ?? 'claude-haiku-3-5-20241022',
    max_tokens: context?.maxTokens ?? 4096,
    system: systemMessages.map(m => m.content).join('\n\n') || undefined,
    messages: nonSystemMessages.map(m => ({
      role: m.role as 'user' | 'assistant',
      content: m.content,
    })),
  });

  const textBlock = response.content.find(block => block.type === 'text');
  return textBlock?.text ?? '';
};

const result = await retryWithValidation(callAnthropic, UserSchema, {
  messages: [
    { role: 'system', content: 'Return a JSON object. No prose, no fences.' },
    { role: 'user', content: 'Generate a user profile for Bob.' },
  ],
  escalation: [
    { afterAttempt: 2, action: 'switch-model', callLLM: callAnthropicSonnet, model: 'claude-sonnet-4-20250514' },
  ],
});
```

### Google Gemini Adapter Pattern

```typescript
import { GoogleGenerativeAI } from '@google/generative-ai';
import { retryWithValidation } from 'llm-retry';

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_API_KEY!);

const callGemini: CallLLMFunction = async (messages, context) => {
  const model = genAI.getGenerativeModel({
    model: context?.model ?? 'gemini-2.0-flash',
    generationConfig: {
      responseMimeType: 'application/json',
      temperature: context?.temperature ?? 0.0,
    },
  });
  const contents = messages
    .filter(m => m.role !== 'system')
    .map(m => ({
      role: m.role === 'assistant' ? 'model' : 'user',
      parts: [{ text: m.content }],
    }));
  const result = await model.generateContent({ contents });
  return result.response.text();
};
```

### Vercel AI SDK Adapter Pattern

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { retryWithValidation } from 'llm-retry';

const callVercelAI: CallLLMFunction = async (messages, context) => {
  const { text } = await generateText({
    model: openai(context?.model ?? 'gpt-4o-mini'),
    messages: messages.map(m => ({
      role: m.role as 'system' | 'user' | 'assistant',
      content: m.content,
    })),
    temperature: context?.temperature ?? 0.0,
  });
  return text;
};
```

### Ollama / Local Model Pattern

```typescript
import { retryWithValidation } from 'llm-retry';

const callOllama: CallLLMFunction = async (messages, context) => {
  const response = await fetch('http://localhost:11434/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: context?.model ?? 'llama3.2',
      messages: messages.map(m => ({ role: m.role, content: m.content })),
      stream: false,
      options: { temperature: context?.temperature ?? 0.0 },
    }),
  });
  const data = await response.json();
  return data.message.content;
};

// Local models benefit most from aggressive repair
const result = await retryWithValidation(callOllama, UserSchema, {
  messages: [{ role: 'user', content: 'Return JSON: { name, age, email }' }],
  repair: { level: 'aggressive' },
  maxRetries: 5,
});
```

---

## 12. API-Level Retry

### Scope

API-level retry handles transient infrastructure failures that prevent the LLM call from completing. These are failures at the HTTP transport layer, not at the output content layer. API-level retry is the inner loop -- it wraps each individual `callLLM` invocation. Output-level retry is the outer loop -- it wraps the entire parse-validate-feedback cycle.

### Retriable Errors

The default `isRetriable` function classifies errors as retriable based on HTTP status codes and error types:

| Error Type | Status Code | Retriable | Rationale |
|---|---|---|---|
| Rate limit | 429 | Yes | The request exceeded the provider's rate limit. Backing off and retrying will succeed once the rate budget replenishes. |
| Server error | 500 | Yes | Internal server error. Often transient. |
| Bad gateway | 502 | Yes | Gateway/proxy issue. Usually resolves quickly. |
| Service unavailable | 503 | Yes | Server temporarily overloaded. Backoff helps. |
| Overloaded (Anthropic) | 529 | Yes | Anthropic-specific overloaded status. |
| Gateway timeout | 504 | Yes | Request timed out at the gateway. |
| Network error | N/A | Yes | `ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`, DNS failures. |
| Request timeout | N/A | Yes | The request exceeded the configured timeout. |
| Authentication error | 401 | No | Invalid API key. Retrying will not help. |
| Permission denied | 403 | No | The API key lacks permission. Retrying will not help. |
| Not found | 404 | No | Invalid endpoint or model. Retrying will not help. |
| Invalid request | 400 | No | Malformed request body. Retrying the same request will fail. |
| Content policy | 400 (varies) | No | The prompt was refused by the provider's content policy. |

The caller can override `isRetriable` to customize which errors trigger API-level retry.

### Exponential Backoff with Jitter

When an API-level retry is triggered, the delay before the next attempt is calculated using exponential backoff with jitter. The base formula is:

```
baseDelay = min(initialDelayMs * multiplier^(attempt - 1), maxDelayMs)
```

Jitter is then applied based on the selected strategy:

| Strategy | Formula | Behavior |
|---|---|---|
| `full` (default) | `random(0, baseDelay)` | Uniformly random between 0 and the full backoff. Provides the best spread across concurrent clients. Recommended by AWS architecture best practices. |
| `equal` | `baseDelay / 2 + random(0, baseDelay / 2)` | Guarantees at least half the base delay. Less aggressive spread but avoids very short delays. |
| `decorrelated` | `random(initialDelayMs, previousDelay * 3)` | Each delay is independent of the backoff formula, based only on the previous delay. Good for highly variable server recovery times. |
| `none` | `baseDelay` | No jitter. All concurrent clients retry at the same time (thundering herd). Not recommended for production use. |

### Retry-After Header

When the provider returns a `429` response with a `Retry-After` header, `llm-retry` respects it. If the `Retry-After` value (in seconds or as an HTTP date) specifies a longer delay than the exponential backoff would calculate, the `Retry-After` value is used. If it specifies a shorter delay, the exponential backoff value is used (to avoid retrying too aggressively). This behavior is controlled by the `respectRetryAfter` option (default: `true`).

### Composing with Provider SDK Retries

Some provider SDKs have built-in retry logic (the OpenAI SDK retries 429 and 5xx errors by default). When using `llm-retry`'s API-level retry alongside a provider SDK's built-in retry, the retries can compound. To avoid this, the caller should either:

1. Disable `llm-retry`'s API-level retry: `apiRetry: false`
2. Or disable the provider SDK's built-in retry: `new OpenAI({ maxRetries: 0 })`

The recommended approach is to disable the provider SDK's retry and let `llm-retry` handle it, because `llm-retry`'s retry logic is configurable and observable (via event hooks and attempt records).

---

## 13. Configuration

### Default Values

| Option | Default | Description |
|---|---|---|
| `maxRetries` | `3` | Maximum output-level retries. Total attempts = maxRetries + 1 (initial + retries). |
| `timeoutMs` | `120000` (2 min) | Overall timeout for the entire retry loop. |
| `repair.level` | `'standard'` | Built-in repair operations: fence strip, JSON extract, trailing comma, comments, single quotes. |
| `repair.useNormalizer` | `false` | Whether to delegate repair to llm-output-normalizer. |
| `apiRetry.enabled` | `true` | Whether API-level retry is active. |
| `apiRetry.maxRetries` | `3` | Maximum API-level retries per LLM call. |
| `apiRetry.initialDelayMs` | `1000` | Initial backoff delay. |
| `apiRetry.maxDelayMs` | `60000` | Maximum backoff delay. |
| `apiRetry.multiplier` | `2` | Exponential backoff multiplier. |
| `apiRetry.jitter` | `'full'` | Jitter strategy. |
| `apiRetry.respectRetryAfter` | `true` | Respect Retry-After headers on 429 responses. |
| `feedback.outputStrategy` | `'full'` | Include the full previous output in feedback. |
| `feedback.includeSchema` | `true` | Include schema description in the first retry's feedback. |
| `preserveHistory` | `true` | Keep full conversation history across retries. |
| `parser` | `JSON.parse` | Parse function for raw output. |
| `escalation` | `[]` | No escalation by default. |

### Configuration Validation

All configuration values are validated at invocation time (not lazily during the retry loop). Invalid values produce clear, actionable error messages:

- `maxRetries` must be a non-negative integer. Negative or non-integer values throw `TypeError: maxRetries must be a non-negative integer, received -1`.
- `timeoutMs` must be a positive number or `Infinity`. Zero or negative values throw `TypeError: timeoutMs must be a positive number, received 0`.
- `escalation[n].afterAttempt` must be between 1 and `maxRetries` inclusive. Values outside this range throw `RangeError: escalation[0].afterAttempt (5) exceeds maxRetries (3)`.
- `apiRetry.initialDelayMs` must be positive. Zero or negative values throw `TypeError`.
- `apiRetry.multiplier` must be >= 1. Values below 1 throw `TypeError`.

### Shorthand Configuration

For common configurations, shorthand options reduce verbosity:

```typescript
// Disable repair entirely
retryWithValidation(callLLM, schema, { repair: false, ... });

// Use a repair level string instead of a config object
retryWithValidation(callLLM, schema, { repair: 'aggressive', ... });

// Disable API-level retry
retryWithValidation(callLLM, schema, { apiRetry: false, ... });
```

---

## 14. Observability

### Event Hooks

Event hooks are the primary observability mechanism. Each hook fires at a specific point in the retry loop and receives context about the current state.

```typescript
const result = await retryWithValidation(callLLM, schema, {
  messages,
  hooks: {
    onAttempt: (attempt, messages) => {
      console.log(`[attempt ${attempt}] Calling LLM with ${messages.length} messages`);
    },
    onRepair: (attempt, original, repaired, repairs) => {
      console.log(`[attempt ${attempt}] Repaired output: ${repairs.join(', ')}`);
    },
    onValidationError: (attempt, errors, rawOutput) => {
      console.log(`[attempt ${attempt}] Validation failed: ${errors.length} errors`);
      for (const err of errors) {
        console.log(`  ${err.path}: ${err.message}`);
      }
    },
    onEscalation: (attempt, action) => {
      console.log(`[attempt ${attempt}] Escalating: ${action}`);
    },
    onSuccess: (attempt, data) => {
      console.log(`[attempt ${attempt}] Success!`);
    },
    onFailure: (attempts) => {
      console.log(`Failed after ${attempts.length} attempts`);
    },
  },
});
```

### Structured Logging

For production systems that use structured logging (JSON logs to stdout), the event hooks can forward to a logger:

```typescript
import pino from 'pino';
const logger = pino();

const hooks: EventHooks = {
  onAttempt: (attempt) => logger.info({ attempt, event: 'llm_retry_attempt' }),
  onRepair: (attempt, _, __, repairs) => logger.info({ attempt, repairs, event: 'llm_retry_repair' }),
  onValidationError: (attempt, errors) => logger.warn({ attempt, errorCount: errors.length, errors, event: 'llm_retry_validation_error' }),
  onEscalation: (attempt, action) => logger.info({ attempt, action, event: 'llm_retry_escalation' }),
  onSuccess: (attempt) => logger.info({ attempt, event: 'llm_retry_success' }),
  onFailure: (attempts) => logger.error({ attemptCount: attempts.length, event: 'llm_retry_failure' }),
};
```

### Result Metadata

The `RetryResult` object provides comprehensive metadata for post-hoc analysis:

- `result.attempts`: Total number of attempts made.
- `result.totalMs`: Wall-clock time for the entire retry loop.
- `result.repaired`: Whether repair was applied during any attempt.
- `result.escalated`: Whether escalation was triggered.
- `result.simplified`: Whether schema simplification was applied.
- `result.history`: Array of `AttemptRecord` objects, one per attempt, with raw output, repair details, validation errors, duration, and token usage.
- `result.totalTokenUsage`: Aggregate token counts across all attempts.

### Cost Tracking

Token usage is tracked per attempt and aggregated in the result. The `callLLM` function can report token usage by setting properties on the `RetryContext`:

```typescript
const callLLM: CallLLMFunction = async (messages, context) => {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages,
  });
  // Report token usage back to llm-retry
  if (context) {
    context.custom = {
      ...context.custom,
      tokenUsage: {
        promptTokens: response.usage?.prompt_tokens ?? 0,
        completionTokens: response.usage?.completion_tokens ?? 0,
        totalTokens: response.usage?.total_tokens ?? 0,
      },
    };
  }
  return response.choices[0].message.content ?? '';
};
```

Alternatively, the `onAttempt` hook's attempt record captures token usage if the `callLLM` function sets it on the context.

---

## 15. Testing Strategy

### Test Categories

**Unit tests: Repair operations** -- Each built-in repair operation is tested individually with targeted input/output pairs. Fence stripping is tested with backtick fences, tilde fences, fences with language tags, nested fences, and unclosed fences. Trailing comma removal is tested with objects, arrays, nested structures, and strings that contain commas. Each repair operation is tested in isolation and in combination with other repairs.

**Unit tests: Validation error formatting** -- Zod errors with various issue types (invalid_type, too_small, too_big, invalid_enum_value, custom, unrecognized_keys) are formatted into feedback messages and verified against expected output. Error paths with nested objects, arrays, and array elements are tested. JSON Schema validation errors (via ajv) are similarly formatted and verified.

**Unit tests: Error feedback message generation** -- The feedback message template is tested with various combinations: single error, multiple errors, errors with and without expected/received values, full output inclusion, truncated output inclusion, errors-only output inclusion, schema inclusion.

**Unit tests: Escalation** -- Each escalation action type is tested: temperature increase (verify the context reflects the new temperature), model switch (verify the new callLLM function is invoked), schema simplification (verify optional fields are removed, refinements are stripped), custom escalation (verify the custom function is called with the correct state). The escalation schedule is tested with multiple actions at different attempt counts.

**Integration tests: Full retry loop** -- End-to-end tests that simulate the complete retry loop with a mock `callLLM` function. The mock returns invalid output on the first N calls and valid output on the last call, verifying that repair, validation, feedback, and retry work together correctly. Test scenarios include:
- First attempt succeeds (no retry needed).
- First attempt has malformed JSON that repair fixes (no retry needed).
- First attempt fails validation, second attempt succeeds after feedback.
- All retries fail, result reports failure with full history.
- Escalation triggers on schedule and changes behavior.
- API-level retry triggers on 429/500 errors within an output-level retry.
- Timeout terminates the loop and returns the best partial result.
- AbortSignal cancels the loop.

**Integration tests: Provider adapters** -- Tests with realistic mock responses matching each provider's response format (OpenAI, Anthropic, Google, Ollama) verify that the adapter pattern works correctly.

**Edge case tests** -- Empty string output, output that is only whitespace, output that is valid JSON but does not match the schema at all (e.g., a string when an object is expected), output that is a JSON array when an object is expected, extremely large output (100KB+), output with Unicode characters, output with control characters.

### Test Organization

```
src/__tests__/
  retry-loop.test.ts              -- Full retry loop integration tests
  repair/
    fence-strip.test.ts           -- Markdown fence stripping
    json-extract.test.ts          -- JSON extraction from prose
    trailing-comma.test.ts        -- Trailing comma removal
    single-quotes.test.ts         -- Single-to-double quote conversion
    truncation.test.ts            -- Truncation completion
    comment-removal.test.ts       -- Comment removal
    custom-repair.test.ts         -- Custom repair functions
  validation/
    zod-validation.test.ts        -- Zod schema validation
    json-schema-validation.test.ts -- JSON Schema validation (with ajv)
    custom-validator.test.ts      -- Custom validator functions
    error-formatting.test.ts      -- Validation error formatting
  feedback/
    message-generation.test.ts    -- Feedback message template
    output-strategies.test.ts     -- Full/truncated/errors-only/none
    schema-inclusion.test.ts      -- Schema description in feedback
  escalation/
    temperature.test.ts           -- Temperature escalation
    model-switch.test.ts          -- Model switching
    schema-simplify.test.ts       -- Schema simplification
    custom-escalation.test.ts     -- Custom escalation functions
    schedule.test.ts              -- Escalation schedule ordering
  api-retry/
    backoff.test.ts               -- Exponential backoff calculation
    jitter.test.ts                -- Jitter strategies
    retriable-errors.test.ts      -- Error classification
    retry-after.test.ts           -- Retry-After header handling
  providers/
    openai.test.ts                -- OpenAI adapter pattern
    anthropic.test.ts             -- Anthropic adapter pattern
    gemini.test.ts                -- Google Gemini adapter pattern
    ollama.test.ts                -- Ollama adapter pattern
  fixtures/
    schemas.ts                    -- Test Zod schemas
    mock-responses.ts             -- Mock LLM responses (valid and invalid)
    mock-errors.ts                -- Mock API errors (429, 500, etc.)
```

### Test Runner

`vitest` (already configured in `package.json`).

---

## 16. Performance

### Latency Overhead

`llm-retry` adds negligible latency to successful first attempts. The overhead consists of: one repair pass (string operations, sub-millisecond for typical output under 10KB), one `JSON.parse` call (sub-millisecond), and one `schema.safeParse` call (sub-millisecond for typical schemas). Total overhead: under 1ms for a successful first attempt. The LLM API call itself (500ms to 30s) dominates the total latency.

For retries, the latency is dominated by the additional LLM calls. Each retry adds one full LLM call round-trip plus the sub-millisecond overhead of repair, parse, and validate. For API-level retries, the exponential backoff delay is added.

### Cost Implications

Each output-level retry makes an additional LLM API call, consuming tokens and incurring cost. The cost of retry is the cost of the additional prompt tokens (the original prompt plus the growing conversation history with feedback messages) plus the cost of the additional completion tokens.

The conversation history grows with each retry because feedback messages and failed responses are appended. For a schema with 5 validation errors, the feedback message may add 500-1000 tokens. Over 3 retries, the cumulative history growth is 1500-3000 tokens of prompt overhead. This is typically a small fraction of the total prompt size.

**Repair saves money.** The repair stage resolves formatting issues without an LLM call. If repair fixes the output on the first attempt, the cost is zero additional tokens. Across a pipeline processing 1000 LLM responses where 5% need fence stripping or comma removal, repair saves 50 LLM calls.

**Escalation trades cost for reliability.** Switching to a stronger model increases per-call cost but reduces the number of retries needed. GPT-4o-mini at $0.15/1M input tokens may need 3 retries, while GPT-4o at $2.50/1M input tokens succeeds on the first retry. The total cost may be comparable, but the latency is lower with escalation.

### Memory Usage

Memory usage is proportional to the conversation history size plus the size of the largest single LLM response. The `AttemptRecord` array stores the raw output and repair output of each attempt, which may be significant for very large responses (100KB+). For typical responses (1-10KB) and 3 retries, the total memory footprint is under 100KB.

The `preserveHistory: false` option reduces memory usage by not storing the full conversation history in the messages array. Only the current messages and feedback are kept, not the full thread of failed attempts.

---

## 17. Dependencies

### Runtime Dependencies

None. `llm-retry` has zero required runtime dependencies. All retry logic, repair operations, error formatting, backoff calculation, and jitter generation are implemented using built-in JavaScript APIs (`JSON.parse`, `Math.random`, `setTimeout`, `String.prototype` methods).

### Peer Dependencies

| Package | Version | Required | Purpose |
|---|---|---|---|
| `zod` | `^3.22.0` | Optional | Schema validation. Required when passing a Zod schema. Not needed when using JSON Schema or a custom validator. |

### Optional Peer Dependencies

| Package | Version | Purpose |
|---|---|---|
| `ajv` | `^8.12.0` | JSON Schema validation. Required when using `validationType: 'json-schema'`. |
| `llm-output-normalizer` | `*` | Comprehensive output repair. Required when using `repair: { useNormalizer: true }`. |
| `zod-to-json-schema` | `^3.22.0` | Convert Zod schemas to JSON Schema for feedback messages. Improves schema description quality in feedback. |

### Development Dependencies

| Package | Purpose |
|---|---|
| `typescript` | TypeScript compiler |
| `vitest` | Test runner |
| `eslint` | Linting |
| `@types/node` | Node.js type definitions |
| `zod` | Used in tests for schema creation |

### Why Minimal Dependencies

The package processes LLM output through repair, parse, and validate operations. The repair and parse operations use string manipulation and `JSON.parse` -- no external library is needed. Backoff and jitter are arithmetic operations. The only external library that provides value beyond what can be trivially implemented is `zod` for schema validation, and it is a peer dependency that the caller already has in their project. Keeping the dependency tree minimal reduces install size, eliminates supply chain risk, and avoids version conflicts in the caller's project.

---

## 18. File Structure

```
llm-retry/
  package.json
  tsconfig.json
  SPEC.md
  README.md
  src/
    index.ts                       -- Public API exports
    retry-loop.ts                  -- Core retry loop implementation
    types.ts                       -- All TypeScript type definitions
    repair/
      index.ts                     -- Repair pipeline orchestration
      fence-strip.ts               -- Markdown fence stripping
      json-extract.ts              -- JSON extraction from prose
      trailing-comma.ts            -- Trailing comma removal
      comment-removal.ts           -- Comment removal
      single-quotes.ts             -- Single-to-double quote conversion
      unquoted-keys.ts             -- Unquoted key quoting
      js-literals.ts               -- JavaScript literal replacement
      truncation.ts                -- Truncation completion
    validation/
      index.ts                     -- Validation dispatcher (Zod / JSON Schema / custom)
      zod-validator.ts             -- Zod schema validation and error extraction
      json-schema-validator.ts     -- JSON Schema validation via ajv
      error-formatter.ts           -- ValidationError formatting from Zod/ajv errors
    feedback/
      index.ts                     -- Feedback message generation
      template.ts                  -- Default feedback template
      schema-description.ts        -- Schema-to-description conversion for feedback
    escalation/
      index.ts                     -- Escalation schedule evaluation
      temperature.ts               -- Temperature escalation action
      model-switch.ts              -- Model switch escalation action
      schema-simplify.ts           -- Schema simplification logic
    api-retry/
      index.ts                     -- API-level retry wrapper
      backoff.ts                   -- Exponential backoff calculation
      jitter.ts                    -- Jitter strategy implementations
      retriable.ts                 -- Error classification (retriable vs non-retriable)
    factory.ts                     -- createRetryLoop factory function
  src/__tests__/
    retry-loop.test.ts
    repair/
      fence-strip.test.ts
      json-extract.test.ts
      trailing-comma.test.ts
      single-quotes.test.ts
      truncation.test.ts
      comment-removal.test.ts
      custom-repair.test.ts
    validation/
      zod-validation.test.ts
      json-schema-validation.test.ts
      custom-validator.test.ts
      error-formatting.test.ts
    feedback/
      message-generation.test.ts
      output-strategies.test.ts
      schema-inclusion.test.ts
    escalation/
      temperature.test.ts
      model-switch.test.ts
      schema-simplify.test.ts
      custom-escalation.test.ts
      schedule.test.ts
    api-retry/
      backoff.test.ts
      jitter.test.ts
      retriable-errors.test.ts
      retry-after.test.ts
    providers/
      openai.test.ts
      anthropic.test.ts
      gemini.test.ts
      ollama.test.ts
    fixtures/
      schemas.ts
      mock-responses.ts
      mock-errors.ts
  dist/                            -- Compiled output (generated by tsc)
```

---

## 19. Implementation Roadmap

### Phase 1: Core Retry Loop (v0.1.0)

Implement the foundation: the retry loop, basic repair, Zod validation, and feedback generation.

1. **Types**: Define all TypeScript types in `types.ts` -- `Message`, `RetryContext`, `CallLLMFunction`, `ValidationError`, `AttemptRecord`, `RetryResult`, `RetryOptions`.
2. **Repair pipeline**: Implement fence stripping, JSON extraction from prose, and trailing comma removal. These three operations cover the most common formatting issues.
3. **Zod validation**: Implement Zod `safeParse` integration with error extraction. Transform `ZodIssue` objects into `ValidationError` objects with path, message, expected, and received.
4. **Error feedback generation**: Implement the default feedback message template. Format validation errors into a numbered list with paths and details. Include the previous output.
5. **Retry loop**: Implement the core loop -- call LLM, repair, parse, validate, format errors, append feedback, retry. Implement `maxRetries` and `timeoutMs` termination.
6. **Public API**: Export `retryWithValidation` as the primary function.
7. **Tests**: Full test suite for repair operations, validation, feedback generation, and the retry loop.

### Phase 2: API-Level Retry and Escalation (v0.2.0)

Add the API-level retry inner loop and escalation strategies.

1. **Exponential backoff**: Implement backoff calculation with all four jitter strategies (full, equal, decorrelated, none).
2. **Retriable error classification**: Implement the default `isRetriable` function for 429/5xx/network errors.
3. **Retry-After handling**: Parse and respect the `Retry-After` header on 429 responses.
4. **API-level retry wrapper**: Wrap the `callLLM` function with the inner retry loop.
5. **Temperature escalation**: Implement temperature increase via `RetryContext`.
6. **Model escalation**: Implement `callLLM` function replacement.
7. **Schema simplification**: Implement programmatic Zod schema simplification (remove optional fields, strip refinements).
8. **Custom escalation**: Implement the custom escalation function interface.
9. **Escalation schedule**: Implement schedule evaluation and action dispatching.
10. **Tests**: Tests for all backoff/jitter calculations, escalation actions, and schedule evaluation.

### Phase 3: Advanced Features (v0.3.0)

Add JSON Schema validation, the factory function, and advanced repair.

1. **JSON Schema validation**: Implement ajv-based validation with dynamic import and error extraction.
2. **Custom validator**: Implement the `ValidatorFunction` interface.
3. **createRetryLoop factory**: Implement the factory function that returns a configured `RetryLoop` instance.
4. **Advanced repair operations**: Implement single-to-double quote conversion, unquoted key quoting, JavaScript literal replacement, comment removal, and truncation completion.
5. **llm-output-normalizer integration**: Implement the `useNormalizer` option with dynamic import.
6. **Event hooks**: Implement all hook invocations at the correct points in the retry loop.
7. **Tests**: Tests for JSON Schema validation, factory function, advanced repair, and hooks.

### Phase 4: Polish and Production Readiness (v1.0.0)

Harden for production use.

1. **Configuration validation**: Validate all options at invocation time with clear error messages.
2. **AbortSignal support**: Implement cancellation via `AbortSignal` at every async boundary (LLM call, backoff delay).
3. **Token usage tracking**: Implement `RetryContext`-based token usage reporting and aggregation in `RetryResult`.
4. **Performance optimization**: Profile the repair pipeline, optimize hot paths, benchmark with large outputs.
5. **Edge case hardening**: Test with empty strings, very large outputs, deeply nested schemas, schemas with many refinements.
6. **Documentation**: Comprehensive README with installation, quick start, configuration reference, provider integration examples, and troubleshooting guide.

---

## 20. Example Use Cases

### Structured Data Extraction with Retries

Extract a structured user profile from an LLM, with automatic retry when the output does not match the schema:

```typescript
import { retryWithValidation } from 'llm-retry';
import { z } from 'zod';
import OpenAI from 'openai';

const openai = new OpenAI();

const UserSchema = z.object({
  name: z.string().min(1),
  age: z.number().int().min(0).max(150),
  email: z.string().email(),
  roles: z.array(z.enum(['admin', 'editor', 'viewer'])).min(1),
  address: z.object({
    street: z.string(),
    city: z.string(),
    state: z.string().length(2),
    zip: z.string().regex(/^\d{5}$/),
  }),
});

type User = z.infer<typeof UserSchema>;

const result = await retryWithValidation<User>(
  async (messages, context) => {
    const response = await openai.chat.completions.create({
      model: context?.model ?? 'gpt-4o-mini',
      messages: messages as any,
      temperature: context?.temperature ?? 0.0,
      response_format: { type: 'json_object' },
    });
    return response.choices[0].message.content ?? '';
  },
  UserSchema,
  {
    messages: [
      {
        role: 'system',
        content: `Extract user data from the following text and return it as a JSON object matching this schema:
          { name: string, age: integer (0-150), email: valid email, roles: array of "admin"|"editor"|"viewer", address: { street, city, state (2-letter), zip (5 digits) } }`,
      },
      {
        role: 'user',
        content: 'Alice Johnson, 30 years old, alice.johnson@example.com, admin and editor, lives at 123 Main St, Portland, OR 97201',
      },
    ],
    maxRetries: 3,
    repair: 'standard',
  },
);

if (result.success) {
  console.log('Extracted user:', result.data);
  console.log(`Took ${result.attempts} attempt(s) in ${result.totalMs}ms`);
} else {
  console.error('Extraction failed:', result.errors);
}
```

### Model Escalation on Failure

Start with a cheap, fast model and escalate to a stronger model if it cannot produce valid output:

```typescript
import { retryWithValidation } from 'llm-retry';
import { z } from 'zod';
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const AnalysisSchema = z.object({
  sentiment: z.enum(['positive', 'negative', 'neutral', 'mixed']),
  confidence: z.number().min(0).max(1),
  topics: z.array(z.object({
    name: z.string(),
    relevance: z.number().min(0).max(1),
  })).min(1).max(10),
  summary: z.string().min(50).max(500),
  keyEntities: z.array(z.object({
    name: z.string(),
    type: z.enum(['person', 'organization', 'location', 'product', 'event']),
  })),
});

const callHaiku = async (messages: any[], context: any) => {
  const response = await anthropic.messages.create({
    model: 'claude-haiku-3-5-20241022',
    max_tokens: 4096,
    messages: messages.filter((m: any) => m.role !== 'system') as any,
    system: messages.find((m: any) => m.role === 'system')?.content,
  });
  return response.content[0].type === 'text' ? response.content[0].text : '';
};

const callSonnet = async (messages: any[], context: any) => {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    messages: messages.filter((m: any) => m.role !== 'system') as any,
    system: messages.find((m: any) => m.role === 'system')?.content,
  });
  return response.content[0].type === 'text' ? response.content[0].text : '';
};

const result = await retryWithValidation(callHaiku, AnalysisSchema, {
  messages: [
    { role: 'system', content: 'Analyze the following text. Return a JSON object with sentiment, confidence, topics, summary, and key entities.' },
    { role: 'user', content: articleText },
  ],
  maxRetries: 4,
  escalation: [
    { afterAttempt: 2, action: 'increase-temperature', delta: 0.3 },
    { afterAttempt: 3, action: 'switch-model', callLLM: callSonnet, model: 'claude-sonnet-4-20250514' },
  ],
  hooks: {
    onEscalation: (attempt, action) => {
      console.log(`Attempt ${attempt}: escalating with ${action}`);
    },
  },
});

console.log(`Model used: ${result.history[result.history.length - 1].escalated ? 'Sonnet (escalated)' : 'Haiku'}`);
```

### Data Pipeline with Cost Tracking

Process a batch of documents with retry and track the total cost:

```typescript
import { createRetryLoop } from 'llm-retry';
import { z } from 'zod';

const EntitySchema = z.object({
  entities: z.array(z.object({
    name: z.string(),
    type: z.string(),
    confidence: z.number().min(0).max(1),
  })),
  relationships: z.array(z.object({
    source: z.string(),
    target: z.string(),
    relation: z.string(),
  })),
});

const retryLoop = createRetryLoop({
  callLLM: async (messages, context) => {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: messages as any,
      temperature: 0.0,
      response_format: { type: 'json_object' },
    });
    if (context) {
      context.custom = {
        tokenUsage: {
          promptTokens: response.usage?.prompt_tokens ?? 0,
          completionTokens: response.usage?.completion_tokens ?? 0,
          totalTokens: response.usage?.total_tokens ?? 0,
        },
      };
    }
    return response.choices[0].message.content ?? '';
  },
  schema: EntitySchema,
  maxRetries: 2,
  repair: 'standard',
});

let totalTokens = 0;
let successCount = 0;
let failCount = 0;

for (const document of documents) {
  const result = await retryLoop.execute([
    { role: 'system', content: 'Extract entities and relationships from the text. Return JSON.' },
    { role: 'user', content: document.text },
  ]);

  totalTokens += result.totalTokenUsage.totalTokens;

  if (result.success) {
    successCount++;
    await saveEntities(document.id, result.data);
  } else {
    failCount++;
    await logFailure(document.id, result.errors);
  }
}

console.log(`Processed ${documents.length} documents: ${successCount} succeeded, ${failCount} failed`);
console.log(`Total tokens used: ${totalTokens}`);
```

### Local Model with Aggressive Repair

Use aggressive repair settings for a local model that produces heavily malformed output:

```typescript
import { retryWithValidation } from 'llm-retry';
import { z } from 'zod';

const ConfigSchema = z.object({
  name: z.string(),
  version: z.string(),
  port: z.number().int().min(1).max(65535),
  features: z.array(z.string()),
  database: z.object({
    host: z.string(),
    port: z.number().int(),
    ssl: z.boolean(),
  }),
});

const result = await retryWithValidation(
  async (messages) => {
    const response = await fetch('http://localhost:11434/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'llama3.2',
        messages: messages.map(m => ({ role: m.role, content: m.content })),
        stream: false,
      }),
    });
    const data = await response.json();
    return data.message.content;
  },
  ConfigSchema,
  {
    messages: [
      { role: 'user', content: 'Generate a server config with name, version, port, features array, and database settings. Return JSON only.' },
    ],
    maxRetries: 5,
    repair: {
      level: 'aggressive',
      custom: [
        // Llama sometimes outputs Python-style booleans and None
        (text) => text.replace(/\bTrue\b/g, 'true').replace(/\bFalse\b/g, 'false').replace(/\bNone\b/g, 'null'),
      ],
    },
    feedback: {
      includeSchema: true,  // Always remind the model of the schema
    },
  },
);
```
