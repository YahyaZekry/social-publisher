# External Integrations & Data Contracts

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-10 (Instagram publish race condition fixed, stale-webhook-registration bug fixed)
> Document exact field contracts — never guess the shape.

## Telegram Bot API

- Trigger: `Telegram Trigger` node, `updates: ["message", "callback_query"]`
  (the `callback_query` entry was added 2026-07-08 for the button menu — see
  below). WF1 has the *only* trigger — WF4 and WF2 used to have their own
  too, but Telegram only allows one webhook per bot, so their logic was
  merged into WF1 (see `history.md`). **Any future platform must follow this
  same pattern.**
- Webhook path: `/webhook/<uuid>/webhook` on the ngrok tunnel.
- Owner: single private chat (`chat.type: "private"`) tested so far.
- **Interaction model is now entirely inline-keyboard buttons, not typed
  commands (redesigned 2026-07-08).** Every preview (LinkedIn and Instagram)
  carries a 5-button `inline_keyboard`: regenerate-both, regenerate-image-only,
  regenerate-text/caption-only, Post it, Discard. Typed `/approve`, `/edit`,
  `/regenerate*`, `/discard` **no longer do anything on either platform** —
  the old typed-command listener chain (`Ignore Commands` → `Is it a
  command?` → `Extract Command` → `Get Pending Approval` → `Has Pending?` →
  `Resume Workflow` → `Update Status`) still exists but is now dead code, kept
  only until `roadmap.md`'s cleanup TODO is done.
- **How a button tap actually resumes a paused execution:** tapping a button
  sends a `callback_query` update (not a `message`) — `Is Callback Query?`
  branches on `$json.callback_query` existing right after the Trigger, before
  `Ignore Commands`, so the two paths never collide. `Extract Callback Data`
  pulls `chat_id`/`telegram_user_id`/`command` (=`callback_query.data`, e.g.
  `li_approve`, `ig_regen_caption`) and fans out to two things in parallel:
  `Answer Callback Query` (resource `callback`, operation `answerQuery`,
  `queryId` = `callback_query.id` — stops the button's loading spinner, and
  since 2026-07-08 also sets `additionalFields.text` to a short per-command
  message like "🔄 Regenerating caption..." so the tap is visibly
  acknowledged even before the actual regenerate finishes) and a **separate,
  parallel** `Get Pending Approval (Callback)` → `Has Pending? (Callback)` →
  `Resume Workflow (Callback)` → `Update Status (Callback)` chain that mirrors
  the old typed-command listener's Supabase lookup/resume logic exactly, just
  reading from `Extract Callback Data` instead of `Extract Command`. Kept as
  a **fully separate node chain** rather than merged into the old one — see
  the node-group gotcha below for why.
- **n8n node-group validation gotcha (hit repeatedly building the button
  menu, 2026-07-08):** this workflow's `nodeGroups` (the colored
  canvas-organization boxes — e.g. "Linkedin Personal", "Telegram Bot",
  "Instagram") are enforced server-side on every API `PUT`: each group's
  member-induced subgraph must be a single weakly-connected component with
  **at most one member receiving external input** (a "root" with no
  in-group predecessor) **and at most one member sending external output**.
  Concretely this means: a member node can *never* have both an in-group
  predecessor and an out-of-group predecessor at the same time (same for
  successors) — inserting *any* new node between two existing members from
  an outside source trips `"must form a single connected subgraph with a
  single entry and exit"` (`invalid-subgraph` in the underlying
  `validateNodeSelectionForGrouping` check, in
  `n8n-workflow/dist/cjs/node-grouping-validation.js`). This is why the
  callback-resume chain above is a fully separate, ungrouped set of nodes
  rather than merging into `Get Pending Approval` etc. — merging would give
  that node both an internal predecessor (`Extract Command`) and an external
  one (`Extract Callback Data`), which always fails. When adding new nodes
  to an existing group, either make sure they only touch the group at
  exactly the one existing entry/exit point, or leave them out of the group
  entirely (grouping is purely cosmetic, never required for a node to
  function).
