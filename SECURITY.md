# Security & Architecture Audit — WACRM

**Scope:** Read-only audit. No code, config, or dependencies were modified.
**Method:** Targeted code/config review across auth, tenant isolation, secrets, data flow, API surface, and third-party integrations. All findings below are evidence-based (file/line cited); no generic OWASP boilerplate is included without a matching code observation.
**Auditor stance:** Senior Software Architect / Security Engineer, read-only engagement.

---

## Executive Summary

This is a WhatsApp Business CRM (Next.js + Supabase Postgres), multi-tenant on an `account_id`-scoped shared database, with a public REST API (`/api/v1`), an MCP server for AI-agent access, WhatsApp webhook ingestion, a cron-driven automation/flow engine, and an opt-in bring-your-own-key AI assistant (OpenAI/Anthropic).

Overall, the application shows **above-average security engineering discipline** for its class: RLS-based tenant isolation with per-role policies, timing-safe HMAC webhook verification, SSRF-guarded outbound HTTP (DNS resolution + private-range blocking + no-redirect-follow), field-allowlisted writes (no mass assignment), AES-256-GCM encryption for stored tokens with correct IV/tag handling, and no telemetry/analytics SDKs phoning data home. Two real, high-severity multi-tenancy vulnerabilities were found in the migration history — **both already identified and patched** by the team (see Tenant Isolation section), referencing advisory `GHSA-fg5p-2qc3-jmxr`.

The most significant *live* issue found is operational, not a code defect: the local `.env.local` file holds a real `SUPABASE_SERVICE_ROLE_KEY`, `ENCRYPTION_KEY`, `META_APP_SECRET`, and a working WhatsApp access token/phone number in plaintext. It is correctly `.gitignore`d, but its mere existence on disk in a shared/exportable project folder is a standing exposure risk.

The one structural (architectural) weakness worth tracking is that the public API / MCP surface intentionally bypasses Postgres RLS (via the service-role client) and relies entirely on hand-written `account_id` filters in each route. Every route reviewed does this correctly today, but there is no database-level backstop if a future route forgets — unlike the dashboard/browser path, which is protected by RLS regardless of application-code correctness.

---

## Architecture / Data Flow

- **Tenancy model:** single shared Postgres database (Supabase); isolation enforced per-row via `account_id` and Row-Level Security policies (155+ `CREATE POLICY` statements across `supabase/migrations/`).
- **Two distinct trust paths into the data:**
  1. **Dashboard/browser path** — Supabase SSR client, user JWT, RLS-enforced on every query. DB-level isolation regardless of app-code bugs.
  2. **Public API / MCP / webhook / cron path** — uses a **service-role client that bypasses RLS**, scoping enforced only by application code appending `.eq('account_id', ctx.accountId)` on every query (`src/lib/auth/api-context.ts`). Correct in every route sampled, but structurally fragile — a missed filter in a future endpoint is a full cross-tenant leak with no DB backstop.
- **MCP server** (`mcp-server/`, published as `wacrm-mcp`) is a thin stateless proxy over `/api/v1` — it holds only the operator's own scoped API key, never a service-role credential, and cannot pivot outside that key's account/scopes.
- **Middleware** (`src/middleware.ts`) enforces session validity only for a fixed allowlist of dashboard page paths and `/api/whatsapp/*` (excluding the webhook); it does **not** gate most of `/api/*`. Every API route sampled independently enforces its own auth (`requireRole`, `requireApiKey`, `getCurrentAccount`), which held true everywhere reviewed — but this means the middleware provides no defense-in-depth backstop for API routes.

---

## Third-Party Data Sharing

| Service | Data sent | Trigger | Risk |
|---|---|---|---|
| **Supabase** (Postgres/Auth/Storage) | All CRM data: auth, contacts/PII, message content, media | Core infra (always) | High — single point of breach for everything |
| **Meta WhatsApp Graph API** | Message content, phone numbers, templates, media, long-lived access token | Core product function (always) | High — required, but a major blast-radius node |
| **OpenAI** (`api.openai.com`) | Full conversation transcript + system prompt (chat); KB document text (embeddings) | Opt-in, account pastes own key (AI draft/auto-reply/knowledge base) | Medium-High — customer PII leaves to a third party whenever this feature is enabled |
| **Anthropic** (`api.anthropic.com`) | Same as OpenAI, when selected as provider | Opt-in, BYOK | Medium-High |
| **Account-configured outbound webhooks** | Message text, `conversation_id`, `contact_id`, `whatsapp_message_id` per subscribed event | Opt-in, admin registers arbitrary HTTPS URL | Medium — SSRF-guarded and HMAC-signed, but any admin can route live message content to any external endpoint they control |
| **npm registry** (`wacrm-mcp`) | None directly | `npx -y wacrm-mcp` fetches latest each run | Low — supply-chain trust surface only |

