# Product archetypes

What the-stack looks like for each kind of product: what every product shares, what each archetype adds, and the seams that let a product move from one archetype to another without a rewrite.

**Status: Proposed (DECISIONS P12).** Decisions referenced here keep their own status in [DECISIONS.md](../DECISIONS.md); this guide only maps them to archetypes. Last updated: 2026-10-15.

## 1. How to use this guide

Kickoff is a PRD session with the user (DECISIONS O10). In it, the agent:

1. **Names the archetypes.** One primary, plus any secondary ones (most products are two or three: "B2B SaaS + AI/agent-native, API later").
2. **Builds the core (§2)** whatever the archetype.
3. **Adds the variants** for each named archetype (§5).
4. **Lays the seams (§6)** for any archetype the product may grow into. They are cheap on day one and expensive to retrofit.
5. **Records all of it** in the project's `AGENTS.md` (alongside its README and `docs/prd/*`): archetypes, variants chosen, seams in place, deviations and why.

Don't build an archetype's variants before the product is that archetype. SSO, white-label builds, split payments, offline sync and CRDT documents arrive with the move, not before. The seams are what make the move cheap.

## 2. The core: invariants for every archetype

| Concern | Rule | Source |
|---|---|---|
| Actions | Every capability is one contract procedure; UI, mobile, public API, MCP, admin and agents all call it. No side doors: no admin tool with database credentials, no agent clicking through our own UI | README §3.2, P9 |
| Risk | Every action has a risk tier (`read` / `reversible` / `irreversible` / `external`); reversible beats approval prompts; the server enforces approval for agents on the risky tiers | P9 |
| Actor | One actor model for users, staff, agents, API keys and system jobs (§3) | P9–P11 |
| Authorization | One `can()` in `core`; tenancy comes from the verified session, never from input | P2 |
| Tenancy | `orgId` on every tenant-owned row, in every archetype; B2C users get a personal org | P2, §6 |
| Types and errors | Parse at the boundary; typed errors with a literal `code` | coding-standards, errors |
| Async | Idempotent webhooks, queues, workflows and outbox | P4 |
| State | Every piece of state has one owner: Postgres for shared truth, Durable Objects for per-entity live state, the browser for view state, the agent for run state. Other copies (search index, cache, offline replica) are derived and can be rebuilt | P5, P9 |
| Money | Once anything is paid or plan-gated: product code reads **entitlements**; billing providers only write them | §6 |
| Usage | Once anything costs per use: metered events go to one append-only **usage ledger** | §6 |
| Audit | Every write records its actor and emits an audit event | P10, §6 |
| Operations | Admin app + admin MCP from day one | P10 (Decided) |
| Quality | Traces end to end, SLOs, performance budgets, product analytics | P3, P6, P8 |
| Delivery | IaC, a stage per PR, vault for dev secrets, host-native prod secrets; fakes for every dependency not under test | README §4–§6, P13 |
| Observability | Traced from day one; the public `requestId` is the trace ID | P3 |

## 3. One actor model

Every request builds one `Actor` from verified credentials. `can()`, audit, rate limits, budgets and the UI ("who changed this") all read it. P9, P10 and P11 each described part of it; this is the shared shape.

```ts
/** Who is acting, on whose authority, inside which boundary. Built once per request from verified credentials, never from input. */
type Actor = {
  principal:
    | { kind: "user"; userId: UserId }          // a person signed in to the product
    | { kind: "staff"; staffId: StaffId }       // our staff, through the admin front door
    | { kind: "apiKey"; keyId: ApiKeyId }       // a machine credential owned by an org
    | { kind: "system"; source: SystemSource }; // webhook, schedule, queue consumer
  scope:
    | { kind: "org"; orgId: OrgId; role: OrgRole }               // member of an org (a personal org in B2C)
    | { kind: "customer"; orgId: OrgId; customerId: CustomerId } // end customer of an org (B2B2C)
    | { kind: "platform" };                                      // staff and system work across orgs
  agent?: {                         // present when an agent drives the call
    agentId: AgentId;
    sessionId: AgentSessionId;
    mode: "delegated" | "service";  // delegated: a person's assistant; service: serves someone else for the principal
    grant: readonly Permission[];   // what this agent session was granted
  };
  support?: {                       // staff working inside a customer's org
    orgId: OrgId;
    grantId: SupportGrantId;
    reason: string;
    expiresAt: Date;
  };
};
```

