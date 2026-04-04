# 🔐 headless-oauth

> **AgentSkill for [OpenClaw](https://openclaw.ai)** — Authorize any OAuth CLI on a headless server, no browser required.

[![clawhub](https://img.shields.io/badge/clawhub-headless--oauth-blue)](https://clawhub.ai/skills/headless-oauth)
[![License: MIT-0](https://img.shields.io/badge/License-MIT--0-green.svg)](https://opensource.org/licenses/MIT-0)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-compatible-purple)](https://openclaw.ai)

---

## The Problem

OAuth requires a browser. On a headless VPS or server, there is no browser. Most CLI tools hang or crash with an unhelpful error when you try to authenticate.

## The Solution

Split the OAuth flow across two machines:

```
VPS / Server                    Your Local Machine
────────────────                ──────────────────
1. Generate auth URL    ──────► 2. Open URL in browser
                                3. Log in + grant permissions
                                4. Copy redirect URL or code
5. Exchange for token   ◄──────
6. Token stored locally ✓
```

This skill teaches your OpenClaw agent exactly how to do this for the most common CLI tools.

---

## Supported Tools

| Tool | Method | Guide |
|------|--------|-------|
| **gog** (Google Workspace) | `--remote --step 1/2` | [references/gog.md](references/gog.md) |
| **gh** (GitHub CLI) | Device flow (`--no-launch-browser`) | [SKILL.md](SKILL.md#github-cli--gh) |
| **gcloud** (Google Cloud) | `--no-launch-browser` | [SKILL.md](SKILL.md#gcloud-google-cloud-cli) |
| Any OAuth CLI | General pattern | [SKILL.md](SKILL.md#applying-this-pattern-to-any-cli) |

---

## Install

```bash
# via clawhub CLI
npx clawhub@latest install headless-oauth
```

Or clone manually:

```bash
git clone https://github.com/IgorIvanter/headless-oauth.git
# Place the folder in your OpenClaw workspace/skills/ directory
```

---

## Quick Example: GitHub CLI on a VPS

```bash
gh auth login --hostname github.com --git-protocol https --no-launch-browser
# → Prints a one-time code like: ABCD-1234
# → Open https://github.com/login/device on your local machine
# → Enter the code — gh polls and completes automatically
```

## Quick Example: Google Workspace (gog CLI)

```bash
export GOG_KEYRING_PASSWORD="your-password"

# On server — generates the auth URL
gog auth add you@gmail.com \
  --services gmail,calendar,drive,contacts \
  --remote --step 1

# Open the printed URL locally, approve, copy redirect URL from address bar
# Then back on server:
gog auth add you@gmail.com \
  --services gmail,calendar,drive,contacts \
  --remote --step 2 \
  --auth-url "http://127.0.0.1:PORT/oauth2/callback?code=...&state=..."
```

---

## Keyring Note

Many CLIs store tokens in a system keyring, which requires a TTY. On a headless server, set the password via env var:

```bash
export GOG_KEYRING_PASSWORD="your-keyring-password"
# Add to ~/.bashrc or your OpenClaw env config
```

---

## Common Errors

| Error | Fix |
|-------|-----|
| `redirect_uri_mismatch` | Use **Desktop app** OAuth client, not Web application |
| `store token: no TTY available` | Set the keyring password env var |
| `Access blocked` | Add your email as test user in Google consent screen |
| Commands fail silently | Set `GOG_ACCOUNT=you@gmail.com` |

---

## Files

```
headless-oauth/
├── SKILL.md              # AgentSkill instructions
└── references/
    └── gog.md            # Full gog CLI setup guide (Google Workspace)
```

---

## About AgentSkills

This is an [AgentSkill](https://openclaw.ai) — a Markdown-based instruction set for OpenClaw agents. When installed, the agent automatically knows how to handle headless OAuth flows for you.

Built by [Igor Ivanter](https://igorivanter.com) · Published on [ClawHub](https://clawhub.ai/skills/headless-oauth)
