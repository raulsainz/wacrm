# wacrm — Project Context

## Framework & Important Notes

<!-- BEGIN:nextjs-agent-rules -->
This version uses Next.js 16 with breaking changes — APIs, conventions, and file structure may differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.
<!-- END:nextjs-agent-rules -->

**Tech Stack:**
- Framework: Next.js 16.3.8, React 19.2.4, `next-intl` for i18n
- Database & Auth: Supabase (Postgres + Auth + Storage), `@supabase/ssr` middleware
- UI: Tailwind CSS, shadcn, Base UI, recharts, `@dnd-kit`, `@xyflow/react` (dag/flow builder)
- Node: 22+ required (`engines.node: >=22.12.0`)

---

## Configuration Requirements

### 5 Required Environment Variables (app will not run without these)

Copy `.env.local.example` → `.env.local` and fill in:

| Variable | Purpose | Notes |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL | Client + server; baked in at build time |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase client key (RLS-scoped) | Used in middleware, client, server-side calls |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase server key (bypasses RLS) | Required for webhook handlers, automations, flows, AI admin clients |
| `ENCRYPTION_KEY` | AES-256-GCM encryption key (64 hex chars / 32 bytes) | Encrypts WhatsApp tokens, webhook secrets, AI provider keys at rest. **Critical:** rotating this key orphans existing encrypted data. Generate: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `META_APP_SECRET` | Meta webhook signature verification (HMAC-SHA256) | Webhook handler fails closed (rejects all requests) if unset. Supports comma-separated list for multi-WABA/multi-app setups |

### Required Non-Env Setup

1. **Supabase project creation & migrations:** Create Supabase account/project. Apply all 59 migration files in `supabase/migrations/` via Supabase CLI — they are **not** auto-run by the app or Docker. Database: Postgres 17.
2. **Meta/WhatsApp setup:** Create Meta app + WhatsApp Business Account (WABA), obtain `META_APP_SECRET`, configure webhook at `https://<your-domain>/api/whatsapp/webhook` (HTTPS required). Refer to `docs/whatsapp-connection-troubleshooting.md` and `docs/multi-waba.md` for multi-WABA/multi-app scenarios.
3. **Per-account credentials (entered in-app, not env vars):**
   - WhatsApp: `phone_number_id`, `access_token`, `verify_token`, `waba_id` (Settings → WhatsApp config)
   - AI providers: OpenAI/Anthropic/OpenAI-compatible API keys (Settings → AI Assistant) — stored encrypted using `ENCRYPTION_KEY`

### Optional Environment Variables

| Variable | Enables | Default |
|---|---|---|
| `META_APP_ID` | Image-header message template submission (Resumable Upload API) | Not set |
| `AUTOMATION_CRON_SECRET` | `/api/automations/cron` & `/api/flows/cron` endpoints for Wait-step automations and flow timeouts. Routes return 503 if unset. Requires external scheduler (Vercel Cron/GitHub Actions) with header `x-cron-secret`. | Not set (503 until configured) |
| `NEXT_PUBLIC_SITE_URL` | Correct domain for invite links | Derived from request Host/X-Forwarded-Host headers if unset |
| `ALLOWED_INVITE_HOSTS` | Defense-in-depth allow-list for invite domains (comma-separated) | All hosts allowed if unset |
| `WHATSAPP_TEMPLATES_DRY_RUN` | Skip real Meta API calls for template submission (useful for CI/local dev without live WABA) | Not set (real calls made) |
| `AI_REQUEST_TIMEOUT_MS` | AI provider call timeout | 30000 |
| `AI_CONTEXT_MESSAGE_LIMIT` | Max messages sent to AI in context | 20 |
| `AI_PROVIDER_ALLOWED_PRIVATE_HOSTS` | Allow-list for private/self-hosted LLM endpoints (e.g. Ollama, vLLM; comma-separated) | Empty (private targets blocked for SSRF protection) |
| `NEXT_PUBLIC_APP_LOCALE` | Default app language (baked in at build time) | `en` |
| `ALLOWED_DEV_ORIGINS` | Dev-only cross-origin HMR allow-list (comma-separated) | Not set |

---

## External Integrations

### WhatsApp Business API (Meta Graph API)

- **No official SDK.** Direct `fetch` calls to `https://graph.facebook.com/v21.0` (see `src/lib/whatsapp/meta-api.ts`).
- **Webhook handler:** `GET/POST /api/whatsapp/webhook` validates `x-hub-signature-256` HMAC (requires `META_APP_SECRET`).
- **Webhook routes:** 
  - `GET` — Meta's verification handshake
  - `POST` — message inbound/status/template-lifecycle events, dispatched to flows, automations, AI auto-reply, outbound webhooks
- **Token encryption:** `ENCRYPTION_KEY` (AES-256-GCM) encrypts WhatsApp `access_token` and `verify_token` at rest.
- **Template submission:** Supports text, media with headers; image headers use Resumable Upload API (requires `META_APP_ID`).

### Supabase (Database, Auth, Storage)

