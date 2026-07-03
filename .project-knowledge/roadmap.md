# Roadmap

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03
> Forward-looking only. Check this before starting any task.

## Current Goal

**The full loop works end-to-end, confirmed live 2026-07-03**: `y: <topic>` →
Groq drafts a post → row written to `pending_approvals` → Telegram preview →
`/approve` → posted for real to personal LinkedIn → Telegram confirmation.
WF1 is now the single self-contained workflow (WF4's logic was merged in).
Next: fix the `Update Status` bug below (cosmetic/bookkeeping, not blocking),
then start on Instagram / company LinkedIn using WF1 as the template.

---

## Known Bugs

- [ ] **`Update Status` writes an invalid `status` value.** It sets
      `status: $node['Extract Command'].json['command']` directly, which is
      the bare command word (`approve`/`edit`/`discard`) — but Supabase's
      `pending_approvals_status_check` constraint rejects `approve` (400,
      `23514`). It almost certainly wants past-tense (`approved`, matching
      the `pending` already used elsewhere). Not blocking — the actual
      LinkedIn post already succeeds via the separate `Post to LinkedIn` path
      — but the row is left stuck at `status = 'pending'` forever, which
      could make a later `/approve` (with no new draft) incorrectly match
      this stale row and try to hit its long-dead `resume_url`. Fix: map the
      command to the correct enum value before writing (check the actual
      constraint's allowed values first rather than guessing).
      *(found: 2026-07-03)*

---

## Active TODOs

- [ ] Move the Supabase `apikey`/`Authorization` headers (used across several
      nodes in WF1) into a proper n8n Header Auth credential instead of
      hardcoded plaintext values on each node. Now doubly important — the
      same raw service_role JWT, a raw Groq API key, and a raw LinkedIn
      bearer token are all duplicated across multiple nodes in this (private)
      repo's exported JSON (see `integrations.md`).
      *(added: 2026-07-02, expanded: 2026-07-03)*
- [ ] Build the actual publish step(s) for Instagram and company LinkedIn —
      only personal LinkedIn (WF1) exists so far. WF1 is the template:
      draft via LLM → Supabase pending row → Telegram preview → Wait node →
      approve/edit/discard routing. *(added: 2026-07-02, refined: 2026-07-03)*
- [ ] Consider a real command parser instead of `startsWith('/')` if more
      commands beyond `/approve`/`/edit`/`/discard` are planned.
      *(added: 2026-07-02)*
- [ ] LinkedIn bearer token on `Post to LinkedIn` is a raw personal access
      token with unknown expiry — confirm how/when it needs refreshing before
      relying on it. *(added: 2026-07-03)*
- [ ] Decide what to do with the now-inactive WF4 workflow (kept as
      `workflows/command-listener-deprecated.json`) — delete it from the n8n
      instance entirely, or leave it as an inert historical reference?
      *(added: 2026-07-03)*
- [ ] Several stale `pending_approvals` rows accumulated during this
      session's testing (from expired/consumed resume URLs) will never
      resolve to `approved`/`rejected` — harmless since lookups always take
      the newest `pending` row, but worth a one-time cleanup query.
      *(added: 2026-07-03)*

---

## Planned Features

- [ ] `/reject` command (or similar) as a counterpart to `/approve` — note
      `/discard` already exists in WF1 and may already cover this.
      *(added: 2026-07-02, refined: 2026-07-03)*
