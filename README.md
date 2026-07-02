# social-publisher

n8n automation that posts content to personal Instagram, personal LinkedIn, and
company LinkedIn — gated behind a Telegram approval step.

## How it works

1. Something (TBD — content generation step not built yet) creates a row in the
   Supabase `pending_approvals` table for a given `telegram_user_id`.
2. The owner sends a command to the Telegram bot (e.g. `/approve`).
3. The `WF4 - Command Listener` n8n workflow looks up the pending approval,
   resumes/updates it, and replies in Telegram.
4. (Not yet built) An actual publish step pushes the approved content to
   Instagram / LinkedIn.

## Runtime

- n8n running in Docker locally (container: `n8n`, image `docker.n8n.io/n8nio/n8n`,
  version 2.28.4), tunneled via ngrok for the Telegram webhook.
- Workflow data lives in n8n's own SQLite volume (`n8n_data`) — this repo is a
  version-controlled *export* of the workflows, not the source of truth at runtime.
- n8n's native git Source Control is Business/Enterprise-only and unavailable on
  this license, so workflows are versioned manually via
  `n8n export:workflow --backup` into `workflows/`.

## Updating this repo after editing a workflow in the n8n UI

```bash
docker exec n8n n8n export:workflow --backup --output=/home/node/.n8n/workflow-backups/
docker cp n8n:/home/node/.n8n/workflow-backups/. ./workflows/
docker exec n8n rm -rf /home/node/.n8n/workflow-backups/
```

Then rename any new files from their n8n workflow ID to something readable.

## External services

- **Telegram Bot API** — approval commands + replies.
- **Supabase** (`xiqohqytcyjiqskkextz`) — `pending_approvals` table.
- **Instagram / LinkedIn APIs** — not yet integrated.

See `.project-knowledge/` for detailed, evolving notes.
