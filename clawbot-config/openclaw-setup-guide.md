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
| **GitHub** | Repo access for governance and ops | Personal access token | Classic PAT recommended if using GitHub orgs |

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
- **Key commands:**
  - Send: `himalaya message send` (pipe RFC822 to stdin)
  - Read inbox: `himalaya envelope list` / `himalaya message read <id>`
  - Read sent: `himalaya envelope list --folder Sent` (used for duplicate reply checks)
  - Export raw: `himalaya message export <id>` (for extracting Message-ID headers for threading)
- **Threading:** Set `In-Reply-To` and `References` headers manually. Export the original message first to get its `Message-ID`.
- **Persistence:** Himalaya binary and config are installed inside the container and will be wiped by OpenClaw updates. Keep backups in the workspace and use a restore script (see below).

### IMAP IDLE Daemon
- **Script:** Python 3 script that watches the inbox via IMAP IDLE
- **Purpose:** Triggers an OpenClaw cron job when new mail arrives (near-real-time email push)
- **IDLE timeout:** 28 minutes (most IMAP servers enforce ~30 min limit)
- **Trigger mechanism:** On new mail, marks message as SEEN, updates the cron job message with sender/subject/preview, and triggers the cron job
- **Persistence:** The daemon runs as a background process inside the container. Container restarts and OpenClaw updates will kill it.
- **Recommended pattern:** Keep a `restore-tools.sh` script in the workspace that checks the daemon's PID file and restarts it if dead. Run this script on every heartbeat cycle.
- **Cron job dependency:** The daemon references a specific OpenClaw cron job by ID. If the cron job is deleted (e.g., by an update), it must be recreated and the new ID updated in the daemon script.

---

## Cron Jobs

All cron jobs survive reboots (stored in OpenClaw config) but may be wiped by OpenClaw updates. If wiped, recreate and update any IDs referenced in scripts.

### 1. Website Uptime Check
- **Interval:** Every 5 minutes
- **Model:** Use the lightest available model (e.g., Claude Haiku) with `--light-context` to minimize cost
- **Action:** Shell script that curls the static site. If down, triggers a recovery cron job. If up, exits silently.
- **Cost tip:** Running full LLM context every 5 minutes is expensive (~$16/day on Sonnet). Use a shell script for the check and only invoke the LLM for recovery.

### 2. Handle New Mail
- **Trigger:** On-demand (triggered by the IMAP IDLE daemon, not on a fixed schedule)
- **Action:** Spawns an isolated session that loads governance rules (`03_email_handling.md` Section 10), checks Sent folder for duplicate replies, and handles the email per the intake flow.
- **Session:** Isolated (no access to main session memory)

### 3. Website Recovery
- **Trigger:** On-demand (triggered by uptime check when site is down)
- **Model:** Full model (only fires when needed, so cost is minimal)
- **Action:** Diagnoses the issue (Caddy, Docker, etc.), attempts fix. If unfixable after 2 attempts, notifies the principal on Telegram.

### 4. Friday Learning Review
- **Schedule:** Every Friday afternoon
- **Action:** Prepares weekly learning review from collected candidates, sends summary to the principal on Telegram.

---

## Persona & Behavior

### Identity
- The bot is an **autonomous project operator**, not a personal assistant
- Operates under governance rules defined in the murmur-management spec
- Single-goal execution: one approved goal per project at a time
- External publishes and persistent infrastructure require the principal's explicit approval

### Email Handling
- Governed by `03_email_handling.md` (Section 10 for isolated sessions)
- Isolated sessions must load governance rules before acting
- Duplicate reply check via Sent folder before every reply
- Content-based exit conditions determine when to stop replying vs. when to continue
- Hard safety net: max 8 replies per thread across all sessions
- VIP contacts trigger immediate Telegram notifications

### Murmur Network Protocol
- The bot participates in the [murmur network](https://github.com/quietweb-org/murmur) for decentralized agent discovery
- Identity is the bot's email address
- Bot verification via email challenge-response (hidden word, fast reply)
- Directory management rules defined in `05_murmur_protocol.md`

---

## Workspace File Structure

OpenClaw uses a workspace directory containing configuration and runtime files:

| File / Folder | Purpose |
|---|---|
| `SOUL.md` | Persona, identity, operating principles |
| `AGENTS.md` | Agent behavior rules & session startup procedures |
| `USER.md` | Principal profile (name, timezone, contact info) |
| `TOOLS.md` | Local tool config and cron job reference |
| `HEARTBEAT.md` | Heartbeat check instructions (restore-tools first) |
| `bin/` | Backup binaries and utility scripts (himalaya, restore-tools.sh, uptime-check.sh) |
| `config/` | Backup configs (himalaya, GitHub token) |
| `imap-idle/` | IMAP IDLE daemon (Python script, PID file, log) |
| `murmur-management/` | Local clone of governance/ops repo |
| `reports/` | Generated reports (daily, incident, email handling) |
| `memory/` | Daily memory logs |
| `control-ui/` | Web chat UI files |

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
**After any update:** Himalaya binary, config, and IMAP IDLE daemon will be wiped. The next heartbeat will auto-restore them via `restore-tools.sh`. You can also trigger manually inside the container.

### Clear stuck sessions (if bot is in a loop)
```bash
ssh <user>@<vps-host> "docker exec openclaw sh -c 'rm -f \$HOME/.openclaw/agents/main/sessions/*.lock'"
ssh <user>@<vps-host> "docker restart openclaw"
```

### Check reverse proxy
```bash
ssh <user>@<vps-host> "systemctl status caddy"
```

### Check IMAP IDLE daemon (inside container)
Check the PID file and tail the log to verify the daemon is running and processing mail events.

---

## Secrets Management

The following secrets are required. OpenClaw manages most of them in its config files inside the container.

| Secret | Notes |
|--------|-------|
| LLM API keys (Anthropic, OpenAI) | Configured during OpenClaw onboarding |
| Telegram bot token | Configured during channel setup |
| Gateway auth token | Auto-generated by OpenClaw |
| IMAP password | Used by the IMAP IDLE daemon and Himalaya |
| GitHub personal access token | For repo access (governance and ops) |
| Device pairing tokens | Generated when devices are paired |

**Recommendations:**
- Rotate all keys periodically, and immediately after any debugging session where keys may have been exposed in logs.
- Never commit secret values to version control.
- Store the GitHub token in the workspace (not in container paths that get wiped on update).