- **All DB access:** Via `@supabase/ssr` middleware + `@supabase/supabase-js` client; no raw Postgres connection string.
- **No `DATABASE_URL`** — use Supabase URL + keys.
- **Auth:** Supabase Auth handles signup/login; `src/middleware.ts` validates session and refresh tokens.
- **Storage:** Two buckets: `chat-media` (inbound WhatsApp media mirror + outbound attachments), `flow-media` (flow builder assets). Uploads are account-scoped via RLS path convention (`account-<account_id>/...`). Max 16 MB per file.
- **Migrations:** 59 SQL files; includes pgvector extension for AI knowledge-base semantic search (migration 030, optional feature).

### AI Providers (Bring-Your-Own-Key)

- **Supported:** OpenAI, Anthropic, OpenAI-compatible (self-hosted Ollama, vLLM, etc.)
- **No SDK.** Raw HTTP calls via `fetch`.
- **Keys stored encrypted** in DB per account (not env vars) using `ENCRYPTION_KEY`.
- **Embeddings:** OpenAI (1536-dim `text-embedding-3-small`) for AI knowledge-base semantic search.
- **Timeouts & tuning:** Configurable via `AI_REQUEST_TIMEOUT_MS`, `AI_CONTEXT_MESSAGE_LIMIT`, `AI_PROVIDER_ALLOWED_PRIVATE_HOSTS`.
- **SSRF protection:** Private/loopback hosts blocked unless in `AI_PROVIDER_ALLOWED_PRIVATE_HOSTS` allow-list.

### Outbound Webhooks (Public API)

- **Mechanism:** Accounts register webhook endpoint URLs + secrets (stored encrypted with `ENCRYPTION_KEY`).
- **Delivery:** Inside `next/server` `after()` callback, HTTPS only, Stripe-style HMAC-SHA256 signatures.
- **SSRF protection:** `isDeliverableUrl` blocks private/loopback targets and redirect chains.
- **No retry queue** — best-effort single-attempt delivery; durable retries would require a queue/worker (not currently implemented).

---

## What's NOT in This Project

- **No Twilio, SendGrid, Resend, or email/SMS SDK.** Supabase Auth handles email transactionally (confirmation, password reset).
- **No Stripe or payment integration.**
- **No Redis, BullMQ, or queue system.** Rate limiting is in-memory single-process (documented scaling limitation). Webhook retry logic is single-attempt.
- **No S3, Cloudinary, or external file storage.** Uses Supabase Storage only.
- **No `DATABASE_URL`.** All DB access via Supabase client.
- **No NextAuth, Clerk, Auth0, or external auth provider.** Supabase Auth only.

---

## Local Development

```bash
npm install
cp .env.local.example .env.local      # fill in required vars (see above)
npm run dev                            # Turbopack dev server, port 3000
npm run typecheck                      # tsc --noEmit
npm run lint                           # ESLint
npm run format                         # Prettier
npm run test / npm run test:watch      # Vitest
```

## Docker Deployment

```bash
docker compose --env-file .env.local up --build -d
```

**Important:** 
- `NEXT_PUBLIC_*` vars are baked in as build args — rebuild (`--build`) whenever any change.
- Supabase is external — apply migrations separately via Supabase CLI.
- No internal database container — the compose file is app-only.

---

## MCP Server (Optional)

Sub-package `mcp-server/` exposes wacrm as MCP tools. Separate setup:

```bash
cd mcp-server
cp .env.example .env
# Set:
# WACRM_BASE_URL=https://your-wacrm-instance
# WACRM_API_KEY=<generated in-app under Settings → API keys>
# WACRM_ENABLE_WRITES=false (optional, off by default)
# WACRM_ENABLE_BROADCASTS=false (optional, requires WACRM_ENABLE_WRITES)
```

See `docs/mcp.md` for full setup.

---

## Key File Structure

- `src/middleware.ts` — Supabase Auth session validation
- `src/lib/supabase/server.ts`, `src/lib/supabase/client.ts` — DB access clients
- `src/lib/whatsapp/` — Meta API calls, webhook signature verification, token encryption
- `src/app/api/whatsapp/webhook/route.ts` — Webhook handler
- `src/lib/flows/`, `src/lib/automations/` — Flow/automation execution engines
- `src/lib/ai/` — AI provider abstraction, embeddings, knowledge-base
- `src/app/api/v1/` — Public REST API (requires API key, not `AUTOMATION_CRON_SECRET`)
- `supabase/migrations/` — 59 SQL migration files (Postgres schema)

---

## Verification Checklist

1. ✓ All 5 required env vars set in `.env.local`
2. ✓ `npm install` succeeds
3. ✓ `npm run dev` boots without missing-env-var errors
4. ✓ Supabase migrations applied (59 files)
5. ✓ Sign up/login works (tests Supabase Auth)
6. ✓ Settings → WhatsApp config page opens (tests Supabase Storage + `ENCRYPTION_KEY`)
7. ✓ Webhook URL configured in Meta app dashboard (HTTPS)
8. ✓ `npm run typecheck && npm run lint` clean