- **Permissions** = what the principal may do in its scope ∩ the agent session's grant (if any) ∩ the org's agent policy (if any). Plan limits are a separate check (`entitled()`), so "forbidden" and "upgrade required" stay different errors.
- **Delegated vs service agents.** A delegated agent works for the person using it (an in-app assistant, an external agent over MCP). A service agent works for the principal while talking to someone else: a support bot answering an org's customers, a voice agent taking calls for a user, a seller's agent negotiating with a buyer. The person it talks to is untrusted input and never gains authority (P9 rule on untrusted content).
- **Support access is not impersonation.** Staff act as themselves inside a customer's org, with a reason and an expiry, visible to that org (P10). There is no "log in as".
- **System actors are real actors.** A webhook or schedule that changes data is audited like anyone else.

## 4. Where archetypes differ

| Archetype | Identity pools | Tenancy unit | Money flow | Dominant state |
|---|---|---|---|---|
| **B2C** | One consumer pool | Personal org | Subscriptions: web merchant of record + store in-app purchase | Records, user content |
| **B2B SaaS** | Org members (SSO/SCIM) | Org | Seats or usage; self-serve plus invoices | Records, workflows |
| **B2B2C** | Org members + end customers | Org + customer | Orgs pay us; end customers pay orgs | Records, per-org config |
| **Marketplace** | One pool, two sides | Two parties per transaction | Split payments, escrow, payouts | Listings, orders, ledger, search index |
| **AI / agent-native** | Plus agent sessions | Org; agent runs per session | Variable cost per request; usage or credits | Run state, records |
| **Realtime / collaborative** | Plus guests via links | Org; one Durable Object per shared entity | Seats | Live shared state, documents |
| **API / developer** | Org members + API keys + OAuth clients | Org; test and live modes | Usage-metered | Records, outbound events |
| **Internal tool** | Staff only | The company (one org) | None | Records, operations |
| **Mobile-first** | Any, plus devices | Any | Store in-app purchase + web link-out | Local replica + server truth |

## 5. Archetypes

Each section lists only what the archetype adds to or changes in the core.

### 5.1 B2C

Many individuals, low revenue each, growth-driven, often mobile.

- **Identity:** one consumer pool. Passkeys, email one-time codes, Sign in with Apple/Google; no passwords by default. WorkOS AuthKit (README default) or Clerk native components for consumer-only mobile apps.
- **Tenancy:** a personal org created at sign-up, hidden in the UI. Public content (profiles, posts) carries an explicit visibility field instead of skipping the org scope.
- **Money:** web checkout through the merchant of record (Polar); store in-app purchase (RevenueCat); both write the same entitlements. A free tier is the norm, so cost per free user is a design input.
- **Surfaces:** prerendered marketing (SEO matters), web app, mobile app.
- **Abuse:** bots at sign-up and on anything free. Bot challenge on sign-up and expensive actions, rate limits per user, device and IP, caps on free usage. Avoid SMS codes where possible (SMS pumping fraud); if SMS is needed, restrict countries and use the provider's fraud guard (O19).
- **User content:** report button, moderation queue (P10 work queues), takedown as a named operation.
- **Compliance:** README §10 launch minimums; data export and deletion as named operations; analytics consent in the EU; age rating and minors' rules if children could plausibly use it.
- **First to break:** the cost of free usage, then database connections and vendor rate limits during a spike.
- **Decisions:** P2 (personal org), P6, P9 (assistant with per-user budgets), P10 (abuse queue), O19.

### 5.2 B2B SaaS

Organizations buy; admins configure; members work. Few users, high value per account; sales-assisted at the top end.

