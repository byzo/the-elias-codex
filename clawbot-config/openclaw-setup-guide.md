# OpenClaw Setup Guide — runtime infrastructure

**Pattern name:** the runtime-config repo (commonly named `<agent>-runtime-config` or `<agent>-bot-config`; this spec uses the directory name `clawbot-config/` for historical reasons).

This document describes the **runtime infrastructure** for running an autonomous agent on [OpenClaw](https://openclaw.ai/) — the host VPS, the reverse proxy, the Docker setup, the daemons, the boot hook, the cron model, and the recovery flow.

It is anonymised. All credentials, hostnames, real domains, and instance-specific identifiers have been removed.

The companion document is `elias-ops-config/ops-repo-guide.md`, which describes the **ops repo** that holds the agent's working data and the in-container scripts and daemons under version control.

---

## What lives where

The pattern uses three locations. Once you internalise the split, everything else slots into place:

| Location | What it holds | Visibility |
|---|---|---|
| **Spec repo** (this repo) | Anonymised governance + setup pattern | Public |
| **Ops repo** (`<agent>-ops`) | Live ops data + the in-container runtime scripts (daemons, restore script, cron manifest, static site source, boot/heartbeat config) | Private |
| **Runtime-config repo** (`<agent>-runtime-config`) | Host-side setup: hostnames, Caddy config, docker-compose, secrets inventory, incident log | Private |

The runtime-config repo is the host-side runbook. It is not normally cloned into the container itself — the agent doesn't need to read it at runtime. The ops repo is what the agent reads and writes from inside the container.

---

## Infrastructure Overview

### VPS
- **Provider:** any cloud VPS provider (commonly OVH, Hetzner, DigitalOcean)
- **OS:** any modern Linux (Ubuntu 24.04 / 25.04 has been used)
- **SSH access:** dedicated non-root user, no sudo. The OpenClaw data directory is owned by a separate system user so the SSH user cannot write to it directly.

### Reverse Proxy
- **Software:** Caddy (recommended — automatic Let's Encrypt TLS) or nginx
- **Routes typically used:**
  - `agent.<domain>` → reverse proxy to the OpenClaw web gateway on `127.0.0.1:<port>` (Control UI / web chat)
  - `<domain>` → static file server from the agent's published static site directory
  - `www.<domain>` → permanent redirect to `<domain>`
- The reverse proxy runs as a separate user from the OpenClaw container. The container's data directory must be readable by the proxy user (e.g. `chmod o+x` on the parent path) for static files to be served. `restore-tools.sh` (see below) checks this on every run.

### Docker
- **Image:** `ghcr.io/openclaw/openclaw:latest`
- **Restart policy:** `always`
- **Port:** bound to localhost only (`127.0.0.1:<port>`); the reverse proxy fronts it
- **Data volume:** host directory mapped to `/home/node/.openclaw` inside the container
- **Environment:** `NODE_ENV=production`
- **Lifecycle:** managed via `docker compose` (`docker-compose.yml` lives in the runtime-config repo)

### Web Gateway (inside OpenClaw)
- Bound to LAN behind the reverse proxy
- Token-based auth with rate limiting (sensible defaults: 10 attempts / 60s window, 5min lockout)
- Trusted proxies set to the Docker network CIDR
- Allowed origins restricted to the Control UI hostname

---

## API Accounts

The agent typically requires keys from the following providers:

| Provider | Purpose | Auth | Notes |
|---|---|---|---|
| **Anthropic** | Primary LLM | API key | Prepaid credits with auto top-up recommended |
| **OpenAI** | Fallback LLM + image model | API key | Standard billing |
| **Google** | Optional fallback LLM | API key | Useful as a third-tier fallback |
| **Telegram** | Bot communication channel | Bot token | Created via @BotFather |
| **Email (IMAP)** | Inbound/outbound email | IMAP login + password | Any IMAP-capable provider |
| **GitHub** | Repo access (ops, runtime-config, etc.) | Classic PAT | Fine-grained PATs may not work across organisations — classic with full repo scope is the safe choice |

OpenClaw stores most of these in `~/.openclaw/openclaw.json` during onboarding. The IMAP password lives in the email CLI's config file (e.g. `~/.config/himalaya/config.toml`). The GitHub token lives at a workspace path that survives container updates (e.g. `<ops-repo>/config/github-token.txt`, gitignored).

---

## AI Models — primary and fallbacks

The recommended pattern is a **three-tier fallback** with a *visible* fallback alert:

- **Primary:** highest-quality model (e.g. Claude Opus)
- **Secondary:** previous-generation primary (same provider) — handles primary-model outages
- **Tertiary:** different-provider fallback (e.g. GPT-5) — handles provider outages

If a session ever falls back to anything but the primary, that **must** be visible to the principal. Silent fallbacks are how a session ends up running on a model with different tool-calling reliability and quietly breaking things. See `model-fallback-watcher` in the daemon trio below.

### Agent settings worth pinning

- `agents.maxConcurrent` — typical: 4
- `agents.maxConcurrentSubagents` — typical: 8
- `compactionMode: safeguard`
- `tools.exec.notifyOnExit: false` and `notifyOnExitEmptySuccess: false` — silences "Exec completed" pings on backgrounded exec exits, which otherwise produce a lot of channel noise during work bursts.

These overrides should live in the ops repo at `config/openclaw-overrides.json` and be deep-merged into the live `~/.openclaw/openclaw.json` on every boot by `restore-tools.sh`. That way they survive container wipes without ever putting secrets in git.

---

## Channels

### Telegram
- **DM Policy:** pairing (the bot pairs with a specific user account)
- **Group Policy:** disabled by default
- **Streaming:** partial
- **Default recipient:** the principal's account

### Web Chat
- Accessed via the OpenClaw Control UI at the `agent.<domain>` hostname
- The Control UI directory is a **symlink** from the workspace into the container's app dist (`/app/dist/control-ui`). Container updates can wipe the symlink — `restore-tools.sh` recreates it.

---

## The daemon trio

Three lightweight daemons run continuously inside the container. All are singleton-enforced (single instance per process name) and all are auto-(re)started by `restore-tools.sh` on every boot.

These daemons are deliberately not LLM-driven — they cost zero tokens and they keep running even when the agent is offline.

| Daemon | Purpose |
|---|---|
| **IMAP IDLE** (Python) | Watches the inbox via IMAP IDLE. On new mail, dispatches the message into the agent (either by triggering a cron job, or — preferred — by sending the message directly into the main agent session via the agent CLI as a fire-and-forget). |
| **uptime-watchdog** (Bash) | Loops every 5 minutes inside the container. Calls one or more `*-uptime-check.sh` scripts (one per service the agent owns: static site, radio, etc.). On failure, attempts a self-fix; if that fails, triggers an LLM-driven recovery cron job. |
| **model-fallback-watcher** (Python) | Watches the active main session for model fallbacks. The moment the resolved model differs from the configured primary, sends one notification to the principal on the configured channel and persists state so it doesn't spam. |

### Why a daemon and not a cron?

Cron jobs spawn an LLM session every tick. At a 5-minute uptime check that is ~288 LLM calls/day per service; on a Sonnet-class model that is dollars/day. A shell daemon costs zero tokens. The LLM only enters the loop when self-fix has failed and recovery is needed.

### The IMAP IDLE pitfalls

Most IMAP providers drop IDLE connections silently after ~10 minutes (some send a clean DONE, some return an empty bytestring with no error). The daemon must:

1. Reconnect aggressively (e.g. every ~10 minutes proactively) rather than relying on the provider's IDLE timeout.
2. After every reconnect, run `SEARCH UNSEEN` **before** re-entering IDLE and dispatch any hits — otherwise mail arriving during the reconnect window is silently lost.
3. Mark messages SEEN only **after** successful dispatch — failures must leave the message UNSEEN for the next reconnect to retry.

These three rules are non-negotiable. They were learned the hard way.

### Daemon location

All daemon scripts live **in the ops repo**, under version control:

```
<ops-repo>/
  bin/
    restore-tools.sh
    start-uptime-watchdog.sh
    start-model-fallback-watcher.sh
    uptime-watchdog.sh
    uptime-check.sh
    <other-service>-uptime-check.sh
    model-fallback-watcher.py
    <email-cli-binary>          # large binary kept here so it survives wipes
  imap-idle/
    <agent>-mail.py             # the IMAP IDLE daemon
    start.sh                    # singleton wrapper
    *.pid, *.log                # runtime, gitignored
```

This placement is itself a lesson — see "What gets wiped" below.

---

## Cron jobs — minimal set

With the daemon trio in place, cron jobs are reduced to a small set of LLM-driven tasks:

| Cron | Schedule | Purpose |
|---|---|---|
| `<service> uptime check` | every 3h | Belt-and-braces secondary check; the watchdog daemon is the primary |
| `<service> recovery` | manual trigger (`3650d` nominal) | LLM-driven recovery, triggered by the watchdog when self-fix fails |
| `Friday Learning Review` | Fri afternoon | Weekly learning review per `02_playbook.md` |

Older variants of this pattern also had `Handle New Mail`, `Morning Briefing`, and `Interview Question` as cron jobs. They have all been migrated:

- `Handle New Mail` → IMAP IDLE daemon talks to the main agent directly via the agent CLI. No isolated session, no cron. The original isolated-session model is documented in `03_email_handling.md` Section 10 as a fallback.
- `Morning Briefing` / `Interview Question` → heartbeat tasks (see below). The cron versions, running in isolated sessions on lighter context, leaked internal monologue into the channel and ignored conditional skip rules. Heartbeat tasks running in the main session with the main model behave reliably.

### Cron-manifest pattern

OpenClaw stores cron jobs in its config directory, which **survives** restarts but **does not survive** container image updates in some setups. The pattern that solves this is a **manifest file** committed to the ops repo:

```
<ops-repo>/config/cron-manifest.json
```

Each entry has: `name`, `schedule`, `purpose`, `last_known_id`, and a `recreate_cmd` that produces an equivalent cron when run. `restore-tools.sh` reconciles live OpenClaw cron jobs against this manifest on every boot:

1. List live cron jobs.
2. For each manifest entry: if the `last_known_id` is missing from the live list, run `recreate_cmd` and patch the new id back into the manifest **and into any scripts that reference it** (e.g. the IMAP daemon's `CRON_JOB_ID` constant).
3. If recreation fails, alert the principal.

The manifest is the **single source of truth** for which cron jobs should exist. If you `cron edit` a payload via the OpenClaw CLI, also update the corresponding `recreate_cmd` so the next restore reproduces the change.

---

## The boot-md hook — single automatic entry point for recovery

OpenClaw's `boot-md` hook fires on `gateway:startup` (every container start). The hook runs the agent on `workspace/BOOT.md`, which instructs the agent to:

1. Run `bash <ops-repo>/bin/restore-tools.sh`.
2. Notify the principal **only if** anything failed.

This is the **single automatic entry point for recovery**. If you change the recovery flow, change `BOOT.md` (canonical copy in `<ops-repo>/BOOT.md`, restored to `workspace/BOOT.md` by `restore-tools.sh`).

The heartbeat is **not** wired to `restore-tools.sh`. Daemons rarely fail; polling the recovery script every 2h would burn tokens for nothing. Run `restore-tools.sh` on container start, plus on demand from a session, plus from incident response — that is enough.

---

## Heartbeat — the periodic-checks file

Heartbeat is a small set of LLM tasks that fire on a schedule. The cadence is typically every 2h during waking hours (e.g. 08:00–23:00 in the principal's timezone), with quiet hours skipped entirely.

The heartbeat is configured in `HEARTBEAT.md` at the workspace root. The canonical copy lives in the ops repo (`<ops-repo>/HEARTBEAT.md`); `restore-tools.sh` restores the workspace copy from canonical if missing.

`HEARTBEAT.md` is a YAML-ish task list. Each task has:

- `name` — identifier
- `interval` — how often to actually run (e.g. `2h`, `24h`, `7d`); OpenClaw skips ticks where no task is due
- `prompt` — what the task does in this tick

Useful tasks that have proven valuable in practice:

- `vip-inbox` (2h) — flag unread mail from VIPs to the principal, no action taken
- `reminders-due` (2h) — surface pending reminders due within a few days; track `Last Alerted` to prevent re-spam
- `blocked-projects` (24h) — nudge stalled projects that haven't moved in 5+ days
- `morning-nudge` (24h) — friendly start-of-day check-in on weekdays in a narrow time window
- `monday-priority-ritual` (7d) — Monday-only project re-prioritisation prompt

Heartbeat tasks are **observe-and-alert only**. They must not take destructive actions. They must reply `NO_REPLY` whenever a task is not due, has nothing to report, or hits any tool failure — never explain failures back to the principal.

---

## restore-tools.sh — the recovery script

`<ops-repo>/bin/restore-tools.sh` is the heartbeat / boot-callable recovery script. It is **idempotent** by design — running it repeatedly is safe. In order it performs (variations possible per instance):

1. **Reverse-proxy traversal permission fix** — `chmod o+x` on the OpenClaw data directory if needed (the proxy user must be able to traverse).
2. **Email CLI binary** — copy from the ops-repo backup to `/home/node/bin/` if missing.
3. **Email CLI config** — copy from the ops-repo backup to `~/.config/<cli>/` if missing.
4. **Static binaries** (e.g. ffmpeg, ffprobe) — copy from `workspace/bin/` to `/home/node/bin/` if missing. (Workspace-root copies are used because these are too large to comfortably commit to git.)
5. **Control UI symlink** — `workspace/control-ui` → `/app/dist/control-ui` if missing.
6. **Static site copy** — copy `<ops-repo>/<site>/` to `workspace/<site>/` if the served directory is missing/empty (the reverse proxy can't follow a symlink across users on some hosts, so a real copy is required).
7. **Other code repos** (e.g. node-based services) — restart if not running. If source files are wiped (working tree empty but `.git` intact), run `git restore .`. If `node_modules` is incomplete, run `npm install`.
8. **IMAP IDLE daemon** — singleton check + auto-start.
9. **uptime-watchdog daemon** — singleton check + auto-start.
10. **model-fallback-watcher daemon** — singleton check + auto-start.
11. **Cron-manifest reconciliation** — for every entry, if the `last_known_id` is not in the live cron DB, run `recreate_cmd` and patch the new id everywhere it's referenced.
12. **openclaw.json overrides** — deep-merge `<ops-repo>/config/openclaw-overrides.json` into the live config (no secrets in this file — it's in git).
13. **Git repo health** — `git status` on each tracked repo.
14. **Workspace integrity check** — refuse and print FATAL if `workspace/.git/` exists. The workspace root **must not** be a git working tree (see "The 2026-04-26 incident" below).

Run manually any time:

```bash
bash <ops-repo>/bin/restore-tools.sh
```

---

## What gets wiped on a container update / fresh boot

The OpenClaw Docker image rebuilds wipe most of the writable layer. After a fresh container, the following are typically **gone or empty**:

- `/home/node/bin/<email-cli>`, `ffmpeg`, `ffprobe`
- `~/.config/<email-cli>/`
- `workspace/control-ui` symlink
- `workspace/<site>/` static site copy
- Working trees of any code repos cloned into the workspace (the `.git` directory survives, but tracked files may be wiped)
- `node_modules/` for any node-based services (often partial)

What **survives** (bind-mounted or on a separate layer):

- The ops repo (entire directory)
- `.git/` directories of cloned code repos
- `workspace/bin/` large binaries (ffmpeg/ffprobe)
- `~/.openclaw/openclaw.json`, `agents/`, `cron/`, `media/`, `logs/`

`restore-tools.sh` brings everything back from canonical sources. The `boot-md` hook calls it on container start.

---

## Bootstrapping a brand-new OpenClaw container

If you're standing up the agent in a fresh container with no existing `workspace/`:

1. **Configure `~/.openclaw/openclaw.json`** — provider keys, channel tokens, gateway controlUi root, agent default model + fallbacks.
2. **Clone the operational repos**:
   ```bash
   cd /home/node/.openclaw/workspace
   TOKEN=$(cat ~/some-secure-place/github-token.txt)  # full-repo classic PAT
   git clone "https://${TOKEN}@github.com/<owner>/<agent>-ops.git"
   # plus any other code repos this agent owns
   ```
3. **Drop static binaries** (`ffmpeg`, `ffprobe`) into `workspace/bin/`. They are too large to commit to git.
4. **First run of restore-tools.sh**:
   ```bash
   bash workspace/<agent>-ops/bin/restore-tools.sh
   ```
   This sets up the email CLI, the Control UI symlink, the static site copy, starts all three daemons, and reconciles the cron manifest.
5. **Verify**:
   - All three daemon processes are running (`pgrep -af` for each).
   - `openclaw cron list` matches the manifest.
   - All hostnames return 200.
6. **Confirm boot-md hook** runs `restore-tools.sh` on container start.

---

## Workspace integrity (the lesson)

> **The workspace root must never be a git working tree.**

This rule was learned from a real incident in which a fallback-model session, helping the principal edit a separate Jekyll repo, accidentally ran `git checkout` inside `workspace/` itself — which had been a working tree of an unrelated public repo. The checkout silently deleted the static site, the daemon scripts, the email CLI, and overwrote workspace-root identity files (`SOUL.md`, `USER.md`, etc.) with stock template boilerplate. Several services were down for hours before the next session noticed.

The fixes that came out of it are now invariants:

1. **`workspace/.git/` must not exist.** `restore-tools.sh` refuses to proceed with FATAL if it finds one.
2. **Anything wipeable must either be in a tracked repo or be re-creatable from one.** This is the rule that pushed every daemon, every script, and the static site source into the ops repo.
3. **`restore-tools.sh` itself must live in a tracked repo** so it can rebuild itself after a wipe.
4. **Model fallbacks must be visible.** Hence the `model-fallback-watcher` daemon: the principal must always know whether the session they're talking to is on the configured primary model.
5. **Auto-disabled monitoring must alert.** Silent disable is itself a failure mode.

---

## Operations runbook (host side)

```bash
# Is the bot running?
ssh <user>@<host> "docker ps"

# Recent logs
ssh <user>@<host> "docker logs openclaw --tail 50"

# Restart
ssh <user>@<host> "docker restart openclaw"

# Update OpenClaw
ssh <user>@<host> "cd ~/openclaw && docker compose pull && docker compose up -d"
# After update: the boot-md hook fires and runs restore-tools.sh automatically.

# Clear stuck sessions
ssh <user>@<host> "docker exec openclaw sh -c 'rm -f /home/node/.openclaw/agents/main/sessions/*.lock'"
ssh <user>@<host> "docker restart openclaw"

# Reverse proxy
ssh <user>@<host> "systemctl status caddy"

# Daemons (inside container)
ssh <user>@<host> "docker exec openclaw bash -c 'pgrep -af \"<agent>-mail.py\"; pgrep -af \"uptime-watchdog.sh\"; pgrep -af \"model-fallback-watcher.py\"'"
```

---

## Diagnostic commands (inside container)

```bash
# Daemon health
pgrep -af "<agent>-mail.py"
pgrep -af "uptime-watchdog.sh"
pgrep -af "model-fallback-watcher.py"

# Active model right now
openclaw status

# Cron registration
openclaw cron list

# Cron manifest validity
python3 -c "import json; json.load(open('<ops-repo>/config/cron-manifest.json'))"

# All hostnames live
for u in https://<domain>/ https://agent.<domain>/; do
  echo -n "$u "; curl -s -o /dev/null -w "HTTP %{http_code}\n" "$u"
done

# Daemon logs
tail -40 /tmp/uptime-check.log
tail -40 <ops-repo>/imap-idle/<agent>-mail.log
tail -40 <ops-repo>/imap-idle/model-fallback.log
```

---

## Secrets management

Values are never stored in the spec repo. The runtime-config repo (private) holds an inventory of *where* each secret lives, never the value itself.

| Secret | Typical location | Notes |
|---|---|---|
| LLM provider API keys | `~/.openclaw/openclaw.json` and the relevant agent `models.json` | Configured during onboarding |
| Telegram bot token | `~/.openclaw/openclaw.json` (`channels.telegram.botToken`) | Created via @BotFather |
| Gateway auth token | `~/.openclaw/openclaw.json` (`gateway.auth.token`) | Auto-generated; rotate if exposed |
| IMAP password | `~/.config/<email-cli>/config.toml` | Backed up at `<ops-repo>/config/<email-cli>/config.toml` |
| GitHub personal access token | `<ops-repo>/config/github-token.txt` (gitignored) | Classic PAT with full repo scope |
| Device pairing tokens | OpenClaw config | Generated when devices pair |

**Rules of thumb:**

- Never commit secret values to git. Backed-up configs in the ops repo must be gitignored.
- Rotate any key that has been exposed in logs, console output, or an unredacted screenshot.
- The `openclaw-overrides.json` file in the ops repo is **not** a place for secrets — it is in git, and exists to hold non-secret config overrides that must survive container wipes.
- If the principal hands you a token in chat, treat the message as compromised material and ask whether to rotate.

---

## Persona & behaviour

The agent's persona, voice, and behavioural rules live in workspace files outside this guide:

- `SOUL.md` — persona and tone
- `IDENTITY.md` — name, creature, vibe, signature emoji, avatar
- `USER.md` — about the principal
- `AGENTS.md` — operating procedures (this spec ships a copy)
- `MEMORY.md` — facts only (infra, contacts, preferences) — not project state

Governance behaviour (single-goal execution, authority boundaries, escalation, learning loop) lives in the four governance files at the root of this spec (`01_constitution.md` … `04_escalation_rules.md`).

---

## Where to go next

- For the ops repo structure (projects, contacts, state, candidates, reviews, in-container scripts), see `elias-ops-config/ops-repo-guide.md`.
- For governance rules, see `01_constitution.md`, `02_playbook.md`, `03_email_handling.md`, `04_escalation_rules.md`.
