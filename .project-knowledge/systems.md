# Systems

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-04

| System | Status | Details |
|--------|--------|---------|
| n8n auth / RBAC | Working | Single user, project `personalOwner`, global role `global:owner`. Global role briefly self-corrected from `global:member` to `global:owner` on 2026-07-02 — see `history.md`. |
| Reverse proxy trust | Working | `N8N_PROXY_HOPS=1` set on the container; confirmed zero "trust proxy" errors in logs since. |
| Telegram Bot | Working | One `Telegram Trigger`, on WF1 (WF4's and WF2's were removed after their logic got merged in — two triggers on one bot credential doesn't work, see `history.md`). |
| Database (Supabase) | Working | REST calls via raw HTTP Request node headers (`apikey` + `Authorization: Bearer ...`), not an n8n credential. `pending_approvals` table — see `integrations.md`. |
| LLM content generation (Groq) | Working for LinkedIn, not for Instagram yet | 4 nodes now call Groq's `llama-3.3-70b-versatile` via raw HTTP Request + hardcoded API key. LinkedIn prompts are solid; Instagram's (`Build Image Prompt`, `Generate Caption`) still produce generic/off-topic output despite two rounds of tuning — open, see `roadmap.md`. |
| Image generation (Pollinations) | Working mechanically, quality issues | Text-to-image via `Build Image Prompt`'s Groq output; images often default to generic stock-photo visuals — see `roadmap.md`. |
| Image hosting (Cloudinary) | Working | Unsigned upload preset, no credential secrecy needed for that part. |
| Instagram publishing | **Working, confirmed live** | `Create Media Container` → `Publish to Instagram` (Meta Graph API), first real post 2026-07-04. Long-lived Page Access Token in place (~Sept 2026). |
| LinkedIn publishing (personal) | **Working, confirmed live** | WF1's `Post to LinkedIn` node (UGC Posts API) — first real post published 2026-07-03. |
| LinkedIn publishing (company) | Not built | No node/workflow yet — will use `s:` prefix. |
| Content generation (what creates a pending approval) | Built for LinkedIn + Instagram | `Save to Supabase` / `Save to Supabase (IG)` write `pending_approvals` rows for `workflow_type: 'linkedin'` / `'instagram'`. Company-LinkedIn equivalent not built. |
