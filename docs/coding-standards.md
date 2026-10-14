# Coding standards

Rules that make agent-written code checkable by the compiler, linters and tests, so trust can be built by verifying structure rather than re-reading everything. Every project copies this file (and [errors.md](errors.md)) and adapts it.

Adapted from the principles in Dillon Mulroy's [coding-standards.md](https://github.com/dmmulroy/cf-workflow-pool/blob/main/docs/coding-standards.md). Rules marked **(Effect)** apply only to projects using Effect TS (decision O1 in [DECISIONS.md](../DECISIONS.md)).

**Two principles behind every rule:**

- **Consistency and clear boundaries beat brevity.** Boilerplate is cheap for agents; every capability, Durable Object and boundary has the same explicit shape in every project, so code is predictable to read and check.
- **Deterministic checks where we can.** Each rule is tagged with how it is checked: `type` (compiler), `lint`, `test`, `ci` (script or build step), or `review` (no tool can check it yet). When a `review` rule keeps being broken, turn it into a lint rule, type or test.

## 1. Monorepo boundaries

- Each `apps/*` is an independent, deployable composition root. An app **never** imports from another app, in code, scripts or tests. `lint`
- `packages/*` hold shared capabilities and **never** import from `apps/*`. `lint`
- Cross-app behaviour goes through a real runtime interface (HTTP/RPC) or a package with an intentional public entrypoint. Shared end-to-end test helpers live in a package. `lint` (package `exports` + restricted imports)
- Code stays in its owning app while it has one consumer; move it to a package when a second consumer appears. Don't create packages for hypothetical reuse. `review`
  - **Deliberate exception:** `packages/contract` and `packages/core` exist from day one, because more clients are expected (the-stack §3.2).
- Apps trend toward thin entrypoints: runtime startup, framework wiring, app config and final dependency composition. `review`

## 2. Environment isolation

- Every environment (PR stage, prod, local) gets its own infrastructure with the same shape. Never route preview, test or local data through production resources because they already exist. `ci` (preview and prod use separate Cloudflare accounts and credentials)
- Non-production resources use generated or stage-qualified names; retention settings let ephemeral stages be destroyed. `ci` (PR-close destroy job)
- When a stack depends on a shared-infrastructure stack, create the shared one first and destroy it last, with the same stage and provider mode. `ci`
- Document every cross-stack dependency: owner, permanent or stage-isolated, how local providers are chosen, creation/destruction order. `review`

## 3. Parse at the boundary, then preserve types

Choose handling by where a value came from:

| Value came from | Do this |
|---|---|
| Outside or storage: HTTP/RPC bodies, DB rows, queue messages, env vars, vendor payloads, files | Decode from `unknown` with the boundary's schema into the strongest domain type |
| A known encoded form (e.g. JSON text whose schema expects a string) | Use a decoder typed for exactly that encoded form |
| An already-parsed domain value | Pass it unchanged; don't re-parse or round-trip it |
| A newly computed value with a new constraint | Establish the constraint with its constructor, refinement or schema |

