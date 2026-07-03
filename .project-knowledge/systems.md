# Systems

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03

| System | Status | Details |
|--------|--------|---------|
| n8n auth / RBAC | Working | Single user, project `personalOwner`, global role `global:owner`. Global role briefly self-corrected from `global:member` to `global:owner` on 2026-07-02 — see `history.md`. |
| Reverse proxy trust | Working | `N8N_PROXY_HOPS=1` set on the container; confirmed zero "trust proxy" errors in logs since. |
| Telegram Bot | Working | One `Telegram Trigger`, on WF1 (WF4's was removed after its logic got merged in — two triggers on one bot credential doesn't work, see `history.md`). |
| Database (Supabase) | Working | REST calls via raw HTTP Request node headers (`apikey` + `Authorization: Bearer ...`), not an n8n credential. `pending_approvals` table — see `integrations.md`. `Update Status` writes a value that fails a check constraint — see `roadmap.md`. |
| LLM content generation (Groq) | Working | WF1's `Generate LinkedIn Post` / `Rewrite with Instructions` nodes call Groq's `llama-3.3-70b-versatile` via raw HTTP Request + hardcoded API key. See `integrations.md`. |
| Instagram publishing | Not built | No node/workflow yet. |
| LinkedIn publishing (personal) | **Working, confirmed live** | WF1's `Post to LinkedIn` node (UGC Posts API) — first real post published 2026-07-03. |
| LinkedIn publishing (company) | Not built | No node/workflow yet. |
| Content generation (what creates a pending approval) | Built (LinkedIn only) | WF1's `Save to Supabase` node writes `pending_approvals` rows for `workflow_type: 'linkedin'`. Instagram/company-LinkedIn equivalents not built. |
