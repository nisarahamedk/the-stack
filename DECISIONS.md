# Decisions log

Everything that still needs a decision lives here, so we can come back to it.
When something is decided, move it into the README and mark it **Decided** here with the commit.

Statuses: **Decided** (in README) · **Proposed** (opinion formed, awaiting approval) · **Open** (needs research or a call) · **Verify** (decided, but not yet proven in a real project).

Last updated: 2026-10-15 (P9–P11 re-examined against every archetype; P12 and O16–O21 added).

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
- **Refinements (2026-10-14, after review):**
  - Consistency and clear boundaries beat brevity; boilerplate is fine for agents. Every Durable Object has the same explicit shape (`ICallClient` / `CallClient` / `CallServer`) in plain TS too, not only with Effect.
  - Every rule is tagged with how it is checked (`type`, `lint`, `test`, `ci`, `review`); prefer deterministic checks, and promote `review` rules that keep breaking.
  - No catch-all for expected errors, but one top-level defect handler per entrypoint (log + Sentry + `500` with `requestId`).
  - The UI maps error codes to its own copy; server messages are never shown verbatim.
  - Every error union discriminates on a literal `code`.
  - JSDoc on every export stays (presence linted).

### P8. Quality targets: performance, reliability, scale
The README promises these; nothing enforces them yet. Proposed defaults per project, adjusted per app:
- **SLOs from day one:** availability and latency targets per critical user flow (e.g. 99.9% success, API p95 < 300 ms at the edge), measured from traces (P3). Error budget decides when reliability work beats feature work.
- **Performance budgets in CI:** web Core Web Vitals (LCP < 2.5 s, INP < 200 ms, CLS < 0.1) on PR previews; JS bundle size limit per route; Worker CPU time per request watched.
- **Load test before launch and on hot-path changes:** k6 (or similar) against the PR stage at ~10× expected peak; find the first bottleneck (DB connections, DO hot keys, vendor rate limits).
- **Failure-mode tests:** each vendor dependency has a test for timeout, 429 and 5xx; each has a timeout, a retry policy that matches idempotency (P4), and a defined degraded behaviour.
- **Scale by design:** stateless Workers; per-entity Durable Objects (no global singletons); DB indexes reviewed with query plans for every new query on a hot path; pagination everywhere (cursor-based).
- **Extensibility:** additive contract changes (§3.2), capabilities behind interfaces (coding-standards §9), feature flags for risky rollouts (P6).
- **Open:** exact default numbers; which load-testing tool runs from GitHub Actions against PR stages; whether SLO burn alerts go to the on-call setup.

