# Product and dashboard guide

For operators, support, sales, and account admins. No code required.

After you sign in you land on the **dashboard**. The left sidebar is the product. This guide walks every area with **what it is**, **who uses it**, and a **worked example**.

New install (env, database, Meta, webhook): **[Setup guide](./setup-guide.md)**.

---

## Sign in and first hour

1. Open the app URL (local default after `npm run dev` is often `http://localhost:3000` or the next free port). See the [setup guide](./setup-guide.md) if the app is not running yet.
2. **Sign up** with email + password (minimum 6 characters). There is no built-in demo user.
3. Confirm email if your Supabase project requires it.
4. An **account** is created for you. You are the **owner**.
5. Go to **Settings → WhatsApp** and connect the Cloud API (admin/owner). Until this is saved and the token is valid:
   - You can still click around the UI.
   - Inbound messages appear only after Meta’s webhook points at this install **and** the number is registered.
   - Outbound sends (inbox, automations, broadcasts) fail with a Meta authentication error.

### Example — first successful conversation

1. Admin saves Phone Number ID, WABA ID, **permanent** System User token, and webhook verify token.
2. Meta App → WhatsApp → Configuration: callback URL = `{your origin}/api/whatsapp/webhook`, same verify token, subscribe to **messages**.
3. A customer texts the business number: “Hi, I want pricing.”
4. Inbox shows a new thread. An agent replies in the composer.
5. The customer sees the reply on WhatsApp. Delivery ticks update in the thread when Meta sends status webhooks.

---

## Dashboard (`/dashboard`)

Home screen for volume and pipeline health. Numbers are **account-scoped** (everyone on the team sees the same workspace data, subject to RLS).

| Widget | What it shows | Example |
| --- | --- | --- |
| Open conversations | Threads currently `open` | 12 open; +3 vs yesterday |
| New contacts today | Contacts created since local midnight | 4 new leads from WhatsApp |
| Open pipeline value | Sum of **open** deal values | `$18,400` in the default currency |
| Agent messages today | Outbound messages with `sender_type = agent` | 47 replies sent today |
| Conversations chart | Volume over 7 / 30 / 90 days | Spike on Monday after a broadcast |
| Pipeline donut | Open deals by stage | 8 in “Qualified”, 2 in “Won” |
| Response time | How quickly agents reply | Most replies under 15 minutes |
| Activity feed | Recent automations, messages, deals | “Welcome Message ran for +1415…” |
| Quick actions | Shortcuts | New automation, new broadcast |

**Example:** A team lead opens Dashboard Monday morning, sees open conversations jumped from 8 to 22 after a weekend broadcast, and drills into **Inbox** to assign overflow.

---

## Inbox (`/inbox`)

Shared WhatsApp inbox for the connected number. One conversation per contact.

### Conversation list

- Filter by status: **open**, **pending**, **closed**.
- Unread counts on the sidebar.
- Search / select a thread to open the transcript.

### Thread

- Inbound bubbles from the customer; outbound from **agent** or **bot** (automations / flows / AI).
- Status on outbound: sending → sent → delivered → read (when Meta reports it), or **failed**.
- Reply in-thread (quote a message).
- Assign the conversation to a teammate (fires `conversation_assigned` automations).
- Send **text**, **media** (image / video / document / audio / voice note), **interactive buttons or lists**, **templates** (outside the 24-hour window you generally need a template).
- **Quick replies**: canned text or interactive snippets from Settings.
- **AI draft**: if AI is configured, generate a suggested reply; the agent edits and sends (human in the loop).

### Contact sidebar (on the conversation)

- Name, phone, tags, custom fields.
- Linked **deals**.
- **Notes** (internal; not sent to WhatsApp).

### Example — 24-hour window vs template

1. Customer messages at 10:00. You can send free-form text until 10:00 the next day (Meta’s customer service window).
2. At 11:00 the next day, a free-form “Just checking in” is rejected by Meta.
3. Agent sends the approved template `order_update` with parameter `A-1042` instead.

### Example — assignment

Support queue is unassigned. Maria assigns the thread to herself. An automation “When conversation assigned → add tag `in-progress`” runs.

---

## Notifications (`/notifications`)

In-app alerts for the signed-in user (assignment, mentions of activity you care about, depending on what the product emits). Unread badge on the sidebar. Open an item to jump to the related conversation or page.

**Example:** You are off the inbox tab; a teammate assigns you a VIP chat. Bell badge increments; you open Notifications and click through to Inbox.

---

## Contacts (`/contacts`)

Directory of people who messaged you or were imported.

