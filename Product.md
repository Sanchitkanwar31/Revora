# wacrm — Technical Product Analysis

> Analysis of the codebase as it stands in this working copy, written for an
> engineer who needs to run, operate, or extend it. Version at time of
> writing: `0.8.0` (package.json). Not an official project doc — see
> `README.md` / `docs/` / wacrm.tech for the maintained docs.

---

## 1. What this is

**wacrm** is a self-hostable CRM built specifically around the **official
WhatsApp Business Cloud API** (Meta). It is distributed as a fork-and-deploy
**template**, not a hosted SaaS product — every install is its own Next.js
app talking to its own Supabase project. There is no central wacrm server;
"the product" is the codebase itself plus whatever Supabase/Meta/Hostinger
resources the operator provisions.

Core domains implemented:

- **Shared inbox** — one WhatsApp number, multiple human agents, per-
  conversation assignment/status.
- **Contacts** — with tags, custom fields, CSV import, phone-based
  deduplication.
- **Sales pipelines** — Kanban-style deals linked to conversations.
- **Broadcasts** — Meta-approved template campaigns with per-recipient
  variable substitution and delivery/read tracking.
- **Automations** — flat trigger → linear/branching step list (webhooks,
  tags, keyword matches, waits).
- **Flows** — a visual node-graph conversational bot builder (richer than
  Automations; suspends on user input, resumes on reply).
- **AI reply assistant** — bring-your-own OpenAI/Anthropic key; draft
  replies, optional autonomous auto-reply with human handoff, and a
  knowledge-base-backed RAG mode.
- **Public REST API** (`/api/v1`) and an **MCP server** (`wacrm-mcp` on
  npm) that wraps it for AI assistants (Claude, Cursor, etc.).
- **Multi-user accounts** with 4-tier RBAC (owner/admin/agent/viewer).
- **Outbound webhooks** for third-party integration.

---

## 2. Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Framework | **Next.js 16** (App Router), `output: "standalone"` | ⚠️ Per `AGENTS.md`, this Next.js major has breaking changes vs. the version most training data assumes — check `node_modules/next/dist/docs/` before writing new Next-specific code. |
| UI runtime | **React 19.2**, TypeScript (`typescript@^6`, strict) | |
| Styling | **Tailwind CSS v4** + shadcn/ui (`components.json`, style `base-nova`) | `class-variance-authority`, `tailwind-merge`, `tw-animate-css` |
| Data / Auth / Storage | **Supabase** (Postgres + Auth + Storage + Realtime), via `@supabase/ssr` + `@supabase/supabase-js` | Row-Level Security is the primary authorization mechanism, not just a defense-in-depth layer |
| WhatsApp | Meta Cloud API (official WhatsApp Business Platform), called server-side only | No third-party WA wrapper (Twilio/360dialog etc.) — direct `graph.facebook.com` calls |
| AI | OpenAI + Anthropic HTTP APIs, bring-your-own-key per account | `src/lib/ai/providers/{openai,anthropic}.ts` behind a shared interface |
| Flow diagram UI | `@xyflow/react` (React Flow) + `@dagrejs/dagre` (auto-layout) | powers the Flows visual builder |
| Drag & drop | `@dnd-kit/*` | Kanban pipelines, list reordering |
| Charts | `recharts` | dashboard analytics |
| i18n | `next-intl` | `en` and `ko` locale files under `messages/` |
| Testing | **Vitest** (`vitest.config.ts`, node environment) | unit/integration tests colocated as `*.test.ts` next to source |
| Lint/format | ESLint 9 flat config + Prettier (with Tailwind class sorting) | |
| Containerization | Multi-stage `Dockerfile` (Alpine, non-root `nextjs` user) + `docker-compose.yml` | |
| Package manager | npm (`packageManager: npm@10.9.9`), Node `>=20` | |

Notably **absent**: no ORM (raw Supabase client + hand-written SQL
migrations), no separate backend service (all server logic lives in Next.js
Route Handlers/Server Actions), no job queue/worker process (background work
is either `after()` in a webhook handler, or an externally-pinged cron GET
endpoint).

---

## 3. Repository layout

