# Agent Stack — reference for building new apps

Point an agent here with: *"Build <app>. Use this reference."*
This file defines the default stack, how a project is set up, and how the dev → QA → prod loop works.
Deviate only when the app genuinely needs it, and say why in the PR.

Last verified: 2026-10-11. Items marked **(unproven)** have not been exercised end to end yet.

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
| Language | TypeScript everywhere | |
| Web | TanStack Start | |
| API | Hono on Cloudflare Workers | One API for web + mobile; authorization lives here (no RLS) |
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
  web/            TanStack Start
  api/            Hono Worker
  mobile/         Expo app
packages/
  db/             Drizzle schema + migrations
  shared/         types, validation (zod), API client
alchemy.run.ts    all infrastructure, per stage
secrets.json      vault names this project uses (no values)
.env.example      non-secret config keys
AGENTS.md         project-specific notes; links back to this reference
```

## 4. Secrets and config

- **Infisical**, two projects shared by all apps:
  - `dev-vault` — readable by agent sessions (dev/test keys only).
  - `prod-vault` — readable only by CI (GitHub OIDC). Never by agent sessions.
- **What goes in the vault:** only values a human copied from a provider dashboard.
  - Account-wide keys → global names: `OPENROUTER_API_KEY`, `RETELL_API_KEY`.
  - Per-app keys → prefixed: `WORKOS_<APP>_API_KEY`.
- **What does not:** anything IaC creates (database branches, webhook URLs, agent IDs, generated tokens) and non-secret config (publishable keys, allowed origins), which lives in code.
- Each repo commits `secrets.json` mapping env var → vault name, e.g. `{ "WORKOS_API_KEY": "WORKOS_MYAPP_API_KEY" }`.
- Alchemy reads secrets directly from Infisical at deploy time.

## 5. Environments

| Stage | Created by | Lifetime | Data |
|---|---|---|---|
| `pr-<n>` | CI on every push to a PR (draft or ready) | Deleted on PR close | Neon branch of seed data |
| `prod` | CI on merge to main | Permanent | Production |

- **The agent opens a draft PR as soon as it starts work** and pushes to it; that PR's stage is its working environment. One agent session = one PR.
- **Only CI deploys.** Agent sessions never hold deploy credentials.
- Each stage gets its own Worker URL, database branch, and provider resources (webhook endpoints, voice agents, products…) pointing at that stage. Previews use dev/test credentials.
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

- Accounts: Cloudflare (Workers Paid), Neon, Infisical, WorkOS, Polar, Resend, OpenRouter, Sentry, Better Stack, PagerDuty, PostHog, Expo, GitHub.
- Infisical: `dev-vault`, `prod-vault`; machine identities `claude-agent` (Universal Auth, read `dev-vault`) and `github-ci` (OIDC, read both).
- Claude cloud environment: `INFISICAL_CLIENT_ID` / `INFISICAL_CLIENT_SECRET` env vars, network allowlist for provider hosts, setup script that installs deps and writes `.dev.vars` from Infisical.
- Mac: GitHub self-hosted runner (iOS lane).
