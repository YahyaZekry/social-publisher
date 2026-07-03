# Roadmap

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03
> Forward-looking only. Check this before starting any task.

## Current Goal

WF1 (LinkedIn Post - Personal) now drafts end-to-end: `y: <topic>` → Groq
generates the post → row written to `pending_approvals` → Telegram preview
reply. Next: fix the webhook-contention bug below so `/approve` can actually
reach WF4 and resume WF1, then exercise the full draft → approve → post loop
for real.

---

## Known Bugs

- [ ] **WF1 and WF4 silently fight over the same Telegram webhook.** Both
      workflows have their own `Telegram Trigger` node using the *same*
      credential (`Telegram account`, id `ZkWfYSlTSfJ42VwR`). Telegram only
      allows one webhook URL per bot, so activating/saving either workflow
      overwrites the other's registration — confirmed via `getWebhookInfo`,
      which currently points at WF1's trigger, not WF4's. Right now, sending
      `/approve` hits WF1's trigger, gets caught by its `Ignore Commands` IF
      node (matches "starts with /"), and is silently dropped — WF4 never
      sees it, so `Resume Workflow` never fires. Needs an architectural fix:
      either one shared Telegram Trigger that fans out by prefix/command to
      both flows, or moving WF1's draft step and WF4's approve step into a
      single workflow. *(found: 2026-07-03)*
- [ ] **WF4's `Update Status` node has an incomplete Supabase credential** —
      its `apikey` header is truncated (missing the JWT header segment) and
      `Authorization` is still the literal placeholder text
      `Bearer YOUR_SUPABASE_SERVICE_ROLE_KEY`, never replaced with a real key
      (unlike `Get Pending Approval`, which was fixed earlier). This node runs
      right after `Resume Workflow`, so `/approve` will 401 here even once the
      webhook-contention bug above is fixed. *(found: 2026-07-03, while
      redacting secrets from the exported JSON for this repo)*

---

## Active TODOs

- [ ] Move the Supabase `apikey`/`Authorization` headers (used by `Get Pending
      Approval` in WF4 and `Save to Supabase` in WF1) into a proper n8n Header
      Auth credential instead of hardcoded plaintext values on each node.
      Now doubly important — the same raw service_role JWT is duplicated
      across two workflows' exported JSON in this (private) repo, alongside
      a raw Groq API key and a raw LinkedIn bearer token (see
      `integrations.md`). *(added: 2026-07-02, expanded: 2026-07-03)*
- [ ] Build the actual publish step(s) for Instagram and company LinkedIn —
      only personal LinkedIn (WF1) exists so far. *(added: 2026-07-02)*
- [ ] Test `/approve`, `/edit [instructions]`, and `/discard` against a real
      pending row end-to-end — blocked on the webhook-contention bug above.
      *(added: 2026-07-02, refined: 2026-07-03)*
- [ ] Consider a real command parser instead of `startsWith('/')` if more
      commands beyond `/approve` are planned (e.g. `/reject`). *(added: 2026-07-02)*
- [ ] LinkedIn bearer token on `Post to LinkedIn` is a raw personal access
      token with unknown expiry — confirm how/when it needs refreshing before
      relying on it. *(added: 2026-07-03)*

---

## Planned Features

- [ ] `/reject` command (or similar) as a counterpart to `/approve`. *(added: 2026-07-02)*