No analytics/telemetry SDK (Sentry, PostHog, Mixpanel, Segment, GA) found in `package.json` or `mcp-server/package.json`; `NEXT_TELEMETRY_DISABLED=1` is explicitly set in the Dockerfile.

---

## Data Leakage Findings

```text
[MEDIUM] Customer message content sent to external LLM providers with no per-message consent or redaction
Status: CONFIRMED
File/Component: src/lib/ai/providers/openai.ts, src/lib/ai/providers/anthropic.ts, src/lib/ai/context.ts, src/app/api/ai/draft/route.ts
Evidence: aiContextMessageLimit() pulls the last N conversation messages (customer text included) and forwards them as `messages` to OpenAI/Anthropic chat completion endpoints. Knowledge-base text is similarly sent to OpenAI embeddings (src/lib/ai/embeddings.ts). Control is account-level opt-in only (enable feature + paste key) — no per-conversation consent or field-level scrubbing.
Risk: Customer PII (phone numbers, names, message content) is disclosed to a third-party AI provider outside the CRM's direct control, governed only by the account owner's choice to enable the feature.
Recommended Fix: Document this clearly in the product's privacy policy / DPA; consider optional PII redaction before sending context to the model.
Architectural Fix: None required — this is a legitimate opt-in feature; ensure it is clearly disclosed to end customers whose messages may be forwarded.
```

```text
[LOW] Account-admin-configured outbound webhooks can exfiltrate message content to any HTTPS endpoint
Status: CONFIRMED (by design)
File/Component: src/lib/webhooks/deliver.ts, src/app/api/whatsapp/webhook/route.ts (~line 835)
Evidence: On message events, a signed JSON payload including message text is POSTed to any URL an account admin registers in webhook_endpoints. SSRF-guarded (src/lib/webhooks/ssrf.ts) and HMAC-signed, so this is not a bug — but it is a data-egress channel controllable by any account admin.
Risk: A malicious or compromised admin account could silently exfiltrate all inbound customer messages to an external collector.
Recommended Fix: Surface registered webhook endpoints prominently in the UI/audit log so other account members can see where data is being sent.
Architectural Fix: None required.
```

No evidence was found of data leaking to unintended third parties (no rogue analytics/telemetry calls, no unexpected outbound hosts beyond Supabase/Meta/OpenAI/Anthropic/user-configured webhooks).

---

## Secret Exposure

```text
[HIGH] Live production-grade secrets present in plaintext .env.local on disk
Status: CONFIRMED
File/Component: D:\dell\Html project\WA chatbot\wacrm-main\.env.local
Evidence: File contains a real SUPABASE_SERVICE_ROLE_KEY (JWT, role=service_role — bypasses all RLS), a real ENCRYPTION_KEY (valid 32-byte AES-256 key — decrypts every stored WhatsApp token and AI provider key in the DB), a real META_APP_SECRET, and a trailing plaintext comment block containing a live WhatsApp Access Token, Phone Number ID, WABA ID, and Webhook Verify Token.
Risk: Anyone who gains access to this file (backup, screen share, zip export, misconfigured sync, laptop theft) gets full database bypass, can decrypt every stored secret in the DB, can spoof/forge webhook signatures, and can send WhatsApp messages as this business.
Recommended Fix: Rotate all four secrets (Supabase service role key, ENCRYPTION_KEY, META_APP_SECRET, WhatsApp access token) as a precaution, move to a secrets manager for any shared/deployed environment, and never keep a live production key set in a local dev file.
Architectural Fix: Use per-environment secret injection (Vault, Doppler, cloud secrets manager) rather than .env files for anything beyond pure local dev with dummy/sandboxed credentials.
```