- **Identity:** a user can belong to many orgs (org switcher; the active org lives in the session). Invites, verified domains with automatic joining, SSO (SAML/OIDC) and SCIM through WorkOS when a customer asks.
- **Tenancy:** org, pooled, RLS backstop (P2). Enterprise variants, only when paid for: dedicated database, data residency, dedicated deployment (O20).
- **Money:** seat or usage subscriptions. Self-serve through the merchant of record; larger customers want invoices, net terms, purchase orders and tax IDs (O17).
- **Org admin console is part of the product:** members and roles, SSO and SCIM setup, billing, audit log with export, API keys, agent policy, support access log. Built from the same admin components as P10, scoped to one org.
- **Trust:** least-privilege roles; a customer-visible audit log; staff support access visible to the org's admins (and approval-gated for orgs that require it); MFA and session rules enforced through the identity provider.
- **Compliance:** DPA and subprocessor list, security page, retention settings, stated backup and recovery targets; SOC 2 through Vanta when a buyer asks (README).
- **First to break:** one large org (a huge import, export or query) slowing everyone. Per-org rate limits, bulk work as queued jobs, query timeouts.
- **Grows into:** API product (customers integrate), B2B2C (customers bring their customers), agent-native (customers bring their agents).
- **Decisions:** P2, P8, P9 (+ org agent policy), P10 (+ support access visibility), O4, O17, O20.

### 5.3 B2B2C / platform

Businesses run their own customer relationships on our product.

- **Identity:** org members as in B2B. End customers: **default one platform identity with a per-org customer profile**, so one person using two orgs in the shared app signs in once and each org sees only its own profile. Isolated per-org pools only for white-label tiers that need them (O15). WorkOS AuthKit supports a flat, org-less user pool (free to 1M MAU), which fits this default; it has no self-serve per-tenant pools (checked 2026-10-15).
- **Tenancy:** the tenant *is* the org, so one `orgId` column everywhere; end-customer rows add `customerId`.
- **Money:** orgs pay us (subscription); end customers pay orgs through a third-party payments processor with payouts (O16); digital goods sold inside a mobile app follow store rules.
- **Surfaces:** tenant console (the B2B org console, grown up), customer app (shared or white-label Expo build), customer web on the org's own domain (Cloudflare for SaaS), public API and webhooks for orgs that integrate.
- **Trust:** no path from one org to another org's customers; staff access through support grants that orgs can see; per-org sending identities so one org can't damage another's email deliverability.
- **Compliance:** for end-customer data we are usually the processor and the org the controller; deletion and export requests arrive through the org.
- **First to break:** the largest org, and native build fan-out once white-label exists.
- **Decisions:** P11, O15, O16, O20.

### 5.4 Marketplace

The platform matches two sides, owns both relationships and takes a cut.

- **Identity:** one platform pool. A person can be buyer and seller; sellers usually act through an org (a business, or a personal org for individuals), verified through the payments processor (KYC/KYB).
- **Tenancy:** a transaction belongs to **two parties** (`buyerOrgId`, `sellerOrgId`); repositories scope by party, not by a single org. The catalog is platform-owned and publicly readable.
- **Money:** split payments, delayed payout or escrow, refunds, disputes, payouts, platform fee, seller tax reporting (O16). No merchant of record allows marketplaces (Polar, Paddle, Lemon Squeezy and Stripe Managed Payments all exclude them, checked 2026-10-15), so this archetype needs a Connect-style processor. Keep our own append-only ledger of balances, reconciled against processor webhooks; the processor holds the money, the ledger explains it.
- **Surfaces:** buyer web and app (public listing pages are a case for per-route SSR, README §3.1), seller console, admin with trust-and-safety queues.
- **Trust and safety is the product:** seller verification, listing and message moderation, review integrity, fraud, disputes, off-platform leakage in messages. In the EU, online-platform duties (notice and action, trader traceability, statements of reasons) apply (O19).
- **Search and matching:** start with Postgres full-text search; move to a search engine when relevance and facets demand it (O21). Neon removed the `pg_search` extension on 2026-09-21, so BM25 inside Neon is not an option today. The index is derived data, rebuilt from Postgres through the outbox.
- **Contention:** two buyers, one item. Database constraints for simple stock; a Durable Object per listing for holds and auctions.
- **First to break:** search quality and latency, then fraud.
- **Decisions:** P2 (party scoping), P4, P10 (work queues), O16, O19, O21.