- **Gotcha (resolved, keep in mind for any future workflow):** don't give a
  second workflow its own `Telegram Trigger` on this same bot credential —
  whichever workflow activates/saves most recently silently wins the webhook,
  and the other stops receiving updates with no error at all.
- **Gotcha (fixed 2026-07-10): repeated API-driven structural edits to an
  already-*active* workflow can leave the webhook's internal registration
  stale.** Symptom: Telegram delivers the message fine (confirmed via
  ngrok's request inspector — correct path, correct
  `X-Telegram-Bot-Api-Secret-Token`, valid update JSON), n8n responds `200`,
  but **no execution is created at all** — the response body is a bare,
  malformed-looking `firstEntryJson` string instead of the normal ack. Silent
  or subtle, not represented anywhere in `docker logs` (a genuinely
  successful execution logs nothing either, so log silence doesn't confirm
  success). Root cause not confirmed with certainty, but strongly correlated
  with having just pushed several structural `PUT`s (new nodes/connections)
  to the workflow via the Public API while it stayed continuously active —
  n8n normally only fully re-registers a webhook's execution plan on the
  inactive→active transition, not on every subsequent update. **Fix:**
  `POST /api/v1/workflows/:id/deactivate` then `POST
  /api/v1/workflows/:id/activate` — confirmed working immediately after (a
  real Telegram message produced a normal execution + reply within
  seconds). **Going forward: after any batch of structural API edits to an
  already-active workflow, deactivate/reactivate before trusting it's live**,
  the same way saving in the UI implicitly would.
