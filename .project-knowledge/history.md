# History

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-10 (Instagram publish race condition fixed, stale-webhook-registration bug fixed)
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
- **Hashtag count was wrong for Instagram, right for LinkedIn** — web
  research found Instagram actually enforces a **hard 5-hashtag cap**
  (rolled out Dec 2025) and both platforms' own data favor 3-5 highly
  specific hashtags over many generic ones (Reels with 4 tags got 47% more
  reach than those with 15+; LinkedIn posts with 11+ hashtags see *less*
  engagement than posts with none). `Generate Caption` was asking for 15-20
  hashtags — fixed to 3-5. `Generate LinkedIn Post` and `Rewrite with
  Instructions` were already correctly set to 3-5, just missing the explicit
  hashtag-formatting rule (no space after `#`, no hyphens) that Instagram's
  caption prompt had already needed — added it to both LinkedIn prompts too,
  preventively. *(fixed: 2026-07-04)*
- `/edit` broke `/approve` for every post that had ever been edited: `Send
  Updated Preview` had no outgoing connection at all, so after a `/edit` the
  execution simply finished instead of looping back to `Wait for Approval`.
  A later `/approve` then tried to resume that already-finished execution
  and 409'd with `"The execution 'N' has finished already"` — silently, no
  Telegram reply. Root-caused by comparing `execution_entity.status` across
  the draft execution (finished right after `Send Updated Preview`, no
  further nodes) against the stale `pending_approvals.resume_url` still
  pointing at it. Fixed by wiring `Send Updated Preview → Wait for
  Approval`, the same loop-back pattern the first preview already used.
  *(fixed: 2026-07-05)*
