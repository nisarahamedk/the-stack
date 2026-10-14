# Agent Stack — reference for building new apps

Point an agent here with: *"Build <app>. Use this reference."*
This file defines the default stack, how a project is set up, and how the dev → QA → prod loop works.
Deviate only when the app genuinely needs it, and say why in the PR.

Open and proposed decisions live in [DECISIONS.md](DECISIONS.md).

Last verified: 2026-10-12. Items marked **(unproven)** have not been exercised end to end yet.

**Proven in a real project (FirstWord, 2026-10-12):** Alchemy v2 PR stages on the dev Cloudflare account with preview + destroy workflows (40+ preview runs); Infisical `dev-vault` login from agent sessions; prod deploy on push to `main` via GitHub Actions + wrangler (successful runs); react-doctor at zero issues in CI.

---

## 1. Principles

1. **Agent-native.** Every feature must be testable by an agent without a human: emulator → sandbox/test mode → real provider.
2. **Secrets live only in the vault.** Code never contains them; anything code can create (IDs, URLs, tokens) is created by IaC, not stored.
3. **Everything is infrastructure as code.** Every environment can be created and destroyed with one command.
4. **An isolated environment per PR.** The agent opens a draft PR early and uses its environment. No shared staging, no queueing.
5. **Hard spending caps wherever a provider offers them**; alerts plus a kill switch where it doesn't.
6. **Portable by default.** Standard Postgres, framework-agnostic app code, platform-specific bits behind small interfaces.

## 2. Default stack

| Concern | Default | Notes |
|---|---|---|
| Repo | Monorepo: pnpm workspaces + Turborepo | `apps/web`, `apps/api`, `apps/mobile`, `packages/*` |
| Language | TypeScript everywhere, Zod 4 at boundaries | Effect TS: not default yet — see §3.1 |
| Web | TanStack Start | See §3.1 for rendering, data and guardrails |
| API | oRPC (contract-first) served by Hono on Cloudflare Workers — **pin the version** | One typed API for every client (web, mobile, partners); typed RPC at `/rpc`, OpenAPI REST at `/api/v1`. Authorization lives here (no RLS). See §3.2 |
| Mobile | Expo (EAS Build / Update / Workflows) | No secrets in the app bundle — publishable keys only |
| Hosting | Cloudflare Workers | AWS / Hetzner / Fly when needed (Alchemy supports them); Vercel projects use Vercel's own previews |
| IaC | Alchemy v2 — **pin the exact version** (beta) | Stages per PR + prod; custom resources for SaaS without a provider |
| Database | Neon Postgres + Drizzle, via Hyperdrive | Branch per environment. D1 only for small Cloudflare-only apps |
| Offline mobile sync | PowerSync (only if offline is required) | |
| Background work | Cloudflare Queues, Cron Triggers, Workflows, Durable Object alarms | |
| File storage | Cloudflare R2, presigned uploads | R2 has no versioning — back up what matters |
| Auth | WorkOS AuthKit | Web: hosted pages + Widgets. Mobile: custom native screens via your API (headless). Consumer-only mobile apps may use Clerk's native components instead |
| Payments | Polar (merchant of record) on web | Mobile: link US users to web checkout; RevenueCat for in-app subscriptions elsewhere |
| Email | Resend | |
| AI models | OpenRouter, behind Cloudflare AI Gateway | Direct providers for realtime voice (OpenAI Realtime / Gemini Live), streaming STT/TTS (Deepgram, ElevenLabs, Cartesia), broad image/video (fal.ai) |
| AI evals | Evalite (in CI) + Langfuse (tracing) | |
| Errors | Sentry | Set a pay-as-you-go budget (hard cap) |
| Uptime / status / paging | Better Stack (free) + PagerDuty Free for phone paging | Rootly / incident.io once there's a team |
| Analytics + feature flags | PostHog | |
| Secrets | Infisical | See §4 |
| CI | GitHub Actions | GitHub-hosted runners; self-hosted Mac runner for iOS |
| Dependencies / security | Renovate (or Dependabot), Dependabot alerts, Cloudflare WAF | |
| Domains / DNS | Cloudflare Registrar + DNS | |
| Compliance (when an enterprise buyer asks) | Vanta (has MCP + Claude Code plugin); cheaper: Drata, Comp AI. Avoid Delve | |

## 3. Project layout

