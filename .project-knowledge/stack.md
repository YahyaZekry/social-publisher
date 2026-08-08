# Stack

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-08-08

## Tech Stack

| Category | Details |
|----------|---------|
| Automation engine | n8n 2.28.4, self-hosted in Docker |
| Container | `n8n` (image `docker.n8n.io/n8nio/n8n`), volume `n8n_data:/home/node/.n8n`, port 5678:5678 |
| Tunnel | ngrok, static domain `ergonomic-password-rosy.ngrok-free.dev` |
| Bot platform | Telegram Bot API |
| Database | Supabase (project ref `xiqohqytcyjiqskkextz`) — `pending_approvals` table |
| LLM | OpenRouter (`openai/gpt-4o-mini`) — LinkedIn post text + Instagram captions; Groq (`llama-3.3-70b-versatile`) — FLUX image prompts. **All 6 API keys now live in n8n credentials (2026-08-08)** — see `integrations.md` for the full list |
| Image generation | Pollinations (`image.pollinations.ai`) — free, no auth, text-to-image only |
| Image hosting | Cloudinary (unsigned upload preset) |
| Publish targets | Personal LinkedIn (built), Instagram (built) — both live; company LinkedIn not yet integrated |
| License | n8n Community/base license (1 entitlement) — no Source Control (Business/Enterprise-only) |

---

## Dev Commands

| Command | What It Does |
|---------|---------------|
| `docker restart n8n` | Restart n8n without losing workflow data |
| `docker logs n8n --tail 50 --timestamps` | Recent n8n logs |
| `docker exec n8n n8n export:workflow --backup --output=/home/node/.n8n/workflow-backups/` | Export all workflows (pretty, one file per workflow) |
| `docker cp n8n:/home/node/.n8n/workflow-backups/. ./workflows/` | Pull exported workflows into this repo |
| `curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo` | Check current Telegram webhook registration |
| `curl -X POST https://api.telegram.org/bot<TOKEN>/deleteWebhook` | Reset Telegram webhook (use if rate-limited or re-registering) |

---

## Environment Variables

| Variable | Used In | What It Enables |
|----------|---------|------------------|
| `WEBHOOK_URL` | n8n container | Public base URL (ngrok) n8n uses when registering webhooks |
| `N8N_PROXY_HOPS` | n8n container | Set to `1` — tells Express to trust 1 hop of `X-Forwarded-For` (fixes "trust proxy" error behind ngrok) |

Bot token and every API key now live as **n8n credentials** (not hardcoded in
the workflow JSON) since 2026-08-08 — the repo's `workflows/main-workflow.json`
contains **no secrets**, only generic `-REPLACE` credential IDs that n8n
re-prompts you to match by name on import. Real values are kept locally in the
gitignored `.credentials.env` (see repo README section 4).
