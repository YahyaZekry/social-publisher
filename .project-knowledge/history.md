# History

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-02
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

---

## Decisions

- n8n's native git Source Control isn't available (Business/Enterprise-only,
  this instance has a 1-entitlement license) — versioning workflows manually
  via `n8n export:workflow --backup` into this repo's `workflows/` folder
  instead. *(2026-07-02)*
- This project was split out of the n8n instance as its own repo rather than
  folded into the existing `Synax-n8n` project — unrelated project, different
  purpose (Synax vs. personal/company social posting). *(2026-07-02)*
