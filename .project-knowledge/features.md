# Features & Workflows

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03

## Features

- **Telegram approval commands** — owner can message the bot to act on a pending
  approval. Currently only `/approve` has been exercised. *(added: 2026-07-01)*
- **LinkedIn post drafting via Telegram** — texting the bot `y: <topic>` generates
  a personal LinkedIn post with Groq and sends it back as a preview, pending
  approval. Drafting half confirmed working end-to-end 2026-07-03; the
  approve/post half is built but not yet exercised (see `roadmap.md` — blocked
  on the webhook-contention bug). *(added: 2026-07-03)*

---

## Workflows

**WF4 - Command Listener** (`workflows/command-listener.json`)

1. `Telegram Trigger` — receives `message` updates via webhook.
2. `Is it a command?` (IF) — passes only if `message.text` starts with `/`.
3. `Extract Command` (Set) — pulls out `telegram_user_id`, `chat_id`, `command`,
   `instructions` (text after the command) via expressions on `$json.message.*`.
4. `Get Pending Approval` (HTTP Request → Supabase) — looks up the latest pending
   row for that user. `Always Output Data` is on (see `integrations.md` gotcha).
5. `Has Pending?` (IF) — branches on whether a row was found.
   - **Found →** `Resume Workflow` (HTTP Request) → `Update Status` (HTTP Request) →
     `Reply - Processing` (Telegram) — tells the user their approval is being processed.
   - **Not found →** `Reply - No Pending` (Telegram) — tells the user there's nothing
     to approve.

Not yet built: the actual Instagram/company-LinkedIn publish steps that presumably
run after `Resume Workflow` for those targets (WF1 below covers personal LinkedIn).

---

**WF1 - LinkedIn Post (Personal)** (`workflows/linkedin-post-personal.json`)

1. `Telegram Trigger` — receives `message` updates via webhook (same bot/credential
   as WF4 — see the webhook-contention bug in `roadmap.md`).
2. `Ignore Commands` (IF) — messages starting with `/` are dropped here (dead end,
   no downstream connection) so `/approve` etc. don't leak into the drafting flow.
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
   its one-time resume URL is what got stored in step 7.
10. `Route Command` (Switch on `$json.body.command`) — branches on what the
    resumed webhook call's body contains:
    - **`approve` →** `Post to LinkedIn` (HTTP Request → LinkedIn `ugcPosts` API)
      → `Confirm Posted` (Telegram).
    - **`edit` →** `Rewrite with Instructions` (HTTP Request → Groq, same model,
      rewrites the post per `$json.body.instructions`) → `Send Updated Preview`
      (Telegram) — loops back to another approve/edit/discard decision.
    - **`discard` →** `Discard` (Telegram) — confirms cancellation.

Not yet exercised end-to-end: steps 9–10 depend on WF4 (or something) actually
calling the stored `resume_url` — currently blocked by the Telegram
webhook-contention bug (`roadmap.md`).
