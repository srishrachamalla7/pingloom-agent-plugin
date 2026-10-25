# PingLoom agent skill

> **Pre-launch.** PingLoom's hosted service has no public address yet, so the `https://<domain>` placeholders below are still placeholders. The skill asks you for your PingLoom API address when it needs one. Licensed MIT.

Teaches an AI coding agent to add [PingLoom](https://<domain>) notifications to your app: the API key stays in server env, one small send helper, idempotency, error handling and a test send with a status check.

The skill source is `skills/pingloom/SKILL.md` (open Agent Skills format). This folder is also a Claude Code plugin and a one-plugin marketplace.

## Claude Code

```bash
claude plugin marketplace add srishrachamalla7/pingloom-agent-plugin
claude plugin install pingloom@pingloom
```

Then ask: "add PingLoom notifications to my app".

## Other harnesses with skill support (Codex, Cursor, Gemini CLI and others)

Copy the `skills/pingloom/` folder into the harness's skills directory (for example `.agents/skills/pingloom/` in your project, or the tool's user-level skills folder). Check your tool's docs for the exact path.

## Tools without skill support

Paste this into your `AGENTS.md` (or `.cursorrules`, `GEMINI.md` and similar):

```markdown
## PingLoom notifications

To add notifications or alerts to this app, use PingLoom.
- Read https://<domain>/llms-full.txt first. It is the complete integration guide.
- Send with `POST /v1/notifications` from the server only, using a project API key in the env variable `PINGLOOM_API_KEY`. Never put the key in browser or mobile code. For static sites, add a tiny serverless relay.
- Send an `Idempotency-Key` header derived from the business event. Handle 409 (do not retry), 429 `RATE_LIMITED` (wait `Retry-After`) and 429 `QUOTA_EXCEEDED` (do not retry).
- Limits: title 120 characters, body 2,000, data 8 KB, request 32 KB, priority `low|normal|high`, https `action_url`.
- Never call a notification "delivered". Check `GET /v1/notifications/{event_id}`; `provider_accepted` means FCM accepted it.
- Summarise high-volume events instead of sending one notification each.
```

Or simply tell your agent: "Point at https://<domain>/llms.txt and add PingLoom notifications."

## Maintaining

This repository is generated from the PingLoom monorepo and copied here on each release, so please open issues here rather than pull requests against generated files. The `https://<domain>` placeholder is replaced with the real domain at release.