### 5.5 AI / agent-native

Agents and people work on the same surface; model calls drive cost and latency.

- **Actors:** delegated and service agents (§3); risk tier on every action; approvals are server-side pending operations (P9, P10).
- **Data:** agent run state in a Durable Object per session or thread, keyed by its domain ID. Retrieval (vectors, knowledge) is filtered by `orgId` inside the retrieval adapter, never by prompt instructions.
- **Money:** cost per request varies. Every run carries a budget per actor and per org, enforced on the server from the usage ledger; pricing is usage or credits, with margin watched per model.
- **Surfaces:** in-app agent (O14), MCP for external agents, admin MCP (P10). Target MCP 2026-07-28 (stateless; OAuth with Client ID Metadata Documents, dynamic client registration deprecated). In B2B, the Enterprise-Managed Authorization extension lets the org's identity provider decide which agents may connect.
- **Quality:** prompts versioned in code; evals in CI gate prompt and model changes; LLM traces share the trace ID; model fallbacks through the AI gateway (O5).
- **Trust:** untrusted content can't widen authority; model output is never an authorization decision; minimal personal data in prompts; zero-retention providers where offered.
- **Compliance:** disclose that it's an agent (P9); per-org choices on data retention and training use; check current EU AI Act transparency duties for the product.
- **First to break:** cost, then provider rate limits.
- **Evidence:** FirstWord exercises this archetype (service agent on realtime voice).
- **Decisions:** P9, O5, O14.

### 5.6 Realtime / collaborative

Several people (and agents) see and change the same thing live.

- **Pick the model by data shape:**

| Shape | Model |
|---|---|
| Records and workflows that update live | Postgres truth + Durable Object pushes that refresh queries (README default) |
| Shared documents (rich text, canvas, slides) | CRDT: Yjs on our Durable Objects, or Liveblocks (O14) |
| High-frequency shared state (games, cursors, live control) | One authoritative Durable Object per room |

- **Tenancy:** one Durable Object per shared entity, keyed by canonical ID, behind a client (coding-standards §8). Authorize on connect; re-check on permission change and disconnect revoked users.
- **Sharing and guests:** share links are capability grants (one resource, one role, expiry), checked by `can()` (O4).
- **Conflicts:** CRDT for documents; version checks and field-level merge for records; an agent never overwrites a field a person is editing (P9).
- **Persistence:** Durable Object storage holds live state; snapshots go to Postgres or R2 per P5.
- **Tests:** two or more browser contexts in one Playwright test.
- **First to break:** a hot room (one Durable Object is one thread). Use WebSocket hibernation; shard presence from content when rooms grow.
- **Decisions:** P5, P9 (shared state), O4, O14.

### 5.7 API / developer product

The public contract is the product; customers are developers, their programs and their agents.

- **Identity:** developer accounts are B2B orgs. API keys are `apiKey` principals: owned by an org, scoped, separate test and live keys, stored hashed, shown once, prefixed so secret scanners can spot leaks. OAuth for third-party apps acting for users; MCP with OAuth for agents.
- **Contract:** `/v1` OpenAPI generated from the contract (coding-standards §7). Additive within a version; a breaking change is a new version with a deprecation window, announced with `Deprecation`/`Sunset` headers and a changelog. `Idempotency-Key` on every non-idempotent write; cursor pagination; one error shape (errors.md); request ID on every response.
- **Test mode:** test keys hit a test data plane inside production (the Stripe pattern). This is a product feature, separate from our PR stages.
- **Limits:** per key and per org, set by plan entitlements; `429` with `Retry-After` and rate-limit headers. Cloudflare's rate-limiting binding is per location and eventually consistent: use it for burst protection, and count billable quotas from the usage ledger.
- **Outbound webhooks:** signed (Standard Webhooks format), retried with backoff, per-endpoint delivery log and replay (O18).
- **Developer experience:** docs and SDKs generated from the OpenAPI spec, `llms.txt`, an MCP server, a public status page (O18).
- **First to break:** one client's burst; then a breaking change slipping through (CI breaking-change check, coding-standards §7).
- **Decisions:** README §3.2, coding-standards §7, P4, P8, O18.

