# Engineering and operations

How to run, configure, and debug a wacrm install. **First-time walkthrough:** [Setup guide](./setup-guide.md). Product behavior: [Product guide](./product-guide.md). Internals: [Architecture](./architecture.md).

## Prerequisites

- Node.js **≥ 20**
- A [Supabase](https://supabase.com) project
- A Meta app with WhatsApp product (for real messaging)
- `ENCRYPTION_KEY` and `META_APP_SECRET` for anything beyond clicking around the UI

Docker is **not** required. Local Supabase CLI is optional; most teams point `.env.local` at a hosted Supabase project.

## Quick start

```bash
git clone <your-fork>
cd wacrm
npm install
cp .env.local.example .env.local
# fill REQUIRED vars — see table below
npm run dev
```

Open the URL Next prints (port **3000**, or 3001/3002 if taken). Sign up at `/signup`.

### Apply database migrations

On a **new** Supabase project, apply every file in `supabase/migrations/` in numeric order (`001` … latest).

**SQL Editor (simple):**

```bash
cat supabase/migrations/*.sql > /tmp/wacrm-all-migrations.sql
```

Paste into Supabase → SQL Editor → Run.

**CLI:**

```bash
npx supabase link --project-ref <project-ref>
npx supabase db push
```

Do not skip or reorder files. Re-running is often safe (`IF NOT EXISTS`) but a clean first pass is best.

Enable **Realtime** on `messages` (and conversations if you use those subscriptions) in the Supabase dashboard if inbox live-updates are missing.

## Environment variables

Full comments live in `.env.local.example`.

### Required

| Variable | Purpose |
| --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase API URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Browser + SSR user client |
| `SUPABASE_SERVICE_ROLE_KEY` | Webhook, engines, API key lookup. **Secret.** |
| `ENCRYPTION_KEY` | 64 hex chars. `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `META_APP_SECRET` | HMAC for inbound webhook. App Settings → Basic. Without it, webhook rejects POSTs. |

### Recommended

| Variable | Purpose |
| --- | --- |
| `NEXT_PUBLIC_SITE_URL` | Canonical origin (sitemap, OG; invite fallback) |
| `NEXT_PUBLIC_APP_LOCALE` | `en` (default) or `ko` |

### Optional

| Variable | Purpose |
| --- | --- |
| `ALLOWED_INVITE_HOSTS` | Hostname allow-list for invite URLs |
| `AUTOMATION_CRON_SECRET` | Protects `GET /api/automations/cron` (Wait steps) |
| `META_APP_ID` | Image-header template resumable upload |
| `WHATSAPP_TEMPLATES_DRY_RUN=true` | Fake template submit in local/CI |
| `AI_REQUEST_TIMEOUT_MS` | Default 30000 |
| `AI_CONTEXT_MESSAGE_LIMIT` | Default 20 |

There is **no** global OpenAI/Anthropic env var. Keys are per-account in Settings → AI / Agents.

## Scripts

| Command | What |
| --- | --- |
| `npm run dev` | Next dev server |
| `npm run build` / `npm start` | Production |
| `npm run lint` / `typecheck` | CI-style checks |
| `npm test` | Vitest |

## Generate a Meta access token

Full Cloud API walkthrough: **[WhatsApp Meta integration](./whatsapp-meta.md)**.

Inbox send, automations, flows, and broadcasts use the token stored in **Settings → WhatsApp**, not an env var.

**Do not** save the temporary token from WhatsApp → API Setup (expires ~24h). That produces Graph **code 190** / `Authentication Error` while inbound webhooks still return 200.

### Permanent System User token

1. [business.facebook.com/settings](https://business.facebook.com/settings) → your Business.
2. **Users → System users** → add one (Admin) if needed.
3. **Add assets**: WhatsApp Business Account + the Meta **app** used for webhooks.
4. **Generate token** → that app → expiration **Never**.
5. Permissions: `whatsapp_business_management`, `whatsapp_business_messaging`.
6. Copy `EAA…` once. Paste into wacrm **Permanent Access Token** → Save → **Test API Connection**.

Also copy **Phone Number ID** and **WhatsApp Business Account ID** from WhatsApp → API Setup.

Docs: [Meta system user tokens](https://developers.facebook.com/docs/whatsapp/business-management-api/get-started#system-user-access-tokens).

### Webhook

Callback: `https://<your-host>/api/whatsapp/webhook`  
Verify token: same string as Settings → Webhook Verify Token.  
Subscribe field: **messages** (and template fields if you use template webhooks).

Production numbers: set two-step PIN in WhatsApp Manager and paste it so wacrm can `POST /{phone_number_id}/register`. Test numbers often have no PIN.

`META_APP_SECRET` must match the app that signs webhook POSTs.

## Cron jobs

| Endpoint | Header | Why |
| --- | --- | --- |
| `GET /api/automations/cron` | `x-cron-secret: $AUTOMATION_CRON_SECRET` | Resume **Wait** steps |
| `GET /api/flows/cron` | `x-cron-secret: $AUTOMATION_CRON_SECRET` | Mark abandoned flow runs `timed_out` (unblocks the next trigger) |

Example:

```bash
curl -sS -H "x-cron-secret: $AUTOMATION_CRON_SECRET" \
  https://crm.example.com/api/automations/cron
```

Hit every minute (or 5) from GitHub Actions, cron, or the host’s scheduler. If unset, the route returns **503** `cron not configured`.

## Local vs other processes on port 3000

If `localhost:3000` is not wacrm, Next will bind **3001** or **3002**. Check the `npm run dev` banner. Unrelated apps on 3000 do not affect Meta; Meta must reach a **public HTTPS** URL (tunnel or deploy).

DNS: if `*.supabase.co` is NXDOMAIN on the machine DNS but works via `8.8.8.8`, login will `Failed to fetch`. Fix resolver/hosts, not application code.

## i18n

UI strings live in `messages/en.json` and `messages/ko.json`. `next-intl` **throws** on missing keys (`MISSING_MESSAGE`) even if `defaultValue` is passed. Add the key to **both** locale files when adding UI copy.

## Logging (what to look at)

| Log prefix | Meaning |
| --- | --- |
| `[webhook] inbound processed` | Message saved; lists `automationTriggers` |
| `[automations] no active automations for trigger` | Nothing Active for that trigger type |
| `[automations] skipped — trigger config did not match` | Keywords / reply ids / tag id mismatch |
| `[automations] running` / `step failed` | Engine executed; send may have failed |
| `[meta-api] Graph API request failed` | Full Graph envelope |
| `[whatsapp/send] failed` | Inbox send mapped to HTTP 502 + message |
| `[inbox] Failed to send message` | Browser console |

Webhook HTTP **200** is not outbound success.

## Common failures

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Login fetch failed | Supabase URL DNS / wrong keys | Check `.env.local`, DNS |
| Webhook 401 | Bad `META_APP_SECRET` or signature | App secret, raw body |
| Inbound works, send 190 | Dead access token | New System User token, re-save |
| Automations never run | Were fire-and-forget (fixed in engine await); or inactive; or Flow consumed; or send 190 | Logs tab + server logs |
| Wait never continues | No cron | Configure `AUTOMATION_CRON_SECRET` + ping |
| Templates submit fail locally | Hitting live Meta | `WHATSAPP_TEMPLATES_DRY_RUN=true` |
| Teammate sees empty WhatsApp settings | Not admin | Role upgrade |
| Invite link wrong host | Spoofed Host / missing site URL | `NEXT_PUBLIC_SITE_URL` / `ALLOWED_INVITE_HOSTS` |

## Public API and MCP

- REST: [public-api.md](./public-api.md) — keys in Settings → API keys.
- MCP: [mcp.md](./mcp.md) and `mcp-server/README.md`. Wrapper around `/api/v1`; writes opt-in via env.

## Security checklist for production

- [ ] Service role and `ENCRYPTION_KEY` only on the server
- [ ] HTTPS for webhook and dashboard
- [ ] Permanent token in DB, not git
- [ ] `META_APP_SECRET` set
- [ ] RLS enabled (comes with migrations)
- [ ] API keys scoped to least privilege
- [ ] Rotate tokens if leaked; revoke API keys in UI

## Deploy notes

The upstream template targets Node hosts (documented for Hostinger). Any Node 20+ host that can run `next start` works. Set the same env vars in the panel. Point Meta webhooks at the **production** origin.

`maxDuration` on the webhook route is 60s so `after()` media + engines can finish after the 200 ack. Serverless platforms that freeze immediately after response will drop work unless they honor Next.js `after()`.
