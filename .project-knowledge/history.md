# History

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03
> Past-only. Append-only — never delete entries.

## Fixed

- n8n logged "X-Forwarded-For header is set but trust proxy is false" → already
  resolved by `N8N_PROXY_HOPS=1` on the container before this session started;
  confirmed zero occurrences across full log history. *(fixed: on/before 2026-07-02)*
- Telegram webhook registration retry storm: workflow activation hit Telegram's
  429, retried, then hit an n8n bug (`TypeError: Cannot read properties of
  undefined (reading 'lastTimeout')` in `active-workflow-manager.ts:828`) which
  rolled the workflow back to inactive. Fixed by clearing the Telegram webhook
  (`deleteWebhook`) and recreating the `n8n` container cleanly (same env vars,
  `n8n_data` volume untouched). *(fixed: 2026-07-02)*
- After the container recreation, publishing the workflow failed with
  "User attempted to access a workflow without permissions" — the browser's
  session had been invalidated by the restart. Fixed by logging out/back in.
  n8n also self-corrected the account's global role from `global:member` to
  `global:owner` around the same time (cause not fully confirmed, but matches
  n8n's known owner-repair-on-login behavior). *(fixed: 2026-07-02)*
- `Get Pending Approval` node returned 401 "Invalid API key": the `apikey`
  header was truncated (only the JWT payload segment, missing header/signature),
  and `Authorization` was missing the `Bearer ` prefix. Fixed by setting both
  headers to the full service_role JWT, with `Bearer ` prefix only on
  `Authorization`. *(fixed: 2026-07-02)*
- `Extract Command` (Set node) had four fields declared with no `value`
  expression, so `telegram_user_id` etc. were always `undefined` downstream,
  causing a Postgres `invalid input syntax for type bigint: "undefined"` error.
  Fixed by wiring each field to `$json.message.*` expressions. *(fixed: 2026-07-02)*
- `/approve` produced no Telegram reply even though the execution showed
  "success": `Get Pending Approval` returned `[]` (genuinely no pending row for
  that user), and in n8n a 0-item node output silently stops every downstream
  node — including the "not found" reply branch, which never got a chance to
  run. Fixed by enabling "Always Output Data" on `Get Pending Approval so the
  `Has Pending?` branch always evaluates. *(fixed: 2026-07-02)*
- WF1 couldn't publish: `Send Preview`, `Confirm Posted`, `Send Updated Preview`,
  and `Discard` (all Telegram nodes) had no credential attached. Fixed by
  attaching the same shared `Telegram account` credential (id `ZkWfYSlTSfJ42VwR`)
  used by WF4. *(fixed: 2026-07-03)*
- WF1's `Extract Content` had the exact same bug class as WF4's old `Extract
  Command`: fields declared with no value expression, so `chat_id` /
  `telegram_user_id` / `content` all evaluated to the literal string
  `"undefined"`. Fixed by wiring `$json.message.chat.id`,
  `$json.message.from.id`, and `$json.message.text.replace(/^y:\s*/i, '')`.
  *(fixed: 2026-07-03)*
- `Save to Supabase` and five other downstream nodes (`Send Preview`,
  `Post to LinkedIn`, `Confirm Posted`, `Rewrite with Instructions`,
  `Send Updated Preview`) all referenced a node called `Extract Post Text`
  that had never actually been added to the canvas — `Generate LinkedIn Post`
  connected directly to `Save to Supabase`, so `$json.post_text` didn't exist
  and `.substring()` on it produced an empty request body (Supabase:
  `PGRST102 Empty or invalid json`). Fixed by adding a Set node named
  `Extract Post Text` between them, pulling `chat_id` / `telegram_user_id`
  from `Extract Content` and `post_text` from `$json.choices[0].message.content`.
  *(fixed: 2026-07-03)*
- Getting the `Extract Content` / `Extract Post Text` expressions to actually
  save correctly took ~5 rounds due to n8n's expression rule: a parameter only
  evaluates as an expression if the raw stored string starts with a literal
  `=` immediately followed by `{{ ... }}`. Every partial version — `{{ }}`
  with no `=`, `==` (an extra `=` typed on top of the one already needed),
  or the two similarly-named nodes' values pasted into each other — silently
  produced a literal fixed string instead of erroring, which is what made
  each attempt look plausible until re-checked against the DB. Worth
  remembering for any future Set/Edit Fields node work. *(fixed: 2026-07-03)*

---

## Decisions

- n8n's native git Source Control isn't available (Business/Enterprise-only,
  this instance has a 1-entitlement license) — versioning workflows manually
  via `n8n export:workflow --backup` into this repo's `workflows/` folder
  instead. *(2026-07-02)*
- This project was split out of the n8n instance as its own repo rather than
  folded into the existing `Synax-n8n` project — unrelated project, different
  purpose (Synax vs. personal/company social posting). *(2026-07-02)*
- WF1 hands off approval to WF4 via n8n's native `Wait` node (`resume: webhook`)
  rather than its own polling: WF1 pauses at `Wait for Approval` and writes
  its one-time `$execution.resumeUrl` into the `pending_approvals.resume_url`
  column. WF4's `Resume Workflow` node is meant to POST to that URL once the
  user approves, which resumes WF1's paused execution directly into
  `Route Command`. This only works if WF4's Telegram Trigger is actually the
  one receiving updates — see the webhook-contention bug in `roadmap.md`.
  *(2026-07-03)*
