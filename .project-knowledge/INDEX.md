# social-publisher — Knowledge Index

> Last updated: 2026-08-08 (all API keys migrated into n8n credentials — Groq/Supabase/LinkedIn/Meta joined OpenRouter and Telegram; workflow JSON and repo are now secret-free)
> Status: Active — LinkedIn and Instagram both fully working live with good content quality
> Stack: n8n (Docker) + Telegram Bot API + Supabase + OpenRouter (text/captions) + Groq (image prompts) + Pollinations + Cloudinary + LinkedIn API + Meta Graph API
> Current goal: company LinkedIn (`s:`) — the credentials-migration roadmap item is done as of 2026-08-08

## What This Project Does
Automates posting to personal Instagram, personal LinkedIn, and company LinkedIn.
A human approves each post via Telegram before it goes out. "WF1 - LinkedIn
Post (Personal)" (exported as `workflows/main-workflow.json`) is the single
self-contained workflow for everything — it started as just LinkedIn (`y:`
prefix) and has since absorbed Instagram (`ig:` prefix) too, since only one
workflow can hold the bot's webhook. Texting `y:` or `ig:` posts your
text/caption **verbatim** (no AI auto-drafting) and, for Instagram,
auto-generates an image too (or uses a photo you send directly). Every
preview carries a **5-button inline-keyboard menu** (regenerate text+image /
image only / text only / Post it / Discard) — typed `/approve`, `/edit`,
`/regenerate*`, `/discard` no longer do anything on either platform, replaced
2026-07-08. **First real LinkedIn post: 2026-07-03. First real Instagram
post: 2026-07-04, with output quality fixed the same day** (see `history.md`
for the FLUX/Llama prompting research behind that fix). LinkedIn image
support and Instagram's own-photo support were both added 2026-07-08.
Company LinkedIn (`s:`) is not built yet. Repo is confirmed safe to make
public (git history was rewritten to scrub an exposed key first — see
`history.md`). Since 2026-07-08, workflow edits are made via n8n's Public
API (self-generated key) rather than manual UI clicks, given how large
recent changes have gotten — see `history.md`'s Decisions section for the
node-group validation constraint this surfaced. **2026-07-10: fixed a real
Instagram publish race condition** (Graph API rejected `/media_publish`
when called before the media container finished processing — now polls
`status_code` until `FINISHED` before publishing) **and a stale
webhook-registration bug** that silently dropped some incoming Telegram
messages after repeated structural API edits to the active workflow (fixed
by deactivate/reactivate) — see `integrations.md` and `history.md`.
**2026-07-12: `Generate Caption` (Instagram) switched from Groq/Llama to
OpenRouter's `openai/gpt-4o-mini`** — same prompt, same response shape, just
a different backend; LinkedIn text and image-prompt generation stay on Groq
— see `integrations.md`. **Same day, a content-quality pass** added
few-shot examples to the LinkedIn text prompts (previously image-only),
clarified the image prompts' worked examples aren't exhaustive, and added
`presence_penalty` to `Generate Caption` now that it runs on real OpenAI
infra — see `integrations.md`'s "Prompt quality pass" section. **A live
test of that fix immediately caught a second, sharper LinkedIn issue**
(invented narrative framing, not just invented facts) — fixed the same day
in a second round, see `integrations.md`'s "Round 2" note. **A third round
the same day replaced the LinkedIn text prompt entirely** rather than
patching again — one generative rule instead of an accumulating
banned-phrase list — see `integrations.md`'s "Round 3" note and
`roadmap.md` for a flagged watch-item on the new few-shot examples.
**2026-08-08: the credentials-migration goal is complete** — every API key
(Groq, Supabase, LinkedIn, Meta, plus OpenRouter and Telegram) now lives in
an n8n credential; the workflow JSON and this repo contain no secrets
(generic `-REPLACE` credential IDs in the export, real values in a gitignored
`.credentials.env`). See `history.md`'s 2026-08-08 decisions.

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
| Tuning content quality or hashtags for either platform | `integrations.md` (Groq section) + `history.md` |
| Building company LinkedIn (`s:`) | `roadmap.md` + `integrations.md` + `features.md` (use WF1's Instagram merge as the template) |
| General orientation (new session) | This file → then pick by task |
| Full audit | All files |
