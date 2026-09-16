# Setup guide

End-to-end path from an empty machine to a working WhatsApp inbox: **clone → database → env → first user → Meta → webhook → test send**.

WhatsApp-only detail (tokens, webhook HMAC, `/register`, templates, Graph errors): **[WhatsApp Meta integration](./whatsapp-meta.md)**.

After setup, use the [product guide](./product-guide.md) for the dashboard. Token rotation, cron, and debugging live in [engineering](./engineering.md).

---

## What you need

| Item | Who | Notes |
| --- | --- | --- |
| Node.js **20+** | Engineer | `node -v` |
| Git + this repo (or your fork) | Engineer | |
| [Supabase](https://supabase.com) project | Engineer | Free tier is enough to start |
| Meta developer account + Business | Admin / engineer | [developers.facebook.com](https://developers.facebook.com) |
| WhatsApp Business number | Business | Meta **test number** is fine for sandbox; production needs a real WABA number |
| Public HTTPS URL for webhooks | Engineer | Production host, or a tunnel (`ngrok`, Cloudflare Tunnel, etc.) for local |

You can complete **steps 1–4** without Meta. The UI works; Inbox send and inbound messages will not.

**Time:** about 30–60 minutes the first time if Meta assets already exist; longer if you are creating a Business and WABA from scratch.

---

## Checklist

Use this as you go. Details are below.

- [ ] 1. Clone, install, copy `.env.local`
- [ ] 2. Create Supabase project; copy URL + anon + service role keys
- [ ] 3. Generate `ENCRYPTION_KEY`; set `META_APP_SECRET` (placeholder until the Meta app exists)
- [ ] 4. Run **all** SQL migrations in order
- [ ] 5. Configure Supabase Auth (redirect URLs, optional confirm-email)
- [ ] 6. `npm run dev` → sign up → you are **owner**
- [ ] 7. Create Meta app, add WhatsApp, copy Phone Number ID + WABA ID
- [ ] 8. Generate a **permanent System User** token (not the 24h temporary token)
- [ ] 9. Set `META_APP_SECRET` from App Settings → Basic; restart the app
- [ ] 10. Save WhatsApp credentials in **Settings → WhatsApp**
- [ ] 11. Point Meta webhook at `https://<public-host>/api/whatsapp/webhook`
- [ ] 12. Subscribe to **messages**; complete `/register` (PIN) if it is a production number
- [ ] 13. Test: phone → Inbox (inbound), Inbox → phone (outbound)
- [ ] 14. Optional: cron, templates dry-run, AI key, invite teammates, API key

---

## 1. Clone and install

```bash
git clone <your-repo-url>
cd wacrm
npm install
cp .env.local.example .env.local
```

---

## 2. Create a Supabase project

1. [supabase.com/dashboard](https://supabase.com/dashboard) → **New project**.
2. Wait until the project is healthy.
3. **Project Settings → API**:
   - **Project URL** → `NEXT_PUBLIC_SUPABASE_URL`
   - **anon public** → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - **service_role** → `SUPABASE_SERVICE_ROLE_KEY` (server only; never commit, never put in client code)

If login later fails with `Failed to fetch`, the URL must resolve from your machine (DNS). Try `8.8.8.8` if corporate DNS cannot find `*.supabase.co`.

---

## 3. Fill `.env.local`

Required block:

```bash
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# 64 hex characters (32 bytes)
ENCRYPTION_KEY=

META_APP_SECRET=
```

Generate the encryption key:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

**Do not rotate `ENCRYPTION_KEY` later** without re-saving WhatsApp, AI keys, and webhook secrets — old ciphertext cannot be decrypted.

Set `META_APP_SECRET` after step 7 (App Settings → Basic → App secret). Until then the app starts, but inbound webhooks are rejected.

Recommended:

```bash
NEXT_PUBLIC_SITE_URL=http://localhost:3000   # or your public origin
NEXT_PUBLIC_APP_LOCALE=en
```

Optional for local template UI without hitting Meta:

```bash
WHATSAPP_TEMPLATES_DRY_RUN=true
```

Optional if you will use automation **Wait** steps or flow timeout sweep:

```bash
AUTOMATION_CRON_SECRET=$(openssl rand -hex 32)
```

Full variable comments: `.env.local.example` and [engineering → Environment](./engineering.md#environment-variables).

---

## 4. Apply database migrations

The app expects every file in `supabase/migrations/` applied in numeric order (`001` … latest). Storage buckets (avatars, chat media, flow media) are created by those migrations.

**Option A — SQL Editor**

```bash
cat supabase/migrations/*.sql > /tmp/wacrm-all-migrations.sql
```

Supabase → **SQL Editor** → New query → paste → **Run**. If a statement errors mid-file, fix or skip already-applied objects and continue; a clean project should run top to bottom.

**Option B — CLI**

```bash
npx supabase link --project-ref <your-project-ref>
npx supabase db push
```

The project ref is the subdomain of `https://<ref>.supabase.co`.

### Auth settings (same project)

**Authentication → URL configuration**

- Site URL: `http://localhost:3000` locally, or your production origin.
- Redirect URLs: include `http://localhost:3000/**` and your production `https://<host>/**` (invite + email confirm).

**Authentication → Providers → Email**

- For the fastest local trial, you can disable **Confirm email**.
- For production, keep confirmation on; users must click the mail before login.

---

## 5. Run the app and create the first user

```bash
npm run dev
```

Open the URL Next prints. If port 3000 is taken, it may be **3001** or **3002**.

1. Go to `/signup`.
2. Email + password (min 6 characters). There is **no** seeded demo user.
3. Sign in. You land on `/dashboard` as **owner** of a new account.

If the dashboard errors on missing tables, migrations did not apply — return to step 4.

---

## 6. Create the Meta app and WhatsApp product

1. [developers.facebook.com](https://developers.facebook.com) → **My Apps** → **Create App** → type **Business**.
2. Add product **WhatsApp** → **Set up**. Link the Business / WABA when asked.
3. **WhatsApp → API Setup**
   - Copy **Phone number ID**
   - Copy **WhatsApp Business Account ID** (WABA ID)
   - Ignore **Temporary access token** for wacrm (it expires ~24 hours)

4. **App Settings → Basic**
   - Copy **App secret** → `META_APP_SECRET` in `.env.local`
   - Optional: **App ID** → `META_APP_ID` (needed to submit **image-header** templates)
5. Restart `npm run dev` after changing env.

---

## 7. Permanent access token (System User)

Outbound send (Inbox, automations, flows, broadcasts) uses this token.

1. [business.facebook.com/settings](https://business.facebook.com/settings) → the Business that owns the WABA.
2. **Users → System users** → **Add** (role **Admin**) if you have none.
3. Open the system user → **Add assets**:
   - WhatsApp account (WABA) — full control
   - The **Meta app** from step 6
4. **Generate token**
   - App: the same app
   - Expiration: **Never**
   - Permissions: `whatsapp_business_management`, `whatsapp_business_messaging`
5. Copy `EAA…` immediately.

Meta reference: [System user access tokens](https://developers.facebook.com/docs/whatsapp/business-management-api/get-started#system-user-access-tokens).

---

## 8. Connect WhatsApp inside wacrm

**Settings → WhatsApp** (owner or admin):

| Field | Value |
| --- | --- |
| Phone Number ID | From API Setup |
| WhatsApp Business Account ID | From API Setup |
| Permanent Access Token | System User token from step 7 |
| Webhook Verify Token | Any long random string **you invent** (e.g. `openssl rand -hex 16`). You will paste the **same** string into Meta. |
| Two-step PIN | Production numbers only. From WhatsApp Manager → Phone numbers → Two-step verification. Leave blank for Meta **test** numbers. |

**Save configuration**, then **Test API Connection**.

If Test fails with **Authentication Error / code 190**, the token is wrong, expired, or missing WhatsApp permissions — generate a new System User token and save again.

---

## 9. Webhook (Meta → your app)

Meta must POST to a URL it can reach on the public internet. `http://localhost:3000` will **not** work.

### Local development

Expose the Next port:

```bash
# example
ngrok http 3000
```

Use the `https://….ngrok-free.app` (or Cloudflare) origin.

### Meta configuration

**App → WhatsApp → Configuration** (wording varies: Configuration / Webhook):

1. Callback URL: `https://<your-public-host>/api/whatsapp/webhook`
2. Verify token: **exactly** the string from Settings → Webhook Verify Token
3. Verify and save (Meta sends GET; wacrm returns the challenge if the token matches a saved config)
4. Subscribe to the **messages** field (and template fields if you use template status webhooks)

If verification fails: app is running, `META_APP_SECRET` is set, WhatsApp config is **saved** (verify token is in the database), URL has no trailing path typo.

### Register the number (production)

Saving credentials is not always enough for inbound. wacrm calls Meta `/register` when you save a PIN. Symptom if skipped: messages go to whichever app last claimed the number.

Test numbers are often pre-registered — PIN optional.

---

## 10. Prove it works

### Inbound

1. From your personal WhatsApp, message the **business / test** number (the number shown in API Setup).
2. **Inbox** should show a new conversation within a few seconds.
3. Server log: `POST /api/whatsapp/webhook` → **200**. That only means Meta reached you.

### Outbound

1. Open that thread in Inbox, send a short text.
2. It should appear on your phone. Delivery ticks follow when status webhooks arrive.
3. If the bubble goes **failed** with code **190**, fix the access token (step 7–8). Inbound can still work.

### First automation (optional)

1. **Automations → New** → template **Welcome Message** (or keyword `hello` → send text).
2. Toggle **Active**.
3. Message the number again. Check **Logs** on that automation if nothing replies.

---

## 11. Optional next steps

| Goal | What to do |
| --- | --- |
| Automation **Wait** steps | Set `AUTOMATION_CRON_SECRET`. Schedule `GET /api/automations/cron` with header `x-cron-secret` every 1–5 minutes. Also ping `GET /api/flows/cron` so abandoned flows time out. |
| Team | **Settings → Members** → invite as agent/admin. Add redirect URLs for production. |
| Templates | Meta → WhatsApp Manager → create/approve templates → **Settings → Templates → Sync**. Local: `WHATSAPP_TEMPLATES_DRY_RUN=true`. |
| AI drafts / auto-reply | **AI Agents → Setup** → paste OpenAI or Anthropic key (encrypted with `ENCRYPTION_KEY`). |
| Public API / MCP | **Settings → API keys** → [public-api.md](./public-api.md), [mcp.md](./mcp.md). |
| Production host | Same env vars on the server. `NEXT_PUBLIC_SITE_URL` = `https://your-domain`. Point Meta webhook at production. [Hostinger walkthrough](https://wacrm.tech/docs/deployment-hostinger). |

---

## 12. Production (short)

1. Deploy Node 20+ (`npm run build` / `npm start` or the host’s Next builder).
2. Set the same env vars (no `.env` in git).
3. `NEXT_PUBLIC_SITE_URL=https://your-domain`.
4. Supabase Auth Site URL + redirects = production origin.
5. Re-save or keep WhatsApp config in the **same** Supabase project.
6. Meta webhook callback = `https://your-domain/api/whatsapp/webhook`.
7. HTTPS is required.

---

## If something fails

| Stage | Symptom | See |
| --- | --- | --- |
| 2–5 | Login fetch failed | DNS / Supabase keys — [engineering](./engineering.md#common-failures) |
| 4 | Dashboard SQL errors | Migrations not applied |
| 9 | Webhook verify 403 | Verify token not saved, or GET URL wrong |
| 10 inbound | Nothing in Inbox | Webhook URL, `messages` subscription, `/register`, `META_APP_SECRET` |
| 10 outbound | Auth error 190 | Permanent token, Test connection |
| Automations | Inbound ok, no bot reply | Active flag, trigger match, Flow consuming, send token, [product guide](./product-guide.md) |

---

## Roles after first signup

The first user is **owner**. They connect WhatsApp and invite the rest of the team. Capability table: [product guide → Team and roles](./product-guide.md#team-and-roles).
