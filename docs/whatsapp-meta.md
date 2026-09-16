# WhatsApp Cloud API (Meta) integration guide

How wacrm talks to the official **WhatsApp Business Cloud API** — credentials, webhooks, tokens, templates, and the failures you will actually see.

This is **not** the unofficial WhatsApp Web / Baileys stack. You need a Meta app, a WhatsApp Business Account (WABA), and Graph API tokens.

Related: first-time install order is in the [setup guide](./setup-guide.md). Dashboard fields are in the [product guide](./product-guide.md). Internals: [architecture](./architecture.md).

---

## How the integration is split

Two layers. Mixing them up is the usual source of “webhook 200 but send fails.”

| Layer | Where it lives | What it is for |
| --- | --- | --- |
| **App / server env** | `.env.local` or host panel | Prove Meta POSTs are real; encrypt stored secrets |
| **Account credentials** | **Settings → WhatsApp** (Postgres `whatsapp_config`, encrypted) | Send and receive on **this** WABA number |

### Server env (engineer)

| Variable | Required | Role |
| --- | --- | --- |
| `META_APP_SECRET` | **Yes** for inbound | HMAC of webhook POST (`X-Hub-Signature-256`). Missing → every webhook rejected. |
| `ENCRYPTION_KEY` | **Yes** | AES-256-GCM for `access_token`, `verify_token`, AI keys, outbound webhook secrets |
| `META_APP_ID` | Optional | Only for **image-header** template submit (resumable upload) |
| `WHATSAPP_TEMPLATES_DRY_RUN=true` | Optional | Local/CI: skip live Meta on template submit |

There is **no** `WHATSAPP_ACCESS_TOKEN` env var. The send token is stored per account in the database after Settings → Save.

### Account fields (owner / admin in the UI)

| Field | Meta name | Used for |
| --- | --- | --- |
| Phone Number ID | Phone number ID on API Setup | Graph path `/{phone-number-id}/messages` |
| WhatsApp Business Account ID | WABA ID | Subscribe app, sync templates |
| Permanent Access Token | System User token | **All outbound** Graph calls |
| Webhook Verify Token | You invent this string | Meta GET challenge ↔ wacrm |
| Two-step PIN | WhatsApp Manager PIN | `POST /{phone-number-id}/register` so **this app** owns inbound |

One `phone_number_id` can belong to **only one** wacrm account (enforced on save).

---

## Mental model

```
                    META_APP_SECRET (env)
                            │
Customer ──WhatsApp──► Meta ──POST /api/whatsapp/webhook──► wacrm
                            │                         HMAC check, then 200
                            │                         after(): save inbox,
                            │                         flows / automations / AI
                            ▼
Agent / automation / API ──► wacrm ──Bearer access_token──► graph.facebook.com
                             decrypt token from whatsapp_config
```

- **Inbound** needs: public HTTPS webhook, matching verify token, valid app secret, number registered to this app, `messages` subscription.
- **Outbound** needs: valid **permanent** token + correct Phone Number ID (and, outside the 24-hour window, an **approved template**).

Those paths are independent. A dead send token does **not** stop inbound. A wrong webhook does **not** stop you from sending if Graph still accepts the token.

---

## 1. Meta app and WhatsApp product

