---
name: headless-oauth
description: >
  Authorize CLI tools that require OAuth on a headless server (no browser).
  Use when setting up any CLI tool that needs OAuth login on a VPS or server
  without a display — e.g., gog (Google Workspace), gh (GitHub), Spotify CLI,
  Twitter CLI, etc. Handles the --remote/--no-launch-browser/device-flow patterns:
  generate auth URL on server, user opens it in a local browser, pastes back the
  redirect URL or one-time code. Includes full setup guide for gog CLI (Google Workspace).
metadata:
  openclaw:
    requires:
      bins: []
    tags:
      - oauth
      - auth
      - headless
      - vps
      - server
      - google-workspace
      - cli
---

# Headless OAuth

Authorize CLI tools that require OAuth on a headless server — no browser needed on the server side.

## The Problem

OAuth requires a browser: open URL → log in → grant permissions → get token.
On a headless VPS, step one is impossible. Most CLIs hang or crash with an unhelpful error.

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
Fix by setting the keyring password via environment variable before running any auth command:

```bash
export KEYRING_PASSWORD="your-password"        # generic
export GOG_KEYRING_PASSWORD="your-password"    # gog CLI
```

Add to `~/.bashrc` or your OpenClaw environment config for persistence.

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

# Step 2: open that URL locally, approve, copy redirect URL from address bar
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

## Applying This Pattern to Any CLI

1. Check `--help` for flags like `--no-launch-browser`, `--remote`, `--manual`, or `--headless`
2. Check docs for "device flow" or "offline access"
3. If none exist: use `ssh -L` port forwarding to tunnel the callback to your local machine