- The adapter that owns an I/O boundary owns its parsing and parse errors. Domain code and services receive parsed values, never request payloads, DB rows or vendor records. `type` (branded domain types: an unparsed value doesn't type-check)
- Keep `unknown` for genuinely untrusted input, private to the adapter. `review`
- Instantiate each codec once, next to its owner; expose a domain-named parser (`parseCallId(input: string)`), not raw library decoders. `review`
- Schemas: Zod 4 in plain TypeScript; Effect Schema **(Effect)**.

## 4. Never throw away type information

- Don't widen a known value to `unknown`, `object` or `Record<string, unknown>` because downstream code is easier to write that way. Signatures state what is actually accepted and returned. `review`
- **Blockers:**
  - `as unknown as T`, `as any as T`, or widen-then-assert sequences; `lint`
  - explicit `any`; `lint`
  - non-null assertions (`x!`); branch or refine instead; `lint`
  - a known domain or library value passed or returned as `unknown`/`object`/generic record; `review`
  - re-parsing or re-narrowing a value that already crossed its boundary; `review`
  - hand-written approximations of a type a library already exports; `review`
  - `typeof`/generic object checks used instead of parsing the real contract. `review`
- A type assertion is allowed only at the smallest unavoidable platform boundary, when the runtime fact is already established and TypeScript can't express it. It needs, directly above it: `lint` (assertions banned except with a suppression)
  - `// SAFETY:` explaining where the value comes from and which invariant guarantees the type;
  - `// Verified by:` naming the test that fails if the invariant breaks;
  - the narrowest lint suppression.
- The rule: **keep evidence you have, create evidence by parsing at the boundary, never fake evidence with an assertion.**

## 5. Optional values

- Pick one representation inside the app and convert at the boundary. `review`
  - Plain TypeScript: `T | undefined` (no `null` internally); decode `null`/missing fields at the boundary.
  - **(Effect)** `Option.Option<T>` everywhere inside the app; encode back to `null`/omitted only where an external contract requires it.

## 6. Errors

Follow [errors.md](errors.md). In short: typed errors discriminated by a literal `code`; explicit translation at every boundary (`type`: exhaustive switch); no catch-all for expected errors, one top-level handler for defects; truthful `retryable`; public errors carry `requestId`; the UI writes its own copy per code.

## 7. API contracts

- The contract (oRPC contract package, or Effect HttpApi **(Effect)**) is the source of truth; OpenAPI is generated from it and never hand-edited. `ci` (regenerate and fail on diff)
- Every HTTP endpoint is versioned from `/v1`; no unversioned aliases. `test` (route list check)
- Contract changes are additive (the-stack §3.2). `ci` (OpenAPI breaking-change check)
- Give every client-supplied value a realistic example; describe where IDs come from (e.g. "returned by createCall"); include success and error examples for each operation; document real auth headers; list only real server URLs. `review`
- Review the OpenAPI diff on every contract change. `review`

## 8. Durable Objects

Every Durable Object has the same explicit shape, whatever the project:

| Piece | Example | Role |
|---|---|---|
| Interface | `ICallClient` | The typed operations callers may use; the only thing callers depend on |
| Client | `CallClient` | Implements the interface; owns the namespace binding, ID derivation and stub creation |
| Server | `CallServer` | The Durable Object class; implements the same operations and owns the state |

- Callers depend on `ICallClient` and never touch stubs, namespaces or bindings. `lint` (restricted imports: DO classes and `env.*` namespaces only inside the client file)
- Each instance is keyed by its **canonical domain ID** (`UserId`, `CallId`), never a display name or caller-provided label. `type` (client takes the branded ID)
- Create stubs and their clients inside the current request or invocation; never keep them across invocations. `review`
- A faithful test implementation of `ICallClient` exists alongside (§9). `review`
- **(Effect)** The server is an Effect HTTP service and the client is an Effect service with Layers, following §9's file order.

## 9. Services and dependencies

- Business logic depends on capabilities through interfaces, provided at the composition root; not on concrete clients imported directly. `lint` (restricted imports of vendor SDKs outside adapters)
- Every service has a production implementation and a **faithful test implementation crossing the same interface**. No partial mocks or weakened behaviour just to make tests easy. `review`
- **(Effect)** One file per capability, in this order: interface `IThing` → service class `Thing` (`Context.Service`) → `makeThing` → `thingLayerWithoutDependencies` → `thingLayer` (production) → `thingTestLayer`. Only tests may import `make*` constructors; runtime code imports Layers and yields the service. Names omit `Tag`. `lint` (`make*` import rule)
- **(Effect)** Code style: `Effect.gen` / `Effect.fn` for business steps, branching and early returns; `.pipe(...)` for cross-cutting policy (error translation, retry, timeout, spans, logging, providing dependencies) and single simple transforms. Import stable modules from `effect` (`import { Effect, Option } from "effect"`), unstable ones from their entrypoints (`effect/unstable/http`). `review`

## 10. Tests

- Don't write tests that restate a schema's declared constraints. Test transformations, normalisation, custom checks, compatibility and past regressions, through the public boundary where callers see the outcome. `review`
- Authorization matrix and cross-tenant isolation tests are mandatory (DECISIONS P2). `test`
- Retry-safety: every queue consumer, alarm, schedule and workflow step has a test that runs it twice. `test`

## 11. Documentation

- Every exported symbol has JSDoc on its original declaration, stating what the signature can't (constraints, failures, side effects, ownership, invariants). `lint` (presence only; content is `review`)
- Use durable domain vocabulary in names, docs, UI copy and errors; keep ticket names, migration phases and framework mechanics out of public docs. `review`
- Expected failures are documented in `@returns` / outcomes; `@throws` only for defects or framework-required behaviour. `review`

## 12. Enforcement

The tools behind the `lint`, `type` and `ci` tags. Configure (oxlint or ESLint + typescript-eslint):

- `no-explicit-any`, `no-non-null-assertion`, `consistent-type-assertions` (forbid `as` except with a `SAFETY` suppression);
- `switch-exhaustiveness-check` (error translation, state machines);
- restricted imports: `apps/*` can't import other apps; `packages/*` can't import `apps/*`; Durable Object classes and namespace bindings only inside their client; vendor SDKs only inside adapters; UI code imports components only from `@/components/ui`; no raw color classes;
- a JSDoc-presence rule on exports (e.g. `jsdoc/require-jsdoc` with `publicOnly`);
- **(Effect)** a rule rejecting project imports matching `make[A-Z]` outside `*.test.*` / `*.spec.*`;
- react-doctor and `eslint-plugin-react-hooks` for React (the-stack §3.1);
- CI: OpenAPI regenerate-and-diff plus a breaking-change check.