- Instagram-style content-quality bug recurred on LinkedIn's `/edit` path
  specifically: with no instructions given, `Rewrite with Instructions`
  reused the same original post text every time and produced near-duplicate
  synonym-level rewrites (e.g. "inefficiency is invisible" → "inefficiency
  is like a parasite"), then a worse failure mode — fabricating specifics
  never mentioned in the one-line input (e.g. "audited my work processes",
  "slashed hours to minutes") wrapped in generic AI-startup jargon ("hidden
  taxes", "trenches of", "crash course", ending every post with the same
  reflective-question structure). Root-caused via web research on
  hallucination mitigation and AI-slop/cliché avoidance (see
  `integrations.md`). Fixed by adding an explicit grounding rule (only use
  facts present in the input, never invent numbers/timelines/backstory) and
  a named banned-phrase list to both `Generate LinkedIn Post` and `Rewrite
  with Instructions`' system prompts, plus a ternary in `Rewrite with
  Instructions`'s user message so a bare `/edit` explicitly demands a
  different hook/structure/metaphor instead of a synonym-swap. *(fixed:
  2026-07-05)*
- Two paste mistakes surfaced while applying the above fix, both worth
  remembering as a recurring bug class: (1) the user pasted `Generate
  LinkedIn Post`'s body (ending `content: $json.content`) into `Rewrite with
  Instructions` by mistake — `$json.content` doesn't exist on the
  resumed-webhook payload (`{signature, command, instructions}`), so
  `JSON.stringify` silently dropped the `content` key and Groq rejected the
  request (`'messages.1.content' is missing`); (2) a second paste attempt
  left the field's existing leading `=` in place and pasted a second `=` on
  top of it, producing `=={{ ... }}` — n8n strips only one `=`, so the
  *literal* leftover `={{ ... }}` template prepended a stray `=` character
  to the JSON body sent to Groq, breaking JSON parsing entirely (Groq:
  `cannot unmarshal string into Go value of type map[string]interface{}`).
  Both diagnosed by reading the live node's stored `body` param directly
  from the DB rather than guessing. *(fixed: 2026-07-05)*
- `ig:` drafts stopped replying entirely: `Generate Caption`'s Body field had
  the same doubled-`=` corruption (`"=={{ JSON..."`) left over from the
  2026-07-04 hashtag-count edit session, silently un-noticed since Instagram
  hadn't been tested since. Fixed by removing the extra leading `=`. A
  workflow-wide scan for the same `==` pattern across every node's
  `body`/`value`/Set-field values found no other occurrences. *(fixed:
  2026-07-05)*
- Built targeted Instagram regenerate (`/regenerateImage` / `/regenerateCaption`
  redo only one half instead of both) — see `features.md` for the final node
  graph. Three bugs surfaced while building it, all fixed same day: (1) the
  new `Skip Caption?` node referenced `$node["Wait for Approval (IG)"]`
  unconditionally, which throws `"hasn't been executed"` on the very first
  draft pass (before any resume has ever happened) rather than returning
  undefined — fixed by wrapping the reference in a try/catch IIFE that
  defaults to `""`; (2) the same doubled-`=`/stray-space expression-prefix
  bug recurred twice more while wiring the new If nodes (`Skip Caption?` got
  `==`, `Route Regenerate Target` got `= ` with a space) — both silently
  broke their conditions exactly like the earlier `/edit` bug; (3) after
  fixing the prefix issue, `Skip Caption?`'s right-hand comparison value was
  `regenerateimage` (lowercase) while the actual command everywhere else was
  `regenerateImage` (camelCase), and the condition was case-sensitive — so it
  never matched. All three found by reading the live node's stored
  parameters and cross-referencing actual execution run-counts per node
  rather than guessing from symptoms alone. Along the way, `Route Regenerate
  Target` (an earlier, more complex two-hop design using `instructions` text
  matching) was replaced with a simpler direct 5-way switch on `Route Command
  (IG)`'s own `command` value, once it became clear the "clickable" preview
  buttons the user relied on are just Telegram's automatic `bot_command`
  entity detection on any `/word` in message text (confirmed zero BotFather
  commands are registered for this bot) — meaning a shared `/regenerate
  <target>` command relying on typed trailing text was fighting against
  Telegram's own tap-to-send-immediately behavior for command-shaped words,
  while dedicated one-word commands work with it. *(fixed: 2026-07-05)*
- Instagram images occasionally generated disturbing/uncanny realistic human
  faces — a known FLUX/diffusion failure mode, not a bug in this workflow's
  code. Web research confirmed the fix isn't "more" or "less" realism as a
  dial, but avoiding the semi-realistic middle ground entirely (people
  generally prefer AI art that reads as clearly stylized *or* convincingly
  photoreal, never in between). Pollinations' API has no negative-prompt
  parameter, so the fix had to be a new paragraph in `Build Image Prompt`'s
  system prompt: when a person appears, prefer keeping them faceless/at a
  distance/in motion/from behind, or explicitly switch to a stated
  illustration/digital-art style rather than photorealistic camera language.
  *(fixed: 2026-07-05)*
- A LinkedIn post with a generated image published straight to LinkedIn
  without ever being shown to the user first — the preview only ever
  rendered the caption text, never the actual image, so a genuinely
  disturbing FLUX generation went live and had to be deleted after the
  fact. Root design flaw, not just a bad generation: no image-prompt can
  guarantee zero bad outputs, so the real fix was structural — `Send
  Preview (LI with Image)` (`sendPhoto`, `binaryData: true`, reusing the
  same Pollinations binary that was uploaded to LinkedIn's asset library)
  now always shows the real image in Telegram before `/approve`-equivalent
  is even reachable, matching how Instagram's preview already worked.
  *(fixed: 2026-07-08)*
- Immediately after adding a new "HARD RULE" paragraph to both
  `Build Image Prompt` system prompts (banning the specific
  describable-expression + photorealistic-camera-language combination that
  caused the bug above), image generation broke completely on both
  platforms — the new paragraph's own example phrases (`'looking
  skeptical'`, `'smiling'`, etc.) were spliced into the *already-escaped*
  body string with their inner quotes left unescaped, terminating the JS
  string literal early. Same bug class as several earlier fixes this
  project (doubled `=`, stray spaces) — any time new text is spliced into
  an existing escaped expression body, its own quotes need escaping too,
  not just quotes present at initial-construction time. *(fixed: 2026-07-08)*
- Instagram's "Regenerate caption" button crashed silently (no Telegram
  reply, no visible error) on any post that used a user-uploaded photo
  instead of an AI-generated image — `Generate Caption`'s user-message
  content referenced `$node['Extract Image Prompt'].json['content']`
  directly with no fallback, and `Extract Image Prompt` never runs on the
  user-photo path. Fixed with the same try/catch-fallback pattern used
  elsewhere, falling back to `Extract Content (IG)` (which always runs).
  *(fixed: 2026-07-08)*
