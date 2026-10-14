# Decisions log

Everything that still needs a decision lives here, so we can come back to it.
When something is decided, move it into the README and mark it **Decided** here with the commit.

Statuses: **Decided** (in README) · **Proposed** (opinion formed, awaiting approval) · **Open** (needs research or a call) · **Verify** (decided, but not yet proven in a real project).

Last updated: 2026-10-14 (P7, O13 added).

---

## Proposed — opinion formed, awaiting approval

### P1. UI system: Tailwind v4 + shadcn (direction decided; details proposed)
- **Decided:** Tailwind + shadcn.
- **Proposed:**
  - shadcn CLI v4 with the TanStack Start template (`shadcn init -t start --monorepo`); Base UI primitives (shadcn's default since 2026-07).
  - Our design system as a **private shadcn registry in its own repo** (e.g. `nisarahamedk/ui`), with a `registry:base` item (tokens, fonts, components) every project installs; pin installs by commit.
  - Agents: shadcn MCP server + official shadcn skill + our registry namespace; `https://ui.shadcn.com/llms.txt`.
  - Lint: no raw colors (semantic tokens only); no direct primitive imports (only `@/components/ui`).
  - Forms: shadcn `Field` components + TanStack Form.
  - Mobile: React Native Reusables + NativeWind consuming the same token values (manual mapping; no official shadcn for React Native).
- Sources: https://ui.shadcn.com/docs/changelog · https://ui.shadcn.com/docs/registry/github · https://github.com/founded-labs/react-native-reusables

### P2. Authorization and tenancy
- **Decided:** organizations (tenants) from day one.
- **Proposed:**
  - Permission vocabulary (roles, permission slugs) defined in **WorkOS RBAC**; arrives in the access token (`org_id`, `role`, `permissions`).
  - Engine = code: one `can(ctx, action, resource)` module in `packages/core/policy`, called by every handler. No external engine to start.
  - `orgId` only from the verified token → oRPC context; never from input.
  - `core` exposes only org-scoped repositories (`where org_id = ctx.orgId`); `org_id` in every table and composite keys.
  - Postgres row-level security on Neon as a backstop (session variable + non-owner role). D1 has no RLS → B2B apps use Postgres.
  - Tests: role × action × same-org/other-org matrix from one policy table; two-org isolation test (also with app filters stubbed, to prove RLS alone holds).
  - Later, per-resource sharing: WorkOS FGA if its hierarchy fits (confirm pricing), else OpenFGA.
  - Avoid: Oso Cloud (assets sold to Tiny, 2026-10-01); Permit.io free tier (20-tenant cap).
- Sources: https://workos.com/docs/reference/authkit/session-tokens/access-token · https://workos.com/changelog/fine-grained-authorization · https://orm.drizzle.team/docs/rls

### P3. Observability and replay (from day one)
- One trace ID end to end: browser/app creates `traceparent`; the Worker reads or creates it and copies it into Queue messages, Workflow params and outbound provider headers (Cloudflare only auto-propagates Worker → Durable Object/service bindings today).
- Cloudflare Workers logs + automatic tracing on; structured JSON logs always carrying `traceId`; custom spans (`tracing.enterSpan`) around domain steps.
- Export via OTLP to **Axiom** (MCP for agents). **Sentry** for errors (Workers, TanStack Start beta, Expo) + Sentry MCP. **Langfuse** for LLM traces.
- Replay capture store in **R2**: redacted inbound requests/webhooks and every provider response, keyed by trace ID, expired after 14–30 days. Provider clients wrapped to record; clock and ID generation injectable. Replay = captured input through fakes serving recorded responses, locally or in tests.
- Agent loop: Sentry/Axiom MCP → failing trace → logs + R2 capture → local replay → regression test.
- Note: Cloudflare observability pricing becomes per-GB ingestion on 2026-12-01.
- Sources: https://developers.cloudflare.com/workers/observability/traces/known-limitations/ · https://developers.cloudflare.com/workers/observability/exporting-opentelemetry-data/ · https://developers.cloudflare.com/observability/pricing/

### P4. Async work, webhooks and idempotency
- **Decided:** idempotency is mandatory.
- **Proposed rules:**
  1. Webhooks: verify signature on raw body + timestamp window → insert into `inbox` with unique `(provider, event_id)` (ignore duplicates) → enqueue → 2xx in under 1s. No outside API calls in the webhook request.
  2. Every handler assumes duplicates and out-of-order delivery; dedupe with database unique constraints; order by provider version/timestamp, not arrival.
  3. Queues for simple jobs/fan-out; Workflows for multi-step, long-waiting or side-effecting jobs. Workflow instance ID = business key, backed by a DB unique constraint (IDs are unique only for 30 days).
  4. Every queue has a dead-letter queue with a consumer.
  5. Outbound side effects (email, charges, AI-triggered actions) via an outbox row in the same transaction; sent with the provider's idempotency key.
  6. Every alarm, schedule, consumer and Workflow step is safe to run twice.
- Sources: https://developers.cloudflare.com/queues/reference/delivery-guarantees/ · https://developers.cloudflare.com/workflows/build/rules-of-workflows/ · https://developers.cloudflare.com/agents/guides/webhooks/

### P5. Where state lives
- Database (Postgres; D1 for small apps) = source of truth for identity, organizations, billing, entitlements, and anything queried across users/entities. Webhooks write there first.
- Durable Objects/Agents own per-entity live state (conversation memory, realtime sessions, coordination, scheduling). Keep synced `setState` small.
- If a Durable Object owns data needed elsewhere: outbox row in its SQLite in the same transaction → alarm copies to the database with idempotent upserts keyed by (entity, sequence).
- If the database owns it: the Durable Object is only a cache/coordinator; write the database first; version-check writes (awaits let requests interleave).
- Never billing or entitlements only in a Durable Object.
- FirstWord check: anything shown across screens/users that currently lives only in the Durable Object.

### P6. Product insight: analytics, replay, RUM, flags, experiments
- Skip FullStory (sales-led pricing). **PostHog** for product analytics, session replay, feature flags and experiments (one SDK). **Sentry** browser tracing for real-user performance, sharing the trace ID with the backend. Cloudflare Web Analytics for marketing pages.
- Rules:
  1. Evaluate flags on the server in the gate/loaders and send results with the page (client-side flags flicker, same as the auth flash).
  2. Flags behind a small `flags` interface in `core`; tests can set flags directly.
  3. Every flag has an owner and expiry; a scheduled agent task removes expired flags and dead code.
  4. Experiments = flag + primary metric + guardrail metric defined before launch; until traffic is real, use flags for gradual rollout, not A/B tests.
  5. Session replay masks all inputs and text by default.
  6. One replay tool (PostHog); Sentry links errors to the PostHog session.
  7. Typed `track()` with an event catalog in code.
- Sources: https://rybbit.com/blog/posthog-pricing · https://amplitude.com/compare/posthog-vs-fullstory

### P7. Code standards that make agent code verifiable — **Decided 2026-10-14**: [docs/coding-standards.md](docs/coding-standards.md), [docs/errors.md](docs/errors.md)
Inspired by Dillon Mulroy's project docs ([coding-standards](https://github.com/dmmulroy/cf-workflow-pool/blob/main/docs/coding-standards.md), [errors](https://github.com/dmmulroy/cf-workflow-pool/blob/main/docs/errors.md)); no license on that repo, so we write our own versions rather than copying.
- **Errors are product interfaces and observability data:** every error names the failed operation, the most specific safe reason, and a way forward (fix input, safe retry, reconcile, or contact support with a request ID). Stable machine-readable codes with variant-specific details; a truthful `retryable` that matches the operation's idempotency; never claim "nothing changed" unless provable; public errors carry `requestId` (= trace ID) and never secrets, SQL, stack traces or vendor bodies.
- **No catch-all error translation:** each known error is translated explicitly at each boundary, so a new failure type fails compilation until someone decides its classification.
- **Parse once at the boundary, then preserve types:** decode unknown input (HTTP, DB rows, queue messages, env, vendor payloads) with the boundary's schema; inner code receives parsed domain types; never widen known values to `unknown`/records; `as unknown as` / `as any` are review blockers.
- **Monorepo boundaries:** apps never import other apps; packages never import apps; shared capability moves to a package when a second consumer appears. (Exception we chose deliberately: `contract` and `core` exist from day one, because more clients are expected.)
- **Durable Objects keyed by canonical domain IDs**; callers use an application-level client, never raw stubs.
- **HTTP endpoints versioned from `/v1`.**
- **JSDoc on every exported symbol**, stating what the signature can't.
- **Test layers/fakes cross the same interface as production** and behave faithfully; no partial mocks.
- Every project copies and adapts both docs. The user will flag any rule they disagree with.

---

## Open — needs research or a call

| # | Topic | What's open | Notes |
|---|---|---|---|
| O1 | **Effect TS** | Adopt fully / backend only / not | Current plan: trial on a backend slice. 2026-10-14: Dillon shared the standards he carries between projects (Effect services + Layers with faithful test layers, tagged errors translated explicitly with `catchTags`, Schema parsing at boundaries, `Option` for optional values, Effect HttpApi with generated OpenAPI, Durable Objects as Effect HTTP services). These are concrete, agent-enforceable rules, which is the strongest argument for Effect so far. New data point (2026-10-14): Dillon Mulroy (Cloudflare, Effect advocate) reports Opus 5.5 "starting to build this trust, especially with code/services written in effect." Candidate trial: webhook/inbox/outbox pipeline (P4), where typed errors and retries pay off most. Effect 4 is ~2 weeks old; RPC/atoms still unstable. |
| O2 | Design-system repo | Create `ui` registry repo; brand tokens | Depends on P1. |
| O3 | Observability backend | Axiom vs Honeycomb | Both have MCP; Axiom proposed. Re-check costs after Cloudflare's 2026-12-01 pricing change. |
| O4 | Per-resource sharing | WorkOS FGA vs OpenFGA | Only when an app needs it; confirm WorkOS FGA pricing. |
| O5 | AI layer details | Prompt versioning, evals as CI gate, per-user cost caps, model fallbacks, personal data in prompts/logs | Not yet discussed in depth. |
| O6 | Agent operating model | Definition of done for agent PRs, merge policy, orchestrator/worker/stack-session handoffs, agent spend per project | |
| O7 | Protect `main` | GitHub Pro for required PR review on private repos | Main safeguard before prod deploys. |
| O8 | Clerk vs WorkOS for FirstWord; Clerk Platform API | Migrate FirstWord auth? Request Platform API access? Separate dev Clerk account? | |
| O9 | FirstWord data | Stay on D1 or move to Neon (RLS, branching) | D1 has no RLS; relevant once orgs/B2B matter. |
| O10 | Project kickoff | Template repo + kickoff prompts (stack session → project orchestrator → workers) | Deferred. |
| O11 | Keeping the-stack current | Quarterly re-verify routine that opens a PR | Deferred. |
| O13 | **Contract layer if Effect wins** | oRPC vs Effect HttpApi | Effect HttpApi also gives a typed contract + OpenAPI + typed client. If O1 lands on "adopt", decide whether oRPC stays. **Recommendation: hold FirstWord's oRPC migration until O1 is decided**, to avoid migrating twice. |
| O12 | One-time setup | Sentry + Better Stack accounts; Mac self-hosted runner (when mobile starts) | |

---

## Verify — decided, not yet proven

| # | Claim | Where it gets proven |
|---|---|---|
| V1 | A redirect thrown from an `ssr: 'data-only'` gate on the first request becomes an HTTP redirect before any HTML | FirstWord PR #11 |
| V2 | Clerk's server-side session check (incl. expired-token handshake) works inside the gate | FirstWord PR #11 |
| V3 | Real WebRTC call with pre-recorded caller audio passes on GitHub-hosted runners | FirstWord real-call test (label-gated) |
| V4 | Alchemy doesn't delete or overwrite manually set Worker secrets | First project that moves prod onto Alchemy |
| V5 | iOS Maestro on the Mac self-hosted runner | First mobile project |
| V6 | PostHog flag evaluation inside Workers (definitions cached, e.g. in KV) | First project using flags |
| V7 | oRPC contract-first + `@orpc/tanstack-query` on TanStack Start / Workers | FirstWord oRPC follow-up PR |

---

## Decided (see README)

Infisical `dev-vault` + host-native prod secrets · two Cloudflare accounts · Alchemy v2 PR stages · GitHub Actions prod deploys · TanStack Start with server gate (`ssr: 'data-only'`) · react-doctor + React Compiler + hooks lint · TanStack Query + Form · oRPC contract-first API (`contract` / `core` / clients) · organizations from day one · Tailwind + shadcn · idempotency mandatory.
