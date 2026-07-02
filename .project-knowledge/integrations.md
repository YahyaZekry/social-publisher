# External Integrations & Data Contracts

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-02
> Document exact field contracts — never guess the shape.

## Telegram Bot API

- Trigger: `Telegram Trigger` node, `updates: ["message"]`.
- Webhook path: `/webhook/<uuid>/webhook` on the ngrok tunnel.
- Commands are plain `message.text` starting with `/` (checked via `startsWith` in
  the `Is it a command?` IF node) — not Telegram's native `bot_command` entity parsing.
- Owner: single private chat (`chat.type: "private"`) tested so far.

## Supabase — `pending_approvals` table

- Base URL: `https://xiqohqytcyjiqskkextz.supabase.co/rest/v1/`
- Read via `Get Pending Approval` node:
  `GET /pending_approvals?telegram_user_id=eq.<id>&status=eq.pending&order=created_at.desc&limit=1&select=*`
- Known columns (from usage, not a schema dump): `telegram_user_id` (bigint — must be
  a real number, not the string `"undefined"`), `status` (`pending` seen; presumably
  `approved`/`rejected` too), `created_at`.
- Auth: `apikey` header + `Authorization: Bearer <service_role JWT>`, both hardcoded
  on the node (not an n8n credential — see `roadmap.md` to fix).
- **Gotcha:** a 0-row response is a 0-item n8n output, which silently stops the whole
  downstream branch (no error, no reply). `Get Pending Approval` has
  **"Always Output Data" enabled** to work around this — do not disable it.

## Instagram / LinkedIn (personal + company)

- Not integrated yet. No credentials, no nodes, no data contract defined.
- When built: document the exact publish payload shape here before wiring the node.