`.env.local` **is** correctly listed in `.gitignore` (`.env*`, with only `.env.local.example`/`.env.example` excluded), so it is not at risk of being committed if/when this project is put under git — this is a disk-hygiene finding, not a repository-hygiene one.

No hardcoded secrets were found anywhere in `src/`, `supabase/migrations/`, `mcp-server/src/`, `docker-compose.yml`, or `Dockerfile` — all reference `process.env.*` exclusively. No backend secret (service-role key, `ENCRYPTION_KEY`, `META_APP_SECRET`, AI provider keys, WhatsApp tokens) is ever returned to the frontend: API routes that read these values from the DB (`ai/config`, `whatsapp/config`) explicitly destructure them out of the response and return only boolean flags / non-sensitive metadata (confirmed in code). `NEXT_PUBLIC_*` usage is limited to genuinely public values (Supabase URL/anon key, site URL, locale).

Encryption (`src/lib/whatsapp/encryption.ts`) is implemented correctly: AES-256-GCM, fresh random 12-byte IV per call, authenticated tag verified on decrypt (tampering throws rather than silently succeeding), no ECB. A legacy CBC decrypt-only path exists for backward compatibility and auto-upgrades rows to GCM on use — an acceptable, self-limiting transitional risk.

---

## Authentication / Authorization

No unpatched authentication or authorization vulnerability was found. Specific checks:

- **Session handling:** `src/middleware.ts` validates sessions via `supabase.auth.getUser()` (server-verified, not just cookie presence), with tested cookie-rotation handling (`middleware.test.ts`).
- **Public API (`/api/v1`):** Bearer keys are resolved server-side to a fixed `accountId` via hashed lookup (SHA-256, never stored plaintext); the account cannot be overridden by request input. Dynamic `[id]` routes double-scope every query by `id` **and** `account_id`, returning 404 (not 403) on cross-account access to avoid confirming existence — reviewed in `contacts/[id]`, `conversations/[id]`.
- **Invitations:** 256-bit CSPRNG tokens, only a SHA-256 hash persisted, expiry enforced, and the `redeem_invitation` RPC blocks redemption unless the caller is the sole owner of an empty personal account — invited users can never self-grant `owner` (DB `CHECK` constraint enforces this).
- **Member management / ownership transfer:** both require `requireRole('admin'|'owner')` at the route layer and re-verify caller role + same-account target inside the underlying RPCs — no cross-account or self-escalation path found.
- **Webhook/cron auth:** WhatsApp webhook POSTs are HMAC-verified (`timingSafeEqual`) before any processing, failing closed if the secret is unset. Cron endpoints require a shared secret compared with `timingSafeEqual`, returning 401/503 otherwise.

```text
[LOW] Middleware provides no baseline authentication for most /api/* routes
Status: CONFIRMED (architectural gap, not an active exploit)
File/Component: src/middleware.ts
Evidence: protectedPaths only covers dashboard pages and /api/whatsapp/* (excluding /webhook); routes like /api/account/*, /api/automations/*, /api/flows/*, /api/v1/*, /api/quick-replies/* rely entirely on each handler independently calling requireRole/requireApiKey/getCurrentAccount. Every route sampled did so correctly.
Risk: A single future route that omits its own auth check would be fully unauthenticated, with no middleware-level backstop.
Recommended Fix: Add a lightweight middleware-level check (e.g., require a valid session or API key presence for any /api/* path not explicitly public) as defense-in-depth, in addition to (not instead of) per-route checks.
Architectural Fix: Not required — patch-level fix only.
```

---

## Tenant Isolation

RLS-based isolation is broadly sound. Two real, **historical** cross-tenant vulnerabilities were found in the migration history, both already fixed by the project team:

```text
[HIGH] Cross-tenant AI knowledge base read via SECURITY DEFINER RPC (historical — FIXED)
Status: CONFIRMED FIXED (see supabase/migrations/032_fix_ai_knowledge_membership.sql)
File/Component: match_ai_knowledge_fts / match_ai_knowledge_semantic RPCs
Evidence: Originally SECURITY DEFINER, filtering only on a caller-supplied p_account_id with no membership check — any authenticated user could read another tenant's AI knowledge base by passing a foreign account_id via PostgREST. Referenced as GHSA-fg5p-2qc3-jmxr. Fixed by switching to SECURITY INVOKER so RLS policy is_account_member(account_id) governs access.
Risk (historical): Full cross-tenant knowledge-base disclosure.
Recommended Fix: Already applied. Add a regression test against a live/test DB (migration comment notes it "was not run against a live database" at fix time).
Architectural Fix: None — fix pattern (INVOKER + RLS) is correct.
```