- Create a contact with an **E.164** phone (`+14155550123`).
- Edit name, email, company, tags, custom fields.
- CSV import (phone required; name, email, company, tags optional).
- Deduping: same phone in the account is treated as one person (find-or-create).
- Open WhatsApp thread from the contact (find-or-create conversation).
- Send a **template** to a contact who has no recent inbound (business-initiated).

### Example — CSV import

CSV:

```csv
phone,name,email,company,tags
+14155550123,Jane Doe,jane@acme.com,Acme,"vip,wholesale"
+14155550124,Sam Lee,,Lee Co,lead
```

After import, Jane appears once even if she later texts from WhatsApp; the inbound message attaches to the same contact.

---

## Pipelines (`/pipelines`)

Kanban sales board. Deals belong to a **pipeline** and a **stage**, optionally linked to a contact / conversation.

- Drag cards between stages.
- Deal value uses the account **default currency** (Settings → Deals).
- Admin/owner manage pipeline and stage names in Settings.

### Example

Pipeline “New business”: stages Lead → Qualified → Proposal → Won / Lost.

A inbound “We want 200 units” creates a contact. Agent creates a deal `$12,000` in Qualified, linked to that contact. Dashboard donut and open-value card update.

---

## Broadcasts (`/broadcasts`)

One-to-many **template** sends (Meta requires approved templates for bulk / business-initiated).

1. **New broadcast** — pick template + language.
2. Choose recipients (contacts / tags / paste numbers) and per-row variables.
3. Launch. Status moves sending → sent. Delivered / read counts rise as Meta status webhooks arrive.
4. If a recipient replies, the inbox thread is the same conversation; broadcast “replied” accounting can increment.

**Limits:** Public API caps **1000 recipients per request**; the UI follows the same idea of batching large lists.

### Example

Template `promo_july` body: `Hi {{1}}, 20% off this week.`

| Recipient | `{{1}}` |
| --- | --- |
| +14155550123 | Jane |
| +14155550124 | Sam |

Jane taps through and replies “yes”; Inbox shows her thread; an automation on keyword `yes` can add tag `promo-interested`.

---

## Automations (`/automations`)

If/then rules. **Not** a full chatbot canvas — for menus and branching bots use **Flows**.

### Triggers

| Trigger | Fires when |
| --- | --- |
| New message received | Any inbound (unless a Flow consumed it) |
| First message from contact | That contact’s first-ever inbound |
| Keyword match | Inbound text contains / equals keywords |
| Button / list reply | Customer tapped an interactive id you set |
| New contact created | Webhook auto-created the contact |
| Conversation assigned | Agent assignment changed |
| Tag added | A tag was applied (including by another automation, with a chain cap) |
| Time-based | Cron expression (needs the Wait/cron worker for schedules that wait) |

### Steps

Send message / buttons / list / template, add or remove tag, assign conversation, update contact field, create deal, wait, condition (if/else), webhook, close conversation.

### Templates you can start from

- Welcome message (first inbound)
- Out of office
- Lead qualifier
- Follow-up reminder

Toggle **Active** to run. **Logs** (`/automations/{id}/logs`) show each run: success / partial / failed and per-step detail (including Meta errors).

### Example — keyword FAQ

1. Trigger: Keyword match, keywords `hours`, `open`, match type contains (not case-sensitive).
2. Step: Send message `We're open Mon–Fri 9:00–18:00 IST.`
3. Customer texts “What are your hours?” → automation sends the reply as **bot**.
4. If the send token is invalid, the log step fails with Meta code 190; the inbound message still appears in Inbox.

### Example — welcome + tag

1. Trigger: First message from contact.
2. Send “Thanks for reaching out — a human will join shortly.”
3. Add tag `new-lead`.
4. Create deal in pipeline “Inbound” stage “Lead” with title from the contact name.

### Wait steps

