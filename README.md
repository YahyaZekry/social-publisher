# social-publisher

n8n automation that drafts and publishes content to personal LinkedIn and
personal Instagram (company LinkedIn planned), gated behind a Telegram
approval step. One Telegram bot, one n8n workflow, one message prefix per
platform.

## How it works

1. You text the bot: `y: <topic>` for LinkedIn, `ig: <topic>` for Instagram.
2. An LLM (Groq) drafts the content — a LinkedIn post, or an image prompt +
   Instagram caption. For Instagram, the image is generated (Pollinations)
   and uploaded (Cloudinary).
3. A row is written to Supabase (`pending_approvals`), and the bot sends you
   a preview in Telegram.
4. You reply:
   - `/approve` — actually publishes it (LinkedIn UGC API / Instagram Graph API).
   - `/edit <instructions>` (LinkedIn) or `/regenerate` (Instagram) — redoes
     the content and sends a new preview.
   - `/discard` — cancels.

Everything lives in **one n8n workflow** (`WF1 - LinkedIn Post (Personal)` —
the name is a holdover from before Instagram was merged in). This is
deliberate, not an accident: Telegram only allows **one webhook per bot**, so
every platform's drafting logic has to live in the same workflow, routed by
message prefix, rather than each getting its own workflow with its own
Telegram trigger. See `.project-knowledge/features.md` for the full
node-by-node breakdown, and `.project-knowledge/history.md` for why this
came up (two earlier workflows silently stole each other's webhook before
being merged in).

`workflows/command-listener-deprecated.json` and
`workflows/instagram-post-personal-deprecated.json` are historical exports
from before their logic got merged into WF1 — kept for reference only, not
needed to run this.

---

## Setting this up from scratch

### 1. Prerequisites

- n8n instance (self-hosted or cloud) reachable at a public URL for Telegram
  webhooks (this project uses Docker + ngrok locally — see `.project-knowledge/stack.md`).
- A Telegram bot ([@BotFather](https://t.me/BotFather)) and your own numeric
  Telegram user ID.
- A Supabase project with a `pending_approvals` table (schema below).
- A [Groq](https://console.groq.com) API key (free tier works).
- A [Cloudinary](https://cloudinary.com) account with an **unsigned** upload preset (Instagram only).
- LinkedIn API access with a valid access token + your LinkedIn person URN (LinkedIn only).
- A Meta Developer app + Facebook Page + linked Instagram Business account, with
  a long-lived Page Access Token (Instagram only) — see
  `.project-knowledge/integrations.md` for the exact `fb_exchange_token` flow
  used to generate one.

### 2. Supabase table

```sql
create table pending_approvals (
  id uuid primary key default gen_random_uuid(),
  telegram_user_id bigint not null,
  workflow_type text not null,        -- 'linkedin' | 'instagram' | ...
  resume_url text,
  content_preview text,
  status text not null check (status in ('pending', 'approved', 'rejected')),
  edit_instructions text,
  created_at timestamptz not null default now()
);
```

(This is reconstructed from usage, not exported from Supabase directly —
double-check against your actual table before relying on it verbatim.)

### 3. Import the workflow

In n8n: **Workflows → Import from File** → `workflows/linkedin-post-personal.json`.

### 4. Fill in every placeholder

The exported JSON has real secrets stripped and replaced with placeholder
tokens. Open the workflow and search for each of these, then replace with
your own values:

| Placeholder | Where | Get it from |
|---|---|---|
| `ZkWfYSlTSfJ42VwR` (credential ID) | Every Telegram node | Create a Telegram credential in n8n (bot token from BotFather), then re-attach it to each Telegram node — the old credential ID won't exist in your instance |
| `REPLACE_WITH_GROQ_API_KEY` | `Generate LinkedIn Post`, `Rewrite with Instructions`, `Build Image Prompt`, `Generate Caption` (4 places) | console.groq.com |
| `REPLACE_WITH_SUPABASE_SERVICE_ROLE_KEY` | `Save to Supabase`, `Get Pending Approval`, `Update Status`, `Save to Supabase (IG)` (both `apikey` and `Authorization` headers on each) | Supabase project settings → API |
| `xiqohqytcyjiqskkextz.supabase.co` (URL) | Same 4 nodes | Your Supabase project URL |
| `REPLACE_WITH_LINKEDIN_ACCESS_TOKEN` | `Post to LinkedIn` | LinkedIn API access token |
| `urn:li:person:kvFSUycND7` | `Post to LinkedIn` body | Your own LinkedIn person URN |
| `REPLACE_WITH_META_PAGE_ACCESS_TOKEN` | `Create Media Container`, `Publish to Instagram` | Meta Graph API long-lived Page Access Token |
| `17841476339271624` (IG Business Account ID) | Both Instagram publish nodes' URLs | Your Instagram Business Account ID |
| `ttloffm8` (Cloudinary upload preset) | `Upload to Cloudinary` | Your Cloudinary unsigned upload preset name |
| `cbolcssr` (Cloudinary cloud name) | `Upload to Cloudinary` URL | Your Cloudinary cloud name |

None of these are optional — the workflow won't run correctly until every
one is replaced with a value for **your own** accounts.

### 5. Activate it

Toggle the workflow **Active**, confirm Telegram's webhook is registered to
it (`curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo`), and text the
bot `y: test` or `ig: test`.

---

## Day-to-day usage

| You send | What happens |
|---|---|
| `y: <topic>` | Drafts a LinkedIn post about that topic |
| `ig: <topic>` | Drafts an Instagram image + caption about that topic |
| `/approve` | Publishes the most recent pending draft |
| `/edit <instructions>` | (LinkedIn) Rewrites the draft per your instructions, sends a new preview |
| `/regenerate` | (Instagram) Redoes the image + caption from the same topic, sends a new preview |
| `/discard` | Cancels the pending draft |

Only one draft is "pending" at a time per user — sending a new `y:`/`ig:`
message before approving/discarding the previous one just creates a newer
row; `/approve` always acts on the most recent one.

## Customizing the writing voice

Each platform's tone lives entirely in its Groq system prompt — there's no
shared prompt:
- LinkedIn: `Generate LinkedIn Post` / `Rewrite with Instructions`
- Instagram: `Build Image Prompt` (image description only) / `Generate Caption`

Edit the `content` field of the `role: 'system'` message in each node's Body
to change voice, length, hashtag rules, or what topics it will/won't cover.

## Updating this repo after editing a workflow in the n8n UI

```bash
docker exec n8n n8n export:workflow --backup --output=/home/node/.n8n/workflow-backups/
docker cp n8n:/home/node/.n8n/workflow-backups/9lo7Lhwn3tg2cC6P.json ./workflows/linkedin-post-personal.json
docker exec n8n rm -rf /home/node/.n8n/workflow-backups/
```

**Before committing**, redact any real secrets the export re-introduces
(Groq key, Supabase key, LinkedIn token, Meta token) back to the
`REPLACE_WITH_*` placeholders shown above — the live n8n instance is
unaffected either way, only this repo's copy needs to stay secret-free.

## Known open issues

See `.project-knowledge/roadmap.md` for the current list — notably,
Instagram's generated images/captions/hashtags are still considered
low-quality as of the last update, and several credentials are still
hardcoded on nodes rather than proper n8n credentials.

---

See `.project-knowledge/` for the full evolving history, architecture notes,
and every bug hit along the way.
