# social-publisher — Knowledge Index

> Last updated: 2026-07-03
> Status: Active (two workflows built: approval listener + personal LinkedIn drafting)
> Stack: n8n (Docker) + Telegram Bot API + Supabase + Groq + LinkedIn API
> Current goal: fix the Telegram webhook-contention bug between WF1 and WF4, then verify the full draft → approve → post loop for real

## What This Project Does
Automates posting to personal Instagram, personal LinkedIn, and company LinkedIn.
A human approves each post via Telegram before it goes out. "WF4 - Command
Listener" handles `/approve` etc.; "WF1 - LinkedIn Post (Personal)" drafts a
post with Groq from a `y: <topic>` Telegram message and writes it to the
approval queue. Drafting is confirmed working end-to-end; posting is built
but not yet exercised (blocked on a webhook-contention bug — see `roadmap.md`).
Instagram and company LinkedIn are not built yet.

---

## Files in This Folder

| File | Contents | Load when... |
|------|----------|--------------|
| `stack.md` | Tech stack, dev commands, env vars | Setting up, touching Docker/n8n config |
| `systems.md` | n8n, Telegram, Supabase, auth | Touching any cross-cutting system |
| `integrations.md` | Telegram Bot API + Supabase data contract | Changing webhook logic or the `pending_approvals` table |
| `features.md` | The Command Listener workflow, step by step | Editing that workflow or adding a new one |
| `roadmap.md` | What's planned, what's known-broken | Starting any task |
| `history.md` | Past fixes and decisions | Debugging something that looks familiar |
| `sessions.md` | Session-by-session log | Reviewing work history |

> Only files that exist are listed here.

---

## Context Loading Guide

| Task | Load these files |
|------|-------------------|
| Editing the Command Listener or LinkedIn Post workflow | `features.md` + `integrations.md` |
| Debugging Telegram webhook / n8n proxy issues | `systems.md` + `history.md` + `roadmap.md` (webhook-contention bug) |
| Building the Instagram/company-LinkedIn publish step | `roadmap.md` + `integrations.md` + `features.md` (use WF1 as the template) |
| General orientation (new session) | This file → then pick by task |
| Full audit | All files |
