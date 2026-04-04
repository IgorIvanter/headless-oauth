# gog CLI — Google Workspace on Headless Server

Full setup guide for gog on Linux VPS.

## Installation

Install gog CLI following the official instructions:
https://github.com/steipete/gogcli

## Google Cloud Console Setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials
2. Create OAuth client → **Desktop app** (NOT Web application — gog uses random ports)
3. Download the credentials file
4. Enable required APIs: Gmail, Calendar, Drive, Contacts, Sheets, Docs

## Store Credentials

```bash
gog auth credentials set /path/to/credentials.json
```

## Keyring

gog stores tokens in a system keyring. On a headless server, provide the keyring password
non-interactively using the environment variable documented in gog's README, set for the
duration of the auth step only.

## Authorize Account (Headless)

```bash
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

## Common Pitfalls

| Problem | Fix |
|---------|-----|
| `redirect_uri_mismatch` | Use Desktop app OAuth client, not Web application |
| Keyring unlock fails | Set the keyring password env var (see gog README) |
| `Access blocked` (testing mode) | Add email as test user in Google consent screen settings |
| Commands fail without error | Check gog README for required account env var |

## Key Commands

```bash
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

gog handles access token refresh automatically.
