# Roadmap

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-04
> Forward-looking only. Check this before starting any task.

## Current Goal

**LinkedIn (`y:`) is fully working, confirmed live** — draft → preview →
`/approve` → real post. **Instagram (`ig:`) is mechanically working as of
2026-07-04** (image generated, uploaded, captioned, previewed, `/approve`
actually posts to Instagram) **but content quality is not good enough yet** —
see Known Bugs. That's the immediate focus before moving on to company
LinkedIn (`s:`).

---

## Known Bugs

- [ ] **Instagram output quality is bad — image, caption, and hashtags all
      feel off.** User's own words: "i hate the tags, the way it's writing
      and the image output." Two rounds of prompt tweaks so far (follow the
      user's topic instead of defaulting to Yahya's AI/athlete persona;
      avoid generic stock-photo imagery; avoid filler hashtags like
      `#newpost`/`#digitalfootprint`) have **not** resolved it as of
      2026-07-04. Needs a harder look at `Build Image Prompt` and
      `Generate Caption`'s prompts — possibly needs concrete before/after
      examples in the system prompt rather than more abstract instructions,
      since abstract "don't do X" instructions haven't been landing well.
      *(found: 2026-07-04)*

---

## Active TODOs

- [ ] **Rotate the Supabase `service_role` key.** It was committed in
      plaintext to this repo's first commit (`5f50384`) before redaction
      started. Recommended regardless of whether git history also gets
      rewritten — a rotated key makes the old exposed one harmless. Needs
      the Supabase dashboard (can't be done from here) + updating it in the
      nodes across WF1 that still hardcode it. Asked 2026-07-03, no
      decision yet on whether to also rewrite git history for the old commit.
      *(added: 2026-07-03)*
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
