# Systems

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-02

| System | Status | Details |
|--------|--------|---------|
| n8n auth / RBAC | Working | Single user, project `personalOwner`, global role `global:owner`. Global role briefly self-corrected from `global:member` to `global:owner` on 2026-07-02 — see `history.md`. |
| Reverse proxy trust | Working | `N8N_PROXY_HOPS=1` set on the container; confirmed zero "trust proxy" errors in logs since. |
| Telegram Bot | Working | Webhook registered at `https://ergonomic-password-rosy.ngrok-free.dev/webhook/<id>/webhook`. Trigger node listens for `message` updates only. |
| Database (Supabase) | Working | REST calls via raw HTTP Request node headers (`apikey` + `Authorization: Bearer ...`), not an n8n credential. `pending_approvals` table — see `integrations.md`. |
| Instagram publishing | Not built | No node/workflow yet. |
| LinkedIn publishing (personal) | Not built | No node/workflow yet. |
| LinkedIn publishing (company) | Not built | No node/workflow yet. |
| Content generation (what creates a pending approval) | Unknown / not built | Nothing in this n8n instance currently writes to `pending_approvals` — needs to be built or is external. |
