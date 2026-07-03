# Systems

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03

| System | Status | Details |
|--------|--------|---------|
| n8n auth / RBAC | Working | Single user, project `personalOwner`, global role `global:owner`. Global role briefly self-corrected from `global:member` to `global:owner` on 2026-07-02 — see `history.md`. |
| Reverse proxy trust | Working | `N8N_PROXY_HOPS=1` set on the container; confirmed zero "trust proxy" errors in logs since. |
| Telegram Bot | Partially working | Only one workflow's `Telegram Trigger` can hold the webhook at a time (WF1 and WF4 share one bot credential) — see the contention bug in `roadmap.md`. Currently registered to WF1. |
| Database (Supabase) | Working | REST calls via raw HTTP Request node headers (`apikey` + `Authorization: Bearer ...`), not an n8n credential, on both WF4 (read) and WF1 (write). `pending_approvals` table — see `integrations.md`. |
| LLM content generation (Groq) | Working | WF1's `Generate LinkedIn Post` / `Rewrite with Instructions` nodes call Groq's `llama-3.3-70b-versatile` via raw HTTP Request + hardcoded API key. See `integrations.md`. |
| Instagram publishing | Not built | No node/workflow yet. |
| LinkedIn publishing (personal) | Built, not yet exercised | WF1's `Post to LinkedIn` node (UGC Posts API) — blocked from a real test by the webhook-contention bug. |
| LinkedIn publishing (company) | Not built | No node/workflow yet. |
| Content generation (what creates a pending approval) | Built (LinkedIn only) | WF1's `Save to Supabase` node writes `pending_approvals` rows for `workflow_type: 'linkedin'`. Instagram/company-LinkedIn equivalents not built. |
