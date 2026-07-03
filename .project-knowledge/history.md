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
- **WF1 and WF4 fighting over one Telegram webhook** (see the bug logged
  2026-07-03 above): fixed by merging WF4's 8 non-trigger nodes (`Is it a
  command?` through `Reply - No Pending`) directly into WF1 via copy/paste
  across workflow tabs, wired off `Ignore Commands`' true ("starts with /")
  branch, which previously dead-ended. WF4 was then deactivated (kept, not
  deleted, as a reference/backup — see `workflows/command-listener-deprecated.json`).
  Telegram now has exactly one webhook registered, on WF1. *(fixed: 2026-07-03)*
- Clipboard copy/paste across workflow tabs silently failed via a "Paste"
  context-menu click (likely a browser clipboard-read permission issue) but
  worked fine with a direct Ctrl+V keystroke on the focused canvas. *(noted: 2026-07-03)*
- **`/approve` resumed nothing, repeatedly, with `404: The workflow for
  execution "N" does not contain a waiting webhook with a matching
  path/method`** — this survived a fresh (never-edited-since) execution, a
  matching workflow version, and a WF1 deactivate/reactivate cycle, so none
  of those were the cause. Root cause, found by directly curling the resume
  URL with different HTTP methods: **n8n's `Wait` node (resume: webhook)
  listens for `GET` by default when no `httpMethod` option is set** — WF4's
  `Resume Workflow` node had always sent `POST`, which never matched. A GET
  to the exact same URL returned `200 "Workflow was started"` immediately.
  Fixed by changing `Resume Workflow` to GET with `command`/`instructions`
  as query parameters instead of a JSON body, and updating `Route Command`'s
  three conditions from `$json.body.command` to `$json.query.command`.
  *(fixed: 2026-07-03)*
- One diagnostic GET call made directly (via curl, to isolate the method-
  mismatch bug above) consumed a real paused execution without a real
  command attached, so that specific draft never got a reply/post — not a
  bug, just a side effect of debugging worth remembering before repeating
  that kind of direct test against a live paused execution. *(2026-07-03)*
- After the GET-method fix, `/approve` still didn't post: `Route Command`
  matched nothing and `Update Status` hit
  `pending_approvals_status_check`. Cause: `Extract Command`'s `command`
  field is `message.text.split(' ')[0]`, which for `/approve` keeps the
  leading `/` — but `Route Command` and the status column both expect the
  bare word. Fixed by appending `.replace(/^\//, '')`. *(fixed: 2026-07-03)*
- Same double-`=` expression mistake as above recurred once on this exact
  fix (`command` briefly held `=approve` — a literal `=` prepended to an
  otherwise-correctly-evaluated result) before landing correctly.
  *(fixed: 2026-07-03)*
- End-to-end confirmed working 2026-07-03: a `y: ...` Telegram message
  produces a real Groq-generated post, a Supabase `pending_approvals` row,
  a Telegram preview, and `/approve` genuinely posts it to LinkedIn via
  `Post to LinkedIn` → `Confirm Posted`. First real, human-confirmed
  success of the full loop.
- `Update Status` was writing the raw command word (`approve`/`edit`/`discard`)
  straight into `pending_approvals.status`, which the table's
  `pending_approvals_status_check` constraint rejects (`approve` isn't a
  valid value). Fixed by mapping it explicitly: `approve` → `approved`,
  `discard` → `rejected`, anything else (i.e. `edit`) stays `pending` since
  that path loops back to another preview rather than finalizing.
  *(fixed: 2026-07-03)*
- Generated posts always defaulted to Synax/work/debugging themes regardless
  of what the user actually wrote (e.g. `y: Testing the bot` → a post about
  debugging a Synax workflow) — the system prompt never told the model it
  *could* write about anything else. Fixed by adding an explicit instruction
  to follow the user's actual topic and not force Synax/work into every
  post, in both `Generate LinkedIn Post` and `Rewrite with Instructions`.
  *(fixed: 2026-07-03)*
- Even after the fix above, every generated post opened with the identical
  "Just spent [some time] [doing something] and..." hook — a strong LLM
  default for this kind of prompt that a generic "strong hook" instruction
  doesn't discourage on its own. Fixed by explicitly telling the model to
  vary its opening structure and never use that specific pattern.
  *(fixed: 2026-07-03)*
- Caught while updating `Rewrite with Instructions`' prompt (not yet actually
  hit in testing): it still read `$json.body.instructions`, left over from
  before the Wait-node resume mechanism switched to query params — `/edit`
  would have received `undefined` as the rewrite instructions. Fixed to
  `$json.query.instructions` to match the rest of the flow. *(fixed: 2026-07-03)*
- Both of the two prompt-update edits above (and the `Update Status` fix)
  needed a second attempt each — the first attempt tested against the *old*
  unsaved text every time, since the paste hadn't actually landed. No
  special reason found; re-verifying each field via the DB after every
  "done" (rather than trusting the UI) caught it immediately both times.
  *(2026-07-03)*

---

## Decisions

- n8n's native git Source Control isn't available (Business/Enterprise-only,
  this instance has a 1-entitlement license) — versioning workflows manually
  via `n8n export:workflow --backup` into this repo's `workflows/` folder
  instead. *(2026-07-02)*
- This project was split out of the n8n instance as its own repo rather than
  folded into the existing `Synax-n8n` project — unrelated project, different
  purpose (Synax vs. personal/company social posting). *(2026-07-02)*
- ~~WF1 hands off approval to WF4 via n8n's native `Wait` node (`resume:
  webhook`) as two separate workflows~~ — superseded 2026-07-03: WF4's logic
  was merged directly into WF1 (see Fixed, above) once it became clear two
  workflows can't each hold their own Telegram Trigger against the same bot.
  WF1 is now the single self-contained workflow for the LinkedIn flow; WF4
  is kept inactive as a historical reference only.
  *(decided: 2026-07-02, superseded: 2026-07-03)*
- Decided **not** to build LinkedIn hashtag-popularity lookup. LinkedIn has
  no public API for trending/popular hashtags — the only way to get that
  data would be scraping LinkedIn's own UI, which violates their ToS and
  risks the account now actively used to post. Sticking with the LLM's own
  hashtag judgment (steerable via the system prompt if needed) rather than
  adding a third-party hashtag-analytics API for a marginal quality gain.
  *(2026-07-03)*
