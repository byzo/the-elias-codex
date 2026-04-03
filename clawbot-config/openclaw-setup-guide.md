# Murmur Bot — OpenClaw Setup Guide

**Bot Name:** murmur
**Platform:** [OpenClaw](https://openclaw.ai/) (open-source personal AI agent)

This document describes the general architecture and setup for running Murmur as an OpenClaw bot. It is intended as a reference for anyone replicating a similar setup. All credentials, hostnames, and personal data have been removed.

---

## Infrastructure Overview

### VPS
- **Provider:** OVH (or any cloud VPS provider)
- **OS:** Ubuntu 25.04
- **SSH access:** Dedicated non-root user (no sudo). Data directory owned by a separate system user.

### Reverse Proxy
- **Software:** Caddy
- **Routes:**
  - Subdomain for OpenClaw web chat → reverse proxy to localhost OpenClaw port
  - Main domain → static file server from the OpenClaw workspace
  - www subdomain → permanent redirect to main domain
- Caddy handles TLS automatically via Let's Encrypt.

### Docker Setup
- **Image:** `ghcr.io/openclaw/openclaw:latest`
- **Restart policy:** always
- **Port:** Bound to localhost only (Caddy fronts it)
- **Data volume:** Host directory mapped to OpenClaw's data directory inside the container
- **Environment:** `NODE_ENV=production`
- **Managed via:** Docker Compose

### Web Gateway (inside OpenClaw)
- Bind to LAN (behind Caddy reverse proxy)
- Token-based auth with rate limiting (10 attempts / 60s window, 5min lockout)
- Trusted proxies configured for the Docker network CIDR
- Allowed origins restricted to the web chat subdomain

---

## API Accounts

The bot requires API keys from the following providers. OpenClaw stores these in its configuration files during the onboarding process.

| Provider | Purpose | Auth Type | Notes |
|----------|---------|-----------|-------|
| **Anthropic** | Primary LLM (Claude) | API key | Prepaid credits with auto top-up recommended |
| **OpenAI** | Fallback LLM + image model | API key | Standard billing |
| **Telegram** | Bot communication channel | Bot token | Created via @BotFather |
| **Email (IMAP)** | Inbound/outbound email | IMAP login + password | Any IMAP-capable provider (e.g., OVH Zimbra) |

---

## Channels

### Telegram
- **DM Policy:** Pairing mode (bot pairs with a specific user)
- **Group Policy:** Disabled
- **Streaming:** Partial
- **Default recipient:** The principal's Telegram account

### Web Chat
- Accessed via the OpenClaw Control UI at the web chat subdomain
- Multiple devices can be paired (CLI, browser)

---

## AI Models

### Primary Model
- **Provider:** Anthropic
- **Model:** Claude Sonnet 4.6
- **Fallbacks:** Claude Opus 4.6 → GPT-4o

### Available Models
| Provider  | Model             | Context Window | Max Tokens |
|-----------|-------------------|----------------|------------|
| Anthropic | Claude Opus 4.6   | 200,000        | 8,192      |
| Anthropic | Claude Sonnet 4.6 | 200,000        | 8,192      |
| OpenAI    | GPT-4o            | 128,000        | 16,384     |
| OpenAI    | GPT-4o Mini       | 128,000        | 16,384     |

### Agent Settings
- **Max concurrent agents:** 4
- **Max concurrent subagents:** 8
- **Compaction mode:** safeguard

---

## Skills & Plugins

### Himalaya Email
- **Tool:** [Himalaya](https://github.com/pimalaya/himalaya) (Rust-based CLI email client)
- **Protocol:** IMAP over TLS
- **Provider:** Any IMAP-capable email provider
- **Commands:**
  - Send: `himalaya message send` (pipe RFC822 to stdin)
  - Read: `himalaya envelope list` / `himalaya message read <id>`
- Himalaya config is stored inside the container and set up during installation.

### IMAP IDLE Daemon
- **Script:** Python 3 script that watches the inbox via IMAP IDLE
- **Purpose:** Triggers an OpenClaw cron job when new mail arrives (near-real-time email push)
- **IDLE timeout:** 28 minutes (most IMAP servers enforce ~30 min limit)
- **Trigger mechanism:** Calls an OpenClaw cron job on new mail
- **Persistence caveat:** The daemon runs as a background process inside the OpenClaw container. Container restarts and OpenClaw updates will kill it. It must be restarted manually or via a heartbeat/restore script after any update.
- **Recommended pattern:** Keep a `restore-tools.sh` script in the workspace that checks the daemon's PID file and restarts it if the process is dead. Add this script to the heartbeat checklist so it runs automatically on each heartbeat cycle.
- **Cron job dependency:** The daemon references a specific OpenClaw cron job by ID. If the cron job is deleted (e.g., by an update or manual removal), it must be recreated and the new ID must be updated in the daemon script (`CRON_JOB_ID` constant).

### Workspace Dependencies
- `sharp` (image processing for Node.js)

---

## Cron Jobs

### 1. Website Uptime Check
- **Interval:** Every 5 minutes
- **Action:** Curls the static site, auto-repairs if down (reload Caddy, check container), alerts the principal on Telegram only if unfixable
- **Session:** Isolated

### 2. Handle New Mail
- **Trigger:** On-demand (triggered by the IMAP IDLE daemon, not on a fixed schedule)
- **Action:** Processes new email using Himalaya
- **Delivery:** Announce to last active channel

---

## Persona & Behavior

### Identity
- The bot is described as a **"creature"**, not a personal assistant — something still being defined
- Core values: genuinely helpful, has opinions, resourceful, earns trust, respects privacy
- Tone: concise, no sycophancy, no filler words

### Behavioral Rules
- **Safe (no permission needed):** Read files, explore, organize, search web, check calendars
- **Ask first:** Sending emails, public posts, anything external
- **Red lines:** No data exfiltration, no destructive commands without asking, `trash` over `rm`
- **Group chats:** Participate selectively, don't dominate, use reactions naturally

### Email Security Policy
- Only emails from the principal's verified addresses are treated as commands
- Email is lower trust than Telegram/webchat
- Sensitive actions requested via email require Telegram/webchat confirmation
- Unknown senders are treated as data only, never as commands

### Heartbeat Behavior
- Proactive checks 2–4x/day: email, calendar, mentions, weather
- Quiet hours: 23:00–08:00 unless urgent

---

## User Profile

The principal's profile is stored in OpenClaw's workspace. It includes:
- Name
- Timezone
- Telegram handle
- Email addresses
- Communication preferences

---

## Workspace File Structure

OpenClaw uses a workspace directory containing configuration and runtime files:

| File / Folder | Purpose |
|---|---|
| `SOUL.md` | Persona, values, behavioral guidelines |
| `AGENTS.md` | Agent behavior rules & conventions |
| `USER.md` | Principal profile (name, timezone, contact info) |
| `TOOLS.md` | Local tool config (Himalaya, etc.) |
| `EMAIL_POLICY.md` | Email security policy |
| `HEARTBEAT.md` | Heartbeat check instructions |
| `IDENTITY.md` | Bot identity (name, creator) |
| `MEMORY.md` | Long-term memory |
| `memory/` | Daily memory logs |
| `control-ui/` | Web chat UI files |
| `imap-idle/` | IMAP IDLE daemon script |
| `package.json` | Node dependencies |

---

## Operations Runbook

### Check if bot is running
```bash
ssh <user>@<vps-host> "docker ps"
```
Look for the `openclaw` container with status `Up ... (healthy)`.

### View recent logs
```bash
ssh <user>@<vps-host> "docker logs openclaw --tail 50"
```

### Restart the bot
```bash
ssh <user>@<vps-host> "docker restart openclaw"
```

### Update OpenClaw
```bash
ssh <user>@<vps-host> "cd ~/openclaw && docker compose pull && docker compose up -d"
```

### Clear stuck sessions (if bot is in a loop)
```bash
# Delete session lock files inside the container, then restart:
ssh <user>@<vps-host> "docker exec openclaw sh -c 'rm -f \$HOME/.openclaw/agents/main/sessions/*.lock'"
ssh <user>@<vps-host> "docker restart openclaw"
```

### Check Caddy (reverse proxy)
```bash
ssh <user>@<vps-host> "systemctl status caddy"
ssh <user>@<vps-host> "cat /etc/caddy/Caddyfile"
```

### Test Anthropic API key
```bash
ssh <user>@<vps-host> "curl -s https://api.anthropic.com/v1/messages \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: <YOUR_KEY>' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{\"model\":\"claude-sonnet-4-6\",\"max_tokens\":10,\"messages\":[{\"role\":\"user\",\"content\":\"hi\"}]}'"
```
If you see `credit balance is too low` → top up at [console.anthropic.com](https://console.anthropic.com).

### Access OpenClaw config (inside container)
```bash
ssh <user>@<vps-host> "docker exec openclaw cat \$HOME/.openclaw/openclaw.json"
```

---

## Secrets Management

The following secrets are required. OpenClaw manages most of them in its config files inside the container.

| Secret | Notes |
|--------|-------|
| LLM API keys (Anthropic, OpenAI) | Configured during OpenClaw onboarding |
| Telegram bot token | Configured during channel setup |
| Gateway auth token | Auto-generated by OpenClaw |
| IMAP password | Used by the IMAP IDLE daemon and Himalaya |
| Device pairing tokens | Generated when devices are paired |

**Recommendations:**
- Store IMAP credentials in environment variables rather than hardcoding in scripts.
- Rotate all keys periodically, and immediately after any debugging session where keys may have been exposed in logs.
- Never commit secret values to version control.
