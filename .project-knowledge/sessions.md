# Session Log

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-02
> Append-only — never edit past entries.

| Date | Summary |
|------|---------|
| 2026-07-02 | Diagnosed and fixed the full Telegram approval chain: confirmed trust-proxy was already fixed, resolved a rate-limit/crash loop by resetting the webhook and recreating the container, fixed a stale-session permission error, fixed a truncated Supabase API key + missing `Bearer` prefix, fixed unset field expressions in `Extract Command`, and fixed a silent-no-reply bug via "Always Output Data". Set up this repo (`social-publisher`) with exported workflows and `.project-knowledge/`, since n8n's native git push isn't available on this license. |