```
src/
  app/
    (auth)/            login, signup, forgot-password — public routes
    (dashboard)/        agents, automations, broadcasts, contacts, dashboard,
                         flows, inbox, notifications, pipelines, settings
    api/
      account/          invitations, members, api-keys, ownership transfer
      ai/                autoreply, config, draft, knowledge, playground, test, usage
      automations/       CRUD + engine + cron
      contacts/
      flows/             CRUD + templates + cron
      invitations/[token]
      quick-replies/
      v1/                PUBLIC REST API — messages, contacts, conversations, broadcasts, webhooks, me
      whatsapp/          broadcast, config, media, react, send, templates, webhook
    join/[token]/       invite acceptance flow
  components/           UI (shadcn-based)
  hooks/                use-auth, use-realtime, use-presence, use-can, ...
  i18n/                 next-intl request config
  lib/                  ALL business logic (see §5) — one folder per domain,
                         heavily unit-tested (`*.test.ts` siblings)
  middleware.ts          Supabase session refresh + route protection
  types/                 shared TS types (flows/automations config shapes, etc.)
supabase/migrations/    36 sequential, idempotent .sql files — the schema of record
mcp-server/             separate npm package (`wacrm-mcp`), thin wrapper over /api/v1
docs/                   docker.md, mcp.md, public-api.md (this repo's own docs;
                         the full self-host guide lives in the sister `wacrm-site` repo)
messages/               en.json, ko.json (next-intl)
```

`src/lib/` is where the real engineering is. Each subfolder is a bounded
context with its own tests:

`account, ai, api, api-keys, auth, automations, broadcast-status, contacts,
conversations, currency, dashboard, flows, inbox, media, presence,
rate-limit, storage, supabase, template-status, themes, utils, webhooks,
whatsapp`.

---

## 4. Architecture & request flow

### 4.1 Multi-tenancy model ("accounts")

Originally single-tenant-per-`auth.users` row; migration `017_account_sharing.sql`
introduced an **account layer** on top:

- `accounts` table — one row per tenant. `owner_user_id` is denormalized for
  fast owner checks; a unique index currently enforces **one account per
  owner** (single-membership design, documented as a locked decision).
- Every domain table gained an `account_id` column, which is now the
  tenancy/isolation key (RLS policies check account membership, not
  `user_id`, going forward). The pre-existing `user_id` columns are kept but
  now mean "the agent who owns/created this row" (audit/assignment), **not**
  tenancy.
- `profiles.account_id` + `profiles.account_role` (enum:
  `owner|admin|agent|viewer`) is how a signed-in user's membership and role
  are resolved.
- A `SECURITY DEFINER` SQL helper `is_account_member(account_id, min_role)`
  is the single predicate both RLS policies and (mirrored) TypeScript role
  helpers (`src/lib/auth/roles.ts`) rely on — kept intentionally symmetric so
  policy changes don't drift between the two layers.
- New signups get a fresh personal account + `owner` role atomically via the
  `handle_new_user` trigger on `auth.users`.
- Invitations (`account_invitations`) store only a SHA-256 hash of the invite
  token; the plaintext is shown exactly once at creation.

RBAC capability predicates (`src/lib/auth/roles.ts`) are the single source of
truth for "can this role do X" and are meant to be called from **both** UI
gates and API route guards:

- `canManageMembers` / `canEditSettings` → admin+
- `canSendMessages` → agent+ (viewers are read-only everywhere)
- `canDeleteAccount` / `canTransferOwnership` → owner only

### 4.2 Auth flow

- Supabase Auth (email/password) issues a JWT stored in cookies via
  `@supabase/ssr`.
