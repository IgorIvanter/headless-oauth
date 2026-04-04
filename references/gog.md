# gog CLI — Google Workspace on Headless Server

Full setup guide for gog v0.12.0 on Linux VPS.

## Installation

```bash
# Check latest version at https://github.com/steipete/gogcli/releases
VERSION=0.12.0
curl -fsSL "https://github.com/steipete/gogcli/releases/download/v${VERSION}/gogcli_${VERSION}_linux_amd64.tar.gz" \
  -o /tmp/gog.tar.gz

# Verify the download (check the release page for checksums)
sha256sum /tmp/gog.tar.gz

tar -xzf /tmp/gog.tar.gz -C /usr/local/bin/ gog
gog --version
```

## Google Cloud Console Setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials
2. Create OAuth client → **Desktop app** (NOT Web application — gog uses random ports)
3. Download `credentials.json`
4. Enable required APIs: Gmail, Calendar, Drive, Contacts, Sheets, Docs

## Store Credentials

```bash
gog auth credentials set /path/to/credentials.json
# → stored at ~/.config/gogcli/credentials.json
```

## Authorize Account (Headless)

```bash
# Set keyring password only for the duration of the auth step
# Do not permanently store this in shell configs or agent env
export GOG_KEYRING_PASSWORD="your-keyring-password"

# Step 1: generate auth URL (run on server)
gog auth add you@gmail.com \
  --services gmail,calendar,drive,contacts,sheets,docs \
  --remote --step 1

# Step 1 output: auth_url https://accounts.google.com/o/oauth2/auth?...
# Open that URL in your local browser, approve permissions.
# After approval, browser redirects to http://127.0.0.1:PORT/oauth2/callback?code=...&state=...
# The page won't load — that's fine. Copy the full URL from the address bar.

# Step 2: exchange code (run on server)
gog auth add you@gmail.com \
  --services gmail,calendar,drive,contacts,sheets,docs \
  --remote --step 2 \
  --auth-url "http://127.0.0.1:PORT/oauth2/callback?code=...&state=..."
```

> **Security note:** `GOG_KEYRING_PASSWORD` is needed only during the auth step and when running gog commands. Avoid storing it permanently in plain text. Prefer injecting it via your secrets manager, a restricted env file, or an ephemeral shell session.

## Common Pitfalls

| Problem | Fix |
|---------|-----|
| `redirect_uri_mismatch` | Use Desktop app OAuth client, not Web application |
| `store token: no TTY available` | Set `GOG_KEYRING_PASSWORD` env var before running the command |
| `Access blocked` (testing mode) | Add email as test user OR publish the app in consent screen settings |
| Commands fail without error | Set `GOG_ACCOUNT=you@gmail.com` |

## Key Commands

```bash
# Set required env vars before running commands
export GOG_KEYRING_PASSWORD="your-keyring-password"
export GOG_ACCOUNT="you@gmail.com"

# Calendar
gog calendar list --limit 10
gog calendar create primary --summary "Meeting" \
  --from "2026-04-05T10:00:00+02:00" \
  --to "2026-04-05T11:00:00+02:00"

# Gmail
gog gmail list --limit 10
gog gmail send --to recipient@example.com --subject "Subject" --body "Text"

# Drive
gog drive list --limit 10

# Sheets
gog sheets read SHEET_ID --range "Sheet1!A1:D10"
```

## Token Lifetime

Google refresh tokens don't expire as long as:
- Not manually revoked in Google Account settings
- The app is used at least once every 6 months (for unpublished apps)
- No new authorization was made (can invalidate old token)

gog handles access token refresh automatically.