```
apps/
  web/            TanStack Start: marketing (prerendered) + app (gated, data-only)
                  day one: also hosts the oRPC handler (/rpc, /api/v1) and webhooks
  api/            (later) Hono Worker hosting the oRPC handler, split out when a 2nd client arrives
  mobile/         (later) Expo app
packages/
  contract/       oRPC contract: procedures + Zod schemas + typed errors — the ONLY backend package clients import
  core/           business logic: plain functions, no HTTP, no React (db/providers passed in)
  db/             Drizzle schema + migrations
alchemy.run.ts    all infrastructure, per stage
secrets.json      vault names this project uses (no values)
.env.example      non-secret config keys
AGENTS.md         project-specific notes; links back to this reference
```

### 3.1 Web app conventions

The linter is the rulebook; this section only records the choices no linter makes.

- **Guardrails (enforced in CI, not prose):**
  - **react-doctor** — pin the version, run with `--no-telemetry`, `--scope changed --blocking error` in CI, and install its agent skill (`npx react-doctor@<version> install`). Its ~60 rules (incl. TanStack Start rules) are the React rulebook; don't restate them here.
  - **React Compiler** on; agents don't hand-write `useMemo`/`useCallback`.
  - **`eslint-plugin-react-hooks`** `recommended` preset.
  - When react-doctor flags something that doesn't apply (e.g. an SSR-only rule on a client-only route), disable that rule in config with a comment saying why — don't work around it in code.
- **Rendering and gating — decide on the server, draw in the browser:**
  - **Signed-in app routes: `ssr: 'data-only'`** on the gated pathless layout routes (TanStack Start selective SSR). On the first request the Worker runs the gate (`beforeLoad`) and critical loaders; React components render only in the browser, so no hydration mismatches and no server-rendered component code. Later navigations run the same gate in the browser using cached state.
  - **Why not `ssr: false`:** a client-only app draws the URL's route before it knows the session or account state, then corrects itself — the redirect/flash bug. Waiting on the client fixes it only while every component obeys the rule, costs 4–5 sequential round trips, and needs auth JS loaded first. A server gate redirects with HTTP before any JS downloads, so a wrong first screen can't be drawn regardless of client code.
  - **Marketing / public pages: prerendered at build time.**
  - **Full SSR (`ssr: true`) only by exception**, per route, for public pages with dynamic content that must be indexed. Note the reason in the route file.
- **Auth and lifecycle gating rules:**
  - **One source of truth:** a single `viewer.state` procedure returns the user's lifecycle state (`anonymous` → `needs_setup` → `active` → …). No cookies or local storage remembering state.
  - **One gate:** a function on the pathless layout routes maps state → allowed route groups (`(public)`, `(auth)`, `(setup)`, `(app)`) and throws `redirect()` otherwise. Components and layouts never redirect. New lifecycle state or route group → update the gate.
  - **Never draw a guess:** while state is unknown, show a neutral pending screen — never the sign-in page or the app layout.
  - **Keep it fresh:** mutations that change lifecycle state invalidate `viewer.state`; refetch it on window focus so changes on other devices are picked up.
  - **Not the security boundary:** the API still authorizes every call. The gate only decides which screen to draw.
  - **Mobile:** Expo keeps the splash screen up until `viewer.state` answers, then routes.
  - **Tests:** one Playwright test per state × route group, loading the page fresh and asserting the *first* rendered screen.
  - **Verify per project (unproven):** a redirect thrown from a `data-only` gate on the first request becomes an HTTP redirect before any HTML, and the auth provider's server-side session check (e.g. Clerk) works inside the gate.
- **Data:** all app data goes through the oRPC contract (§3.2) via `@orpc/tanstack-query`: loaders `ensureQueryData(orpc.x.queryOptions(...))`, components `useSuspenseQuery`, writes `useMutation` + invalidation. Gate and critical loaders call the same procedures on the server for the first load. No `createServerFn` for app data — only web-only chores (e.g. session cookies). URL state lives in validated search params.
- **Forms:** TanStack Form with the same Zod schema as the contract procedure.
- **Live data:** TanStack Query plus Durable Object / Cloudflare Agents SDK WebSocket pushes that invalidate or update queries. TanStack DB once it reaches 1.0.
- **Effect TS — not the default (decision open):** Effect 4 shipped 2026-10-01; models mostly know v3, and RPC/HTTP/SQL/atoms are in `effect/unstable/*`. Next new project trials it on the backend only (Effect 4 pinned, Effect repo vendored for agents, Effect kept behind Promise boundaries so web/mobile stay plain TypeScript). Decide after the trial; revisit full-stack Effect when RPC/atoms are stable.