- Sending a photo with an `ig: <caption>` caption fell through to the
  "start your message with ig:" reminder instead of being recognized — all
  prefix/command checks only read `message.text`, which is empty on photo
  messages (the text lives in `message.caption` instead). This had been
  logged as an open, lower-priority bug in `roadmap.md` since 2026-07-04;
  fixed properly once "post your own photo" was actually built, by having
  every check read `(message.text || message.caption || '')`. *(fixed:
  2026-07-08)*
- Button taps had no visible confirmation beyond the button's own brief
  loading-spinner animation, easy to miss — especially since the actual
  regenerate could take several seconds afterward, making it look like the
  tap hadn't registered at all. Fixed by giving `Answer Callback Query` a
  per-command `additionalFields.text` (a short Telegram toast like "🔄
  Regenerating caption...") instead of an empty acknowledgment. *(fixed:
  2026-07-08)*
- Instagram's `Build Image Prompt` was still only running the *original*
  single-example system prompt (plus the later "HARD RULE" patch spliced on
  top) — the earlier 3-example diversity fix (Example A/B/C: person-not-
  facing-camera photorealistic, no-person still-life photorealistic,
  person-front-and-center illustration-style) had only ever been handed to
  the user as paste-ready text for manual application, and was never
  actually pasted in before the session moved on to other work. LinkedIn's
  equivalent node got the 3-example version from the start since it was
  built fresh via the API. Caught when the user directly asked whether
  image-style variety had actually been added; fixed by porting the exact
  same 3-example structure onto Instagram's node, keeping its
  Instagram-specific wording (e.g. "AI/tech/running imagery" instead of
  LinkedIn's "AI/tech/corporate imagery"). *(fixed: 2026-07-08)*
- **Instagram publish failed with "Media ID is not available"** — user
  reported a post attempt "didn't work"; root-caused via the failed
  execution's error log to a genuine Instagram Graph API race condition:
  `Publish to Instagram` called `/media_publish` immediately after
  `Create Media Container` returned, before Instagram had finished
  asynchronously processing the image (Meta error code 9007, subcode
  2207027, `error_user_msg: "The media is not ready for publishing, please
  wait for a moment"`). A second symptom was a follow-on `409` when the user
  tapped "Post it" again on the same stale, already-errored execution.
  Fixed by inserting a poll-until-ready loop between the two publish steps
  — see `integrations.md`'s Instagram section for the exact node chain.
  Verified locally against a faithful Python re-implementation of n8n's
  actual node-grouping-validation algorithm (read directly from
  `n8n-workflow/dist/cjs/graph/graph-utils.js` inside the container this
  time, not just the earlier paraphrase) before pushing — passed on the
  first live attempt. *(fixed: 2026-07-10)*
- **Real, separate bug: some Telegram messages silently produced zero
  reply and zero execution**, despite n8n returning `200` to Telegram.
  Confirmed genuine (not user error, not a second bot) by inspecting the
  actual raw webhook payloads via ngrok's local request inspector — correct
  secret token, correct path, valid `ig: test` update, but no matching
  execution ever appeared in n8n, and the `200` response body was an
  unexplained bare `firstEntryJson` string. Correlated with this session
  having pushed several structural API `PUT`s to the workflow while it
  stayed continuously active. Fixed by deactivating then reactivating the
  workflow via the Public API, which forces n8n to fully re-register the
  webhook; confirmed fixed by a subsequent real message producing a normal
  execution and reply within seconds. See `integrations.md` for the
  general gotcha this surfaces for future API-driven edits. *(fixed:
  2026-07-10)*
- **Non-issue, resolved by clarification:** a screenshot showing a
  Telegram chat named "The Bear Code" with a long, unrelated slash-command
  menu (`/subagents`, `/prisma_cli`, `/taskflow`, etc.) looked like it might
  be a second system competing for the bot's webhook. Ruled out concretely:
  (1) grepped the entire live workflow JSON for `setMyCommands` and found
  nothing — nothing in this project could have set that menu; (2) confirmed
  only WF1 is active among the 3 workflows in this n8n instance; (3) the
  actual webhook payload for a real `ig: test` message showed the sender's
  Telegram account itself is `is_bot: false, username: "TheBearCode"` — i.e.
  it's the user's own personal account/branding (matches the Meta Facebook
  Page already named "Bear Code" in this same project, see `integrations.md`),
  not a separate bot. The command menu itself was a one-time BotFather
  setting, unrelated to message delivery; the user cleared it directly.
  *(resolved: 2026-07-10)*

---

## Decisions

- Built the 2026-07-08 session's large feature set (LinkedIn image support,
  the unified 5-button menu replacing typed commands on both platforms,
  Instagram's user-photo support) via n8n's **Public REST API**
  (`/api/v1/workflows/:id`, a self-generated API key) instead of the
  manual drag-and-click UI process used for everything before it — this
  session's changes were large enough (60+ nodes touched across several
  passes) that manual per-node UI instructions would have been far slower
  and more error-prone. Every batch of changes was validated locally first
  (a hand-written Python re-implementation of n8n's node-group validator,
  plus expression-syntax checks for the recurring doubled-`=`/unescaped-
  quote bug classes) before pushing, and the live workflow was re-fetched
  and diffed after each push to confirm it landed correctly. Discovered
  along the way that n8n's Public API enforces a **node-group topology
  constraint** on every save (each group must be a single connected
  component with at most one external entry and one external exit) that
  the manual UI apparently doesn't enforce as strictly when dragging nodes
  around by hand — see `integrations.md` for the exact rule and how it
  shaped the button-resume chain's design (kept fully separate from the
  old typed-command listener rather than merged in). Direct curl calls
  against the live LinkedIn API (outside of n8n) were attempted once to
  verify the image-upload mechanism before building it, but stopped after
  one call at the user's/system's request in favor of testing exclusively
  through real n8n executions triggered via Telegram.

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
- Debugging a "message doesn't arrive" report is done via **inspecting real
  user-sent messages** (n8n's execution list/detail via the Public API,
  ngrok's local request inspector for raw payloads) rather than firing a
  synthetic webhook POST at the live workflow to reproduce it — the latter
  would trigger a genuine end-to-end run (image generation, a real Telegram
  message, a path toward an actual publish), which is explicitly off-limits
  per the earlier decision to test exclusively through real Telegram
  messages (see the 2026-07-08 entry above). This boundary held again
  2026-07-10 when a manual reproduction attempt was declined. Extracting
  credentials directly from n8n's local database (rather than asking the
  user for them) is similarly off-limits, even though it's the project's
  own self-hosted instance — ask the user directly for API keys instead.
  *(2026-07-10)*
- LinkedIn text output "felt rigged" — diagnosed as a prompt-content problem,
  not a sampling-parameter problem, after checking Groq's own API reference
  (`presence_penalty`/`frequency_penalty` are documented as accepted but
  silently unsupported on Groq's models). Fixed by applying the same
  few-shot-examples technique already proven on the image prompts
  (2026-07-04) to the LinkedIn text prompts for the first time — a
  banned-phrase blocklist alone is reactive and gives the model nothing
  concrete to sound like instead. `presence_penalty` was added only to
  `Generate Caption`, since that node is the one exception now running on
  OpenRouter/genuine OpenAI infra where the parameter actually works (see
  `integrations.md`, "Prompt quality pass"). Deferred collapsing the two
  byte-identical LinkedIn-text nodes into one shared upstream node — a
  real node-graph change, left for later unless requested. *(2026-07-12)*
