# social-publisher — Knowledge Index

> Last updated: 2026-07-04 (Instagram quality fixed, repo made public-ready)
> Status: Active — LinkedIn and Instagram both fully working live with good content quality
> Stack: n8n (Docker) + Telegram Bot API + Supabase + Groq + Pollinations + Cloudinary + LinkedIn API + Meta Graph API
> Current goal: proper n8n credentials instead of hardcoded values, then build company LinkedIn (`s:`)

## What This Project Does
Automates posting to personal Instagram, personal LinkedIn, and company LinkedIn.
A human approves each post via Telegram before it goes out. "WF1 - LinkedIn
Post (Personal)" (exported as `workflows/main-workflow.json`) is the single
self-contained workflow for everything — it started as just LinkedIn (`y:`
prefix) and has since absorbed Instagram (`ig:` prefix) too, since only one
workflow can hold the bot's webhook. Texting `y:` or `ig:` drafts a
post/image+caption, sends a Telegram preview, and
`/approve`/`/edit`/`/regenerate`/`/discard` control what happens next —
approving actually publishes. **First real LinkedIn post: 2026-07-03. First
real Instagram post: 2026-07-04, with output quality fixed the same day**
(see `history.md` for the FLUX/Llama prompting research behind that fix).
Company LinkedIn (`s:`) is not built yet. Repo is confirmed safe to make
public (git history was rewritten to scrub an exposed key first — see
`history.md`).

---

## Files in This Folder

| File | Contents | Load when... |
|------|----------|--------------|
| `stack.md` | Tech stack, dev commands, env vars | Setting up, touching Docker/n8n config |
| `systems.md` | n8n, Telegram, Supabase, auth | Touching any cross-cutting system |
| `integrations.md` | Telegram Bot API + Supabase data contract | Changing webhook logic or the `pending_approvals` table |
| `features.md` | WF1, step by step (LinkedIn + Instagram draft halves + shared approve half) | Editing that workflow or adding a new one |
| `roadmap.md` | What's planned, what's known-broken | Starting any task |
| `history.md` | Past fixes and decisions | Debugging something that looks familiar |
| `sessions.md` | Session-by-session log | Reviewing work history |

> Only files that exist are listed here.

---

## Context Loading Guide

| Task | Load these files |
|------|-------------------|
| Editing WF1 (any platform's drafting or approve/post side) | `features.md` + `integrations.md` |
| Debugging Telegram webhook / Wait-node resume issues | `integrations.md` (Wait-node GET gotcha) + `history.md` |
| Fixing Instagram output quality (image/caption/hashtags) | `roadmap.md` (Known Bugs) + `integrations.md` (Groq section) |
| Building company LinkedIn (`s:`) | `roadmap.md` + `integrations.md` + `features.md` (use WF1's Instagram merge as the template) |
| General orientation (new session) | This file → then pick by task |
| Full audit | All files |