### 3.2 API boundary — built for more clients later

Every app starts as landing page + web app; mobile, partners, CLIs and agents come later. The boundary is the **contract**, not the deployment.

- **oRPC, contract-first.** `packages/contract` defines every procedure (input/output Zod schemas, typed errors). Clients import only the contract, never server code. The same router serves typed RPC (`/rpc`) for first-party TypeScript clients and OpenAPI REST (`/api/v1`, spec generated) for everything else.
- **Day one: one Worker.** The web app's Worker hosts the oRPC handler plus plain Hono/server routes for webhooks. When a second client arrives, move the handler to `apps/api` — contract and clients don't change.
- **Thin handlers.** A handler does auth, validation (from the contract) and calls `packages/core`. All business logic lives in `core`; never in components, routes or handlers.
- **One API for all clients.** Web gets no private shortcuts mobile lacks (no server functions for app data).
- **Contract changes are additive** (new fields/procedures; no renames or removals without a deprecation window) — shipped mobile apps lag behind deploys, same discipline as DB migrations.
- **Agents:** point them at `https://orpc.dev/llms.txt`; pin the oRPC version and upgrade deliberately (Renovate).
- **Why oRPC over tRPC:** contract-first (clients decoupled from server code) and OpenAPI built in. **Risk:** single maintainer (verified 2026-10-12: 1.x since 2025, ~1.8M weekly downloads). **Exit plan:** the contract is plain Zod and logic lives in `core`, so if oRPC is abandoned, serve the same contract with Hono OpenAPI routes and generate clients from the spec; only handler glue changes.

## 4. Secrets and config

- **Dev keys → Infisical `dev-vault`** (env `dev`), shared by all apps. Dev keys must be *readable* by several consumers (agent sessions, PR CI, IaC creating per-PR provider resources), so they need one readable source.
- **Prod keys → the production host's own write-only secret store**, never Infisical. On Cloudflare: Worker secrets on each app's Worker in the prod account, set by the human (dashboard or `wrangler secret put`). Only the running app reads them; values can't be read back via API or dashboard, and each app's secrets are bound to its own Worker. Rotation: reissue at the provider, paste into Cloudflare. Other hosts: Vercel env vars / AWS Secrets Manager.
- **What goes in `dev-vault`:** only values a human copied from a provider dashboard.
  - Account-wide keys → global names: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `RETELL_API_KEY`, `OPENROUTER_API_KEY`.
  - Per-app keys → `<APP>_<PROVIDER>_<KEY>`, e.g. `FIRSTWORD_CLERK_SECRET_KEY`.
- **What does not:** anything IaC creates (database branches, webhook URLs, agent IDs, generated tokens, R2 S3 credentials) and non-secret config (publishable keys, allowed origins, endpoints), which lives in code.
- Each repo commits `secrets.json` mapping env var → vault name, e.g. `{ "CLERK_SECRET_KEY": "FIRSTWORD_CLERK_SECRET_KEY" }`.
- Alchemy reads dev secrets from Infisical at deploy time for PR stages. For `prod` it must not manage (and must not delete) the manually set Worker secrets **(unproven — verify per project)**.
- **Machine identities:**
  - `coding-agent` — Universal Auth, Viewer on `dev-vault` only. Its client ID/secret are the only credentials stored in the Claude environment.
  - `github-ci-preview` — GitHub OIDC (no stored secret), subject `repo:<owner>/*:pull_request` (one identity for all your repos), Viewer on `dev-vault`.
  - No prod identity: CI never reads production secrets. Agent vs CI are separate identities so GitHub stores no Infisical secret and each can be revoked independently.
  - `coding-agent` reads every app's dev keys — acceptable only because `dev-vault` holds test/sandbox keys with spending caps. Per-app scoping needs paid custom roles or more identities than the free plan allows.
