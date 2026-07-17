# Systems

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-04

| System | Status | Details |
|--------|--------|---------|
| n8n auth / RBAC | Working | Single user, project `personalOwner`, global role `global:owner`. Global role briefly self-corrected from `global:member` to `global:owner` on 2026-07-02 — see `history.md`. |
| Reverse proxy trust | Working | `N8N_PROXY_HOPS=1` set on the container; confirmed zero "trust proxy" errors in logs since. |
| Telegram Bot | Working | One `Telegram Trigger`, on WF1 (WF4's and WF2's were removed after their logic got merged in — two triggers on one bot credential doesn't work, see `history.md`). |
| Database (Supabase) | Working | REST calls via raw HTTP Request node headers (`apikey` + `Authorization: Bearer ...`), not an n8n credential. `pending_approvals` table — see `integrations.md`. |
| LLM content generation | **Working, split across two providers (2026-07-17)** | **Text + captions → OpenRouter `openai/gpt-4o-mini`** (3 nodes: `Rewrite with Instructions`, `Rewrite with Instructions (Both)`, `Generate Caption`) via a shared **Header Auth n8n credential** (`Authorization`) — the project's first credential. **Image prompts → Groq `llama-3.3-70b-versatile`** (2 nodes: `Build Image Prompt`, `Build Image Prompt (LI)`), still hardcoded key. See `integrations.md`. |
| Image generation (Pollinations) | **Working well** | Text-to-image via `Build Image Prompt`'s Groq output, now structured around how FLUX actually responds to prompts — see `history.md`. |
| Image hosting (Cloudinary) | Working | Unsigned upload preset, no credential secrecy needed for that part. |
| Instagram publishing | **Working, confirmed live** | `Create Media Container` → `Publish to Instagram` (Meta Graph API), first real post 2026-07-04. Long-lived Page Access Token in place (~Sept 2026). |
| LinkedIn publishing (personal) | **Working, confirmed live** | WF1's `Post to LinkedIn` node (UGC Posts API) — first real post published 2026-07-03. |
| LinkedIn publishing (company) | Not built | No node/workflow yet — will use `s:` prefix. |
| Content generation (what creates a pending approval) | Built for LinkedIn + Instagram | `Save to Supabase` / `Save to Supabase (IG)` write `pending_approvals` rows for `workflow_type: 'linkedin'` / `'instagram'`. Company-LinkedIn equivalent not built. |
