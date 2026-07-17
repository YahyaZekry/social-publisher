# Roadmap

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-17 (OpenRouter credential + IG two-message caption split)
> Forward-looking only. Check this before starting any task.

## Current Goal

**LinkedIn (`y:`) and Instagram (`ig:`) are both fully working, confirmed
live**, and now share one interaction model: typing `y:`/`ig:` posts your
text/caption **verbatim** (no AI auto-drafting), auto-generates an image
where needed (LinkedIn: only on request; Instagram: always, since it can't
publish without media), and shows a 5-button inline-keyboard menu —
Regenerate text/caption+image, image only, text/caption only, Post it,
Discard — instead of typed `/approve`/`/edit`/`/regenerate*` commands, which
no longer do anything on either platform. Instagram also accepts a photo +
`ig: <caption>` directly, using your own photo instead of generating one.
**2026-07-17: all text/caption generation moved to OpenRouter
(`openai/gpt-4o-mini`) behind the project's first n8n credential, and the
Instagram preview was split into two Telegram messages (image, then
full-caption text + menu) to beat the 1024-char photo-caption limit.**
Next: finish moving the remaining hardcoded keys into n8n credentials, then
company LinkedIn (`s:`).

---

## Known Bugs

*(none open as of 2026-07-10 — see `history.md` for everything fixed this
session: Instagram's "Media ID is not available" publish race condition, and
a stale-webhook-registration bug that silently dropped incoming Telegram
messages after repeated API-driven structural edits to the active workflow)*

---

## Active TODOs

- [ ] **Watch for a "we'll see" tic in LinkedIn text output.** The
      2026-07-12 round-3 rewrite of `Rewrite with Instructions`/`(Both)`
      kept 2 of 3 few-shot examples ending on a near-identical hedge ("no
      idea if it'll work", "we'll see") — flagged as a risk before applying
      (examples anchor output more than instructions do) but applied as-is
      per user's call. If real regenerations start reusing that exact
      hedge, swap one example's ending for a plain non-hedged statement.
      *(added: 2026-07-12)*
- [ ] **Clean up the now-dead typed-command infrastructure**: `Ignore
      Commands`, `Is it a command?`, `Extract Command`, `Get Pending
      Approval`, `Has Pending?`, `Resume Workflow`, `Update Status`, `Reply -
      Processing`, `Reply - No Pending` (the `Telegram Bot` node group) are
      unreachable now that both platforms route entirely through the
      button/`callback_query` mechanism instead — kept alive deliberately
      during the button-menu rollout in case of rollback, safe to remove now
      that both platforms are confirmed working end to end. *(added:
      2026-07-08)*
- [ ] **Rotate the Supabase `service_role` key.** Was committed in plaintext
      to this repo's first commit before a 2026-07-04 history rewrite scrubbed
      it (confirmed clean across all commits + GitHub after a force-push).
      Since the repo was never public before that rewrite, this is now low
      urgency — the key was likely never actually seen outside this project
      — but rotating is still good hygiene since it's used from a live
      workflow with plaintext copies on multiple nodes. *(added: 2026-07-03,
      downgraded: 2026-07-04 after history rewrite)*
- [ ] Move the remaining hardcoded keys into proper n8n credentials.
      **Started 2026-07-17: OpenRouter is done** — all three text/caption
      nodes (`Rewrite with Instructions`, `Rewrite with Instructions (Both)`,
      `Generate Caption`) now use a shared **Header Auth credential**
      (`Authorization`), the project's first credential. **Still hardcoded:**
      Supabase `apikey`/`Authorization` (service_role), Groq key (2
      image-prompt nodes), LinkedIn token, Meta/Instagram access token.
      *(added: 2026-07-02, expanded: 2026-07-03/04/08, partially done:
      2026-07-17)*
- [ ] Build the actual publish step for company LinkedIn (`s:`) — personal
      LinkedIn and Instagram are both done now. WF1 is the template for the
      pattern: draft/verbatim capture → Supabase pending row → Telegram
      preview with the 5-button menu → Wait node → button-driven routing.
      Note from 2026-07-03: the company-page prompt will likely want the
      *opposite* instruction from the personal ones — it should probably only
      talk about Synax/AI automation/work, rather than avoiding it.
      *(added: 2026-07-02, refined: 2026-07-03, 2026-07-04)*
- [ ] LinkedIn bearer token on `Post to LinkedIn`/`Post to LinkedIn
      (Image)`/`Register LinkedIn Upload`/`Upload Image to LinkedIn` is a raw
      personal access token with unknown expiry — confirm how/when it needs
      refreshing before relying on it. (The Instagram/Meta token, by
      contrast, is a known long-lived Page Access Token good until ~Sept
      2026 — see `integrations.md`.) *(added: 2026-07-03)*
- [ ] The now-empty/inactive WF4 and WF2 workflows still exist **inside the
      live n8n instance** (not deleted, just inactive with 0/near-0 nodes).
      Still open: delete the two dead workflows from n8n itself, or leave
      them as inert? *(added: 2026-07-03, still open re: live n8n)*
- [ ] Several stale `pending_approvals` rows accumulated during testing will
      never resolve to `approved`/`rejected` — harmless since lookups always
      take the newest `pending` row, but worth a one-time cleanup query.
      *(added: 2026-07-03, expanded: 2026-07-04)*

---

## Planned Features

- [ ] **AI image-to-image manipulation** of a user-uploaded photo (e.g.
      restyle, artistic filter) — Pollinations only does text-to-image, so
      this needs a different image-to-image API (e.g. Replicate, Stability
      AI) as a new integration, plus a way to parse what manipulation the
      user wants from their caption text. The prerequisite "post own photo"
      feature this depended on is now done (2026-07-08) for Instagram.
      *(added: 2026-07-04)*
- [ ] **LinkedIn support for a user-uploaded photo**, mirroring what
      Instagram now has — currently LinkedIn's image path only supports
      AI-generated images (via the "Regenerate image" button); sending your
      own photo alongside `y: <text>` isn't wired up yet the way `ig:` now
      is. *(added: 2026-07-08)*
