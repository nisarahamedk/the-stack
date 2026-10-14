# Error design

Errors are part of the product and part of observability. A good error lets a person or an agent understand what happened, decide what to do next, and find the matching trace, without exposing private details.

Adapted from the principles in Dillon Mulroy's [errors.md](https://github.com/dmmulroy/cf-workflow-pool/blob/main/docs/errors.md), which build on Wix UX's ["When life gives you lemons, write better error messages"](https://wix-ux.com/when-life-gives-you-lemons-write-better-error-messages-46c5223e1a2f).

## Writing the message

1. **Say what failed and why.** Name the operation and give the most specific safe reason you know. Never "Something went wrong" when the cause is classified.
   - Good: `Call call_8f2… was not found. Check the call ID and try again.`
   - Bad: `Something went wrong.`
2. **Say what to do next.** Every error ends with one way forward:
   - fix the named input and retry;
   - retry the same operation, only when that is safe;
   - check or reconcile the named resource, when the outcome is uncertain;
   - contact support with the request ID.
3. **Only reassure what you can prove.** Say "your rules were not changed" only if the failed operation could not have changed them. After a partial or multi-service operation, say the outcome is uncertain and point to where to check.
4. **Retry advice must match idempotency.** Never suggest a retry when the first attempt may have partly succeeded or a retry would create a second operation.
5. **Calm and direct.** No jokes, no apologies, no alarm, no blame on the user or a vendor.
6. **No implementation jargon in public errors.** Use the product's vocabulary, not table names, Durable Object mechanics, stack traces, vendor exception names or internal service names.

## What every error carries

Errors are typed values, not strings. Each carries:

- a stable machine-readable `code` (a literal, unique and searchable in the source);
- the `operation` that failed;
- safe domain IDs (`callId`, `orgId`) when known;
- a classified `reason`;
- for input errors, the field violations and the accepted constraints;
- `retryable`, truthful for this operation;
- at public boundaries, a `requestId` equal to the trace ID (see DECISIONS P3).

Model errors as **discriminated unions on a literal `code`, with variant-specific details**, never a free-form `details: Record<string, unknown>` that lets the code and details disagree. Every error union uses `code` as its discriminant, so translation can switch on it exhaustively. **(Effect)** `_tag` maps one-to-one to `code`.

## Internal vs public

- **Internal errors** keep enough safe structure for logs and traces: operation, entity, capability, reason, and the underlying cause when it is safe to keep. Messages start with a specific, searchable phrase.
- **Public errors** (the API contract) contain exactly: `code`, an operation-specific `message`, variant-specific safe `details`, `requestId`, `retryable`.
- **Never expose:** credentials, tokens or JWT claims, raw request bodies, SQL, stack traces, private causes, vendor response bodies. Redaction keeps the useful safe context (operation, requested ID).

## Translating at boundaries

- Translate each known error **explicitly** at every boundary (handler, adapter, webhook consumer).
- **No catch-all** that silently maps unknown failures to a generic response. A new error type must fail type-checking until someone decides its code, message, status, retry guidance and telemetry.
  - Plain TypeScript / oRPC: an exhaustive `switch` on the error's `code` with a `never` check in the default branch; oRPC typed errors declared per procedure in the contract.
  - Effect: `Effect.catchTags` listing each tag; leave unhandled tags in the error channel on purpose.
- **One exception: a single top-level handler for defects.** Unexpected throws (bugs, broken invariants) that are not typed errors get caught once, at the outermost layer of each entrypoint (fetch, queue, alarm, workflow): log with the trace ID, report to Sentry, return `500` with `requestId` and `retryable: false`. It never handles an expected, typed error; those must already be translated.

## Showing errors in the UI

- The UI maps each error `code` to **its own user-facing copy**, written for that screen. Never render a server `message` verbatim; it is written for API callers and operators.
- Unknown codes (an older client meeting a newer server) fall back to a generic message plus the `requestId`, so support can find the trace.
- Field violations attach to their form fields; `retryable` decides whether a retry action is offered.

## HTTP status

Choose by what the caller can do, not by what broke inside:

| Status | Meaning |
|---|---|
| 400 | The caller can fix malformed or invalid input |
| 401 | Not authenticated |
| 403 | Authenticated but not allowed |
| 404 | Valid-looking ID, unknown resource |
| 409 | Current state conflicts with the request (including idempotency conflicts) |
| 415 | Unsupported media type |
| 422 | Well-formed but fails a business rule the caller can change |
| 429 | Rate limited; include when to retry |
| 500 | Internal invariant broken or non-transient internal failure |
| 503 | A required dependency is temporarily unavailable; `retryable: true` |

## Review checklist

Before adding or changing an error:

1. Does it name the operation or outcome that failed?
2. Is the reason the most specific safe one known?
3. Is the context in typed fields, not buried in prose?
4. Does it give a way forward (fix, safe retry, reconcile, escalate)?
5. Are reassurance and `retryable` provably true?
6. Calm, direct, no blame, no jargon?
7. Can the code or message be found by searching the source?
8. Does the public `requestId` match a trace?
9. Are secrets, private causes and raw external data excluded?
10. Is every boundary translation explicit, with no catch-all (except the top-level defect handler)?
11. Does the UI have its own copy for this code?