- **Debugging technique (introduced 2026-07-10): ngrok's local inspector
  API** (`http://localhost:4040/api/tunnels` for tunnel status,
  `http://localhost:4040/api/requests/http?limit=N` to list recent requests,
  `http://localhost:4040/api/requests/http/<id>` for the full raw
  request/response including headers and base64-encoded body) shows exactly
  what hit the tunnel — including the full Telegram update JSON — without
  ever needing to fabricate a synthetic test webhook against the live
  workflow (which is off-limits here, see `history.md`'s Decisions). This is
  how the stale-registration bug above was actually confirmed: real user
  messages, inspected after the fact, rather than a simulated one.
- **Gotcha (fixed 2026-07-08):** all prefix/command checks (`Ignore
  Commands`, `Is it a command?`, `Has y: prefix?`, `Has ig: prefix?`,
  `Extract Content (IG)`, `Extract Raw Post Text`) used to only look at
  `message.text`. A photo sent *with a caption* puts that text in
  `message.caption` instead — `message.text` is entirely absent on those
  messages, so a photo+caption message used to fall through every prefix
  check and just get the reminder text. Fixed by having every one of those
  checks read `($json.message.text || $json.message.caption || '')` instead
  of `$json.message.text` alone.
- **File download** (used for Instagram's user-photo path, built
  2026-07-08): this Telegram node version (confirmed from the installed
  node's source, `Telegram.node.js`) has a `resource: "file"`, `operation:
  "get"` mode that takes a `fileId` parameter — with its `download` option
  set to `true`, it fetches the file's path via Telegram's API and downloads
  the actual binary in one node (`Download Photo (IG)`), no separate HTTP
  calls needed. The photo's `file_id` comes from the *last* entry in
  `message.photo[]` (Telegram sends multiple resolutions smallest-first, so
  `photo[photo.length - 1]` is the highest-res one) — see `Get Largest Photo
  File ID`.
- **Inline keyboard JSON shape** (confirmed from the installed node's source,
  not guessed): `replyMarkup: "inlineKeyboard"` plus a top-level
  `inlineKeyboard` param shaped `{ rows: [ { row: { buttons: [ { text,
  additionalFields: { callback_data } } ] } }, ... ] }` — one `{row:...}`
  entry per keyboard row, one or more buttons per row. Answering a callback
  query is `resource: "callback"`, `operation: "answerQuery"`, `queryId`,
  `additionalFields: { text, show_alert, cache_time, url }` (`text` is
  capped at 200 characters by Telegram's API).

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
- **Gotcha (cost a lot of debugging time, fixed 2026-07-05):** every branch
  that can lead back to a second approval decision must loop back to the
  `Wait` node — a dead-end branch doesn't just stop harmlessly, it finishes
  the execution entirely. `Send Updated Preview` (fired after `/edit`) had
  no outgoing connection, so the execution finished right there instead of
  re-entering `Wait for Approval`. The stale `pending_approvals` row still
  pointed at that now-finished execution's resume URL, so the *next*
  `/approve` or `/edit` failed with `409: "The execution 'N' has finished
  already"` — silently, no Telegram reply, and easy to misread as a
  token/signature problem. The same resume URL/signature stays valid across
  multiple loops through the same execution, so no extra Supabase update
  was needed after wiring the loop-back — just the missing connection.

## Supabase — `pending_approvals` table

- Base URL: `https://xiqohqytcyjiqskkextz.supabase.co/rest/v1/`
- Read via `Get Pending Approval` node:
  `GET /pending_approvals?telegram_user_id=eq.<id>&status=eq.pending&order=created_at.desc&limit=1&select=*`
- Written via `Save to Supabase` node:
  `POST /pending_approvals` with body
  `{ telegram_user_id, workflow_type: 'linkedin', resume_url, content_preview, status: 'pending' }`
- Updated via `Update Status` node:
  `PATCH /pending_approvals?id=eq.<id>` with body
  `{ status: <mapped status>, edit_instructions }` — `command` is mapped
  (`approve`→`approved`, `discard`→`rejected`, anything else stays `pending`)
  rather than written raw, since the `pending_approvals_status_check`
  constraint rejects bare command words like `approve`.
- Known columns (from usage, not a schema dump): `telegram_user_id` (bigint — must be
  a real number, not the string `"undefined"`), `workflow_type` (`'linkedin'` seen —
  presumably distinguishes Instagram/company-LinkedIn rows once those exist),
  `resume_url` (`$execution.resumeUrl` — a one-time n8n Wait-node resume link),
  `content_preview` (first 200 chars of the generated post text), `status`
  (`pending`/`approved`/`rejected` confirmed used; exact full constraint
  definition still not directly inspected, see `roadmap.md`), `created_at`.
- Auth: `apikey` header + `Authorization: Bearer <service_role JWT>`, hardcoded on
  all three nodes above (not an n8n credential — see `roadmap.md` to fix; this
  key was exposed in this repo's first commit and should be rotated regardless
  of redaction — see `roadmap.md`).
- **Gotcha:** a 0-row response is a 0-item n8n output, which silently stops the whole
  downstream branch (no error, no reply). `Get Pending Approval` has
  **"Always Output Data" enabled** to work around this — do not disable it.
- **Gotcha:** n8n expressions only evaluate if the stored value starts with a literal
  `=` immediately followed by `{{ ... }}` — anything else (missing `=`, doubled `=`,
  `{{ }}` alone) is silently treated as a literal string, which reads as a valid
  request but produces wrong/`null` values or a PostgREST `PGRST102 Empty or
  invalid json`/check-constraint error depending on where it ends up.

## Groq (LLM — post/caption/image-prompt generation)

- **`y:`/`ig:` no longer auto-draft on entry (redesigned 2026-07-08).**
  Originally `y: <topic>`/`ig: <topic>` always ran the topic through Groq
  immediately. Now both post the typed text/caption **verbatim** and only
  call Groq when the "Regenerate text/caption" or "...+ image" button is
  tapped afterward. `Generate LinkedIn Post` (the old always-on-entry
  LinkedIn drafting node) is now dead/deleted — `Rewrite with Instructions`
  (and its clone `Rewrite with Instructions (Both)`, used by the
  regenerate-both button so image generation can chain off the fresh text)
  handles 100% of LinkedIn text generation now, always reading its rewrite
  seed from `Extract Raw Post Text` (the sole LinkedIn entry point now,
  regardless of whether the text ever actually gets rewritten). Instagram's
  `Generate Caption` was already reusable for this via `Skip Caption?` (see
  `features.md`); its trigger condition flipped from "skip only when
  regenerating the image" to "skip *unless* explicitly regenerating the
  caption or both" so a verbatim caption is the default and AI-writing one
  is opt-in.
- Used by 4 nodes now: WF1's `Rewrite with Instructions` (+ its clone
  `Rewrite with Instructions (Both)`) for LinkedIn, and `Build Image Prompt`
  (also cloned as `Build Image Prompt (LI)` for LinkedIn's own image
  generation). `Generate Caption` (Instagram) moved off Groq to OpenRouter —
  see new subsection below.
- `POST https://api.groq.com/openai/v1/chat/completions`, OpenAI-compatible chat
  format, model `llama-3.3-70b-versatile`. LinkedIn nodes: `max_tokens: 600`.
  Instagram: `Build Image Prompt` uses `max_tokens: 200`.
- Auth: `Authorization: Bearer <Groq API key>` header, hardcoded on all
  remaining Groq nodes (not an n8n credential — see `roadmap.md`).

### OpenRouter (Instagram caption generation, switched 2026-07-12)

- `Generate Caption` (Instagram's caption-writing node, still same node —
  only its API target changed) now calls
  `POST https://openrouter.ai/api/v1/chat/completions` with
  `model: 'openai/gpt-4o-mini'`, `max_tokens: 700` (unchanged from the Groq
  version — long reflective caption + hashtags still needs the headroom).
  Response shape is identical to Groq/OpenAI (`choices[0].message.content`),
  so `Set Post Data` (which reads `$node["Generate Caption"].json.choices[0]
  .message.content`) needed no changes.
- Auth: `Authorization: Bearer <OpenRouter API key>` header, hardcoded on
  this one node (same "not yet an n8n credential" pattern as the Groq/Meta/
  LinkedIn keys — see `roadmap.md`).
- System prompt content (Yahya's voice/length/hashtag rules) carried over
  unchanged from the Groq version — it's plain OpenAI-format chat messages,
  which gpt-4o-mini follows natively, so no rewrite was needed beyond the
  API/model swap. All other Groq-backed nodes (LinkedIn text, both image
  prompts) are unaffected and still run on Llama via Groq.
- LinkedIn system prompt: Yahya's voice/length/hashtag rules, plus explicit
  instructions to (a) write about whatever topic the user actually gave it
  rather than defaulting to Synax/work content, and (b) vary the opening
  line rather than always starting with "Just spent [time]..." — both real,
  observed failure modes before the 2026-07-03 prompt update (see
  `history.md`).
- **Grounding + anti-slop rules** (added 2026-07-05, both `Generate LinkedIn
  Post` and `Rewrite with Instructions` share the identical system prompt):
  surfaced specifically on the `/edit`-with-no-instructions path, where the
  model started fabricating specifics never in the input (e.g. "audited my
  work processes", "slashed hours to minutes") wrapped in generic AI-slop
  phrasing ("hidden taxes", "trenches of", "crash course", "blind spots")
  and always ending on the same reflective-question structure. Research
  (hallucination-mitigation + AI-slop/cliché literature, see `history.md`)
  pointed at two concrete fixes rather than vague "sound more human"
  instructions: (1) an explicit grounding rule — only use facts/details
  actually present in the input, never invent numbers/timelines/backstory,
  and a short/vague input should produce a short/honest post rather than a
  padded one; (2) a **named** banned-phrase list (naming the exact clichés
  outperforms a generic "avoid AI jargon" instruction) plus an instruction
  to vary the ending instead of always closing with a rhetorical question.
  `Rewrite with Instructions`'s user-message also got a ternary: when
  `$json.query.instructions` is empty (bare `/edit`), it now explicitly
  demands a different hook/structure/metaphor instead of a synonym-level
  reword of the same idea. **Confirmed working live 2026-07-05.**
- Instagram system prompts (`Build Image Prompt`, `Generate Caption`):
  **working well as of 2026-07-04**, after a harder rewrite than the
  LinkedIn one needed. `Build Image Prompt` is structured specifically
  around how FLUX (Pollinations' underlying model) actually responds —
  natural-language prose rather than keyword tags, subject stated first
  (FLUX weighs earlier words more heavily), explicit camera/lighting/shot
  vocabulary, plus one worked example (Llama 3.3 responds better to
  few-shot examples than to abstract "don't do X" rules alone — this is
  what the first two prompt rounds were missing). `Generate Caption` adds
  an explicit hashtag-format rule (no space after `#`, no hyphens,
  CamelCase — both were silently breaking hashtag parsing) and a flexible
  length rule (short by default, up to ~2000 characters for a topic that
  calls for real reflection) instead of a rigid character cap.
- **Hashtag counts** (updated 2026-07-04 based on actual platform data —
  not guesses): both platforms now target **3-5 highly specific hashtags**,
  never more. LinkedIn (`Generate LinkedIn Post`, `Rewrite with
  Instructions`) was already at 3-5. Instagram (`Generate Caption`) was
  wrongly set to 15-20 — Instagram enforces a **hard 5-hashtag cap** as of
  Dec 2025, and both platforms' own engagement data favor fewer, more
  specific tags over many generic ones. All three prompts now also specify
  the exact formatting rule (no space after `#`, no hyphens, CamelCase for
  multi-word tags) since both errors silently break hashtag parsing on
  their respective platforms.
- No hashtag-popularity data source for either platform — the model picks
  hashtags from its own training knowledge; a real trending-hashtags lookup
  was considered and declined for LinkedIn (no public API for it, would need
  ToS-violating scraping, see `history.md`) — same reasoning would apply to
  Instagram.
- User message for LinkedIn is either the raw topic (`Extract
  Content.content`) or, for edits, the original post + `$json.query.instructions`
  concatenated. For Instagram, `Build Image Prompt` and `Generate Caption`
  both read from `$node["Extract Content (IG)"]`/`$node['Extract Image
  Prompt']` respectively (not `$json` directly) so the `/regenerate` loop-back
  reaches the original topic correctly.
- Response shape consumed: `choices[0].message.content` (plain text, no
  JSON/structured output requested).

## Pollinations (Instagram image generation)

- Used by `Generate Image (Pollinations)`.
- `GET https://image.pollinations.ai/prompt/<url-encoded image prompt>?width=1080&height=1080&nologo=true&model=flux`
  — no auth/API key needed. Response format set to `file` (binary). **No
  negative-prompt parameter exists on this endpoint** — anything to avoid has
  to be steered via the positive prompt text itself, there's no `negative:`
  field to hand it.
- Prompt comes from `Build Image Prompt`'s Groq output (short vivid scene
  description, no text-in-image).
- **Uncanny/disturbing realistic faces (fixed 2026-07-05):** FLUX (like most
  diffusion models) frequently distorts faces/hands/anatomy on detailed
  photorealistic human portraits. Research confirmed the fix isn't a realism
  dial — people prefer AI art that's clearly stylized *or* clearly photoreal,
  never the semi-realistic middle. Since there's no negative prompt param
  (above), `Build Image Prompt`'s system prompt now has an explicit rule:
  when a person appears in the scene, prefer them faceless/at a
  distance/mid-motion/from behind, or switch the whole scene to a named
  illustration/digital-art style instead of photorealistic camera language.

## Cloudinary (image hosting for Instagram)

- Used by `Upload to Cloudinary`.
- `POST https://api.cloudinary.com/v1_1/cbolcssr/image/upload`, multipart
  form: `file` (binary from Pollinations), `upload_preset: 'ttloffm8'`
  (an *unsigned* preset — not a secret, safe to have client-side/in a public
  repo).
- Response's `secure_url` field is the public HTTPS URL handed to both
  Telegram (`Send Preview (IG)`) and Instagram's Graph API.

## LinkedIn (personal)

- Used by WF1's `Post to LinkedIn` (text-only) / `Post to LinkedIn (Image)`
  nodes, fired only after the "Post it" button (`li_approve`).
- `POST https://api.linkedin.com/v2/ugcPosts`, `X-Restli-Protocol-Version: 2.0.0`.
- Auth: `Authorization: Bearer <LinkedIn access token>` header, hardcoded on
  every LinkedIn node (not an n8n credential — see `roadmap.md`;
  expiry/refresh behavior unknown, also tracked in `roadmap.md`).
- Text-only body: `author: 'urn:li:person:kvFSUycND7'`, `lifecycleState:
  'PUBLISHED'`, `specificContent['com.linkedin.ugc.ShareContent']
  .shareCommentary.text` = the current post text, `shareMediaCategory:
  'NONE'`, `visibility['com.linkedin.ugc.MemberNetworkVisibility']:
  'PUBLIC'`.
- **Confirmed working live 2026-07-03** — first real post published this way.
- **Image support (added 2026-07-08):** LinkedIn posts are text-only by
  default — an image is only attached if the "Regenerate image" or
  "Regenerate text+image" button was tapped at least once. Unlike Instagram,
  this still uses the older (but confirmed still-functional) unversioned
  `v2/assets` Assets API rather than the newer `/rest/images` Images API,
  since that's the API generation this account's token was already granted
  under — mixing generations risks an incompatible asset-URN type. Flow:
  (1) `Register LinkedIn Upload` — `POST
  https://api.linkedin.com/v2/assets?action=registerUpload` with
  `registerUploadRequest: { owner, recipes: ['urn:li:digitalmediaRecipe:feedshare-image'],
  serviceRelationships: [{identifier: 'urn:li:userGeneratedContent',
  relationshipType: 'OWNER'}] }` → returns `value.asset` (the asset URN) and
  `value.uploadMechanism['com.linkedin.digitalmedia.uploading.MediaUploadHttpRequest'].uploadUrl`.
  (2) `Attach LI Image Binary` (Code node) re-attaches the Pollinations
  binary onto the current item via `$('Generate Image (LI
  Pollinations)').item.binary.data` — needed because the HTTP Request node in
  between doesn't carry input binary through to its own output. (3) `Upload
  Image to LinkedIn` — `PUT` the binary straight to that `uploadUrl`
  (`contentType: "binaryData"`, `inputDataFieldName: "data"`) — LinkedIn
  requires the *same* OAuth bearer token on this PUT too (unlike video
  uploads, which don't). (4) `Post to LinkedIn (Image)` references
  `shareMediaCategory: 'IMAGE'`, `media: [{status: 'READY', media: <asset
  URN>, title: {text: 'Image'}}]`. **The generated image is always shown
  back in Telegram (`Send Preview (LI with Image)`, `sendPhoto` with
  `binaryData: true` reusing the same Pollinations binary) before any
  `/approve`-equivalent is even possible** — an earlier version of this
  build skipped that and published an image straight to LinkedIn without
  ever showing it first, which produced a genuinely bad (disturbing) image
  the user had to delete — see `history.md`. No prompt can fully guarantee a
  good generation every time, so the show-before-publish step is the actual
  safety net, not the prompt quality.

## Instagram — Meta Graph API (personal, via Facebook Page "Bear Code")

- Used by `Create Media Container` → `Publish to Instagram` (two-step publish,
  fired only after `/approve`).
- Key IDs: Meta App (BearCode) `1528187988974071` · Facebook Page ID
  (Graph API, **not** the number in the page's `profile.php?id=` URL)
  `1175585338972824` · Instagram Business Account ID `17841476339271624` ·
  Instagram username `yahya_zekry`.
- **Step 1**: `POST /v20.0/17841476339271624/media` with `image_url`
  (Cloudinary's `secure_url`), `caption`, `access_token` → returns `creation_id`
  (as `$json.id`).
- **Step 2**: `POST /v20.0/17841476339271624/media_publish` with
  `creation_id`, `access_token`.
- **Publish race condition (fixed 2026-07-10):** calling step 2 immediately
  after step 1 sometimes failed with `400 Media ID is not available` /
  `"The media is not ready for publishing, please wait for a moment"` (Meta
  error code 9007, subcode 2207027) — Instagram processes the container
  (fetching/transcoding the image from `image_url`) asynchronously, and the
  container isn't always ready the instant `/media` returns. Fixed by
  inserting a poll between the two steps: `Wait Before Media Check` (3s,
  `resume: timeInterval`) → `Check Media Status` (`GET
  /v20.0/<container-id>?fields=status_code`) → `Track Poll Attempt` (Set
  node, increments a `poll_attempt` counter via the same self-referential
  try/catch pattern used elsewhere — `$node["Track Poll Attempt"]` reading
  its own most recent prior run within the execution, defaulting to `1` the
  first time) → `Route Media Status` (Switch: `status_code == 'FINISHED'` →
  `Publish to Instagram`; `status_code == 'IN_PROGRESS' && poll_attempt < 8`
  → loop back to `Wait Before Media Check`; anything else, via
  `options.fallbackOutput: 'extra'` → `Notify Media Failed (IG)`, a Telegram
  message telling the user to retry instead of a silent failure). Since
  `Publish to Instagram` is now fed from outside the `Instagram` node group
  (the polling chain deliberately sits outside every group, same pattern as
  the callback-resume chain), its body expression had to change from
  `$json.id` to `$node['Create Media Container'].json['id']` — the Switch
  node passes through `Track Poll Attempt`'s output, which no longer has an
  `id` field. `Publish to Instagram` and `Confirm Posted (IG)` were removed
  from the `Instagram` node group for the same reason (an external
  predecessor would otherwise make one of them a second group "root",
  which n8n's node-group validator rejects — see the node-group validation
  gotcha under Telegram Bot API above, and `history.md`'s Decisions for how
  this was verified locally before pushing).
- Auth: `access_token` is a **long-lived Page Access Token** (hardcoded
  directly in the request body on both nodes, not a header — same
  "move to a credential" TODO as everything else, see `roadmap.md`),
  derived via the `fb_exchange_token` flow from a long-lived user token.
  Good until **~Sept 2026** (tied to the underlying user token's expiry,
  not a short-lived token) — refresh steps documented separately by the
  user outside this repo. The *first* token used here was short-lived and
  expired same-day; don't confuse the two failure modes if this breaks
  again: "Session has expired" (auth/expiry) vs. "Cannot parse access token"
  (malformed value — e.g. stray whitespace, seen once during setup).
- **Confirmed working live 2026-07-04** — first real Instagram post
  published this way (content quality issues aside, see `roadmap.md`).
- **User's own photo (added 2026-07-08):** `ig: <caption>` sent as a photo's
  caption (not plain text) now uses that actual photo instead of generating
  one — `Has Photo? (IG)` checks `$node["Telegram Trigger"].json.message.photo`
  right after `Extract Content (IG)`; if present, `Get Largest Photo File ID`
  → `Download Photo (IG)` → `Upload User Photo to Cloudinary` feeds the same
  `Set Post Data`/`Save to Supabase (IG)`/`Send Preview (IG)` path the
  AI-generated-image branch already used, so the existing 5-button menu
  (including "Regenerate image only", which swaps in an AI-generated image
  instead) works unchanged. `Set Post Data`'s `cloudinary_url` field tries
  `Upload to Cloudinary` (AI path) first, falling back to `Upload User Photo
  to Cloudinary` (user-photo path) via try/catch, since exactly one of the
  two ever runs per execution.

## LinkedIn (company)

- Not integrated yet. No credentials, no nodes, no data contract defined.
- When built: document the exact publish payload shape here before wiring
  the node. Note from `roadmap.md`: the prompt for this one will likely
  want to *always* talk about Synax/work, the opposite of the personal
  LinkedIn prompt's "don't force it" instruction.
