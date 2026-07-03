# Session Log

> Part of social-publisher/.project-knowledge/ | Last updated: 2026-07-03
> Append-only — never edit past entries.

| Date | Summary |
|------|---------|
| 2026-07-02 | Diagnosed and fixed the full Telegram approval chain: confirmed trust-proxy was already fixed, resolved a rate-limit/crash loop by resetting the webhook and recreating the container, fixed a stale-session permission error, fixed a truncated Supabase API key + missing `Bearer` prefix, fixed unset field expressions in `Extract Command`, and fixed a silent-no-reply bug via "Always Output Data". Set up this repo (`social-publisher`) with exported workflows and `.project-knowledge/`, since n8n's native git push isn't available on this license. |
| 2026-07-03 | Got WF1 (LinkedIn Post - Personal) publishing and drafting end-to-end: fixed a missing Telegram credential on 4 nodes, fixed unset `Extract Content` field expressions (same bug class as WF4), and added a missing `Extract Post Text` node that 6 other nodes referenced but never existed. Diagnosed each step directly from the sqlite execution data rather than trusting the UI, since several rounds of manual expression edits silently saved as literal strings instead of real expressions. Confirmed via live Telegram test: `y: <topic>` now generates a real LinkedIn post via Groq and returns a preview. Along the way, discovered WF1 and WF4 silently contend for the same Telegram webhook (logged as a bug in `roadmap.md`) — `/approve` won't reach WF4 until that's resolved. Exported the updated workflow to `workflows/linkedin-post-personal.json` and pushed. |