1. [developers.facebook.com](https://developers.facebook.com) → **My Apps** → **Create App** → **Business**.
2. **Add product** → **WhatsApp** → Set up. Link the Business Manager / WABA.
3. **WhatsApp → API Setup**
   - Copy **Phone number ID**
   - Copy **WhatsApp Business Account ID**
   - Do **not** treat **Temporary access token** as production (≈ 24 hours).
4. **App Settings → Basic**
   - **App secret** → `META_APP_SECRET` (restart the Node process after changing env).
   - **App ID** → `META_APP_ID` if you will submit image-header templates.

Use the **same** Meta app for: webhook, System User token, and `META_APP_SECRET`. A token from app A and a secret from app B will fail HMAC or Graph auth.

---

## 2. Permanent System User token

wacrm labels this **Permanent Access Token**.

1. [business.facebook.com/settings](https://business.facebook.com/settings) → the Business that owns the WABA.
2. **Users → System users** → Add if needed (role **Admin**).
3. **Add assets**
   - WhatsApp Business Account — full control
   - The **Meta app** from step 1
4. **Generate token**
   - App = that Meta app
   - Expiration = **Never**
   - Permissions (both):
     - `whatsapp_business_management`
     - `whatsapp_business_messaging`
5. Copy `EAA…` once. Paste into Settings → WhatsApp. Re-enter it whenever you click **Save** (the UI does not reuse the masked value).

Official: [System user access tokens](https://developers.facebook.com/docs/whatsapp/business-management-api/get-started#system-user-access-tokens).

### Why the temporary token breaks you

| Token | Lifetime | Typical result in wacrm |
| --- | --- | --- |
| API Setup “Temporary access token” | ~24h | Inbox send / automations: `Authentication Error` Graph **code 190** |
| System User, never expire | Until you revoke it | Stable send + template sync |

---

## 3. Save credentials in wacrm

**Settings → WhatsApp** (owner/admin). Always save through the UI / `POST /api/whatsapp/config` — **do not** insert tokens into Postgres yourself. The API encrypts with `ENCRYPTION_KEY` and calls Meta to verify the Phone Number ID.

On save wacrm:

1. Rejects if another account already claimed this `phone_number_id`.
2. `GET /{phone-number-id}` with your token (health check).
3. Encrypts access + verify tokens.
4. Optionally **`/register`** if you sent a PIN (see below).
5. Returns `registered: true | false` and any `registration_error`.

Then click **Test API Connection**. That is the outbound check.

---

## 4. Webhooks

Endpoint: `GET|POST /api/whatsapp/webhook`

### Verify (GET)

Meta calls:

`GET /api/whatsapp/webhook?hub.mode=subscribe&hub.challenge=…&hub.verify_token=…`

wacrm compares `hub.verify_token` to **decrypted** `whatsapp_config.verify_token` rows. First match wins; it returns the challenge as plain text.

The config row **must already be saved** before you click Verify in Meta.

### Callback (POST)

1. Read **raw** body.
2. Verify `X-Hub-Signature-256: sha256=<hex>` with `META_APP_SECRET` (fail closed).
3. Respond **200** `{ "status": "received" }` immediately.
4. `after()`: persist messages, then flows / automations / AI.

**200 does not mean** a reply was sent. It only means Meta’s POST was accepted.

### Configure in Meta

**App → WhatsApp → Configuration** (or Webhooks):

| Setting | Value |
| --- | --- |
| Callback URL | `https://<public-host>/api/whatsapp/webhook` |
| Verify token | **Exact** string from Settings → Webhook Verify Token |
| Fields | At least **`messages`**. Add template fields if you sync template status from webhooks. |

`localhost` is not reachable. Locally use **ngrok**, Cloudflare Tunnel, or similar HTTPS, then use that origin as the callback.

### Signature failures

If `META_APP_SECRET` is wrong or missing, logs show `[webhook] rejected request with invalid signature` (or “not set”) and Meta retries. Fix env, restart, do not disable verification.

---

## 5. Register the phone number (`/register`)

Saving Phone Number ID + token is **not** the same as claiming inbound for this app.

Meta: `POST /{phone-number-id}/register` with the 6-digit **two-step verification PIN** from WhatsApp Manager → Phone numbers.

| Number type | PIN | What wacrm does |
| --- | --- | --- |
| Meta **test** number | Often none | Save without `/register`; banner may say not registered. Test inbound can still work if Meta already bound the test number to the app. |
| **Production** number | Required | Paste PIN on save so this app receives events. Otherwise inbound stays on whichever app last registered. |

Symptom of a stolen inbound path: webhook never fires (or fires on another project) while send still works.

Use **Verify registration** on the WhatsApp settings page after save. It checks Graph phone info, webhook subscription, and local `registered_at`.

---

## 6. Message types and the 24-hour window

| Direction | Allowed |
| --- | --- |
| Customer → you | Always (within Meta policy) |
| You → customer **inside** 24h of last customer message | Free-form **text**, media, interactive buttons/lists |
| You → customer **outside** that window | **Approved template** only (`send_template` / Inbox template / broadcast) |

Interactive **button / list ids** must match automations (`interactive_reply`) and Flow edges exactly (`yes` ≠ `Yes`).

Reactions, location, stickers, documents, voice notes are stored inbound when Meta sends them; outbound media uses a **public URL** Meta can fetch.

---

## 7. Templates

- Created and approved in **Meta WhatsApp Manager** (or submitted from **Settings → Templates**).
- **Sync from Meta** pulls approved names/languages into `message_templates`.
- Broadcasts **only** send templates.
- Image-header sample upload needs `META_APP_ID` + `META_APP_SECRET`.
- Local UI without a live WABA: `WHATSAPP_TEMPLATES_DRY_RUN=true`.

---

## 8. Subscribe the WABA to the app

On save, wacrm may call `POST /{waba-id}/subscribed_apps` so the WABA emits webhooks to this app. If templates or inbound never arrive, confirm the WABA is subscribed to the **same** app whose secret and token you use.

---

## 9. End-to-end test plan

1. **Test API Connection** in Settings → green / live.
2. Personal WhatsApp → business/test number: “hello”.
3. Inbox shows a thread. Server: `POST /api/whatsapp/webhook` **200** and `[webhook] inbound processed`.
4. Reply from Inbox. Phone receives it. Failed bubble + code **190** → token, not webhook.
5. Optional: Active automation on `hello` → bot reply. Failures go to **Automations → Logs**.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Webhook GET verify fails | Config not saved; verify token mismatch; app down | Save Settings first; identical string; public HTTPS |
| POST 401 / invalid signature | Wrong or empty `META_APP_SECRET` | App Settings → Basic; restart Node |
| No inbound at all | Callback URL, no `messages` field, number not `/register`’d, events going to another app | Meta Configuration; PIN + save; only one webhook app |
| Inbound OK, send **Authentication Error** / **190** | Temporary or revoked token; wrong app; missing WhatsApp permissions | New System User token; re-save; Test connection |
| Send OK, automations silent | Rule inactive; keyword mismatch; Flow consumed the message; send 190 on the step | Logs; trigger type; disable conflicting flow |
| “Recipient not in allowed list” | Cloud API **test** number allow-list | Add the customer number in API Setup; or go live |
| Decrypt / “token corrupted” | `ENCRYPTION_KEY` changed after save | Restore old key **or** re-enter WhatsApp + AI secrets |
| Phone Number ID already claimed | Another account in this database | Use a different number or that account |
| Template submit image header fails | No `META_APP_ID` | Set env and restart |
| ngrok works then dies | Tunnel URL changed | Update Meta callback URL |

Graph errors in logs look like:

`Authentication Error (HTTP 401, type=OAuthException, code=190, fbtrace_id=…). Hint: the WhatsApp access token is invalid or expired…`

---

## 11. Security

- Never commit `.env.local` or paste System User tokens into git.
- Service role and `ENCRYPTION_KEY` stay on the server.
- Rotating `ENCRYPTION_KEY` orphans encrypted WhatsApp tokens — operators must re-save Settings → WhatsApp.
- Webhook URL is public; HMAC (`META_APP_SECRET`) is what stops spoofed inbound.
- Least privilege: System User needs WhatsApp permissions only, not every Business asset.

---

## 12. Code map

| Path | Role |
| --- | --- |
| `src/app/api/whatsapp/webhook/route.ts` | Verify GET + inbound POST |
| `src/lib/whatsapp/webhook-signature.ts` | HMAC |
| `src/app/api/whatsapp/config/route.ts` | Save, verify, `/register` |
| `src/app/api/whatsapp/config/verify-registration/route.ts` | Diagnostic |
| `src/lib/whatsapp/meta-api.ts` | Graph helpers + error formatting |
| `src/lib/whatsapp/send-message.ts` | Shared outbound (inbox + API) |
| `src/lib/whatsapp/encryption.ts` | Token encrypt/decrypt |
| `src/components/settings/whatsapp-config.tsx` | Settings UI |

Meta Graph version is pinned in `meta-api.ts` (currently **v21.0**).