```text
[HIGH] Self-privilege-escalation / cross-tenant hijack via unrestricted profiles UPDATE (historical — FIXED)
Status: CONFIRMED FIXED (see supabase/migrations/034_fix_profiles_update_rls.sql)
File/Component: profiles table RLS policy (originally from migration 017)
Evidence: Original profiles_update policy only checked auth.uid() = user_id, not which columns changed — an authenticated user could UPDATE profiles SET account_role='owner' or account_id=<victim> on their own row. Fixed with a BEFORE UPDATE trigger (enforce_profile_privilege_columns) rejecting changes to account_role/account_id from the authenticated role.
Risk (historical): Self-granted ownership/admin, or hijacking membership into another tenant's account.
Recommended Fix: Already applied. Add a regression test (migration notes manual-only validation).
Architectural Fix: None — trigger-based column-level protection is the right pattern given Postgres RLS can't do column-level UPDATE restriction natively.
```

```text
[MEDIUM] Public API / MCP surface has no database-level tenant-isolation backstop
Status: CONFIRMED (architectural, not an active exploit — every current route is correctly scoped)
File/Component: src/lib/auth/api-context.ts, all src/app/api/v1/** routes
Evidence: requireApiKey() returns a service-role (RLS-bypassing) Supabase client; api-context.ts comments explicitly state "every downstream query MUST be explicitly filtered by ctx.accountId." All v1 routes reviewed (contacts, conversations, messages, broadcasts, webhooks) do this correctly today.
Risk: Any future /api/v1 route (or MCP tool) that omits the account_id filter is an immediate full cross-tenant data leak, with no RLS policy to catch the mistake — unlike the dashboard path.
Recommended Fix: Add an automated test suite that asserts every v1 route filters by account_id (e.g., a contract test that queries with two seeded accounts and asserts zero cross-account rows returned).
Architectural Fix: Consider issuing a scoped Postgres role/JWT claim per API key (so RLS applies even on this path) instead of using the service-role client, removing reliance on per-route code discipline.
```

No table reviewed (contacts, conversations, messages, accounts, whatsapp_config, api_keys, automations, flows, broadcasts, ai_knowledge_chunks) was found with RLS disabled or an overly permissive (`USING (true)`) policy in the current migration state.

---

## API / Application Security

No SQL injection, command injection, or path traversal was found — all DB access goes through the parameterized Supabase query builder or `.rpc()` calls with structured arguments; no `child_process` usage exists in the codebase. No XSS was found — `dangerouslySetInnerHTML` usage is limited to two files rendering fixed i18n markup, never user-supplied content. No mass assignment was found — every write route uses an explicit field allowlist rather than spreading the request body into `.update()`/`.insert()`. SSRF is actively mitigated for both outbound webhook delivery and the automation engine's `send_webhook` step (DNS-resolved private-range blocking + no-redirect-follow + timeout); the flows engine has no generic HTTP-request node, so its media-send path only ever hands URLs to Meta's own servers, not to this app's.

```text
[LOW] Uploaded media content-type is trusted from the client, not server-validated
Status: LIKELY
File/Component: src/lib/storage/upload-media.ts (~line 109)
Evidence: contentType: file.type is passed through to Supabase Storage from the client-supplied File object with no server-side MIME/allowlist check; file size is capped (16MB) and the storage path is sanitized against traversal, but content-type is not re-derived from file bytes.
Risk: If the storage bucket serves objects with the stored (client-supplied) content-type and public read access, an attacker could upload an HTML/SVG file typed as an image and have it served as executable content from the bucket's origin — a stored-XSS vector scoped to that bucket's domain. Could not confirm bucket-level content-type/MIME enforcement from application code alone (Supabase bucket config is out of scope for this static review).
Recommended Fix: Validate/derive content-type server-side from file magic bytes rather than trusting file.type; explicitly allowlist permitted MIME types for the media bucket.
Architectural Fix: Configure the Supabase Storage bucket itself with a strict allowed-MIME-types list and force Content-Disposition: attachment (or a sandboxed subdomain) for user-uploaded content, independent of app-layer checks.
```

