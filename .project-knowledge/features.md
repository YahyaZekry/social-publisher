# Features & Workflows

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-10 (Instagram publish race condition fixed, stale-webhook-registration bug fixed)

## Features

- **LinkedIn post drafting + approval via Telegram** — texting the bot
  `y: <text>` posts that text **verbatim** (no AI drafting on entry — see
  `integrations.md`'s Groq section for why this changed 2026-07-08), shows a
  5-button menu (regenerate text+image / image only / text only / Post it /
  Discard), and only calls Groq if a regenerate button is tapped. LinkedIn
  posts are text-only unless an image is explicitly generated via the menu —
  once one is generated, it's shown back as a real photo in Telegram before
  any publish is possible. **Confirmed working end-to-end live on
  2026-07-03**, redesigned around the button menu + image support
  2026-07-08. *(added: 2026-07-01, confirmed working: 2026-07-03, image
  support + button menu: 2026-07-08)*
- **Instagram post drafting + approval via Telegram** — texting `ig:
  <caption>` posts that caption **verbatim** and auto-generates an image via
  Pollinations (Instagram can't publish without media, so this part stays
  automatic even though the caption doesn't). Sending an actual **photo**
  with `ig: <caption>` as its caption uses that photo instead of generating
  one. Same 5-button menu as LinkedIn (regenerate caption+image / image only
  / caption only / Post it / Discard) replaces the old typed
  `/approve`/`/regenerate*`/`/discard` commands entirely. **Confirmed
  working live end-to-end with genuinely good output quality as of
  2026-07-05**; verbatim-by-default + button menu + user-photo support added
  2026-07-08. *(added: 2026-07-04, quality fixed: 2026-07-04, targeted
  regenerate: 2026-07-05, button menu + verbatim default + user photo:
  2026-07-08)*

---

## Workflows

**WF1 - LinkedIn Post (Personal)** (`workflows/main-workflow.json`)

This is the *only* active workflow, and the single hub for every platform —
it started as just LinkedIn, absorbed WF4's approve-handling logic on
2026-07-03, then absorbed WF2's Instagram-drafting logic on 2026-07-04, each
time because a second workflow with its own `Telegram Trigger` on the same
bot credential silently steals the one available webhook. Both WF4 and WF2
are now inactive, empty/near-empty, kept only as reference exports. **Any
future platform (company LinkedIn, etc.) must follow the same pattern —
merged into this one workflow, never its own standalone one.**

### Entry / prefix detection

1. `Telegram Trigger` — receives `message` **and** `callback_query` updates
   (the latter added 2026-07-08 for the button menu).
2. `Is Callback Query?` (IF) — `$json.callback_query` existing branches
   straight into the **button-resume chain** below; everything else falls
   through to `Ignore Commands` exactly as before 2026-07-08.
3. `Ignore Commands` (IF) — text/caption starting with `/` branches into the
   now-dead typed-command listener (see `roadmap.md` — pending cleanup).
4. `Has y: prefix?` (IF) — `(message.text || message.caption || '')` must
   start with `y:` (case-insensitive; the caption fallback was added
   2026-07-08 to fix photo-caption messages falling through).
   - **No →** `Has ig: prefix?` (same fallback logic) → `Extract Content
     (IG)` or `Ask for prefix (IG)`.
   - **Yes →** `Extract Raw Post Text` (Set) — `chat_id`/`telegram_user_id`
     from `message.chat.id`/`message.from.id`; `post_text` = the text after
     `y:` via `.replace(/^y:\s*/i, '')`, used **verbatim** (this node used to
     be the separate `yraw:` entry point before `y:` and `yraw:` were
     unified 2026-07-08 — `Extract Content`/`Generate LinkedIn Post`/the old
     `Extract Post Text` are all deleted now).

### LinkedIn: preview → button menu → publish

5. `Save to Supabase` (HTTP Request → POST `pending_approvals`) — inserts
   `telegram_user_id`, `workflow_type: 'linkedin'`, `resume_url`
   (`$execution.resumeUrl`), `content_preview`, `status: 'pending'`.
6. `Send Preview` (Telegram, plain text + 5-button `inline_keyboard`:
   `li_regen_both`, `li_regen_image`, `li_regen_text`, `li_approve`,
   `li_discard`) — text is always the **current** post text, resolved as
   `Rewrite with Instructions`'s latest output if it ever ran, else
   `Extract Raw Post Text`'s verbatim text (a try/catch fallback, since
   `Rewrite with Instructions` may never have executed yet).
7. `Wait for Approval` (`Wait`, `resume: webhook`) — pauses; **only resumes
   on GET**, see `integrations.md`.
8. `Route Button (LI)` (Switch on `$json.query.command`, the renamed/
   repurposed old `Route Command` — 5 outputs matching the button values):
   - **`li_approve` →** `Was Image Generated? (LI)` (try/catch check on
     whether `Upload Image to LinkedIn` ever ran this execution) →
     `Post to LinkedIn (Image)` or plain `Post to LinkedIn` → `Confirm
     Posted`.
   - **`li_discard` →** `Discard`.
   - **`li_regen_text` →** `Rewrite with Instructions` (rewrites from the
     *original* `Extract Raw Post Text` seed, never from a previous
     rewrite — the prompt explicitly demands "a genuinely different take"
     on bare regenerate) → `Has Image? (LI)` (try/catch check) → shows
     either `Send Preview` (no image yet) or `Attach Preview Image Binary` →
     `Send Preview (LI with Image)` (an image already exists from an
     earlier regenerate) → loops back to `Wait for Approval`.
   - **`li_regen_image` →** `Build Image Prompt (LI)` directly (skips the
     rewrite).
   - **`li_regen_both` →** `Rewrite with Instructions (Both)` (a clone of
     `Rewrite with Instructions`, kept as a separate node instance so its
     one fixed downstream wire can go straight to image generation instead
     of the has-image branch) → same `Build Image Prompt (LI)` target as
     `li_regen_image` (both converge there; whichever ran most recently
     wins the "current text" reference).
9. Image chain (shared by `li_regen_image` and `li_regen_both`):
   `Build Image Prompt (LI)` (Groq — same anti-uncanny-face system prompt as
   Instagram's, see `integrations.md`, generalized to "LinkedIn post") →
   `Extract Image Prompt (LI)` → `Generate Image (LI Pollinations)` →
   `Register LinkedIn Upload` → `Attach LI Image Binary` → `Upload Image to
   LinkedIn` → `Attach Preview Image Binary` → `Send Preview (LI with
   Image)` (`sendPhoto`, `binaryData: true`, same 5-button menu) → loops
   back to `Wait for Approval`. **The image is always shown here before any
   publish is possible** — see `integrations.md` for why that specific
   ordering matters.

### Instagram: entry → user-photo branch → button menu → publish

10. `Has ig: prefix?` (IF, caption-aware) → `Extract Content (IG)` (Set) —
    `chat_id`/`telegram_user_id`/`content` (the caption, verbatim,
    `.replace(/^ig:\s*/i, '')`).
11. `Has Photo? (IG)` (IF, added 2026-07-08) — checks
    `$node["Telegram Trigger"].json.message.photo` for existence.
    - **Yes (user sent their own photo) →** `Get Largest Photo File ID`
      (Set — grabs `message.photo[photo.length-1].file_id`, the
      highest-res entry) → `Download Photo (IG)` (Telegram, `resource:
      "file"`, `operation: "get"`, `download: true` — fetches and downloads
      in one node) → `Upload User Photo to Cloudinary` (same
      Cloudinary preset as the AI path) → `Set Post Data` directly (skips
      the whole AI-image chain below).
    - **No →** `Build Image Prompt` (Groq — turns `content` into a FLUX
      prompt) → `Extract Image Prompt` → `Generate Image (Pollinations)` →
      `Upload to Cloudinary` → `Skip Caption?`.
12. `Skip Caption?` (IF, condition flipped 2026-07-08: was "skip only when
    regenerating the image", now "skip *unless* the command is
    `ig_regen_caption` or `ig_regen_both`" — caption defaults to verbatim,
    AI-writing one is opt-in via the button menu) — True → `Set Post Data`
    directly. False → `Generate Caption` (Groq; its `content` reference
    tries `Extract Image Prompt` first, falls back to `Extract Content
    (IG)` directly via try/catch — needed because `Extract Image Prompt`
    never runs on the user-photo path) → `Set Post Data`.
13. `Set Post Data` (Set) — `cloudinary_url` tries `Upload to Cloudinary`
    (AI path) first, falls back to `Upload User Photo to Cloudinary`
    (user-photo path) via try/catch; `caption` tries `Generate Caption`
    first, falls back to `Extract Content (IG)`'s verbatim text;
    `chat_id`/`telegram_user_id` reference `Extract Content (IG)` directly
    (simplified 2026-07-08 — it runs on every path, unlike
    `Extract Image Prompt`).
14. `Save to Supabase (IG)` → `Send Preview (IG)` (Telegram `sendPhoto`,
    same 5-button pattern: `ig_regen_both`, `ig_regen_image`,
    `ig_regen_caption`, `ig_approve`, `ig_discard`) → `Wait for Approval
    (IG)` (same GET-only gotcha as LinkedIn's).
15. `Route Command (IG)` (Switch on `$json.query.command`, 5 outputs, values
    updated 2026-07-08 to match the button data instead of old typed
    commands):
    - **`ig_approve` →** `Create Media Container` → the publish-readiness
      poll (below) → `Publish to Instagram` → `Confirm Posted (IG)`.
    - **`ig_regen_both` / `ig_regen_image` →** both loop back to `Build
      Image Prompt` (`Skip Caption?` is what decides whether `Generate
      Caption` also reruns).
    - **`ig_regen_caption` →** straight to `Generate Caption`, image
      untouched.
    - **`ig_discard` →** `Discard (IG)`.

16. Publish-readiness poll (added 2026-07-10, sits between `Create Media
    Container` and `Publish to Instagram`, deliberately kept **outside**
    the `Instagram` node group — same reasoning as the button-resume chain
    below): `Wait Before Media Check` (`Wait`, `resume: timeInterval`, 3s)
    → `Check Media Status` (HTTP GET
    `/v20.0/<container-id>?fields=status_code`, container ID read from
    `$node["Create Media Container"].json["id"]`) → `Track Poll Attempt`
    (Set — `poll_attempt` increments via a self-referential try/catch
    reading its own most recent prior run, defaulting to `1`) → `Route
    Media Status` (Switch): `status_code == 'FINISHED'` → `Publish to
    Instagram`; `status_code == 'IN_PROGRESS' && poll_attempt < 8` → loops
    back to `Wait Before Media Check`; anything else (via
    `fallbackOutput: 'extra'`) → `Notify Media Failed (IG)` (tells the user
    to retry instead of failing silently). `Publish to Instagram`'s body
    now reads the container ID from `$node['Create Media Container']`
    rather than `$json.id`, since `$json` at that point is whatever `Route
    Media Status` passed through. Fixes a real Instagram Graph API race
    condition — see `integrations.md` and `history.md`.

### Button-resume chain (shared by both platforms, parallel to the old typed-command listener)

17. `Is Callback Query?` TRUE → `Extract Callback Data` (Set — `chat_id`,
    `telegram_user_id`, `command` = `callback_query.data`,
    `callback_query_id`) → fans out to:
    - `Answer Callback Query` (Telegram, `resource: callback`, `operation:
      answerQuery`) — stops the button spinner and shows a short toast like
      "🔄 Regenerating caption..." (added 2026-07-08 so a tap is visibly
      confirmed even before the regenerate finishes).
    - `Get Pending Approval (Callback)` → `Has Pending? (Callback)` →
      `Resume Workflow (Callback)` → `Update Status (Callback)` — a
      **separate, parallel** copy of the old typed-command listener's
      Supabase lookup/resume logic, reading from `Extract Callback Data`
      instead of `Extract Command`. Kept fully separate rather than merged
      into the old chain because of an n8n node-group validation
      constraint — see `integrations.md`.

**Old typed-command listener** (was WF4, now dead code pending cleanup —
`roadmap.md`): `Is it a command?` → `Extract Command` → `Get Pending
Approval` → `Has Pending?` → `Resume Workflow` → `Update Status` → `Reply -
Processing` / `Reply - No Pending`. Still reachable via `Ignore Commands`
but nothing routes a real button/command value there anymore since
`Route Button (LI)`/`Route Command (IG)` only match the new button-data
strings.