### P9. Agent-native apps: people and agents work on the same product
Prompted by Builder.io's Agent-Native (beta, 2026-10-14): one action definition serves the UI, agent tools, HTTP, CLI, MCP and A2A; shared data; the agent sees what is on screen. **Take the ideas, not the framework:** it brings its own auth, orgs, storage and sync, which overlap our WorkOS / oRPC / Postgres / Durable Object choices, and it is beta.
- **One action layer:** every capability is a contract procedure (§3.2) that the UI, the agent, the public API and an MCP server all call. No screen-driving agents inside our own apps.
- **Agents act on behalf of a person, never above them:** the actor is `{ user }` or `{ agent, onBehalfOf, sessionId }`; `can()` (P2) allows the user's permissions intersected with what that agent was granted. Every write records the actor, so the UI can show who changed what.
- **Risk level on every action** in the contract: `read` / `reversible` / `irreversible` / `external` (sends money, messages or calls). The level decides whether the agent may run it directly, must propose it, or must ask first.
- **Reversible beats approve:** prefer drafts, suggestions and undo over confirmation dialogs; agent writes on medium-risk actions land as proposals in the same state the person edits (suggestion mode). Approval prompts only for irreversible or external effects, with approve / reject / **edit** (Magentic-UI users asked for the third option).
- **Show outcomes, not traces:** review shows what changed (a diff of the result); a trace of steps is for debugging. Studies found traces raise confidence without raising accuracy.
- **Shared, live state:** agent changes appear in the UI as they happen (Durable Object push, P5), not by polling; the UI shares a small typed view context (route, selection) so the agent understands "this".
- **Visible, interruptible agent:** disclosed as an agent; status always shown (working / waiting for you / done / failed); stop takes effect immediately; a person always owns the outcome (Linear's Agent Interaction Guidelines).
- **Untrusted content stays untrusted:** text an agent reads from users, callers, email or the web cannot widen what it may do; `external` actions need a person when the agent has read untrusted input in that session.
- **Three kinds of shared state, each with one owner:**
  - *Domain state* (calls, rules, records): server-owned (Postgres / Durable Objects). Changed only through actions, by people and agents alike; pushed live to the UI.
  - *View state* (route, selection, open drafts): browser-owned. The UI sends the agent a typed view context; the agent changes it only through typed **UI tools** (`navigate`, `select`, `highlight`, `fillDraft`), never the DOM. The agent never overwrites a field the person is editing.
  - *Run state* (plan, progress, streamed text, pending approvals): agent-owned, streamed to the UI.
- **Where the protocols fit:** AG-UI is the wire between our in-app agent and our UI (run events, tool calls, interrupts, state snapshot/patch). It carries run and view state only, never a second copy of domain truth. WebMCP registers the same UI tools and actions for an agent already in the person's browser (origin trial from Chrome 149). MCP serves headless external agents; MCP Apps embeds our UI in Claude/ChatGPT. All four call the same actions.
- **Watch, don't build yet:** MCP Apps (render our UI inside Claude/ChatGPT; final spec, supported by both) when a project wants it; WebMCP (flag-gated in Chrome, no mainstream agent calls it yet); A2A.
- **Open:** O14 (agent runtime and UI wire); how the MCP server is generated from the contract (oRPC vs Effect, O13); where risk level lives in the contract; proposal storage shape.
- **Re-examined against every archetype (2026-10-15). Proposed amendments** ([docs/archetypes.md](docs/archetypes.md) §3, §7):
  - *Holds everywhere:* one action layer, risk tiers, reversible beats approve, outcomes over traces, untrusted content, three kinds of state.
  - **One shared `Actor`** (archetypes §3) for P9, P10 and P11. It adds `apiKey` and `system` principals. It also splits agents into **delegated** (a person's assistant) and **service** (works for the principal while talking to someone else: an org's support bot answering its customers, a voice agent taking calls for a user, a seller's agent). P9 only described delegated agents; service agents appear in B2B2C, marketplace and AI products.
  - **Org agent policy (B2B, B2B2C):** org admins can turn agents off, limit tools or risk tiers, and set data retention. `can()` intersects it with the person's permissions and the session grant. Enterprise buyers ask for this before anything else.
  - **Approvals are server-side pending operations**, the same object as P10's. They are shown over AG-UI and push, and can be approved from any device. Mobile and multi-device use need this, and it answers "proposal storage shape".
  - **Risk level lives in procedure metadata** (with audience, exposure and idempotency; P12). That answers "where risk level lives".
  - **Budgets:** every agent run has a per-actor and per-org budget, enforced on the server from the usage ledger (P12). B2C free tiers and usage-priced API products break without it.
  - **External agents:** target MCP 2026-07-28. It is stateless; OAuth uses Client ID Metadata Documents, and dynamic client registration is deprecated. In B2B, the Enterprise-Managed Authorization extension lets the org's identity provider decide which agents connect. Risk tiers map to OAuth scopes.
  - *Variants:* B2C → per-user budgets; B2B → org policy; B2B2C → service agents serving end customers; marketplace → agents buying or selling for people (always `external`, always approval; agentic-commerce protocols are a watch item, not yet researched); realtime → the agent has presence like a collaborator; API product → MCP + OAuth is a first-class surface; mobile → push approvals.

### P10. Back-office admin
- **Admin is another client of the actions, never a side door to the database.** Admin-only procedures live in the contract under an internal `admin` namespace (excluded from public OpenAPI) and run through `core`, so invariants, idempotency, audit and errors stay the same. No Retool-style tools holding prod DB credentials.
- **Separate app, separate front door:** `apps/admin` on `admin.<domain>`, behind Cloudflare Access (free up to 50 users) **and** staff sign-in with staff roles checked in `can()`. Two independent locks; customer sessions never reach it.
- **Staff roles, least privilege:** e.g. `support_read`, `support`, `billing`, `engineer`, `owner`. Personal data masked by default; revealing it is an audited action.
- **One actor model for staff, agents and users:** `{ staff, onBehalfOf?, reason }`, same shape as P9. "View as user" is read-only by default; acting as a user is time-limited, needs a reason, shows a banner, is recorded with both identities, and never covers payouts, credentials or deletion. No "log in as user".
- **Every admin write is audited:** who, on behalf of whom, what changed (before/after), reason, request ID. Append-only, kept longer than app logs.
- **Operations, not raw edits:** each admin action is a named operation (`refundPayment`, `extendTrial`, `transferOwnership`) with a preview of what will change, idempotency key, typed confirmation for irreversible ones, and bulk operations as queued jobs with progress.
- **Customer 360 is the main screen:** one page per org/user with a timeline of everything (account, billing, usage, key domain events, errors, support), linking out by ID to traces (P3), Sentry, PostHog, Polar, WorkOS. Don't rebuild what those dashboards already do.
- **Agent-first ops — Decided 2026-10-15:** every app ships an admin MCP server from day one, so AI agents can work on the platform on behalf of staff.
  - Tools are generated from the admin contract (same operations as the admin UI); no tool exists that the UI's operations don't.
  - Sign-in through the staff identity provider (OAuth), behind Cloudflare Access. The actor is `{ agent, onBehalfOf: staff, sessionId }`; permissions are the staff member's role intersected with the scopes granted to that agent session.
  - **The server enforces approval, not the client:** `read` tools run directly; `reversible` writes run and are undoable; `irreversible` and `external` writes create a pending operation that a person approves (or edits) in the admin UI.
  - Personal data is masked in tool results by default; revealing it is a separate, audited tool. Customer-supplied text in results is untrusted and cannot widen what the session may do.
  - Per-session limits on bulk size and rate; one switch disables all agent access.
  - Every tool call is audited with the session ID. The server exists in every PR stage with seed data.
- **Reads that can't hurt prod:** admin queries use a read-only DB role (or replica) with timeouts; ad-hoc SQL only through that role, logged. Analytics belongs in PostHog or a warehouse, not admin.
- **Built with our stack:** TanStack Start + shadcn + TanStack Table, shared admin components in the private registry (data table, entity page, audit timeline, operation dialog with reason/confirm, impersonation banner). Exists in every PR stage with seed data, so agents can test it.
- **Open:** audit log store (Postgres table vs R2/Axiom); retention period.
- **Re-examined against every archetype (2026-10-15). Proposed amendments:**
  - *Holds everywhere:* admin as a client of the actions, named operations, every write audited, admin MCP with server-enforced approval (the Decided part is unchanged; target MCP 2026-07-28, see P9).
  - **One namespace name:** `admin.*` in every archetype (P11's `platform.*` becomes `admin.*`).
  - **Customers can see support access.** This rule moves from P11 into P10 because B2B needs it as much as B2B2C: org admins see a log of staff access to their org, and an org can require its approval first. In B2C, access is recorded but not shown.
  - **Work queues:** a `case` primitive (assignee, status, SLA, linked entities, agent triage, human decision) for moderation, disputes, KYC review, abuse and support. Entity pages alone don't cover B2C abuse or marketplace trust and safety.
  - **Audit store (answers the open item):** one append-only `audit_event` table in Postgres. `core` writes it for every write by any actor, not only admin. It is partitioned by month and archived to R2. A customer-visible subset feeds the org audit log (B2B). Use WorkOS Audit Logs only when an enterprise wants SIEM streaming ($125/month per connection, checked 2026-10-15). Retention stays open: B2B enterprise usually asks for a year or more.
  - **Admin components are scope-agnostic**, so the B2B org console and the B2B2C tenant console are built from the same kit.
  - **The internal-tool archetype is P10 as the whole product.**
  - *Variants:* B2C → deletion, export and abuse operations; API product → key revocation and quota operations; realtime → inspect live sessions and force-disconnect through the Durable Object client; mobile → minimum-supported-version switch; AI → per-org cost view and a per-org agent kill switch.

### P11. B2B2C: tenants with their own customers and mobile apps
Three audiences, one contract, one action layer:

| Tier | Who | Surface | Contract namespace |
|---|---|---|---|
| Platform | our staff + their agents | `apps/admin` + admin MCP (P10) | `platform.*` (internal) |
| Tenant | the business's staff + their agents | tenant console inside the product web app; public API `/v1` + tenant MCP | `tenant.*` |
| End customer | the business's customers | mobile app (Expo) and optional web | `customer.*` |

- **Principals:** `platformStaff`, `tenantMember { tenantId, role }`, `endCustomer { tenantId, customerId }`, and `agent { onBehalfOf }` over any of them. `can()` checks the principal type first; every row carries `tenantId` (RLS backstop, P2), end-customer rows also carry `customerId`.
- **Identity pools are separate:** tenant staff are org members (WorkOS organizations, SSO/SCIM for big tenants). End customers are a per-tenant user pool: the same email at two tenants is two unrelated accounts. The auth provider for end customers is open (O15).
- **Tenant console is the product, not an admin afterthought:** same P10 rules (named operations, audit, preview/confirm), scoped to one tenant, agent-native (P9). Tenant admins see a log of platform support access to their data; platform staff reach tenant data only through time-limited support access with a reason.
- **Tenant configuration is typed data:** branding tokens, enabled features and entitlements (by plan), store metadata, domains, notification senders, in one schema-validated `tenantConfig`. Apps read it at runtime; nothing tenant-specific is hard-coded.
- **Mobile, two modes from one Expo codebase:**
  - *Shared app (default):* one app in the stores; the customer picks or is linked to their tenant (invite link, code, QR); branding applied at runtime. Matches Apple's accepted "picker" model.
  - *White-label (premium tier):* per-tenant binary generated from `tenantConfig` via `app.config.ts` + EAS build profiles; one EAS project per tenant (updates can't share a URL); published under **the tenant's own Apple/Google developer accounts** (App Store guideline 4.2.6 rejects template apps not submitted by the content provider). Per-tenant store credentials and push keys live in the vault, scoped per tenant. JS changes ship to all tenants via EAS Update; native changes trigger a build fan-out.
- **Web for tenants' customers on their own domains:** Cloudflare for SaaS custom hostnames (100 free, then $0.10/month each).
- **Per-tenant sending identity:** email from the tenant's verified domain (Resend domains), SMS/push credentials per tenant where white-labelled.
- **Money flows are two different systems:** tenants pay us (subscription billing, README default); end customers paying tenants needs a marketplace/Connect-style processor with payouts, not a merchant of record. Digital goods sold inside the mobile app fall under store in-app purchase rules.
- **Isolation tiers:** pooled (shared DB, `tenantId` + RLS) by default; per-tenant database or Durable Object namespace only for tenants who pay for it.
- **Open:** O15 end-customer auth provider; marketplace payments provider; whether white-label is offered at all for a given product.
- **Re-examined against every archetype (2026-10-15). Proposed amendments:**
  - *Holds:* three audiences, one action layer; the tenant console is the product; typed tenant configuration; two mobile modes; custom domains; per-tenant sending identity.
  - **The tenant is the org.** Use one `orgId` column everywhere (not `tenantId`), so a B2B product that grows into B2B2C needs no migration. End-customer rows add `customerId`.
  - **Namespaces:** the primary audience lives at the contract root, and only secondary audiences get a prefix (`admin.*`, `customer.*`). A `tenant.*` prefix would force renaming every B2B procedure when the product grows into B2B2C.
  - **End-customer identity, default flipped.** The default becomes one platform identity with a per-org customer profile. That fits the shared app (the default mobile mode): a person who uses two orgs signs in once, and each org sees only its own profile. Per-org isolated pools are reserved for white-label tiers that need them. Checked 2026-10-15: WorkOS AuthKit supports a flat, org-less pool (free to 1M MAU) but has no self-serve per-tenant pools, so the old default was also the expensive one. O15 updated.
  - **Configuration and entitlements are separate.** `orgConfig` holds what the org chose (branding, domains, senders). Entitlements hold what it paid for, and billing writes them. P11 had them together.
  - **Isolation tiers corrected:** pooled → dedicated database (a Neon project per org, routed by `orgId`) → regional (an EU database plus Durable Object jurisdiction) → dedicated deployment. "Per-tenant Durable Object namespace" is dropped: a namespace is per class binding, so per-tenant objects come from keying (Workers for Platforms if per-tenant code is ever needed). Tracked in O20.
  - **Org API keys** are `apiKey` principals. **Service agents** serving end customers for an org follow P9.
  - **Third-party payments** move out to O16, shared with marketplaces.

### P12. Archetype guide and day-one seams
- **Proposed:** [docs/archetypes.md](docs/archetypes.md) becomes the kickoff guide. Each project names its archetypes, builds the core, adds only the variants for the archetypes it is today, lays the seams, and records all of this in its `AGENTS.md`.
- **Day-one seams in every product** (archetypes §6.1):
  - `orgId` on every tenant-owned row, with personal orgs in B2C and internal tools;
  - the shared `Actor`;
  - action metadata in the contract (audience, risk tier, exposure, idempotency) as oRPC procedure metadata;
  - entitlements that product code reads and only billing providers write;
  - an append-only usage ledger;
  - an audit stream for every write;
  - our own `user`/`org` tables, with identity provider IDs as foreign links;
  - typed `orgConfig`;
  - Durable Objects behind a client;
  - an additive `/v1` contract.
- **Amends P2:**
  - B2C users get a personal org.
  - Marketplace transactions are scoped by party (`buyerOrgId`, `sellerOrgId`).
  - `orgId` comes from the verified session: for personal orgs it is derived on the server from the user, not carried in the token. Our `org` table is the tenancy source of truth; a WorkOS organization is linked only when SSO or managed membership is needed.
- **Why:** each seam costs a table, a type or a field on day one. Retrofitting one means migrating every table or every handler, and moving between archetypes is the common path (B2C → B2B, B2B → API, any → agent-native).
- **Alternatives considered:** per-archetype starter templates (they drift apart and double maintenance); laying seams only when needed (cheap until the move, then the most expensive migrations there are).
- **Revisit if:** a seam goes unused across the first three products, or its day-one cost turns out to matter (e.g. personal orgs at consumer scale).

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
| O16 | **Third-party payments** (marketplace, B2B2C) | Processor, charge model, who carries disputes, KYC flow, seller tax reporting | Checked 2026-10-15: no merchant of record allows marketplaces (Polar, Paddle and Lemon Squeezy prohibit them; Stripe Managed Payments excludes Connect). **Lean: Stripe Connect**, Accounts v2 for new integrations (GA for Direct users is on Stripe's Q4 2026 roadmap). Keep our own balance ledger, reconciled against webhooks. |
| O17 | **Sales-led B2B billing** | Invoices, net terms, purchase orders, quotes, tax IDs | Polar covers self-serve. Options: Stripe Billing + Stripe Tax (we become the seller of record), or Stripe Managed Payments once its Invoicing ships (private preview Q4 2026, GA Q1 2027 per Stripe's roadmap). Decide when the first deal needs it. Entitlements (P12) keep the choice swappable. |
| O18 | **API product kit** | Keys, limits, outbound webhooks, docs, SDKs | Keys: our own hashed table vs Unkey (free 150K verifications/month). Limits: Cloudflare rate-limiting binding (GA; per location, eventually consistent) for bursts, usage ledger for quotas. Webhooks: Queues + Standard Webhooks format vs Svix. Docs: Scalar (MIT renderer; hosted docs paid) vs Mintlify. SDKs: Stainless closed new sign-ups after joining Anthropic (2026-05); Speakeasy open-sourced its generator (AGPLv3, 2026-09; check the licence of generated code); Fern is now Postman; Hey API (MIT, TypeScript, pre-1.0); Cloudflare Forge (Apache-2.0, early). **Lean:** own keys, Scalar, Hey API for TypeScript; decide other languages when a customer asks. |
| O19 | **Abuse, fraud and moderation** (B2C, marketplace, user content) | Bot defence, sign-up fraud, SMS pumping, moderation pipeline, KYC, EU platform duties | Turnstile is free and unlimited (checked 2026-10-15). OpenAI moderation (free) as a first pass, with P10 work queues for human decisions. KYC through the payments processor; Stripe Identity $1.50 per verification. Marketplaces in the EU carry Digital Services Act duties (notice and action, trader traceability). |
| O20 | **Residency and dedicated tenancy** (B2B enterprise, B2B2C) | When and how to offer an EU region, a dedicated database or a dedicated deployment | Routing by `orgId`; Neon project per region or per org; Durable Object jurisdiction. Verify Hyperdrive and Durable Object jurisdiction behaviour before promising it to a customer. |
| O21 | **Search** (marketplace, content-heavy apps) | When Postgres full-text search stops being enough, and what replaces it | Neon removed `pg_search` on 2026-09-21; its `lakebase_text` (BM25) is in preview. Next step: Meilisearch (MIT; cloud from $20/month) or Typesense. The index is derived data, fed through the outbox (P4). |
| O12 | One-time setup | Sentry + Better Stack accounts; Mac self-hosted runner (when mobile starts) | |

---

### O14. In-app agent runtime and UI wire
Three layers to pick: **runtime** (where the agent runs and keeps state), **model + wire** (tool loop and the stream to the UI), **UI** (chat and approval components).
- Runtime: **Cloudflare Agents SDK** (agent = Durable Object with SQLite; `setState` synced to clients over WebSocket; chat agent with client tools and human-in-the-loop approvals; scheduling). Fits P5 and our infra. Alternatives: Mastra, LangGraph (separate backend; only if we leave Workers).
- Model + wire: **Vercel AI SDK v6** (most proven; `useChat`, client tools, tool approval; own stream format; Cloudflare's chat agent builds on it) vs **TanStack AI** (RC Aug 2026, not 1.0; AG-UI is its official protocol; same family as our web stack) vs **Effect AI** (`effect/unstable/ai`, no UI layer; only if O1 = Effect). CopilotKit (AG-UI originator, heaviest, hosted runtime) not by default.
- Collaborative documents (rich text, canvas, slides): **Liveblocks** (or Yjs on our own Durable Objects) only when the product's core surface is a shared document. Then the document is that surface's domain state; agents still write through a server action that checks `can()` and records the actor before calling Liveblocks, never with the raw secret key. Liveblocks Comments + Notifications is a reasonable buy for any app that needs comments. Not the default for record-and-workflow apps (FirstWord): there, Postgres + Durable Objects + actions already give live shared state with validation and authz.
- UI: our shadcn components; assistant-ui (headless, shadcn-based) if chat UI grows. Generative UI specs (A2UI, MCP Apps) not needed in-app.
- Lean: Cloudflare Agents SDK + AI SDK as baseline; re-check TanStack AI at 1.0. Our three-state rules (P9) and actions keep the wire swappable.
- Decide with a FirstWord spike (agent proposes a screening rule while the user edits rules).

### O15. End-customer auth for B2B2C
**Updated 2026-10-15 (P11 re-examination):** the default is now one platform identity with a per-org customer profile; isolated per-org pools only for white-label tiers. That makes WorkOS AuthKit without organizations (flat pool, free to 1M MAU, verified) the lead candidate, since it is already our provider; per-org pools would need another provider or extra WorkOS environments arranged through support. Original framing: need per-tenant user pools, phone/email OTP, passkeys, Sign in with Apple, tenant-branded screens, works with Expo native screens. WorkOS is strongest for tenant staff (orgs, SSO, SCIM) but is B2B-first; candidates for end customers: WorkOS AuthKit without organizations (flat pool, tenant scoping in our DB), Clerk, Better Auth (self-hosted, in our DB), Stytch. Decide on: per-tenant isolation of the same email, branding, cost at consumer scale, native mobile support.

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
