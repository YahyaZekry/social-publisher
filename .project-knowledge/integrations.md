# External Integrations & Data Contracts

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03
> Document exact field contracts — never guess the shape.

## Telegram Bot API

- Trigger: `Telegram Trigger` node, `updates: ["message"]`. WF1 now has the
  *only* one — WF4 used to have its own too, but Telegram only allows one
  webhook per bot, so its logic was merged into WF1 (see `history.md`).
- Webhook path: `/webhook/<uuid>/webhook` on the ngrok tunnel.
- Commands are plain `message.text` starting with `/` (checked via `startsWith`) —
  not Telegram's native `bot_command` entity parsing.
- Owner: single private chat (`chat.type: "private"`) tested so far.
- **Gotcha (resolved, keep in mind for any future workflow):** don't give a
  second workflow its own `Telegram Trigger` on this same bot credential —
  whichever workflow activates/saves most recently silently wins the webhook,
  and the other stops receiving updates with no error at all.

## n8n `Wait` node webhook resume (used by `Wait for Approval`)

- Resume URL: `$execution.resumeUrl`, format
  `https://.../webhook-waiting/<executionId>?signature=<hash>`.
- **Gotcha (cost a lot of debugging time):** with no `httpMethod` option set,
  the Wait node's resume webhook only accepts **`GET`** — a `POST` to the
  exact correct URL/signature fails with `404: The workflow for execution
  "N" does not contain a waiting webhook with a matching path/method`, which
  reads like a stale-registration or wrong-execution error but isn't. Pass
  any data needed on resume (`command`, `instructions`) as **query
  parameters**, read on the other side as `$json.query.<name>`, not
  `$json.body.<name>`.
- The resume URL/signature itself is stable and matches exactly what the
  Wait node reports in its own execution metadata — if a resume 404s, check
  the HTTP method before suspecting the URL or workflow version.

## Supabase — `pending_approvals` table

- Base URL: `https://xiqohqytcyjiqskkextz.supabase.co/rest/v1/`
- Read via `Get Pending Approval` node:
  `GET /pending_approvals?telegram_user_id=eq.<id>&status=eq.pending&order=created_at.desc&limit=1&select=*`
- Written via `Save to Supabase` node:
  `POST /pending_approvals` with body
  `{ telegram_user_id, workflow_type: 'linkedin', resume_url, content_preview, status: 'pending' }`
- Updated (partially — see `roadmap.md` bug) via `Update Status` node:
  `PATCH /pending_approvals?id=eq.<id>` with body
  `{ status: <bare command word>, edit_instructions }`.
- Known columns (from usage, not a schema dump): `telegram_user_id` (bigint — must be
  a real number, not the string `"undefined"`), `workflow_type` (`'linkedin'` seen —
  presumably distinguishes Instagram/company-LinkedIn rows once those exist),
  `resume_url` (`$execution.resumeUrl` — a one-time n8n Wait-node resume link),
  `content_preview` (first 200 chars of the generated post text), `status`
  (`pending` confirmed valid; a `pending_approvals_status_check` constraint
  rejects at least `approve` — exact allowed values not yet confirmed, see
  `roadmap.md`), `created_at`.
- Auth: `apikey` header + `Authorization: Bearer <service_role JWT>`, hardcoded on
  all three nodes above (not an n8n credential — see `roadmap.md` to fix).
- **Gotcha:** a 0-row response is a 0-item n8n output, which silently stops the whole
  downstream branch (no error, no reply). `Get Pending Approval` has
  **"Always Output Data" enabled** to work around this — do not disable it.
- **Gotcha:** n8n expressions only evaluate if the stored value starts with a literal
  `=` immediately followed by `{{ ... }}` — anything else (missing `=`, doubled `=`,
  `{{ }}` alone) is silently treated as a literal string, which reads as a valid
  request but produces wrong/`null` values or a PostgREST `PGRST102 Empty or
  invalid json`/check-constraint error depending on where it ends up.

## Groq (LLM — LinkedIn post generation)

- Used by WF1's `Generate LinkedIn Post` and `Rewrite with Instructions` nodes.
- `POST https://api.groq.com/openai/v1/chat/completions`, OpenAI-compatible chat
  format, model `llama-3.3-70b-versatile`, `max_tokens: 600`.
- Auth: `Authorization: Bearer <Groq API key>` header, hardcoded on both nodes
  (not an n8n credential — see `roadmap.md`).
- System prompt fixes Yahya's voice/length/hashtag rules for personal LinkedIn;
  user message is either the raw topic (`Extract Content.content`) or, for edits,
  the original post + `$json.query.instructions` concatenated.
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
- **Confirmed working live 2026-07-03** — first real post published this way.

## Instagram / LinkedIn (company)

- Not integrated yet. No credentials, no nodes, no data contract defined.
- When built: document the exact publish payload shape here before wiring the node.
