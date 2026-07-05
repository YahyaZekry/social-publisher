# Features & Workflows

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-05 (targeted Instagram regenerate + anti-uncanny-face image prompt)

## Features

- **LinkedIn post drafting + approval via Telegram** — texting the bot
  `y: <topic>` generates a personal LinkedIn post with Groq, sends it back as
  a preview, and waits; `/approve` posts it for real, `/edit <instructions>`
  regenerates it, `/discard` cancels. **Confirmed working end-to-end live on
  2026-07-03** — first real post published this way. *(added: 2026-07-01,
  confirmed working: 2026-07-03)*
- **Instagram post drafting + approval via Telegram** — texting `ig: <topic>`
  generates an AI image (Pollinations) + caption (Groq), previews both, and
  `/approve` posts to Instagram for real; `/discard` cancels. Regenerating
  is now three separate commands: `/regenerate` redoes both image and
  caption, `/regenerateImage` redoes only the image (caption untouched),
  `/regenerateCaption` redoes only the caption (image untouched) — see the
  Instagram draft half below for how targeting works. **Confirmed working
  live end-to-end with genuinely good output quality as of 2026-07-05**,
  including a fix for FLUX generating disturbing/uncanny realistic human
  faces — see `history.md`. *(added: 2026-07-04, quality fixed: 2026-07-04,
  targeted regenerate + anti-uncanny-face fix: 2026-07-05)*

---

## Workflows

**WF1 - LinkedIn Post (Personal)** (`workflows/main-workflow.json` — filename
updated 2026-07-04 since it now covers every platform, not just LinkedIn)

This is now the *only* active workflow, and the single hub for every
platform — it started as just LinkedIn, absorbed WF4's approve-handling
logic on 2026-07-03, then absorbed WF2's Instagram-drafting logic on
2026-07-04, each time because a second workflow with its own `Telegram
Trigger` on the same bot credential silently steals the one available
webhook. Both WF4 and WF2 are now inactive, empty/near-empty, kept only as
reference exports (`command-listener-deprecated.json`,
`instagram-post-personal-deprecated.json`). **Any future platform (company
LinkedIn, etc.) must follow the same pattern — merged into this one
workflow, never its own standalone one.**

**Draft half:**
1. `Telegram Trigger` — receives `message` updates via webhook.
2. `Ignore Commands` (IF) — `message.text` starting with `/` branches into the
   **approve half** below instead of the drafting flow.
3. `Has y: prefix?` (IF) — `message.text` must start with `y:` (case-insensitive).
   - **No →** `Ask for prefix` (Telegram) — sends usage instructions back.
   - **Yes →** continues to step 4.
4. `Extract Content` (Set) — `chat_id`, `telegram_user_id` from `message.chat.id` /
   `message.from.id`; `content` = the text after `y:` via
   `message.text.replace(/^y:\s*/i, '')`.
