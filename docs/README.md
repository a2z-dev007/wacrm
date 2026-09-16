# wacrm documentation

wacrm is a self-hostable CRM for the official **WhatsApp Business Cloud API**. One WhatsApp number is shared by a team: inbox, contacts, pipelines, broadcasts, automations, conversation flows, and an optional bring-your-own-key AI assistant.

This folder is the in-repo source of truth for how the product works. Hosted marketing docs live at [wacrm.tech/docs](https://wacrm.tech/docs).

## Pick a starting point

| Audience | Read this |
| --- | --- |
| **First-time install** (engineer + admin) | [Setup guide](./setup-guide.md) |
| **WhatsApp / Meta Cloud API** | [WhatsApp Meta integration](./whatsapp-meta.md) |
| Operators, CS, sales, admins (non-engineering) | [Product & dashboard guide](./product-guide.md) |
| Engineering (how the system is built) | [Architecture](./architecture.md) |
| Engineering (run, deploy, debug) | [Engineering & operations](./engineering.md) |
| Integrations / backend scripts | [Public API](./public-api.md) |
| AI assistants (Claude, Cursor, …) | [MCP server](./mcp.md) |

## One-page mental model

```
Customer WhatsApp  →  Meta Cloud API  →  POST /api/whatsapp/webhook
                                              │
                                              ├─ save contact + conversation + message
                                              ├─ Flows (menus / bots) — if they consume the message, skip most automations
                                              ├─ Automations (rules)
                                              └─ AI auto-reply (optional; only if no flow consumed the message)

Agent in browser   →  Inbox / API / Automations / Flows
                                              │
                                              └─ POST graph.facebook.com  (needs a valid Permanent Access Token)
```

**Inbound can work while outbound fails.** Webhooks prove Meta can *reach you*. Sending (inbox, automations, broadcasts) proves your **access token** and Phone Number ID are valid.

## Roles at a glance

| Role | Typical person | Can do |
| --- | --- | --- |
| **Owner** | Founder / IT owner | Everything, including transfer ownership |
| **Admin** | Team lead | Settings, members, WhatsApp, templates, API keys |
| **Agent** | Support / sales | Inbox, contacts, deals, broadcasts, automations |
| **Viewer** | Stakeholder | Read only |

Details and examples: [Product guide → Team & roles](./product-guide.md#team-and-roles).