A **Wait** parks the run. A scheduler must call `GET /api/automations/cron` with header `x-cron-secret`. Without that, waits never resume. Engineering: [Engineering guide → Cron](./engineering.md#cron-jobs).

### Flows vs automations

If an active **Flow** consumes the inbound (customer is inside a menu), **new_message_received**, **keyword_match**, and **interactive_reply** automations are skipped for that message. First-inbound / new-contact automations can still fire.

---

## Flows (`/flows`) — Beta

Visual WhatsApp bots: buttons, lists, collect input, conditions, tags, handoff to a human, end.

Typical shape: **Start** → **Send buttons** → branches → **Handoff** or **End**.

Triggers: keyword, first inbound, or manual only.

**Runs** (`/flows/{id}/runs`) show the last executions and captured variables.

### Example — welcome menu

1. Trigger keyword `menu`.
2. Send buttons: Pricing / Support / Talk to a person (`ids`: `pricing`, `support`, `human`).
3. Pricing branch sends a price list; Support sends a FAQ; Human **handoffs** (conversation goes to agents, flow stops consuming).
4. Customer later types freely; Inbox agents reply as usual. Automations can run again because the flow is no longer consuming.

---

## AI Agents (`/agents`)

Bring **your own** OpenAI or Anthropic key (encrypted at rest). No global vendor key in env.

| Tab | Purpose |
| --- | --- |
| Setup | Provider, key, system prompt, auto-reply on/off, knowledge base |
| Playground | Test replies before going live |
| Usage | Token / call stats (admin+) |

**Inbox:** “Draft with AI” fills the composer; the agent sends.

**Auto-reply:** Optional. Runs only if a Flow did **not** consume the message. Knowledge base (FAQs, policies) uses Postgres full-text search; semantic search needs `pgvector` plus an embeddings key in settings.

**Handoff:** After a cap of auto-replies, the bot can stop and leave a summary for humans.

### Example

You paste an OpenAI key, add a knowledge article “Return policy: 30 days with receipt.” Customer asks “Can I return this?” Auto-reply cites the policy. After three bot turns, handoff — Maria sees the summary in Inbox.

---

## Settings (`/settings?tab=…`)

Deep-linkable tabs.

### Account group

| Tab | For | Notes |
| --- | --- | --- |
| Overview | Everyone | Shortcuts and live counts |
| Profile | You | Display name, avatar, email |
| Security | You | Password, sessions |
| Appearance | You | Theme |

### Workspace group (mostly admin/owner)

| Tab | For | Notes |
| --- | --- | --- |
| WhatsApp | Connect Cloud API | Phone Number ID, WABA ID, **permanent token**, verify token, 2FA PIN for `/register` |
| Templates | Meta templates | Sync from Meta, submit new, dry-run in dev |
| Quick replies | Canned replies | Text or interactive |
| Fields & tags | Taxonomy | Tags + custom contact fields |
| Deals & currency | Pipelines | Default currency, pipeline setup |
| Team members | Invite / roles | Owner, admin, agent, viewer |
| API keys | Public API | Shown once; hashed at rest |

### Example — invite a teammate

1. Admin → Settings → Members → invite `sam@company.com` as **agent**.
2. Sam opens the invite link, signs up or signs in, redeems.
3. Sam sees the same inbox and contacts. Sam cannot change WhatsApp credentials.

### Example — rotate a leaked API key

1. Settings → API keys → Revoke the old key.
2. Create a new key with only `contacts:read` if the integration is a reporting script.
3. Copy `wacrm_live_…` into the script. The old key fails with `unauthorized` on the next call.

---

## Team and roles

| Capability | Owner | Admin | Agent | Viewer |
| --- | --- | --- | --- | --- |
| Read inbox, contacts, dashboard | ✓ | ✓ | ✓ | ✓ |
| Send messages, move deals, run broadcasts, edit automations | ✓ | ✓ | ✓ | |
| WhatsApp, templates, tags, pipelines, members, API keys | ✓ | ✓ | | |
| Transfer ownership / destroy account | ✓ | | | |

Solo use: you remain owner; no invites needed.

---

## WhatsApp rules operators should know

Deep dive for engineers/admins: **[WhatsApp Meta integration](./whatsapp-meta.md)**.

1. **Permanent System User token** — not the 24-hour “temporary access token” on API Setup. See [Engineering → Meta token](./engineering.md#generate-a-meta-access-token).
2. **Test connection** in Settings after saving. Inbox send is the real check for outbound.
3. **Webhook 200 OK** only means Meta’s POST was accepted. It does not mean your automation replied.
4. **Sandbox / test numbers** vs production: test numbers skip two-step PIN; production numbers need PIN so wacrm can `/register` the number to this app. Otherwise inbound may keep going to a previous app.
5. Interactive **button ids** must match automation `interactive_reply` or Flow edges exactly (`yes` ≠ `Yes`).

---

## Suggested operating playbooks

**Morning stand-up:** Dashboard → open conversations and response time → Inbox unassigned → Pipelines stuck deals.

**New campaign:** Approve template in Meta → Sync templates → Broadcast → watch delivered/read → Inbox for replies → optional keyword automation.

**Incident — customers write, agents cannot reply:** Check Settings → Test API Connection → regenerate System User token → re-save. Inbound history is already in Inbox.

**Incident — automations silent:** Confirm Active; confirm trigger (keyword vs any message); check Logs; confirm no Flow is consuming; confirm cron if using Wait; confirm outbound token if the step is a send.
