---
name: headless-oauth
description: >
  Authorize any OAuth CLI on a headless server (no browser). Use when setting up
  CLI tools that require OAuth login on a VPS or server without a display —
  gog (Google Workspace), gh (GitHub CLI), gcloud, mcporter (MCP servers), and more.
  Implements three patterns: generate-URL / paste-back, device flow, and manual
  callback relay (for tools that start a local HTTP callback server). Includes a
  full setup guide for gog (Google Workspace).
version: 1.1.0
metadata:
  openclaw:
    emoji: "🔐"
    homepage: https://github.com/IgorIvanter/headless-oauth
    requires:
      bins: []
      env: []
---

# Headless OAuth

Authorize CLI tools that require OAuth on a headless server — no browser needed on the server side.

## ⚠️ Agent Context: You Are on a Remote Server

**You (the agent) are running on a remote VPS. The user is on a separate local machine with a browser.**

This means:
- You cannot open a browser yourself
- `localhost` and `127.0.0.1` on the server are NOT accessible from the user's browser
- The user must open all auth URLs on their own machine
- When a redirect goes to `http://127.0.0.1:PORT/callback?code=...`, it will **fail to load** in the user's browser — that is expected and normal
- The user should copy the full URL from the address bar (even if the page shows an error) and send it to you
- You then forward that URL to the waiting server process via `curl`

Always make this explicit when asking the user to authorize. Example:
> "Open this URL in your browser, log in, and approve. The page will likely fail to load — that's fine. Copy the full URL from the address bar and send it to me."

---

## The Pattern

Split the browser flow across two machines:

```
SERVER                          LOCAL MACHINE
------                          -------------
1. Generate auth URL    →       2. Open URL in browser
                                3. Log in + grant permissions
                                4. Copy redirect URL or code
5. Exchange for token   ←
6. Store token locally
```

Most OAuth CLIs support this via flags like:
- `--no-launch-browser` — gh (GitHub CLI), gcloud
- `--remote --step 1/2` — gog (Google Workspace CLI)
- `--manual` — some generic CLIs
- Device flow (code + URL, no redirect) — gh, some others

---

## Keyring on Headless Servers

Many CLIs store tokens in a system keyring, which requires a TTY to prompt for a password.
Fix by setting the keyring password via environment variable **before** running any auth command:

```bash
export KEYRING_PASSWORD="your-password"        # generic
export GOG_KEYRING_PASSWORD="your-password"    # gog CLI
```

> **Security note:** Set this variable only for the duration of the auth step or command — do not store it permanently in shell configs or agent-global environment. Use a secrets manager or ephemeral shell session instead.

---

## Google Workspace — gog CLI

See [`references/gog.md`](references/gog.md) for the full setup guide.

**Quick reference:**

```bash
export GOG_KEYRING_PASSWORD="your-keyring-password"
export GOG_ACCOUNT="you@gmail.com"

# Step 1: generate auth URL (run on server)
gog auth add you@gmail.com \
  --services gmail,calendar,drive,contacts,sheets,docs \
  --remote --step 1
# → prints: auth_url https://accounts.google.com/o/oauth2/auth?...

# Step 2: open URL locally, approve, copy redirect URL from address bar.
# Then on server:
gog auth add you@gmail.com \
  --services gmail,calendar,drive,contacts,sheets,docs \
  --remote --step 2 \
  --auth-url "http://127.0.0.1:PORT/oauth2/callback?code=...&state=..."

# Verify
gog auth list
gog calendar list --limit 3
```

> **Important:** Use **Desktop app** OAuth client type in Google Cloud Console — not Web application.
> gog uses a random port for the callback, which Web clients reject with `redirect_uri_mismatch`.

---

## GitHub CLI — gh

```bash
gh auth login --hostname github.com --git-protocol https --no-launch-browser
# → prints a one-time code and a URL
# Open https://github.com/login/device locally, enter the code
# gh polls and completes automatically once you approve
```

---

## gcloud (Google Cloud CLI)

```bash
gcloud auth login --no-launch-browser
# → prints a URL
# Open in local browser, log in, copy the verification code shown, paste back
```

---

## Generic Device Flow

If a CLI supports device flow (prints a short code + URL):

1. Note the code and URL printed by the CLI
2. Open the URL on any device
3. Enter the code
4. CLI polls and completes automatically — no redirect URL to copy

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `redirect_uri_mismatch` | Use Desktop app OAuth client, not Web application |
| `store token: no TTY available` | Set the CLI's keyring password env var |
| `Access blocked` (testing mode) | Add your email as test user in Google consent screen settings |
| Commands fail silently | Set `GOG_ACCOUNT=you@gmail.com` or equivalent |
| Token expired | Re-run auth steps; most CLIs handle refresh automatically |

---

## Manual Callback Relay (mcporter, custom OAuth servers)

Some tools (e.g. `mcporter`) start a local HTTP server on the server to catch the OAuth callback,
but the user's browser can't reach `127.0.0.1` on the remote server.

**How to handle it:**

1. Start the auth command with a long timeout so it doesn't expire:
   ```bash
   MCPORTER_OAUTH_TIMEOUT_MS=300000 mcporter auth <server> --log-level info
   ```
2. The tool prints a line like:
   ```
   If the browser did not open, visit https://... manually.
   ```
   Send that URL to the user.
3. Tell the user:
   > "Open this URL, log in and approve. The page will fail to load — that's normal. Copy the full URL from the address bar and send it to me."
4. The user sends back something like:
   `http://127.0.0.1:PORT/callback?code=...&state=...`
5. Forward it to the waiting server with curl:
   ```bash
   curl -s "http://127.0.0.1:PORT/callback?code=...&state=..."
   ```
6. The tool receives the code, exchanges it for a token, and completes authorization.

---

## Applying This Pattern to Any CLI

1. Check `--help` for flags like `--no-launch-browser`, `--remote`, `--manual`, or `--headless`
2. Check docs for "device flow" or "offline access"
3. If the tool starts a local callback server but has no headless flag — use the Manual Callback Relay pattern above
4. If none of the above work: use `ssh -L` port forwarding to tunnel the callback to your local machine
