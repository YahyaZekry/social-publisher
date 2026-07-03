# External Integrations & Data Contracts

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03
> Document exact field contracts — never guess the shape.

## Telegram Bot API

- Trigger: `Telegram Trigger` node, `updates: ["message"]`.
- Webhook path: `/webhook/<uuid>/webhook` on the ngrok tunnel.
- Commands are plain `message.text` starting with `/` (checked via `startsWith` in
  the `Is it a command?` IF node) — not Telegram's native `bot_command` entity parsing.
- Owner: single private chat (`chat.type: "private"`) tested so far.
- **Gotcha:** WF4 and WF1 both have their own `Telegram Trigger` node on the *same*
  bot credential. Telegram only allows one webhook URL per bot — whichever workflow
  was activated/saved most recently silently wins, and the other stops receiving
  updates entirely (no error). See the open bug in `roadmap.md` before adding a
  third workflow with its own trigger on this bot.

## Supabase — `pending_approvals` table

- Base URL: `https://xiqohqytcyjiqskkextz.supabase.co/rest/v1/`
- Read via WF4's `Get Pending Approval` node:
  `GET /pending_approvals?telegram_user_id=eq.<id>&status=eq.pending&order=created_at.desc&limit=1&select=*`
- Written via WF1's `Save to Supabase` node:
  `POST /pending_approvals` with body
  `{ telegram_user_id, workflow_type: 'linkedin', resume_url, content_preview, status: 'pending' }`
- Known columns (from usage, not a schema dump): `telegram_user_id` (bigint — must be
  a real number, not the string `"undefined"`), `workflow_type` (`'linkedin'` seen —
  presumably distinguishes Instagram/company-LinkedIn rows once those exist),
  `resume_url` (WF1's `$execution.resumeUrl` — a one-time n8n Wait-node resume link),
  `content_preview` (first 200 chars of the generated post text), `status` (`pending`
  seen; presumably `approved`/`rejected` too), `created_at`.
- Auth: `apikey` header + `Authorization: Bearer <service_role JWT>`, hardcoded on
  both nodes (not an n8n credential — see `roadmap.md` to fix; now duplicated across
  two workflows' committed JSON).
- **Gotcha:** a 0-row response is a 0-item n8n output, which silently stops the whole
  downstream branch (no error, no reply). `Get Pending Approval` has
  **"Always Output Data" enabled** to work around this — do not disable it.
- **Gotcha:** n8n expressions only evaluate if the stored value starts with a literal
  `=` immediately followed by `{{ ... }}` — anything else (missing `=`, doubled `=`,
  `{{ }}` alone) is silently treated as a literal string, which reads as a valid
  request but produces wrong/`null` values or, here, a PostgREST `PGRST102 Empty or
  invalid json` when the "expression" resolves into the JSON body untouched.

## Groq (LLM — LinkedIn post generation)

- Used by WF1's `Generate LinkedIn Post` and `Rewrite with Instructions` nodes.
- `POST https://api.groq.com/openai/v1/chat/completions`, OpenAI-compatible chat
  format, model `llama-3.3-70b-versatile`, `max_tokens: 600`.
- Auth: `Authorization: Bearer <Groq API key>` header, hardcoded on both nodes
  (not an n8n credential — see `roadmap.md`).
- System prompt fixes Yahya's voice/length/hashtag rules for personal LinkedIn;
  user message is either the raw topic (`Extract Content.content`) or, for edits,
  the original post + `$json.body.instructions` concatenated.
- Response shape consumed: `choices[0].message.content` (plain text post, no
  JSON/structured output requested).

## LinkedIn (personal)

- Used by WF1's `Post to LinkedIn` node, fired only after `/approve`.
- `POST https://api.linkedin.com/v2/ugcPosts`, `X-Restli-Protocol-Version: 2.0.0`.
- Auth: `Authorization: Bearer <LinkedIn access token>` header, hardcoded on the
  node (not an n8n credential — see `roadmap.md`; expiry/refresh behavior unknown,
  also tracked in `roadmap.md`).
- Body: `author: 'urn:li:person:kvFSUycND7'`, `lifecycleState: 'PUBLISHED'`,
  `specificContent['com.linkedin.ugc.ShareContent'].shareCommentary.text` = the
  approved post text, `shareMediaCategory: 'NONE'`,
  `visibility['com.linkedin.ugc.MemberNetworkVisibility']: 'PUBLIC'`.
- Not yet exercised for real — blocked on the webhook-contention bug (`roadmap.md`).

## Instagram / LinkedIn (company)

- Not integrated yet. No credentials, no nodes, no data contract defined.
- When built: document the exact publish payload shape here before wiring the node.
