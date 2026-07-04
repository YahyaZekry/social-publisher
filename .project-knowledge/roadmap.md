# Roadmap

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-04 (Instagram quality fix)
> Forward-looking only. Check this before starting any task.

## Current Goal

**LinkedIn (`y:`) and Instagram (`ig:`) are both fully working, confirmed
live** — draft → preview → `/approve` → real post, with genuinely good
content quality on both (see `history.md` for the 2026-07-04 fix). Next:
proper n8n credentials instead of hardcoded values, then company LinkedIn
(`s:`).

---

## Known Bugs

*(none open as of 2026-07-04 — see `history.md` for what was just fixed)*

---

## Active TODOs

- [ ] **Rotate the Supabase `service_role` key.** Was committed in plaintext
      to this repo's first commit before a 2026-07-04 history rewrite scrubbed
      it (confirmed clean across all commits + GitHub after a force-push).
      Since the repo was never public before that rewrite, this is now low
      urgency — the key was likely never actually seen outside this project
      — but rotating is still good hygiene since it's used from a live
      workflow with plaintext copies on multiple nodes. *(added: 2026-07-03,
      downgraded: 2026-07-04 after history rewrite)*
- [ ] Move the Supabase `apikey`/`Authorization` headers, Groq key, LinkedIn
      token, and Meta/Instagram access token (all used across many nodes in
      WF1 now) into proper n8n credentials instead of hardcoded plaintext
      values on each node. Now quadruply important with Instagram merged in
      too. *(added: 2026-07-02, expanded: 2026-07-03, 2026-07-04)*
- [ ] Build the actual publish step for company LinkedIn (`s:`) — personal
      LinkedIn and Instagram are both done now. WF1 is the template for the
      pattern: draft via LLM → Supabase pending row → Telegram preview →
      Wait node → approve/edit/discard routing. Note from 2026-07-03: the
      company-page prompt will likely want the *opposite* instruction from
      the personal ones — it should probably only talk about Synax/AI
      automation/work, rather than avoiding it.
      *(added: 2026-07-02, refined: 2026-07-03, 2026-07-04)*
- [ ] Consider a real command parser instead of `startsWith('/')` if more
      commands beyond `/approve`/`/edit`/`/discard`/`/regenerate` are planned.
      *(added: 2026-07-02)*
- [ ] LinkedIn bearer token on `Post to LinkedIn` is a raw personal access
      token with unknown expiry — confirm how/when it needs refreshing before
      relying on it. (The Instagram/Meta token, by contrast, is now a known
      long-lived Page Access Token good until ~Sept 2026 — see
      `integrations.md`.) *(added: 2026-07-03)*
- [ ] The now-empty/inactive WF4 and WF2 workflows still exist **inside the
      live n8n instance** (not deleted, just inactive with 0/near-0 nodes).
      Their repo-side exports (`command-listener-deprecated.json`,
      `instagram-post-personal-deprecated.json`) were removed from this repo
      on 2026-07-04 as pointless clutter — `history.md`/`sessions.md` already
      cover everything about them. Still open: delete the two dead workflows
      from n8n itself, or leave them as inert? *(added: 2026-07-03, resolved
      re: repo files 2026-07-04, still open re: live n8n)*
- [ ] Several stale `pending_approvals` rows accumulated during testing
      (from expired/consumed resume URLs, rows that predate the
      status-mapping fix, and now Instagram test rows too) will never
      resolve to `approved`/`rejected` — harmless since lookups always take
      the newest `pending` row, but worth a one-time cleanup query.
      *(added: 2026-07-03, expanded: 2026-07-04)*
- [ ] Prefix-matching (`Has y: prefix?` / `Has ig: prefix?`) only checks
      `message.text` — a photo sent with a caption puts the text in
      `message.caption` instead, so `ig: <caption>` on a photo message
      currently falls through to the reminder instead of being recognized.
      Relevant now that image-to-image / "post my own photo" is planned
      (see Planned Features). *(added: 2026-07-04)*

---

## Planned Features

- [ ] `/reject` command (or similar) as a counterpart to `/approve` — note
      `/discard` already exists in WF1 and may already cover this.
      *(added: 2026-07-02, refined: 2026-07-03)*
- [ ] **Post the user's own uploaded photo instead of an AI-generated one**,
      with an AI-generated caption. Send a photo + caption to the bot; use
      Telegram's `file` resource (`operation: get`, `download: true` — this
      n8n version's Telegram node handles the full fetch-and-download in one
      step, confirmed from the installed node's source) to pull the actual
      binary instead of Pollinations, upload that to Cloudinary, then run
      the existing caption pipeline unchanged. Needs the caption-vs-text
      prefix-matching bug above fixed first. Smaller lift than image-to-image
      below — reuses almost the whole existing pipeline.
      *(added: 2026-07-04)*
- [ ] **AI image-to-image manipulation** of a user-uploaded photo (e.g.
      restyle, artistic filter) — bigger lift than the above: Pollinations
      only does text-to-image, so this needs a different image-to-image API
      (e.g. Replicate, Stability AI) as a new integration, plus a way to
      parse what manipulation the user wants from their caption text.
      Sequence after the simpler "post own photo" feature above, not before.
      *(added: 2026-07-04)*
