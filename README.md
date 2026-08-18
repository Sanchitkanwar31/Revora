# Revora — CRM for WhatsApp

> Shared inbox, contacts, sales pipelines, broadcasts, and no-code
> automations for the official WhatsApp Business Cloud API.

<!-- TODO: re-add a CI status badge here once a private CI pipeline is wired up. -->
<!-- TODO: re-add a deploy banner/link here once a hosting target is chosen. -->

## What you get out of the box

- **Shared inbox** on the official WhatsApp Business API — multiple
  agents working one number, per-conversation assignment, status, and
  notes.
- **Contacts + tags + custom fields**, CSV import, deduplication.
- **Sales pipelines** (Kanban) with deals linked to conversations.
- **Broadcasts** with Meta-approved templates, delivery + read
  tracking, per-recipient variable substitution.
- **No-code automations** — triggers on inbound messages, new
  contacts, keywords, or schedule; conditional branches, waits,
  tags, webhooks. Visual builder.
- **AI reply assistant** — bring your own OpenAI or Anthropic key
  (stored encrypted; no per-seat AI fee, your data stays yours).
  One-click AI-drafted replies in the inbox, plus an optional
  auto-reply bot with a per-conversation cap and clean human handoff.
  Add a **knowledge base** (FAQs, policies, product docs) and it
  answers from your own content — hybrid retrieval (Postgres full-text,
  or semantic pgvector when an embeddings key is set).
- **Real-time dashboard** — response times, daily volume, pipeline
  value, cross-module activity feed.
- **Team accounts** — invite teammates by link, role-based access
  (owner / admin / agent / viewer), ownership transfer. Every install
  is account-scoped, so one shared inbox can be staffed by a whole
  team. Solo use stays single-user with zero setup.
- **Account management** — email, password, avatar, global sign-out.
- **Public REST API** (`/api/v1`) with scoped, revocable API keys —
  build your own automations on top of your CRM. See
  [docs/public-api.md](./docs/public-api.md).
- **MCP server** — drive your CRM from Claude, Cursor, and other AI
  assistants over the [Model Context Protocol](https://modelcontextprotocol.io).
  Read-only by default, opt-in writes. See [docs/mcp.md](./docs/mcp.md)
  (server in [`mcp-server/`](./mcp-server)).

## Security

- Token encryption (AES-256-GCM) for WhatsApp access tokens, verify
  tokens, and AI provider keys.
- Row-Level Security (RLS) on every table as the primary tenant
  isolation mechanism.
- HMAC-verified inbound webhooks; HMAC-signed outbound webhooks.
- SSRF protection on outbound webhook targets.
- CSP, HSTS, and other standard security headers.
- Rate limiting on sends, broadcasts, the public API, and AI calls.

## Quick start

```bash
git clone https://github.com/Sanchitkanwar31/Revora.git
cd Revora
npm install
cp .env.local.example .env.local   # fill in Supabase + Meta creds
npm run dev
```

Open <http://localhost:3000>. You'll be redirected to `/login` (or
`/dashboard` if already signed in).

Prefer containers? See [docs/docker.md](./docs/docker.md) for the
Dockerfile + Docker Compose setup.

## Deployment

<!-- TODO: document the chosen production hosting target (Vercel / Railway / VPS + Docker / etc.) here. -->

This is a standard Next.js 16 app (`output: "standalone"`) with an
external Supabase project for Postgres + Auth + Storage — it runs
anywhere Node.js 20+ does.

## Documentation

- [docs/docker.md](./docs/docker.md) — Docker build + Compose setup.
- [docs/mcp.md](./docs/mcp.md) — MCP server setup.
- [docs/public-api.md](./docs/public-api.md) — public REST API reference.

<!-- TODO: link an internal setup/runbook doc here once one exists. -->

## Stack

- **App** — Next.js 16 (App Router), React 19, TypeScript, Tailwind v4.
- **Data** — Supabase (Postgres + Auth + Storage + RLS).
- **WhatsApp** — Meta Cloud API (official WhatsApp Business API).

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).
