---
name: pingloom
description: Add PingLoom push notifications and alerts to the user's app or backend. Use when the user asks to "add notifications", "add PingLoom", "send me an alert/push when X happens", or "notify my team from my server". Wires up a server-side helper for POST /v1/notifications with an API key kept in server env, an idempotency key, error handling and a test send.
---

# Add PingLoom notifications to an app

PingLoom is notification-as-a-service. A server calls `POST /v1/notifications` with a project API key. PingLoom stores the event, then sends a minimal push through FCM to the phones of people who follow the channel. The full message lives in the team inbox.

Full reference for the API: `https://<domain>/llms-full.txt`. If that is reachable, prefer it over memory for any detail not below.

## Hard rules

1. The API key (`pl_test_...` in dev and staging, `pl_live_...` in production) is a server secret. Read it from the environment variable `PINGLOOM_API_KEY`. Never put it in browser code, a mobile app, `NEXT_PUBLIC_*` or `VITE_*` variables, or git. Add it to `.env.example` as an empty value and make sure `.env` is git-ignored.
2. Never call PingLoom from client-side code. Client code calls the user's own server, which calls PingLoom.
3. Never tell the user a notification was "delivered". The status is `provider_accepted` (FCM accepted the send). That is not proof a phone showed it. Statuses: `accepted`, `queued`, `provider_accepted`, `simulated_accepted` (test setups, no real push), `no_devices`, `retrying`, `failed`.
4. Never put secrets, tokens or personal data in `title`, `body` or `data`. Pushes pass through Google FCM.

## Steps

### 1. Detect the stack

Look at the project files: `package.json` (Node, Next.js, Express, Nest), `pyproject.toml` or `requirements.txt` (Python, FastAPI, Django, Flask), `vercel.json` or `netlify.toml` (serverless), or only HTML/JS with no server (static site). Use the language and error-handling style already in the project. Node 18+ has `fetch`; do not add an HTTP dependency just for this.

### 2. Key and base URL

Ask the user to create a project and an API key in the PingLoom dashboard (Project, API keys, Create key; the key is shown once and needs the `notifications:send` scope, plus `events:read` to check status). They put it in the server env as `PINGLOOM_API_KEY`. Also read `PINGLOOM_API_URL` (default `https://<domain>`; replace with the real API origin the user gives you).

Do not ask the user to paste the key into the chat. Write the variable name into the env file and let them fill it.

### 3. Static site or browser-only app

If there is no server, add a tiny serverless relay (a Vercel/Netlify/Cloudflare function) that holds the key. The relay must:

- accept only `POST`;
- accept only a fixed allow-list of event names, and build title and body itself (never forward browser text);
- check the `Origin` header matches the site;
- cap volume (for example a fixed idempotency key per hour bucket so repeats are not metered).

### 4. Write one helper

One small function, one file. Node (ESM, no dependencies):

```js
// pingloom.js - server side only
const API_URL = process.env.PINGLOOM_API_URL ?? "https://<domain>";

export async function notify({ title, body, channel = "general", priority = "normal", data, actionUrl, idempotencyKey }) {
  const key = process.env.PINGLOOM_API_KEY;
  if (!key) {
    console.warn("PINGLOOM_API_KEY is not set; skipping notification");
    return null;
  }
  const headers = { Authorization: `Bearer ${key}`, "Content-Type": "application/json" };
  if (idempotencyKey) headers["Idempotency-Key"] = idempotencyKey;
  const payload = { title, body, channel, priority, data, action_url: actionUrl };
  for (let attempt = 0; attempt < 3; attempt++) {
    let res;
    try {
      res = await fetch(`${API_URL}/v1/notifications`, {
        method: "POST",
        headers,
        body: JSON.stringify(payload),
        signal: AbortSignal.timeout(5000),
      });
    } catch (err) {
      if (attempt === 2) throw err;
      continue;
    }
    if (res.status === 202) return res.json();
    const { error } = await res.json().catch(() => ({}));
    if (res.status === 429 && error?.code === "RATE_LIMITED") {
      const wait = Number(res.headers.get("Retry-After") ?? 1);
      await new Promise((r) => setTimeout(r, Math.min(wait, 10) * 1000));
      continue;
    }
    if (res.status >= 500 && attempt < 2) continue;
    // 400/401/403/409/413/415/422 and QUOTA_EXCEEDED will not succeed on retry
    throw new Error(`PingLoom ${res.status} ${error?.code ?? ""} ${error?.message ?? ""} (request_id ${error?.request_id ?? "?"})`);
  }
}
```

Python (standard library plus `requests`, or swap in `httpx`):