### 5.8 Internal tool / back office

Staff-only; P10's admin pattern is the whole product.

- **Identity:** company identity provider behind Cloudflare Access; staff roles held in our database; no public sign-up.
- **Tenancy:** one org (the company), still on every row, so a tool that becomes a product (a common path) doesn't need a migration.
- **Surfaces:** the admin app shape (P10) and an admin MCP so agents can run operations, with server-side approvals.
- **Trust:** least privilege, audit, masked personal data, read-only database role for reports.
- **Lighter:** relaxed quality targets, no marketing, no billing.
- **Decisions:** P10.

### 5.9 Mobile-first

The phone app is the product; web is secondary or marketing.

- **Releases:** stores lag deploys. Additive contract changes (README §3.2); feature flags (P6); EAS Update for JavaScript; native changes need a build. `viewer.state` returns a **minimum supported app version** so old builds can be forced to update; old versions keep working inside a stated window.
- **Offline (only when required):** PowerSync with sync rules partitioned by org and user. Writes upload through contract actions, never straight to the database, so `can()`, audit and idempotency still apply.
- **Push:** device tokens per session; notification preferences; sending is an outbox side effect (P4). Approvals waiting on a person (P9) arrive as push notifications.
- **Identity:** native screens through our API (README); refresh tokens in secure storage; device session list with remote sign-out; passkeys.
- **Money:** store in-app purchase (RevenueCat) and web link-out where allowed (README §10), both writing entitlements. Store fees moved on 2026-10-01 (Apple's new EU terms; Google Play US link-out fees began), so re-check README §10 before pricing a mobile product.
- **Links:** universal and app links; deferred deep links for invites.
- **Clients retry on bad networks**, so every write carries an idempotency key from the device.
- **Tests:** Maestro flows (V5), Expo web build in CI.
- **First to break:** an old app version against a changed contract.
- **Decisions:** README mobile rows, P6, P11 (white-label), V5.

## 6. Extension points

### 6.1 Seams to lay on day one

Seams are plumbing around whatever domain model the PRD defines, not a generic domain model; the domain model stays product-specific and minimal. Seven seams go into every product. Three wait for a product **trait** (not an archetype label) and go in the day that trait appears.

| Seam | Lay it | Day-one cost | Unlocks | Retrofit cost without it |
|---|---|---|---|---|
| `orgId` on every tenant-owned row; personal org in B2C and internal tools | Always | One table, one column | B2C → B2B, internal → B2B, B2B → B2B2C | Backfill every table, rewrite every query |
| One actor model (§3) | Always | One type, built per request | Agents, staff, API keys, system jobs, support access | A special case in every handler for each new caller |
| Action metadata in the contract: audience, risk tier, exposure (RPC / OpenAPI / MCP), idempotency | Always | A few fields per procedure | Public API, MCP, admin, agent approvals generated instead of hand-built | Re-classify every procedure later |
| Entitlements read by product code; providers write them | When anything is paid or plan-gated | One table, one webhook path | Web + in-app purchase + invoices + marketplace; plan changes without code | Provider checks scattered through the code |
| Usage ledger | When anything costs per use (AI calls, metered API, usage pricing) | One append-only table | AI budgets, API quotas, usage billing | No history to bill or cap from |
| Audit stream for every write | Always | One table, written by `core` | Customer audit log, admin timeline, agent review | History lost before the day it's needed |
| Our own `user` and `org` tables in Postgres (the provider keeps authentication; its IDs are stored as plain columns) | Always | Two tables | Swap the identity provider; add a second identity pool | Provider IDs as primary keys everywhere |
| Typed `orgConfig` (separate from entitlements) | When orgs configure anything | One schema | White-label, enterprise settings, per-org features | Org-specific branches in code |
| Durable Objects keyed by domain ID behind a client | Always (when Durable Objects are used) | The coding-standards §8 shape | Realtime, agents, per-entity scheduling | Global singletons that hot-spot |
| Additive contract, `/v1` from day one | Always | Discipline + CI check | Mobile, public API, partners | Breaking shipped clients |

### 6.2 Moves between archetypes

| Move | What gets added | Relies on |
|---|---|---|
| B2C → B2B | Org invites and roles, org console, SSO/SCIM, invoices, audit export | Personal org, actor model, entitlements, audit stream |
| B2B → B2B2C | `customer` scope and namespace, end-customer identity, `orgConfig` branding, third-party payments, custom domains | `orgId` = tenant, own identity tables, actor model |
| B2B or B2C → marketplace | Second side, party-scoped repositories, payouts, ledger, trust and safety, search | Entitlements, usage ledger, audit, work queues |
| Any → API product | API keys, rate limits, test mode, outbound webhooks, docs, SDKs | Contract-first, `/v1`, action metadata (exposure), actor model |
| Any → agent-native | Agent sessions, risk tiers enforced, pending operations, budgets | Actor model, action metadata (risk), audit, usage ledger |
| Any → realtime | Durable Object per entity, live pushes, presence | Durable Object shape, server-owned domain state |
| Web → mobile-first | Expo app, push, in-app purchase, minimum-version gate, offline if needed | One API (no web-only server functions), additive contract, `viewer.state`, entitlements |
| Internal → B2B | Public sign-up, billing, org console | `orgId` on every row, actor model |
| One region → residency | Regional database and Durable Object jurisdiction, routing by org | `orgId` on every row; Durable Objects behind a client |

## 7. Decision map

● core for this archetype · ◐ applies with a variant · – doesn't apply

| Decision | B2C | B2B | B2B2C | Mkt | AI | RT | API | Int | Mob |
|---|---|---|---|---|---|---|---|---|---|
| P2 authorization and tenancy | ◐ personal org | ● | ◐ + customer | ◐ party scope | ● | ● | ● | ◐ one org | ● |
| P3 observability | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| P4 idempotency | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| P5 state | ● | ● | ● | ● | ● | ● | ● | ● | ◐ local replica |
| P6 product insight | ● | ● | ● | ● | ● | ● | ◐ API usage | – | ● |
| P8 quality targets | ● | ● | ● | ● | ● | ● | ● | ◐ relaxed | ● |
| P9 agent-native | ◐ budgets | ◐ org policy | ◐ service agents | ◐ agentic commerce | ● | ◐ agent presence | ◐ MCP + OAuth | ● | ◐ push approvals |
| P10 admin | ◐ abuse queue | ◐ support visibility | ● | ◐ T&S queues | ● | ◐ live sessions | ◐ key ops | ● the product | ◐ min version |
| P11 B2B2C | – | ◐ org console | ● | – | – | – | – | – | ◐ white-label |
| O4 per-resource sharing | ◐ | ● | ◐ | – | – | ● | – | – | – |
| O14 in-app agent runtime | ◐ | ◐ | ◐ | ◐ | ● | ◐ | – | ● | ◐ |
| O15 end-customer auth | – | – | ● | – | – | – | – | – | ◐ |
| O16 third-party payments | – | – | ● | ● | – | – | – | – | – |
| O17 sales-led billing | – | ● | ◐ | – | – | – | ◐ | – | – |
| O18 API product kit | – | ◐ | ◐ | – | – | – | ● | – | – |
| O19 abuse and moderation | ● | – | ◐ | ● | ◐ | ◐ | ◐ | – | ● |
| O20 residency and dedicated tenancy | – | ◐ | ◐ | – | – | – | ◐ | – | – |
| O21 search | ◐ | ◐ | ◐ | ● | ◐ | – | – | – | – |

## 8. Not covered yet

Candidates to add when a product needs them:

- **Content / media** (publishing, SEO-led, CMS, paywalls).
- **Single-merchant commerce** (storefront, inventory, fulfilment; often Shopify-backed rather than built).
- **Data / analytics product** (ingestion, warehouse, query cost per tenant).
- **Hardware / IoT companion** (device identity, provisioning, firmware updates).
- **Regulated overlay** (health, finance, children): not an archetype but a layer on top of any of them, changing hosting, audit, retention and vendor choice.
