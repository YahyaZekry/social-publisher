# History

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-04 (Instagram quality fix)
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
- **WF2 - Instagram Post (Personal) built and merged into WF1**, same
  reasoning as the WF4 merge: a second workflow with its own `Telegram
  Trigger` would contend for the one webhook. WF2's 17 non-trigger nodes
  were merged in via a locally-edited, paste-ready JSON fragment (built by
  editing a user-supplied workflow export file directly, not by touching the
  live DB), with 8 colliding node names disambiguated with an `(IG)` suffix.
  WF2 itself was left active=0 with 0 nodes (its content lives in WF1 now).
  *(fixed: 2026-07-04)*
- The user-supplied source file for WF2 used a Set-node field schema
  (`{name, type, value}`) that doesn't match what this n8n instance's
  typeVersion 3 Set node actually expects (`{name, stringValue}`, no `type`
  key) — pasting it in meant `Extract Content (IG)`, `Extract Image Prompt`,
  and `Set Post Data` silently passed through their input completely
  unchanged, contributing none of their defined fields, with no error at
  all. Fixed by having the fields re-entered directly through the node UI
  (not re-pasted as JSON) so n8n serializes them per its own real schema.
  *(fixed: 2026-07-04)*
- Same source file wired `Route Command`'s "regenerate" branch to `Extract
  Content` (now `Extract Content (IG)`) — which expects a Telegram message
  shape, not resume data, and would have crashed on `/regenerate`. Fixed by
  rewiring it to `Build Image Prompt` instead, using
  `$node["Extract Content (IG)"].json.content` (rather than `$json.content`)
  in that node's body so it correctly reaches back to the original topic
  regardless of whether it's the first pass or a regenerate loop-back.
  *(fixed: 2026-07-04)*
- `Send Preview (IG)`'s `sendPhoto` operation had no way to specify which
  photo to send — the source file used a `photoUrl` parameter that doesn't
  exist on this n8n version's Telegram node (confirmed straight from the
  installed node's source: `body.photo = this.getNodeParameter('file', i)`).
  Silently dropped on import, so this went unnoticed until testing produced
  "Bad Request: there is no photo in the request". Fixed by setting the
  correct `file` parameter to
  `={{ $node["Set Post Data"].json["cloudinary_url"] }}` instead.
  *(fixed: 2026-07-04)*
- Instagram/Meta Graph API access token expired mid-session (a short-lived
  token). User generated a proper long-lived Page Access Token (via the
  `fb_exchange_token` flow, good until ~Sept 2026) to replace it in
  `Create Media Container` / `Publish to Instagram` — but the first two
  attempts to paste it in still failed: once because an *old* paused
  execution is permanently bound to the workflow snapshot from when it
  started (confirmed via mismatched `workflowVersionId`), so re-testing the
  same stale draft can never pick up a token change — needs a fresh draft;
  and once because the pasted token had 2 stray leading spaces, which Meta's
  API rejected as "Cannot parse access token". *(fixed: 2026-07-04)*
- First real Instagram post attempt exposed the same "ignores the actual
  topic" problem WF1 had, plus a new one: hashtags were generic filler
  (`#newpost`, `#digitalfootprint`, `#onlinepresence`) that don't match the
  "authentic, raw, no fluff" voice, and generated images defaulted to
  generic corporate stock-photo visuals (city skylines, sunsets) regardless
  of topic. Prompt updates were made to `Generate Caption` (explicit
  anti-generic-hashtag instruction) and `Build Image Prompt` (avoid stock-
  photo defaults, invent one concrete grounded scene) — **user reports these
  are still not good enough as of 2026-07-04; open, see `roadmap.md`.**
  *(attempted: 2026-07-04, not resolved)*
- **Instagram output quality — actually resolved.** Root cause of the image
  problem: the prompt was asking the model for a vague "vivid description"
  with no real technique behind it. Web research on FLUX (the model
  Pollinations uses) prompting found it (a) strongly prefers natural-language
  prose over comma-separated keyword tags, (b) weighs earlier words more
  heavily, so the subject must come first and never get buried, and (c)
  responds well to explicit camera/lighting/shot-type vocabulary
  ("low angle shot", "35mm", "shallow depth of field") rather than vague
  adjectives like "stunning". Separately, Llama 3.3 prompting guidance
  confirmed few-shot examples in the system prompt outperform purely
  abstract "don't do X" instructions. `Build Image Prompt` was rewritten
  around a subject→environment→lighting→camera/style structure with one
  concrete worked example. Confirmed live: a Monster Energy / gaming-desk
  test produced a genuinely well-composed, on-topic image (not stock-photo
  generic). *(fixed: 2026-07-04)*
- Caption still had two issues once the image was fixed: (1) hashtag
  formatting was inconsistent — sometimes a space after `#` (breaks Telegram/
  Instagram hashtag parsing entirely) or a hyphen inside a multi-word tag
  (also breaks it, since Instagram only continues a hashtag through letters/
  numbers/underscores); (2) length was hard-capped under 300 characters with
  no way to write a longer, essay-style caption when a topic actually called
  for one. Fixed `Generate Caption`'s prompt: added an explicit hashtag-
  format rule (no space after `#`, no hyphens, CamelCase) and replaced the
  fixed length cap with "match length to the topic, up to ~2000 characters
  for something that calls for real reflection" — also bumped `max_tokens`
  from 400 to 700 so a longer caption + hashtags doesn't get truncated.
  Confirmed live on a genuinely reflective topic ("the struggle is real...")
  — produced a real short-essay caption with correctly-formatted hashtags
  and a matching, well-composed image. *(fixed: 2026-07-04)*
- Repo's exported workflow file renamed from `linkedin-post-personal.json`
  to `main-workflow.json` — the old name stopped making sense once the file
  covered LinkedIn + Instagram (+ eventually company LinkedIn) in one
  workflow. *(2026-07-04)*
- Git history rewritten (`git filter-branch`, all commits, force-pushed) to
  scrub the real Supabase `service_role` key that had been exposed in the
  first commit before redaction practices started. Verified clean across
  every commit and on GitHub's remote afterward; local backup refs then
  removed and garbage-collected. Done specifically so the repo could be made
  public without carrying that exposure. *(fixed: 2026-07-04)*

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
- Repo confirmed safe to make public on 2026-07-04, after the git-history
  rewrite fully scrubbed the exposed Supabase key and every account-specific
  ID (Supabase ref, LinkedIn URN, Instagram Business Account ID, Cloudinary
  cloud name/preset) was genericized out of the README and workflow
  exports — those aren't classified as secrets but the user didn't want
  them public either. *(2026-07-04)*
