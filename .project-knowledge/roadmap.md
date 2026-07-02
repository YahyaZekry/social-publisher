# Roadmap

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-02
> Forward-looking only. Check this before starting any task.

## Current Goal

Telegram approval loop is now working end-to-end (webhook → lookup → reply). Next:
build whatever creates `pending_approvals` rows, and the actual Instagram/LinkedIn
publish workflows.

---

## Known Bugs

*(none open as of 2026-07-02 — see `history.md` for what was just fixed)*

---

## Active TODOs

- [ ] Move the Supabase `apikey`/`Authorization` headers on `Get Pending Approval`
      (and any future Supabase calls) into a proper n8n Header Auth credential
      instead of hardcoded plaintext values on the node. *(added: 2026-07-02)*
- [ ] Decide what writes rows into `pending_approvals` (another n8n workflow? an
      external content-generation step?) and build it. *(added: 2026-07-02)*
- [ ] Build the actual publish step(s): Instagram, personal LinkedIn, company
      LinkedIn. *(added: 2026-07-02)*
- [ ] Test `/approve` against a real pending row (only the "no pending" branch has
      been verified so far). *(added: 2026-07-02)*
- [ ] Consider a real command parser instead of `startsWith('/')` if more commands
      beyond `/approve` are planned (e.g. `/reject`). *(added: 2026-07-02)*

---

## Planned Features

- [ ] `/reject` command (or similar) as a counterpart to `/approve`. *(added: 2026-07-02)*