```python
# pingloom.py - server side only
import os
import time
import requests

API_URL = os.environ.get("PINGLOOM_API_URL", "https://<domain>")


def notify(title, body=None, channel="general", priority="normal", data=None, action_url=None, idempotency_key=None):
    key = os.environ.get("PINGLOOM_API_KEY")
    if not key:
        print("PINGLOOM_API_KEY is not set; skipping notification")
        return None
    headers = {"Authorization": f"Bearer {key}"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    payload = {"title": title, "body": body, "channel": channel, "priority": priority, "data": data, "action_url": action_url}
    payload = {k: v for k, v in payload.items() if v is not None}
    for attempt in range(3):
        try:
            res = requests.post(f"{API_URL}/v1/notifications", json=payload, headers=headers, timeout=5)
        except requests.RequestException:
            if attempt == 2:
                raise
            continue
        if res.status_code == 202:
            return res.json()
        error = (res.json() if res.content else {}).get("error", {})
        if res.status_code == 429 and error.get("code") == "RATE_LIMITED":
            time.sleep(min(float(res.headers.get("Retry-After", 1)), 10))
            continue
        if res.status_code >= 500 and attempt < 2:
            continue
        raise RuntimeError(f"PingLoom {res.status_code} {error.get('code')} {error.get('message')} (request_id {error.get('request_id')})")
```

What the helper must get right:

- Fire and forget. A failed notification must never break the user's request: call it without `await` in a request path, or catch and log the error. Log only `code`, `message` and `request_id`, never the key.
- Idempotency: pass an `Idempotency-Key` derived from the business event (for example `order-1042-paid`), 1 to 255 printable ASCII characters. The same key with the same body returns the original event for 24 hours. The same key with a different body returns 409 `IDEMPOTENCY_CONFLICT`: that is a bug in the caller, do not retry.
- 429 has two causes. `RATE_LIMITED` means wait `Retry-After` and retry. `QUOTA_EXCEEDED` means the monthly plan is used up: retrying will not help; log it and tell the user.
- Errors always look like this, and 5xx responses are safe to retry with the same idempotency key:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request has invalid fields.",
    "request_id": "req_01J9ZQ8M3K",
    "details": [{ "field": "title", "message": "Title is required.", "code": "REQUIRED" }]
  }
}
```

### 5. Field limits

Only `title` is required.

| Field | Rule |
| --- | --- |
| `title` | 1 to 120 characters |
| `body` | up to 2,000 characters |
| `channel` | slug matching `^[a-z0-9][a-z0-9-]{0,39}$`, default `general`; must exist in the project, else 422 `INVALID_CHANNEL` |
| `priority` | `low`, `normal` or `high` |
| `data` | JSON object, at most 8 KB; not sent in the push |
| `action_url` | absolute `https` URL; PingLoom never fetches it |
| `event_type`, `external_id`, `occurred_at` | optional; see the full reference |

The whole request body is at most 32 KB. Any unknown field returns 422 `UNKNOWN_FIELD`. Example body:

```json
{
  "channel": "payments",
  "title": "Payment received: $49.00",
  "body": "Order 1042 was paid.",
  "priority": "normal",
  "data": { "order_id": "1042" },
  "action_url": "https://example.com/orders/1042"
}
```

### 6. Suggest which events to send

Read the codebase for places where something worth a human's attention happens, then propose a short list and let the user pick before wiring them. Good candidates: payment succeeded or failed, new signup, deploy finished or failed, background job failed, error rate spike, queue backed up, security events (new admin login, password reset flood), milestones.

Use one channel per kind of stream (`payments`, `deploys`, `errors`) so people follow only what they need. Use `high` priority sparingly, for things someone should act on now.

For high-volume events (page views, every signup on a busy product, every request), do not send one notification each: it buzzes phones and burns the monthly quota. Send a daily or hourly summary instead ("Yesterday: 412 signups, 58 orders"), or only the first event past a threshold. Never send user-typed content.

### 7. Verify

1. Make sure the key is set in the server env, then send one test event through the new helper (a one-off script or the dev server).
2. Expect HTTP 202 with an `event_id` and `status` `accepted`.
3. Read it back, using a key with the `events:read` scope:

```
GET /v1/notifications/{event_id}
```

```bash
curl "$PINGLOOM_API_URL/v1/notifications/$EVENT_ID" -H "Authorization: Bearer $PINGLOOM_API_KEY"
```

4. Report the status in plain words. `provider_accepted` means FCM accepted the send; `simulated_accepted` means a test setup without Firebase, no real push was made; `no_devices` means nobody has the mobile app signed in and following the channel (the event is still in the web inbox). Do not say "delivered".

Finish by summarising for the user: files changed, the env variable to set, which events now send, and anything you did not verify (for example, that a real phone showed the alert).
