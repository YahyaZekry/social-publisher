# Features & Workflows

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-02

## Features

- **Telegram approval commands** — owner can message the bot to act on a pending
  approval. Currently only `/approve` has been exercised. *(added: 2026-07-01)*

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

Not yet built: whatever creates the `pending_approvals` row in the first place, and
the actual Instagram/LinkedIn publish step that presumably runs after `Resume Workflow`.
