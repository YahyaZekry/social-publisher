# social-publisher — Knowledge Index

> Last updated: 2026-07-03
> Status: Active — personal LinkedIn loop fully working end-to-end (confirmed live)
> Stack: n8n (Docker) + Telegram Bot API + Supabase + Groq + LinkedIn API
> Current goal: fix the minor `Update Status` value bug, then build Instagram / company LinkedIn using WF1 as the template

## What This Project Does
Automates posting to personal Instagram, personal LinkedIn, and company LinkedIn.
A human approves each post via Telegram before it goes out. "WF1 - LinkedIn
Post (Personal)" is now a single self-contained workflow: texting the bot
`y: <topic>` drafts a post with Groq, sends a Telegram preview, and
`/approve`/`/edit`/`/discard` control what happens next — approving actually
posts to LinkedIn. **First real post published this way on 2026-07-03.**
Instagram and company LinkedIn are not built yet.

---

## Files in This Folder

| File | Contents | Load when... |
|------|----------|--------------|
| `stack.md` | Tech stack, dev commands, env vars | Setting up, touching Docker/n8n config |
| `systems.md` | n8n, Telegram, Supabase, auth | Touching any cross-cutting system |
| `integrations.md` | Telegram Bot API + Supabase data contract | Changing webhook logic or the `pending_approvals` table |
| `features.md` | WF1, step by step (draft half + merged-in approve half) | Editing that workflow or adding a new one |
| `roadmap.md` | What's planned, what's known-broken | Starting any task |
| `history.md` | Past fixes and decisions | Debugging something that looks familiar |
| `sessions.md` | Session-by-session log | Reviewing work history |

> Only files that exist are listed here.

---

## Context Loading Guide

| Task | Load these files |
|------|-------------------|
| Editing WF1 (drafting or approve/post side) | `features.md` + `integrations.md` |
| Debugging Telegram webhook / Wait-node resume issues | `integrations.md` (Wait-node GET gotcha) + `history.md` |
| Building the Instagram/company-LinkedIn publish step | `roadmap.md` + `integrations.md` + `features.md` (use WF1 as the template) |
| General orientation (new session) | This file → then pick by task |
| Full audit | All files |
