# Features & Workflows

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03

## Features

- **LinkedIn post drafting + approval via Telegram** — texting the bot
  `y: <topic>` generates a personal LinkedIn post with Groq, sends it back as
  a preview, and waits; `/approve` posts it for real, `/edit <instructions>`
  regenerates it, `/discard` cancels. **Confirmed working end-to-end live on
  2026-07-03** — first real post published this way. *(added: 2026-07-01,
  confirmed working: 2026-07-03)*

---

## Workflows

**WF1 - LinkedIn Post (Personal)** (`workflows/linkedin-post-personal.json`)

This is now the *only* active workflow — it originally shared duties with a
separate "WF4 - Command Listener" workflow, but both had their own Telegram
Trigger on the same bot credential, and Telegram only allows one webhook per
bot. WF4's approve-handling nodes were merged directly into WF1 on
2026-07-03 (see `history.md`); WF4 itself is now inactive, kept only as
`workflows/command-listener-deprecated.json` for reference.

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

**Approve half** (was WF4, merged in via `Ignore Commands`' true branch):
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
      `instructions` as query params) → calls back into step 9/10 above,
      resuming the paused draft execution → `Update Status` (HTTP Request,
      PATCH — currently writes an invalid `status` value, see `roadmap.md`)
      → `Reply - Processing` (Telegram).
    - **Not found →** `Reply - No Pending` (Telegram).