- **Cloud accounts are split:** a dev account (all PR stages; the agent has full access for provisioning and debugging) and a prod account. For Cloudflare: two accounts under one login.
- **Prod deploys:** CI on `main` uses a deploy token for the prod account, stored as GitHub repo secrets `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID` (repository secrets, not environment secrets). Cloudflare account tokens can't yet be scoped to a single Worker ("com.cloudflare.edge.worker.script is not a supported resource type"), so the token reaches every Worker in the prod account. Accepted while solo; revisit when per-Worker scoping works or an app needs its own account.
- **Deploy pipeline: GitHub Actions** (workflow in the repo: migrations → deploy → smoke test → rollback; logs visible to agents on the PR/commit). Cloudflare **Workers Builds** (Git integration) is fine for simple apps without a database, but its auto-created token is broad and lacks D1, its commands live in the dashboard, and its branch previews would land in the prod account.
- **The real prod boundary is who can deploy:** deployed code can read its own secrets. Protect `main` with required PR review (GitHub Pro for private repos; environment approval gates aren't available for private repos below Enterprise). Agents only open PRs; the human approves merges. Plus: MFA on GitHub, Cloudflare, Infisical; pin Actions to SHAs; commit lockfiles; Renovate with a minimum release age.

## 5. Environments

| Stage | Created by | Lifetime | Data |
|---|---|---|---|
| `pr-<n>` | CI on every push to a PR (draft or ready) | Deleted on PR close | Neon branch of seed data |
| `prod` | CI on merge to main | Permanent | Production |

- **The agent opens a draft PR as soon as it starts work** and pushes to it; that PR's stage is its working environment. One agent session = one PR.
- **PR stages live in the dev cloud account; `prod` lives in the prod account.** CI deploys both. The agent provisions through code (edit `alchemy.run.ts`, push) and may use its dev-account token directly to debug (tail logs, query a PR database, clean up). It has no access to the prod account.
- Each stage gets its own Worker URL (`*.workers.dev` on the dev account), database, and provider resources (webhook endpoints, voice agents, products…) pointing at that stage. Previews use dev/test credentials.
- CI comments the preview URL (and mobile preview link) on the PR and posts test results there.

## 6. The dev → QA → prod loop

Climb only as far as the change needs.

1. **Local (seconds):** typecheck, lint, unit tests, app running locally, provider emulators (WorkOS emulator, Mailpit), replayed signed webhooks, Expo web build in headless Chromium.
2. **PR environment (push → ~1–3 min):** CI deploys `pr-<n>` and runs Playwright against it with real providers and real webhooks; the agent reads the results from the PR.
3. **Special lanes (CI jobs on the PR):** real-time media, devices, Maestro mobile flows — see §7.
4. **Ready for review:** the agent marks the PR ready when everything is green; the human can open the preview URL.
5. **Prod (on merge):** run migrations (must be backward compatible — never drop/rename in the same deploy), deploy, run post-deploy smoke test, roll back automatically on failure.

## 7. Test lanes — where each kind of test runs

| Lane | Needs | Runs on |
|---|---|---|
| Outbound API calls | HTTPS out | Anywhere, including the agent's cloud container |
| Webhooks / callbacks | A public URL | A deployed stage (no tunnels — they don't work from cloud agent containers) |
| Identity, email, payments | No human | Emulators / test modes: WorkOS emulator, Mailpit or Resend receiving, Polar sandbox (card 4242…) |
| Real-time media (WebRTC voice/video) | UDP + fake mic/camera | GitHub-hosted runner; Chromium fake-media flags or a server-SDK bot publishing a pre-recorded `.wav`/`.y4m` **(unproven)** |
| Your own media server (needs inbound UDP) | Open UDP | Fly Machines / Hetzner |
| iOS | macOS | Self-hosted runner on the Mac **(unproven)**, or EAS Workflows |
| Android | Emulator | GitHub-hosted Linux runner or EAS Workflows |

Agent cloud containers: HTTPS only through a TLS-inspecting proxy — no inbound connections, no UDP.

## 8. Adding any provider

Before integrating a provider, answer and record in the project's `AGENTS.md`:

1. **Secrets:** account-wide or per-app? Add to the vault and `secrets.json`.
2. **Per-environment resources:** what must exist per stage (webhook endpoint, agent, product)? Create it in `alchemy.run.ts` (write a custom resource if no provider exists).
3. **Test path:** emulator → sandbox → real. Use the cheapest that proves the feature.
4. **Fixtures:** test users, sample payloads, scripted media.
5. **Lane:** which row of §7.
6. **Cost and cap:** cost per test run; set a hard cap if offered, otherwise an alert.
7. **Cleanup:** what gets deleted when a stage is destroyed.

## 9. Guardrails

- **Hard caps available:** OpenRouter (per key), Sentry, GitHub Actions, Vercel, AWS. Set them via IaC.
- **Alerts only:** Cloudflare (delayed ~1 day), Neon, Expo EAS, Retell. Route alerts to a kill-switch Worker that disables the offending key or stage.
- PR stages are deleted on PR close; a weekly job deletes stages of PRs left open and inactive for 14 days.
- Backups: D1 Time Travel (30 days on paid), Neon PITR (6 hours on free — upgrade before launch), copy important R2 objects elsewhere.

## 10. Launch minimums

- Privacy policy linked in the app and store listings.
- In-app account deletion (Apple) plus a web deletion page (Google Play).
- DPAs with providers that process user data; prefer zero-data-retention for AI calls.
- Sentry + uptime monitoring + paging on; spending caps set.
- App stores: digital subscriptions — US users may be linked to web checkout (Apple: no commission today; Google: 10% on subscriptions within 24h of the link). Elsewhere offer in-app purchase (RevenueCat).

## 11. Starting a new app — checklist for the agent

1. Create the monorepo from §3; add `AGENTS.md` linking here.
2. Write `alchemy.run.ts` with stages per §5; pin the Alchemy version.
3. List needed secrets in `secrets.json`; ask the human to add any missing vault values (never paste them in chat).
4. Add providers per §8.
5. Add CI workflows: preview (create stage + comment URL), destroy on close, prod deploy with migration + smoke test + rollback, special lanes as needed.
6. Set spending caps and alerts per §9.
7. Prove the loop: open a draft PR, see CI deploy its stage and pass one end-to-end test, merge, verify prod.

## 12. One-time human setup (shared by all apps)

- **Accounts:** Cloudflare, Neon, Infisical, WorkOS, Polar, Resend, OpenRouter, Sentry, Better Stack, PagerDuty, PostHog, Expo, GitHub. Start on free plans; upgrade when a quota bites (Cloudflare free = 10 D1 databases per account).
- **Cloudflare — two accounts under one login** (account switcher → Create Account):
  - **Prod account:** per app, an account API token named `<app>-prod-deploy`: Entire account → Workers: Editor (Admin only if CI must create new Workers), D1: Write, Workers AI: Read, Account Settings: Read; plus a zone policy (change the policy scope from "Entire Account" to the app's zone) → Workers Routes: Write. → GitHub repo secrets `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID`. App secrets go on the Worker as Worker secrets.
  - **Dev account:** enable a `workers.dev` subdomain; create an account API token (Start from scratch, Entire account) with Workers: Admin (Editor can't create/delete Workers), D1, Workers AI, Workers KV Storage, Workers R2 Storage, Queues, Hyperdrive, Secrets Store, AI Gateway, Vectorize (Read + Write each), Account Settings: Read. Not "Workers Scripts (Legacy)", not "Full account access" (it can create more tokens and invite members). No IP filtering. → `dev-vault` as `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID`.
- **Infisical** (US cloud): project `dev-vault`; identities per §4.
  - `coding-agent` can be created inside `dev-vault` (Create New → Viewer). Then open it → Universal Auth → copy the **Client ID** and create a **Client Secret** (shown once). Leave trusted IPs open.
  - `github-ci-*` identities: create inside their vault (Create New → Viewer), no secret. Add Auth Method → OIDC Auth: discovery URL and issuer `https://token.actions.githubusercontent.com`, CA cert blank, subject per §4, audience `https://github.com/<owner>`, claims blank. Remove Universal Auth.
  - Repos created after 2026-07-15 use an immutable subject format with numeric IDs (`repo:<owner>@<owner_id>/<repo>@<repo_id>:...`); add `repo:<owner>@<owner_id>/*:...` when the first such repo appears. Never use `repo:<owner>*` (matches other users' names).
- **Claude cloud environment** (environment menu → Edit):
  - Environment variables (not Network secrets — Infisical login sends the secret in the body): `INFISICAL_CLIENT_ID`, `INFISICAL_CLIENT_SECRET`.
  - Allowed domains: `app.infisical.com`, `api.cloudflare.com`, plus each provider's API host.
  - The Infisical CLI (`npm i -g @infisical/cli`) needs `SSL_CERT_FILE=/root/.ccr/ca-bundle.crt` behind the session proxy.
  - Changes apply to **new** sessions only.
- **Mac:** GitHub self-hosted runner (iOS lane).
