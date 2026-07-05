# External Integrations & Data Contracts

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-05 (targeted Instagram regenerate + anti-uncanny-face image prompt)
> Document exact field contracts — never guess the shape.

## Telegram Bot API

- Trigger: `Telegram Trigger` node, `updates: ["message"]`. WF1 now has the
  *only* one — WF4 and WF2 used to have their own too, but Telegram only
  allows one webhook per bot, so their logic was merged into WF1 (see
  `history.md`). **Any future platform must follow this same pattern.**
- Webhook path: `/webhook/<uuid>/webhook` on the ngrok tunnel.
- Commands are plain `message.text` starting with `/` (checked via `startsWith`) —
  not Telegram's native `bot_command` entity parsing on the *receiving* side.
- Owner: single private chat (`chat.type: "private"`) tested so far.
- **Tappable commands need no BotFather setup (discovered 2026-07-05):** this
  bot has **zero** commands registered via BotFather's `/setcommands` — the
  `/approve`, `/edit`, `/regenerate` etc. words that appear tappable in
  Telegram are just Telegram's own client auto-detecting any `/word` pattern
  inside message text as a `bot_command` entity and linkifying it, confirmed
  from actual Telegram API responses on our own outbound preview messages
  (e.g. `Send Updated Preview`'s response included
  `{"offset":209,"length":8,"type":"bot_command"}` matching where `/discard`
  sits in the text). **Important side effect:** tapping one of these
  auto-linked words sends *only that word* — Telegram does not let you tap
  then keep typing more text after it. This is why a shared `/regenerate
  <target>` command relying on typed trailing text (`instructions`) fights
  against the tappable UX; dedicated single-word commands (`/regenerateImage`,
  `/regenerateCaption`) work with it instead of against it — see
  `features.md` and `history.md`.
- **Gotcha (resolved, keep in mind for any future workflow):** don't give a
  second workflow its own `Telegram Trigger` on this same bot credential —
  whichever workflow activates/saves most recently silently wins the webhook,
  and the other stops receiving updates with no error at all.
- **Gotcha (open, see `roadmap.md`):** all prefix checks (`Has y: prefix?`,
  `Has ig: prefix?`) only look at `message.text`. A photo sent *with a
  caption* puts that text in `message.caption` instead — `message.text` is
  entirely absent on those messages, so a photo+caption message currently
  falls through every prefix check and just gets the reminder text.
- **File download**: this Telegram node version (confirmed from the
  installed node's source, `Telegram.node.js`) has a `resource: "file"`,
  `operation: "get"` mode that takes a `fileId` parameter — with its
  `download` option set to `true`, it fetches the file's path via Telegram's
  API and downloads the actual binary in one node, no separate HTTP calls
  needed. Relevant for the planned "post user's own photo" feature (see
  `roadmap.md`) — the photo's `file_id` comes from
  `message.photo[n].file_id` (Telegram sends multiple resolutions; last one
  is highest-res).

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

- Used by 4 nodes: WF1's `Generate LinkedIn Post` + `Rewrite with
  Instructions` (LinkedIn), and `Build Image Prompt` + `Generate Caption`
  (Instagram).
- `POST https://api.groq.com/openai/v1/chat/completions`, OpenAI-compatible chat
  format, model `llama-3.3-70b-versatile`. LinkedIn nodes: `max_tokens: 600`.
  Instagram: `Build Image Prompt` uses `max_tokens: 200`, `Generate Caption`
  uses `max_tokens: 700` (bumped from 400 so a longer reflective caption +
  hashtags doesn't get truncated).
- Auth: `Authorization: Bearer <Groq API key>` header, hardcoded on all four
  (not an n8n credential — see `roadmap.md`).
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

## LinkedIn (company)

- Not integrated yet. No credentials, no nodes, no data contract defined.
- When built: document the exact publish payload shape here before wiring
  the node. Note from `roadmap.md`: the prompt for this one will likely
  want to *always* talk about Synax/work, the opposite of the personal
  LinkedIn prompt's "don't force it" instruction.