- `src/middleware.ts` runs on (almost) every request:
  - Calls `supabase.auth.getUser()`, which **transparently refreshes**
    expired access tokens (rotating the refresh token in the process). The
    middleware carefully copies the refreshed `Set-Cookie` headers onto
    *whatever* response it ultimately returns (redirect or JSON) — a subtle
    but load-bearing fix (referenced as issue #288) because naively
    returning a fresh `NextResponse` would silently drop the rotated
    cookies and eventually wedge the session.
  - Redirects signed-in users away from `/login`, `/signup`,
    `/forgot-password` (honoring a pending `?invite=` token by routing to
    `/join/<token>` instead of `/dashboard`).
  - Redirects signed-out users away from protected paths
    (`/dashboard`, `/inbox`, `/contacts`, `/pipelines`, `/broadcasts`,
    `/automations`, `/settings`) to `/login`.
  - Returns 401 JSON for unauthenticated `/api/whatsapp/*` calls (webhook
    excluded — that path authenticates via HMAC signature instead, see §6).
- Server Components / Route Handlers get a per-request Supabase client via
  `src/lib/supabase/server.ts` (cookie-bound, RLS-respecting, i.e. acts *as*
  the signed-in user).
- Server-only privileged code (webhook processing, automation/flow engines,
  the public API's key-auth path) uses a **service-role** admin client
  (`SUPABASE_SERVICE_ROLE_KEY`) which **bypasses RLS** — every one of those
  code paths is therefore responsible for doing its own tenancy checks in
  application code (e.g. `runAutomationsForTrigger` explicitly re-verifies a
  caller-supplied `contactId` belongs to the claimed `accountId` before
  touching it, precisely because the service-role client won't stop it).

### 4.3 Public API auth (separate from user auth)

`/api/v1/*` authenticates via **API keys**, not cookies:
`Authorization: Bearer wacrm_live_...`. Keys are SHA-256 hashed at rest
(`src/lib/api-keys/`), scoped (`messages:send`, `contacts:read`,
`contacts:write`, `conversations:read`, `broadcasts:send`,
`webhooks:manage`), account-bound, and rate-limited per key (120 req/min).
Only admins/owners can mint keys. This is the same surface the **MCP
server** (`mcp-server/`) wraps — it is a thin proxy, not a second
implementation, so every safety property (scopes, rate limits) is enforced
once, server-side.

### 4.4 Inbound WhatsApp message pipeline

`POST /api/whatsapp/webhook` (`src/app/api/whatsapp/webhook/route.ts`) is the
single most architecturally interesting file in the repo:

1. Reads the **raw** request body (not `request.json()`, to preserve exact
   bytes) and verifies `X-Hub-Signature-256` via HMAC-SHA256 against
   `META_APP_SECRET` (`verifyMetaWebhookSignature`). Invalid signature → 401
   (intentionally not 200, so Meta's delivery dashboard surfaces
   misconfiguration loudly instead of silently swallowing events).
2. Returns `200` immediately, then does the actual work inside Next.js's
   `after()` callback — explicitly **not** a detached/floating promise,
   because on serverless runtimes (this app targets Vercel per the code
   comments) the function can be frozen the instant the response is sent,
   which previously caused partial writes (contact/conversation created,
   message row missing — referenced as issue #301).
3. Branches on webhook field type: template lifecycle events go to
   `handleTemplateWebhookChange`; message `status` updates go to
   `handleStatusUpdate`; actual inbound messages go to `processMessage`.
4. `handleStatusUpdate` enforces a **forward-only status ladder**
   (`pending → sent → delivered → read → replied`, with `failed` only valid
   as a terminal branch from `pending`/`sent`) so out-of-order or replayed
   Meta webhooks can't regress a message's displayed status. It mirrors
   status onto both `messages` and `broadcast_recipients` (aggregate
   triggers recompute broadcast counters), then fires the
   `message.status_updated` outbound webhook last, so a slow subscriber
   can't delay the DB writes.
5. `processMessage` does, in order: find-or-create contact (phone-dedup via
   `src/lib/contacts/dedupe.ts`, race-safe via unique-constraint-then-retry)
   → find-or-create conversation (oldest-row-wins under races, also
   unique-constraint-guarded) → emit `conversation.created` webhook if new →
   short-circuit for reactions (upsert `message_reactions`, never a
   `messages` row) → parse content by type (text/image/video/document/
   audio/sticker/location/interactive) → resolve swipe-reply parent →
   insert the `messages` row → bump conversation preview/unread → reopen a
   closed conversation if the customer wrote again (issue #409) → flag a
   matching broadcast recipient as `replied` → **dispatch to the Flows
   engine** → conditionally dispatch to the **Automations engine** (content
   triggers are suppressed if a Flow already consumed the message, but
   relationship triggers like `new_contact_created` still fire) →
   conditionally dispatch to the **AI auto-reply** (only for plain text,
   only if no Flow/interactive-reply consumed it) → emit the
   `message.received` outbound webhook.
6. Media (image/video/audio/document/sticker) is **not** stored by URL
   directly — the webhook verifies the media exists with Meta
   (`getMediaUrl`) and stores a proxy path (`/api/whatsapp/media/{id}`); the
   actual bytes are fetched on demand through that route, keeping the Meta
   access token server-side only.

### 4.5 Outbound sends

`src/lib/whatsapp/send-message.ts`, `meta-api.ts`, `broadcast-core.ts`,
`template-send-builder.ts` etc. wrap the Graph API for text/media/template/
interactive (buttons/list) sends, template CRUD (including the
resumable-upload flow Meta requires for image-header templates —
`template-header-handle.ts`, needs `META_APP_ID`), and template
status/lifecycle normalization. All Meta HTTP calls are server-side only —
the CSP's `connect-src` deliberately does **not** include
`graph.facebook.com`.

### 4.6 Automations vs. Flows — two different engines, by design

- **Automations** (`src/lib/automations/engine.ts`): trigger-based
  (`new_contact_created`, `first_inbound_message`, `new_message_received`,
  `keyword_match`, `interactive_reply`, tag events, conversation-assigned,
  schedule) firing a **flat/branching step list**: send message/template/
  buttons/list, tag/untag, update contact field, create deal, assign
  conversation, call a webhook, or **wait** (suspends and resumes via an
  external cron ping — `GET /api/automations/cron`, gated by
  `AUTOMATION_CRON_SECRET`, 503 until configured). Runs are logged to
  `automation_logs` per-step for auditability. Never throws to its caller —
  failures are caught and recorded, since dispatch is fire-and-forget from
  the webhook.
- **Flows** (`src/lib/flows/engine.ts`): a **visual node-graph** bot builder
  (edited with `@xyflow/react`) for actual back-and-forth conversations —
  send message/media/buttons/list, collect input, branch on conditions, set
  tags, and **suspend at nodes that need customer input**, resuming on the
  next tap/reply. Concurrency-safety is handled at the DB layer:
  idempotency keyed on `meta_message_id` (refuses to double-advance a run
  for the same inbound message, protecting against Meta retries),
  optimistic `UPDATE ... WHERE current_node_key = ...` preconditions for
  concurrent taps, and a partial unique index
  (`idx_one_active_run_per_contact`) so two simultaneous "start" events for
  the same contact can't create two active runs. Flows also have their own
  cron endpoint (`GET /api/flows/cron`) for any wait-style nodes.
- When both could apply to one inbound message, **Flows win**: if the flow
  runner "consumes" the message, the webhook suppresses the *content-level*
  automation triggers (`new_message_received`, `keyword_match`,
  `interactive_reply`) for that inbound so the customer isn't simultaneously
  routed through a menu and matched against a keyword automation.

### 4.7 AI reply assistant

- **Bring-your-own-key**: each account pastes its own OpenAI or Anthropic
  key under Settings → AI Assistant. The key is AES-256-GCM-encrypted at
  rest using the same `ENCRYPTION_KEY` as WhatsApp tokens. There is **no**
  global provider key / env var — the app makes zero AI calls unless an
  account configures one.
- `src/lib/ai/providers/{openai,anthropic}.ts` implement a shared interface
  (`shared.ts`: usage normalization, network/HTTP error mapping to a typed
  `AiError`, consecutive-same-role message merging for providers like
  Anthropic that require strict alternation).
- Two triggers into the assistant: **draft** (`POST /api/ai/draft`, agent
  clicks "Draft with AI" in the inbox) and **auto-reply**
  (`dispatchInboundToAiReply`, fully autonomous, only runs when no Flow/
  interactive-reply consumed the inbound and the account opted in).
- **Knowledge base / RAG** (migration `030_ai_knowledge.sql`): FAQs/policies/
  docs, chunked (`src/lib/ai/chunk.ts`) and retrieved via **hybrid search**
  — Postgres full-text search out of the box, upgraded to semantic
  (`pgvector`, `CREATE EXTENSION IF NOT EXISTS vector`) automatically once
  the account also sets an embeddings key. No separate vector DB required.
- **Auto-reply handoff** (`src/lib/ai/handoff.ts`): a per-conversation cap on
  autonomous replies, with a clean handoff to a human agent once exceeded —
  this bounds both cost and the "bot loop" failure mode.
- Config knobs: `AI_REQUEST_TIMEOUT_MS` (default 30000),
  `AI_CONTEXT_MESSAGE_LIMIT` (default 20, how many recent messages become
  model context).
- Rate limiting is layered per-user (`aiDraft`: 20/min) **and** per-account
  (`aiDraftAccount`: 60/min, `aiAutoReplyAccount`: 30/min) — the per-account
  cap exists specifically so N agents each under their individual limit
  can't collectively stampede the account's one shared BYO provider key past
  its own rate limit.

---

## 5. Data model (Supabase/Postgres)

Schema of record is `supabase/migrations/*.sql` — **36 sequential,
hand-written, idempotent migrations** (`IF NOT EXISTS` for tables/indexes,
`DROP POLICY IF EXISTS` + recreate for policies, since Postgres has no
`CREATE POLICY IF NOT EXISTS`). There is no migration framework/ORM (no
Prisma/Drizzle) — these are applied directly via the Supabase CLI or SQL
editor.

Key tables (initial schema, `001_initial_schema.sql`, later extended by
`account_id` in `017`):

- `profiles` (1:1 with `auth.users`, auto-created via trigger)
- `contacts`, `tags`, `contact_tags`, `custom_fields`,
  `contact_custom_values`, `contact_notes`
- `conversations`, `messages` (+ `message_reactions`, `interactive_reply_id`,
  `reply_to_message_id` added later), `whatsapp_config` (encrypted access
  token + verify token per account)
- `message_templates` (Meta template lifecycle mirror)
- `pipelines`, `pipeline_stages`, `deals`
- `broadcasts`, `broadcast_recipients` (status aggregated up to the parent
  via trigger)
- Later additions by migration: `automations` + `automation_logs` (006–007),
  `flows` + flow nodes/edges/runs (010, 012, 016), `accounts` +
  `account_invitations` + membership RPCs (017–020), `api_keys` (026),
  `notifications` (027), `webhook_endpoints` (028), `ai_reply_config` +
  usage tracking (029, 031, 033), `ai_knowledge` (+pgvector) (030, 032),
  `interactive_messages` support (035), conversation/contact dedup indexes
  (022, 036).

Authorization pattern used everywhere: **every table has RLS enabled**;
policies are either simple owner checks (`auth.uid() = user_id`, legacy
pre-017 pattern) or, post-`017`, membership + role checks via the
`is_account_member(account_id, min_role)` `SECURITY DEFINER` helper. Foreign
keys mostly `ON DELETE CASCADE`, with one deliberate exception
(`004_contact_delete_set_null.sql` — presumably preserves message history
when a contact is deleted; not fully inspected).

Realtime: `messages` and `conversations` are added to the
`supabase_realtime` publication so the inbox UI can subscribe to live
updates (`src/hooks/use-realtime.ts`) without polling.

---

## 6. Security posture

The README markets "real security primitives" and the code backs it up:

- **Token encryption**: AES-256-GCM (`src/lib/whatsapp/encryption.ts`) for
  WhatsApp access tokens, verify tokens, and AI provider keys, keyed by
  `ENCRYPTION_KEY` (32 random bytes, hex-encoded). Ciphertext format is
  `iv:ciphertext:authTag`; a **legacy CBC format** (`iv:ciphertext`, no
  auth tag) is still *decryptable* for backward compatibility but never
  produced by new writes — call sites opportunistically re-encrypt legacy
  rows to GCM on read (see the webhook GET-verification handler). Rotating
  `ENCRYPTION_KEY` orphans every previously-encrypted token; users must
  re-save WhatsApp settings.
- **Webhook authenticity**: inbound Meta webhooks are HMAC-SHA256 verified
  over the raw body against `META_APP_SECRET`; outbound wacrm webhooks
  (`/api/v1/webhooks`) are themselves HMAC-signed
  (`t=<unix>,v1=<hex>` over `${t}.${rawBody}`) so subscribers can verify
  wacrm's identity too.
- **SSRF protection**: outbound webhook targets (`src/lib/webhooks/ssrf.ts`)
  must be `https://` and resolve to a public address — localhost, RFC1918,
  link-local, and cloud metadata (`169.254.169.254`) are refused at
  delivery time.
- **RLS everywhere** (see §5) as the primary tenant-isolation mechanism, not
  just app-layer checks — meaningful because a bug in a Route Handler still
  can't leak cross-tenant data through the anon-key client.
- **API keys**: stored as SHA-256 hashes only, shown once, scoped, per-key
  rate limited, revocable (revocation effective on next request, audit trail
  retained).
- **Invitation tokens**: stored as SHA-256 hashes only, plaintext shown once.
- **CSP**: currently shipped as `Content-Security-Policy-Report-Only` (not
  enforced yet — explicitly staged that way in `next.config.ts` pending a
  clean bill of health across routes), alongside enforced HSTS,
  `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
  `Referrer-Policy: strict-origin-when-cross-origin`, and a
  `Permissions-Policy` that denies camera/geolocation/payment/usb (mic
  allowed same-origin only, for the inbox voice-note recorder).
- **Rate limiting** (`src/lib/rate-limit.ts`): in-memory, fixed-window,
  per-process. Explicitly documented as **not multi-instance safe** — a
  horizontally scaled deploy silently loses the shared limit; the code
  points at swapping in Redis/Upstash behind the same interface if that
  matters to an operator. Pre-tuned buckets exist per concern (message send,
  broadcast dispatch, reactions, invitation peek/redeem, admin actions,
  public API, AI draft/auto-reply — both per-user and per-account variants).
- **Invite-link host allow-listing** (`ALLOWED_INVITE_HOSTS`, optional):
  defense-in-depth against a spoofed `Host`/`X-Forwarded-Host` header being
  reflected into an invite URL on a bare (non-proxied) deployment.
- **Middleware session handling**: see §4.2 — the refreshed-cookie-forwarding
  fix is a real, previously-shipped bug (#288), not a hypothetical.

---

## 7. Setup requirements

### 7.1 Required accounts/services

1. **Supabase project** (Postgres + Auth + Storage). Free tier is enough to
   start. `pgvector` extension is available out of the box for the optional
   semantic knowledge-base search.
2. **Meta for Developers app** with the WhatsApp Business Platform product
   added — gives you `META_APP_SECRET` (and `META_APP_ID` if you'll submit
   templates with an image header), a phone number ID, and a WABA ID (the
   latter two are entered per-account in-app under WhatsApp settings, not as
   env vars).
3. A **publicly reachable HTTPS URL** for the webhook — WhatsApp will not
   deliver to `http://` or an unreachable host, so local dev needs a tunnel
   (ngrok / Cloudflare Tunnel / localtunnel — all three are pre-allow-listed
   in `next.config.ts`'s `allowedDevOrigins` for Next 16's dev-origin CSRF
   guard).
4. Optional: an **OpenAI and/or Anthropic API key** (per-account, entered in
   the UI, not env) if you want the AI assistant.

### 7.2 Environment variables (`.env.local`, from `.env.local.example`)

**Required — app won't start / core features fail without these:**

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon (public) key — client + RLS-respecting server calls |
| `SUPABASE_SERVICE_ROLE_KEY` | Bypasses RLS; used only server-side (webhook, automation/flow engines, public-API key auth). Never expose to the client. |
| `ENCRYPTION_KEY` | 64 hex chars (32 bytes) for AES-256-GCM. Generate: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `META_APP_SECRET` | Verifies inbound webhook HMAC signatures; without it every webhook POST is rejected |

**Recommended:**

| Var | Purpose |
|---|---|
| `NEXT_PUBLIC_SITE_URL` | Canonical origin for sitemap/OG/invite links when the request-derived origin would be wrong (e.g. cron jobs) |
| `NEXT_PUBLIC_APP_LOCALE` | Default locale (`en`) |

**Optional:**

| Var | Purpose |
|---|---|
| `ALLOWED_INVITE_HOSTS` | Comma-separated hostname allow-list for invite URLs on bare (non-proxied) deployments |
| `AUTOMATION_CRON_SECRET` | Required if using Automation "Wait" steps — protects `GET /api/automations/cron` |
| `META_APP_ID` | Required only for template image-header submission (Resumable Upload is app-scoped) |
| `WHATSAPP_TEMPLATES_DRY_RUN` | `true` to stub out real Meta calls when submitting templates — useful in CI/local dev without a real WABA |
| `AI_REQUEST_TIMEOUT_MS` | Per-call AI provider timeout, default 30000 |
| `AI_CONTEXT_MESSAGE_LIMIT` | Recent messages sent as AI context, default 20 |
| `ALLOWED_DEV_ORIGINS` | Extra hosts allowed for Next 16 dev-origin CSRF check (beyond the pre-baked ngrok/cloudflare/localtunnel wildcards) |

> This working copy's `.env.local` already has the four required keys set
> (values not inspected/echoed here for safety) — the app should be able to
> start locally as-is, contingent on those values being real/valid.

### 7.3 Database setup

No migration runner is wired into `npm run build`/`start` — migrations are
applied manually against the Supabase project (via Supabase CLI or the SQL
editor, running `supabase/migrations/*.sql` in numeric order). This is a
manual step an operator must not forget on first setup **and** after every
pull that adds a new migration file.

### 7.4 Local development

```bash
git clone <your fork>
cd wacrm
npm install
cp .env.local.example .env.local   # fill in Supabase + Meta creds
# apply supabase/migrations/*.sql to your Supabase project
npm run dev
```

Serves on `http://localhost:3000`; `/` → redirected to `/login` or
`/dashboard` per middleware. For the WhatsApp webhook to actually receive
anything locally, tunnel port 3000 and register the tunnel HTTPS URL as the
webhook callback URL in the Meta app.

### 7.5 Docker

```bash
cp .env.local.example .env.local
docker compose --env-file .env.local up --build -d
```

- `NEXT_PUBLIC_*` vars are **build args** (baked into the client bundle at
  image build time — changing them requires `--build`).
- Everything else is injected at **runtime** via `env_file` — changeable
  with just a restart.
- `HOST_PORT` (not `PORT`) controls the published host port; `PORT` is
  pinned to `3000` inside the container deliberately so it can't drift from
  the Compose port mapping / healthcheck.
- Supabase is **external** — no DB container is included; migrations still
  need to be applied out-of-band.
- Nothing is scheduled inside the container — if using Automation "Wait"
  steps or Flow wait-nodes, an **external pinger** must hit
  `GET /api/automations/cron` and `GET /api/flows/cron` with header
  `x-cron-secret: $AUTOMATION_CRON_SECRET`. Both endpoints return `503`
  until that secret is configured.

### 7.6 Recommended production target

The project is built/tested/documented against **Hostinger Managed Node.js**
hosting (one-click Git deploy, this repo's own `main` deploys that way) but
is plain Next.js underneath and runs anywhere Node 20+ does (Vercel,
Railway, a VPS + Docker). The `output: "standalone"` build target exists
specifically so the Docker image doesn't need `node_modules` or the Next CLI
at runtime.

### 7.7 CI

`.github/workflows/ci.yml` runs on every PR/push to `main`: install → lint
(ESLint) → typecheck (`tsc --noEmit`) → test (Vitest) → build (`next build`).
CI uses dummy env values (fake Supabase URL/anon key, an all-zero
`ENCRYPTION_KEY`, a placeholder `META_APP_SECRET`) — no real external
service is touched during CI. `vitest.config.ts` sets the same dummy
`ENCRYPTION_KEY`/`META_APP_SECRET` for the test environment so unit tests of
`encryption.ts`/`webhook-signature.ts` are deterministic.

---

## 8. MCP server (`mcp-server/`)

A **separate npm package**, published as `wacrm-mcp`, versioned/built
independently (own `package.json`, `tsconfig.json`). It is a thin MCP tool
wrapper around `/api/v1` — it does not talk to Supabase or Meta directly,
and inherits every safety property of the public API (scopes, per-key rate
limits) rather than re-implementing them.

Configured entirely via env vars passed by the MCP client:
`WACRM_BASE_URL`, `WACRM_API_KEY` (required); `WACRM_ENABLE_WRITES`,
`WACRM_ENABLE_BROADCASTS` (optional, opt-in — write/broadcast tools aren't
even *registered* with the model unless set, on top of the API key's own
scopes and an explicit `confirm: true` requirement for broadcasts). Logs to
stderr since stdout is the MCP transport.

---

## 9. Internationalization

`next-intl` with a plugin wired in `next.config.ts`
(`createNextIntlPlugin("./src/i18n/request.ts")`). Two locales ship today:
`messages/en.json`, `messages/ko.json`. `NEXT_PUBLIC_APP_LOCALE` sets the
default. There's a dedicated `src/i18n/messages.test.ts`, presumably
asserting key-parity between locale files (not opened in this pass).

---

## 10. Testing strategy

Vitest, `node` environment, tests colocated as `<module>.test.ts` next to
the module they cover (not a separate `__tests__` tree) — a large fraction
of `src/lib/**` has a matching test file, especially the trickiest logic:
phone dedup, webhook signature verification, status-ladder transitions
(indirectly, via `broadcast-status.test.ts`), flow edge/layout/fallback
logic, automation validation, encryption round-trips, template
builders/validators, SSRF guard, rate limiting, RBAC role ranking. No
end-to-end/browser test tooling (no Playwright/Cypress) is present in
`package.json` — testing is unit/integration-level against pure functions
and mocked/stubbed I/O boundaries (admin-client modules exist specifically
so engine logic can be tested without a live Supabase connection).

Run: `npm test` (single run) / `npm run test:watch`.

---

## 11. Notable engineering judgment calls worth knowing before touching this code

These are things the code comments themselves flag as non-obvious — worth
preserving here since they're easy to accidentally regress:

1. **`after()`, not a floating promise**, for webhook post-processing —
   serverless function freezing after response-send previously dropped
   partial writes (issue #301). Any new async work triggered from the
   webhook route must go through the same `after()` block, awaited.
2. **Cookie forwarding in middleware** — any new early-return branch in
   `middleware.ts` must go through `withRefreshedCookies(...)` or sessions
   will silently wedge after idle periods (issue #288).
3. **Status transitions are forward-only** on a fixed ladder — a naive
   "just overwrite with the latest webhook status" would let Meta's
   out-of-order/replayed delivery-status webhooks regress a message's
   displayed state.
4. **Find-or-create races** (contact, conversation) are handled by
   attempting the insert and catching the unique-constraint violation,
   not by locking — see `findOrCreateContact`/`findOrCreateConversation`.
   A prior implementation using `.single()` for existence checks caused a
   duplicate-conversation snowball under concurrent delivery retries
   (issue #363).
5. **Flows consume before Automations** — order matters when adding new
   trigger types; content-level automation triggers must stay suppressed
   when a Flow run consumed the inbound message, but relationship-level
   triggers (new contact / first message) must not be.
6. **Service-role client bypasses RLS** — every engine/route that uses
   `supabaseAdmin()` is personally responsible for tenancy checks that RLS
   would otherwise provide for free. New service-role code paths need an
   explicit "does this row belong to this account" check, not an assumption
   that the database will catch it.
7. **CSP is report-only, not enforced**, by deliberate staged rollout — don't
   assume it's blocking anything in production yet.
8. **Rate limiter is per-process, in-memory** — true only for a
   single-instance deploy; horizontally scaling silently and quietly
   defeats every limit in `RATE_LIMITS` unless swapped for a shared store.

---

## 12. Explicit non-goals / current limitations (from the docs, not inferred)

- Not designed for multi-account-per-user (one owned account per user is a
  locked design decision as of migration 017).
- Public API webhook delivery is **best-effort, single-attempt** — no
  retry-with-backoff queue yet; consumers must dedupe on delivery `id` and
  reconcile via the read endpoints for anything that matters.
- Public API has no deals/pipelines/templates/flows endpoints yet (messages,
  contacts, conversations, broadcasts, webhooks only).
- No built-in multi-instance rate limiting or job scheduling — both assume
  operators needing that scale will swap in external infrastructure
  (Redis-backed limiter, external cron for Automation/Flow waits) using the
  hooks the code already exposes.
- This is explicitly **not** intended to take upstream feature PRs — the
  project's stated contribution model is fork → customize → deploy.