Webhook signature verification (WhatsApp inbound POSTs) is done correctly: raw body is read and HMAC-verified via a fail-closed, timing-safe compare before any JSON parsing or processing.

---

## Architectural Gaps (Summary)

1. Public API/MCP path relies on app-code tenant scoping with no DB-level (RLS) backstop — see Tenant Isolation §3 above. **Patch now (tests), consider redesign (scoped role) longer-term.**
2. Middleware provides no baseline auth for most API routes — single point of developer error. **Patch.**
3. In-memory, per-process rate limiter (`src/lib/rate-limit.ts`) silently stops enforcing across horizontally scaled replicas — documented as a known limitation, not yet hit in the current single-instance deployment. **Patch when scaling out (move to Redis/shared store).**
4. Uploaded file content-type trust — **Patch (server-side MIME validation + bucket config).**

No component in this codebase was found to require a full **rebuild**. Nothing indicates a fundamentally broken security model — the RLS + service-role-with-manual-scoping pattern is a legitimate, common design; it just needs the defense-in-depth items above.

---

## Threats / Risks (Ranked)

1. **Operational secret exposure** (`.env.local` with live keys on disk) — highest realistic risk today, purely operational.
2. **Future regression in v1/MCP tenant scoping** — no RLS backstop; risk is latent, not current.
3. **AI/webhook data egress** — by design, but a meaningful privacy-posture fact for the product's DPA/ToS.
4. **Uploaded-file content-type trust** — plausible but unconfirmed stored-XSS vector, contingent on bucket configuration not visible from app code.

---

## Patch vs Redesign vs Rebuild

| Item | Classification |
|---|---|
| `.env.local` secret hygiene | Patch (rotate + process change) |
| Middleware baseline auth gap | Patch |
| v1/MCP tenant-scoping test coverage | Patch (add regression tests immediately) |
| v1/MCP RLS backstop (scoped Postgres role) | Redesign (longer-term hardening, not urgent — current code is correct) |
| Upload content-type validation | Patch |
| In-memory rate limiter | Patch (swap for shared store) when horizontally scaling |
| Historical AI-knowledge / profiles RLS bugs | Already fixed — no action beyond regression tests |
| **Rebuild** | None identified |

---

## Prioritized Remediation Plan

1. Rotate `SUPABASE_SERVICE_ROLE_KEY`, `ENCRYPTION_KEY`, `META_APP_SECRET`, and the WhatsApp access token found in `.env.local`; move to a secrets manager for any non-local environment.
2. Add automated regression tests for the two historically-fixed cross-tenant bugs (migrations 032, 034) so they cannot silently reappear.
3. Add a contract test asserting every `/api/v1/**` route filters by `account_id` (two-tenant seed data, assert zero leakage).
4. Add a lightweight middleware-level auth check for `/api/*` as defense-in-depth alongside existing per-route checks.
5. Enforce server-side MIME validation (magic-byte check) and a strict bucket-level allowed-content-type list for uploaded media.
6. Document the AI-provider and outbound-webhook data-egress paths in the product's privacy policy / customer-facing DPA.
7. Plan the shared-store rate limiter migration before any horizontal scale-out.

---

## Final Security Assessment

```text
Sensitive data leakage: PARTIALLY VERIFIED
Third-party data sharing: YES
Secret exposure: PARTIALLY VERIFIED
Tenant isolation risk: POTENTIAL
Critical findings: 0
High findings: 3
Overall security rating: 7.5/10
Production ready: AFTER REMEDIATION
Components requiring rebuild: None
```

Notes on the summary line items:
- **Sensitive data leakage — PARTIALLY VERIFIED**: confirmed, intentional egress to OpenAI/Anthropic (opt-in) and account-configured webhooks; no evidence of unintended leakage to unexpected third parties.
- **Secret exposure — PARTIALLY VERIFIED**: real secrets found in a local, correctly-gitignored `.env.local`; no secrets found reachable from the frontend or hardcoded in source.
- **High findings (3)**: the two historical, already-patched RLS/privilege-escalation bugs (listed for completeness/traceability) and the `.env.local` live-secrets exposure.
- **Tenant isolation risk — POTENTIAL**: current code is correctly scoped everywhere reviewed; the risk is the *absence of a DB-level backstop* on the API/MCP path, not a present breach.