5. `Generate LinkedIn Post` (HTTP Request → Groq `chat/completions`,
   `llama-3.3-70b-versatile`) — writes the post from `content` using a fixed
   system prompt (voice/length/hashtag rules for Yahya's personal brand).
6. `Extract Post Text` (Set) — `chat_id` / `telegram_user_id` carried forward from
   `Extract Content`; `post_text` = `$json.choices[0].message.content` (the
   generated post).
7. `Save to Supabase` (HTTP Request → POST `pending_approvals`) — inserts
   `telegram_user_id`, `workflow_type: 'linkedin'`, `resume_url` (`$execution.resumeUrl`),
   `content_preview` (first 200 chars of `post_text`), `status: 'pending'`.
8. `Send Preview` (Telegram) — sends the full generated post back with
   `/approve`, `/edit [instructions]`, `/discard` instructions.
9. `Wait for Approval` (`Wait` node, `resume: webhook`) — pauses this execution;
   its one-time resume URL is what got stored in step 7. **Only resumes on a
   `GET` request** — this is n8n's default for the Wait node's webhook mode
   when no HTTP method is configured; see `integrations.md` gotcha.
10. `Route Command` (Switch on `$json.query.command`) — branches on the query
    string of whatever request resumed it:
    - **`approve` →** `Post to LinkedIn` (HTTP Request → LinkedIn `ugcPosts` API)
      → `Confirm Posted` (Telegram).
    - **`edit` →** `Rewrite with Instructions` (HTTP Request → Groq, same model,
      rewrites the post per `$json.query.instructions`) → `Send Updated Preview`
      (Telegram) — loops back to another approve/edit/discard decision.
    - **`discard` →** `Discard` (Telegram) — confirms cancellation.

**Instagram draft half** (was WF2, merged in off `Has y: prefix?`'s false branch):
10a. `Has ig: prefix?` (IF) — `message.text` starting with `ig:`.
    - **No →** `Ask for prefix (IG)` (Telegram) — usage instructions.
    - **Yes →** `Extract Content (IG)` (Set) — same pattern as `Extract Content`,
      `ig:` instead of `y:`.
11a. `Build Image Prompt` (HTTP Request → Groq) — turns the topic into a short
    image-generation prompt; reads `$node["Extract Content (IG)"].json.content`
    (not `$json.content`) so this also works correctly when reached via the
    `/regenerate` loop-back, not just the first pass.
12a. `Extract Image Prompt` (Set) — carries `image_prompt`, `chat_id`,
    `telegram_user_id`, `content` forward.
13a. `Generate Image (Pollinations)` (HTTP Request, GET, `response.responseFormat:
    "file"`) — `https://image.pollinations.ai/prompt/<encoded prompt>` returns
    the image as binary.
14a. `Upload to Cloudinary` (HTTP Request, multipart form) — uploads the binary,
    unsigned upload preset `ttloffm8`; response's `secure_url` is the public URL
    Instagram's API needs.
15a. `Generate Caption` (HTTP Request → Groq) — writes the Instagram caption +
    hashtags from `$node['Extract Image Prompt'].json['content']`.
16a. `Set Post Data` (Set) — `cloudinary_url`, `caption`, `chat_id`,
    `telegram_user_id` carried forward.
17a. `Save to Supabase (IG)` (HTTP Request → POST `pending_approvals`) — same
    shape as LinkedIn's, `workflow_type: 'instagram'`.
18a. `Send Preview (IG)` (Telegram, `sendPhoto`) — the `file` parameter (not
    `photoUrl` — that field doesn't exist on this node version, see
    `integrations.md`) is the Cloudinary URL; caption includes
    `/approve` / `/regenerate` / `/discard` instructions.
19a. `Wait for Approval (IG)` (`Wait`, `resume: webhook`, same GET-only gotcha
    as the LinkedIn one).
20a. `Route Command (IG)` (Switch on `$json.query.command`, 5 outputs as of
    2026-07-05):
    - **`approve` →** `Create Media Container` → `Publish to Instagram`
      (Graph API two-step publish) → `Confirm Posted (IG)`.
    - **`regenerate` →** loops back to `Build Image Prompt` — regenerates
      both image and caption from the same original topic.
    - **`regenerateImage` →** also loops back to `Build Image Prompt` (same
      target as bare `regenerate`), but `Skip Caption?` (see below) stops it
      short of `Generate Caption` — only the image changes.
    - **`regenerateCaption` →** goes straight to `Generate Caption`, skipping
      the whole image chain entirely — only the caption changes, the
      existing `cloudinary_url` is reused untouched.
    - **`discard` →** `Discard (IG)`.
    - `Send Preview (IG)`'s text advertises all three regenerate variants;
      Telegram auto-links any `/word` pattern in message text as a tappable
      command with **zero BotFather registration needed** — this project has
      no BotFather commands configured at all, confirmed 2026-07-05.
21a. `Skip Caption?` (If node, sits between `Upload to Cloudinary` and
    `Generate Caption`) — condition checks
    `$node["Wait for Approval (IG)"].json.query.command == 'regenerateImage'`,
    wrapped in a try/catch IIFE since this node also runs on the very first
    draft pass, before `Wait for Approval (IG)` has ever executed (referencing
    it directly there throws `"hasn't been executed"` rather than returning
    undefined). True → `Set Post Data` directly (caption untouched, since
    `Generate Caption` simply didn't re-run — `$node["Generate Caption"]`
    still resolves to its last real output). False → `Generate Caption`
    (normal path, also correctly used by the initial draft pass since the
    condition safely evaluates to `""` before any resume exists).
    `Set Post Data`'s `caption` field had to change from `$json.choices[0]...`
    (assumes `Generate Caption` is the immediate predecessor) to the named
    reference `$node["Generate Caption"].json.choices[0]...` so it resolves
    correctly regardless of which path reached it.

**Approve half** (was WF4, merged in via `Ignore Commands`' true branch — shared
by both LinkedIn and Instagram, and will be by any future platform too, since
it's generic: it just looks up the newest pending row and calls its stored
`resume_url`, regardless of which platform/Wait-node it belongs to):
11. `Is it a command?` (IF) — passes only if `message.text` starts with `/`
    (redundant with `Ignore Commands` above, harmless).
12. `Extract Command` (Set) — pulls `telegram_user_id`, `chat_id`, `command`
    (the word after `/`, slash stripped via `.replace(/^\//, '')`), and
    `instructions` (text after the command).
13. `Get Pending Approval` (HTTP Request → Supabase) — looks up the latest
    `pending` row for that user. `Always Output Data` is on (see
    `integrations.md` gotcha).
14. `Has Pending?` (IF) — branches on whether a row was found.
    - **Found →** `Resume Workflow` (HTTP Request, **GET**, `command` +
      `instructions` as query params) → calls back into whichever Wait node
      the pending row's `resume_url` points at, resuming that paused draft
      execution → `Update Status` (HTTP Request, PATCH — maps `approve`/
      `discard` to `approved`/`rejected`, anything else stays `pending`) →
      `Reply - Processing` (Telegram).
    - **Not found →** `Reply - No Pending` (Telegram).
