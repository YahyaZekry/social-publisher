# social-publisher — Knowledge Index

> Last updated: 2026-07-02
> Status: Active (early — one workflow built, publish steps not started)
> Stack: n8n (Docker) + Telegram Bot API + Supabase
> Current goal: get the Telegram approval loop fully reliable, then build the actual Instagram/LinkedIn publish workflows

## What This Project Does
Automates posting to personal Instagram, personal LinkedIn, and company LinkedIn.
A human approves each post via Telegram before it goes out. Currently only the
Telegram approval side ("WF4 - Command Listener") exists; the publish step is
not built yet.

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
| Editing the Command Listener workflow | `features.md` + `integrations.md` |
| Debugging Telegram webhook / n8n proxy issues | `systems.md` + `history.md` |
| Building the Instagram/LinkedIn publish step | `roadmap.md` + `integrations.md` |
| General orientation (new session) | This file → then pick by task |
| Full audit | All files |
