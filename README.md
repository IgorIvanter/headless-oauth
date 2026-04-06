# 🔐 headless-oauth

> **AgentSkill for [OpenClaw](https://openclaw.ai)** — Handle OAuth for terminal-based and server-based tools on a headless machine, no browser on the server required.

[![clawhub](https://img.shields.io/badge/clawhub-headless--oauth-blue)](https://clawhub.ai/skills/headless-oauth)
[![License: MIT-0](https://img.shields.io/badge/License-MIT--0-green.svg)](https://opensource.org/licenses/MIT-0)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-compatible-purple)](https://openclaw.ai)

---

## The Problem

OAuth often assumes the browser and the tool live on the same machine. On a headless VPS or server, that assumption breaks. Terminal-based tools, MCP servers, and server-side workflows can hang, crash, or fail with unhelpful errors when authentication starts on the server but must be completed on your laptop.

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

This skill teaches your OpenClaw agent how to handle that split cleanly for terminal-based tools, MCP environments, and other server-side workflows that rely on OAuth.

---

## Three Patterns

| Pattern | How it works | Example tools |
|---------|-------------|---------------|
| **Generate URL / paste back** | Tool prints auth URL, user opens locally, pastes redirect URL or code back | gog, gcloud |
| **Device flow** | Tool prints a short code + URL, user enters the code, server-side process polls automatically | gh (GitHub CLI) |
| **Manual callback relay** | Tool starts a local HTTP server for callback; user copies the failed redirect URL from browser, agent relays it back manually | ClickUp MCP, mcporter, MCP servers |

---

## Read the article

I wrote a fuller explanation of the pattern here:

**[How to Handle OAuth for OpenClaw Workflows on a Headless Server](https://igorivanter.com/how-to-authorize-any-cli-on-a-headless-server-without-a-browser/)**

If you want the reasoning, tradeoffs, and real-world examples behind this skill, start there.

## Install

```bash
npx clawhub@latest install headless-oauth
```

Or clone manually and place in your OpenClaw `workspace/skills/` directory.

---

## Quick Example: GitHub CLI on a VPS

```bash
gh auth login --hostname github.com --git-protocol https --no-launch-browser
# → Prints a one-time code like: ABCD-1234
# → Open https://github.com/login/device on your local machine
# → Enter the code — gh polls and completes automatically
```

---

## Keyring Note

Some tools store tokens in a system keyring that requires an interactive terminal to unlock.
Check the tool's documentation for a non-interactive option. Set any required credential
only for the duration of the auth step — do not persist it in shell configs.

---

## Common Errors

| Error | Fix |
|-------|-----|
| `redirect_uri_mismatch` | Use **Desktop app** OAuth client, not Web application |
| Keyring unlock fails | Check tool docs for a non-interactive keyring option |
| `Access blocked` | Add your email as test user in the Google consent screen |
| Commands fail silently | Check docs for a required account identifier option |

---

## Files

```
headless-oauth/
└── SKILL.md    # AgentSkill instructions (all three patterns)
```

---

## About AgentSkills

This is an [AgentSkill](https://openclaw.ai) — a Markdown-based instruction set for OpenClaw agents. When installed, the agent automatically knows how to handle headless OAuth flows for you.

Built by [Igor Ivanter](https://igorivanter.com) · Published on [ClawHub](https://clawhub.ai/skills/headless-oauth)
